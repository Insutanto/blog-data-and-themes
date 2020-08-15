---
title: PySpark UDF的坑与绕
tags: Spark
category: Technology
excerpt: 使用Python UDF时，发现pyspark.daemon把CPU全部占满
created: 2019-02-22
image: ./images/lukasz-szmigiel-jFCViYFYcus-unsplash.jpg
image_caption: Photo by Luasz Szmigiel on Unsplash
author: insutanto
---

最近使用Spark跑一些千万级别的数据，因为算法是Python写的，所以用到了PySpark，刚开始不太了解PySpark的原理，导致中间遇到很多问题。

之前其他同事尝试PySpark mllib做机器学习相关的任务，有遇到程序卡死，我的程序后来也遇到了。我将数量级下降到百万级别后，在单机模式下能够正常跑完任务；后来换了一台单机跑全量数据，32核使用local[16]，跑16个线程，使用Python UDF时，发现pyspark.daemon把CPU全部占满。

在使用 vmstat 定位之后，发现每秒cs (content switch)高达700万次，导致内核消耗的cpu时间超过50%，使用yarn提交到一台8核机器上时，发现每秒cs高达600万次，内核消耗cpu的时间超过了惊人的60%，而空闲和等待IO的占比为0%。

![CS切换问题](https://raw.githubusercontent.com/momentxu/git_files/master/images/2019/03/28/pyspark_cs_overhead-1553753770199.png)
(图片来自《Learning PySpark》 —— Tomasz Drabas, Denny Lee)

“其中白色部分是新增的Python进程，在Driver端，通过Py4j实现在Python中调用Java的方法，即将用户写的PySpark程序”映射”到JVM中，例如，用户在PySpark中实例化一个Python的SparkContext对象，最终会在JVM中实例化Scala的SparkContext对象；在Executor端，则不需要借助Py4j，因为Executor端运行的Task逻辑是由Driver发过来的，那是序列化后的字节码，虽然里面可能包含有用户定义的Python函数或Lambda表达式，Py4j并不能实现在Java里调用Python的方法，为了能在Executor端运行用户定义的Python函数或Lambda表达式，则需要为每个Task单独启一个Python进程，通过socket通信方式将Python函数或Lambda表达式发给Python进程执行。

语言层面的交互总体流程如下图所示，实线表示方法调用，虚线表示结果返回。”——《PySpark 的背后原理》 sharkdtu.com

![Relationship between PySpark and Spark](https://raw.githubusercontent.com/momentxu/git_files/master/images/2019/03/28/pyspark_connect_to_jvm-1553753777680.jpeg)
(图片来自《PySpark 的背后原理》 sharkdtu.com)

PySpark是借助Py4j实现Python调用Java，来驱动Spark应用程序，本质上主要还是JVM runtime，Java到Python的结果返回是通过本地Socket完成。

虽然这种架构保证了Spark核心代码的独立性，但是在大数据场景下，JVM和Python进程间频繁的数据通信导致其性能损耗较多，恶劣时还可能会直接卡死，所以建议对于大规模机器学习或者Streaming应用场景还是慎用PySpark，尽量使用原生的Scala/Java编写应用程序，对于中小规模数据量下的简单离线任务，可以使用PySpark快速部署提交。”——《PySpark 的背后原理》 sharkdtu.com

总结下其实就是以下两点：

1. 使用PySpark时，大数据量下，尽量使用Python调Java，而不要用Java调Python（不使用Python UDF）
2. 尽量使用Scala或者Java写Spark程序，简单程序可以写PySpark