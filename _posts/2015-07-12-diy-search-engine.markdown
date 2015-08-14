---
layout: post
title:  "使用elasticsearch DIY 自己的搜索引擎(一)"
date:   2015-07-13  16:17:47
categories: python
tags: python elasticsearch
---

如果你拥有百万份、千万份数量级的文档的时候，如何快速的从浩如烟海的文档中搜索到自己感兴趣的文档呢？

答案就是，你需要一个搜索引擎。

搜索引擎...那不是谷歌、百度这种“技术超强”的公司才拥有的技术吗？

现在，来自开源社区的elasticsearch，已经为我们提供了一套强大的解决方案，并且非常易用。

#1、概念

##1.1 反向索引

反向索引，在很多的文献中也叫“倒排索引”，是被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。它是文档检索系统中最常用的数据结构。

以英文为例，下面是要被索引的文本：

- T0 : "it is what it is"
- T1:"what is it"
- T2:"it is a banana"

我们就能得到下面的反向文件索引：

"a":      {2}
"banana": {2}
"is":    {0, 1, 2}
"it":    {0, 1, 2}
"what":  {0, 1}

全文检索即是通过反向索引，实现在大量文档中快速检索的。

##1.2 中文分词

中文分词指的是使用计算机自动对中文文本进行词语的切分，即像英文那样使得中文句子中的词之间有空格以标识。中文自动分词被认为是中文自然语言处理中的一个最基本的环节。
中文与英文有一个非常大的区别，就是中文是没有空格来分割单词的，这就造成在机器理解中文的困难.

## 1.3 Lucene

Lucene是Apache基金会下的王牌开源项目之一， 它由java编写， 提供对全文检索的各种支持，封装层次较低，可以根据所需进行自定义的扩展，学习曲线比较陡峭，
Solr与elasticsearch均基于Lucene开发。 Lucene也是全文检索领域事实上的王者。如同Linux之于服务器，Windows之于桌面办公。

#2. elasticsearch 介绍

Elasticsearch 是一个分布式可扩展的实时搜索和分析引擎。它能帮助你搜索、分析和浏览数据，而往往大家并没有在某个项目一开始就预料到需要这些功能。Elasticsearch 之所以出现就是为了重新赋予硬盘中看似无用的原始数据新的活力。

无论你是需要全文搜索、结构化数据的实时统计，还是两者的结合，这本指南都会帮助你了解其中最基本的概念，从最基本的操作开始学习 Elasticsearch。之后，我们还会逐渐开始探索更加复杂的搜索技术，你可以根据自身的学习的步伐。

Elasticsearch 并不是单纯的全文搜索这么简单。我们将向你介绍讲解结构化搜索、统计、查询过滤、地理定位、自动完成以及你是不是要查找的提示。我们还将探讨如何给数据建模能提升 Elasticsearch 的性能，以及在生产环境中如何配置、监视你的集群。


#3. 部署elasticsearch
## 3.1 JAVA依赖

elasticsearch依赖于JRE1.8，故在部署elasticsearch之前请先确认JAVA1.8的运行环境。并将环境变量JAVA_HOME 指向JRE的目录。

## 3.2 elasticsearch.yml配置

cluster.name : testcluster

node.name : testnode

elasticsearch本身的默认配置就已经优化的不错，没有问题的情况下可以保持默认配置。

需要注意的一点是，如果你想使用自定义的排序方式， 就需要开启groovy脚本支持，配置如下:

script.groovy.sandbox.enabled: true

如果你想简单点，可以使用Docker部署elasticsearch环境：

Docker run -d -p 9200:9200 -p 9300:9300 -v $pwd/esdata:/usr/share/elasticsearch/data elasticsearch:latest
