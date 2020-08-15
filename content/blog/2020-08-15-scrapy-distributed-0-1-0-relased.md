使用Scrapy开发一个分布式爬虫？你知道最快的方法是什么吗？一分钟真的能 开发好或者修改出 一个分布式爬虫吗？

话不多说，先让我们看看怎么实践，再详细聊聊细节~

## 快速上手

### **Step 0:**

首先安装 Scrapy-Distributed :

```
pip install scrapy-distributed
```

(非必须)如果你没有所需要的运行条件，你可以启动两个 Docker 镜像进行测试 (RabbitMQ 和 RedisBloom):

```bash
# pull and run a RabbitMQ container.
docker run -d --name rabbitmq -p 0.0.0.0:15672:15672 -p 0.0.0.0:5672:5672 rabbitmq:3
# pull and run a RedisBloom container.
docker run -d --name redis-redisbloom -p 0.0.0.0:6379:6379 redislabs/rebloom:latest
```

### Step 1 (非必须):

如果你有一个现成的爬虫，可以跳过这个 Step，直接到 Step 2。

创建一个爬虫工程，我这里以一个 sitemap 爬虫为例:

```bash
scrapy startproject simple_example
```

然后修改 spiders 文件夹下的爬虫程序文件:

```python
from scrapy_distributed.spiders.sitemap import SitemapSpider
from scrapy_distributed.queues.amqp import QueueConfig
from scrapy_distributed.dupefilters.redis_bloom import RedisBloomConfig

class MySpider(SitemapSpider):
    name = "example"
    sitemap_urls = ["http://www.people.com.cn/robots.txt"]
    queue_conf: QueueConfig = QueueConfig(
        name="example", durable=True, arguments={"x-queue-mode": "lazy", "x-max-priority": 255}
    )
    redis_bloom_conf: RedisBloomConfig = RedisBloomConfig(key="example:dupefilter")

    def parse(self, response):
        self.logger.info(f"parse response, url: {response.url}")
```

### **Step 2:**

只需要修改配置文件 settings.py 下的`SCHEDULER`, `DUPEFILTER_CLASS` 并且添加 RabbitMQ 和 Redis 的相关配置，你就可以马上获得一个分布式爬虫，Scrapy-Distributed 会帮你初始化一个默认配置的 RabbitMQ 队列和一个默认配置的 RedisBloom 布隆过滤器。

```
SCHEDULER = "scrapy_distributed.schedulers.DistributedScheduler"
SCHEDULER_QUEUE_CLASS = "scrapy_distributed.queues.amqp.RabbitQueue"
RABBITMQ_CONNECTION_PARAMETERS = "amqp://guest:guest@localhost:5672/example/?heartbeat=0"
DUPEFILTER_CLASS = "scrapy_distributed.dupefilters.redis_bloom.RedisBloomDupeFilter"
BLOOM_DUPEFILTER_REDIS_URL = "redis://:@localhost:6379/0"
BLOOM_DUPEFILTER_REDIS_HOST = "localhost"
BLOOM_DUPEFILTER_REDIS_PORT = 6379
REDIS_BLOOM_PARAMS = {
    "redis_cls": "redisbloom.client.Client"
}
BLOOM_DUPEFILTER_ERROR_RATE = 0.001
BLOOM_DUPEFILTER_CAPACITY = 100_0000
```

你也可以给你的 Spider 类，增加两个类属性，来初始化你的 RabbitMQ 队列或 RedisBloom 布隆过滤器:

```python
class MySpider(SitemapSpider):
		......
    queue_conf: QueueConfig = QueueConfig(
        name="example", durable=True, arguments={"x-queue-mode": "lazy", "x-max-priority": 255}
    )
    redis_bloom_conf: RedisBloomConfig = RedisBloomConfig(key="example:dupefilter")
		......
```

### **Step 3:**

```
scrapy crawl <your_spider>
```

检查一下你的 RabbitMQ 队列 和 RedisBloom 过滤器，是不是已经正常运行了？

可以看到，Scrapy-Distributed 的加持下，我们只需要修改配置文件，就可以将普通爬虫修改成支持 RabbitMQ 队列 和 RedisBloom 布隆过滤器的分布式爬虫。在拥有 RabbitMQ 和 RedisBloom 环境的情况下，修改配置的时间也就一分钟。😂

## 关于Scrapy-Distributed

目前 Scrapy-Distributed 主要参考了Scrapy-Redis 和 scrapy-rabbitmq 这两个库。

如果你有过 Scrapy 的相关经验，可能会知道 Scrapy-Redis 这个库，可以很快速的做分布式爬虫，如果你尝试过使用 RabbitMQ 作为爬虫的任务队列，你可能还见到过 scrapy-rabbitmq 这个项目。诚然 Scrapy-Redis 已经很方便了，scrapy-rabbitmq 也能实现 RabbitMQ 作为任务队列，但是他们存在一些缺陷，我这里简单提出几个问题。

1. Scrapy-Redis 使用 Redis 的 set 去重，链接数量越大占用的内存就越大，不适合任务数量大的分布式爬虫。
2. Scrapy-Redis 使用 Redis 的 list 作为队列，很多场景会有任务积压，会导致内存资源消耗过快，比如我们爬取网站 sitemap 时，链接入队的速度远远大于出队。
3. scrapy-rabbitmq 等 RabbitMQ 的 Scrapy 组件，在创建队列方面，没有提供 RabbitMQ 支持的各种参数，无法控制队列的持久化等参数。
4. scrapy-rabbitmq 等 rabbitmq 框架的 Scheduler 暂未支持分布式的 dupefilter ，需要使用者自行开发或接入相关组件。
5. Scrapy-Redis 和 scrapy-rabbitmq 等框架都是侵入式的，如果需要用这些框架开发分布式的爬虫，需要我们修改自己的爬虫代码，通过继承框架的 Spider 类，才能实现分布式功能。

于是，Scrapy-Distributed 框架就在这个时候诞生了，在非侵入式设计下，你只需要通过修改 settings.py 下的配置，框架就可以根据默认配置将你的爬虫分布式化。

为了解决Scrapy-Redis 和 scrapy-rabbitmq 存在的一些痛点，Scrapy-Distributed 做了下面几件事:

1. 采用了 RedisBloom 的布隆过滤器，内存占用更少。
2. 支持了 RabbitMQ 队列声明的所有参数配置，可以让 RabbitMQ 队列支持 lazy-mode 模式，将减少内存占用。
3. RabbitMQ 的队列声明更加灵活，不同爬虫可以使用相同队列配置，也可以使用不同的队列配置。
4. Scheduler 的设计上支持多个组件的搭配组合，可以单独使用 RedisBloom 的DupeFilter，也可以单独使用 RabbitMQ 的 Scheduler 模块。
5. 实现了 Scrapy 分布式化的非侵入式设计，只需要修改配置，就可以将普通爬虫分布式化。

目前框架还有很多功能正在添加，感兴趣的小伙伴可以持续关注项目仓库的动向，有什么想法也可以一起讨论。

Scrapy-Distributed 的 github 仓库地址：[https://github.com/Insutanto/scrapy-distributed](https://github.com/Insutanto/scrapy-distributed)