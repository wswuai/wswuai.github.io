---
layout: post
title:  "Alfred workflow 开发指南"
date:   2015-05-23 20:38:00
categories: python
tags: python alfred
---

小帽子alfred是mac上最为传奇的效率作品， 今天，我们一起来探索alfred workflow 的世界吧!

# 1. alfred 简介
小帽子是 Mac 平台上最为传奇的效率作品，誉为神兵利器毫不为过。

由于中文网络上尚无系统的alfred workflow 开发教程，便有了写一个教程的心思，以期抛砖引玉，为alfred吸引更多的开发者加入。

-------

# 2.alfred 插件开发

## 2.1 alfred 插件开发概述

Workflow 是alfred2.0推出的最激动人心的特性，通过与脚本语言的交互，workflow可以支持任意操作，把您日常的重复性事务封装在脚本中，大大的提高工作效率。

Workflow 支持php、bash、perl、ruby以及python作为脚本语言，并内置脚本语言解释器，并通过stdio的形式在各个脚本模块中传递参数。

在代码中插入 {query}块可以接收上一个脚本输出的内容。形成完整的控制链条。 最后由alfred输出至 Output 模块， 在Output模块中， 我们可以启动浏览器、将内容复制到剪切板、 启动通知中心、甚至执行bash脚本。

在日常的使用中，我们通常通过关键字来调用某一模块，例如“find xxx" 即是调用find内建模块 query内容为xxx。 在workflow的开发中， 开发者可以自定义自己编写模块的关键字，只要不与其他模块冲突即可。
 
在workflow的结构中，数据流通过alfred的控制线进行传递，每一个脚本模块的STDIO输出会被alfred替换到 下一个脚本的{query}块中。

如下图所示：

![image](/images/alfredtutorial/1.png)

其中，我们使用『拖动』的方式，控制数据在workflow的流向:

![image](/images/alfredtutorial/2.png)

以上的 有道翻译workflow的数据流向是这样的。

通过yd作为关键字启动workflow， 如果直接回车，使用OpenURL，启动youdao网站。如果使用Cmd+回车的方式，
则执行Run Scirpt 并将其输出到剪切板，并用Post Notification的方式，在通知中心显示。

## 2.2 Alfred的XML格式概述

-------

     在Alfred脚本执行的最后一部中， 所有数据要被封装到XML格式中被alfred调用，其格式为

``` XML
<items>
    <item autocomplete = "autocompletex" uid = "123321" arg = "argsx" >
    <title >title</title>
    <subtitle >subtitle</subtitle>
    <icon >icon</icon>
    </item>
</items>
```
运行之后的效果是：


![image](/images/alfredtutorial/3.png)


根节点为items 其中包括任意多个item节点，每一个item节点代表本次查询结果的一行。
每一个item节点包括若干parameter与childnode，其含义为：
uid： 每一个item要有一个独立的uid，不可重复
valid：值为yes 或者 no, 若为no，该行结果不可被选择
autocomplete ： 自动补全的值， 使用tab可以令alfred自动补全为 autocompelete属性的值.
arg：作为下一个模块的参数传递

&lt;title>  该行item的标题，也是主要显示的位置。
&lt;subtitle> 该行字标题位置，会被显示为灰色小字
&lt;icon> 该行图标的文件名，其大小为64X64 pixels

## 2.3 {query} 与 stdio

-------

alfred模块间数据是如何传递的，曾经困扰我很久。

其数据流是这样的， 你在关键字之后的文字会作为第一个模块的query, 由alfred替换代码中{query}的位置， 该模块print到stdio的数据会像第一个模块一样，去替换第二个模块的{query}位置，直到最后生成XML文件， 只有最后一步是不一样的，也就是xml文件输出之后， 用户在不同的条目上按回车， alfred将把该条目对应的item的arg属性的值传递给下一个模块。


## 2.4 python 开发示例

-------

下面，我将用开发一个百度词典为例，详细讲解开发alred插件中的每一个步骤。

###2.4.1 Script Filter
Script Filter 的作用是提供一个Alfred workflow的起始点，可以通过点击右上角的加号，从Inputs中选择。

它的界面如下图所示：

![image](/images/alfredtutorial/4.png)

在上图中可以看出， 输入 “bdc hello” 之后， alfred会启动一个python解释器，并运行Scirpt栏中的代码。

Script中的{query}会被alfred自动的取代为「hello」.

也就是，此时 实际运行的代码是 

``` python
import bddict
bddict.query('hello')
```


-------

其中 ， bddict.py 的代码如下：

``` python

# -*- coding: utf_8 -*- 
# author xiaogo
import sys
import hashlib
import time
import math
import base64
import urllib2 
import urllib
import re
import json
import alfredxml

client_id = '1xML4eLvqG8runr4uHcDPU6f'

def is_cn_char(i):
    return 0x4e00<=ord(i)<0x9fa6

def bdDict(From,To,Q):
    resp = urllib2.urlopen("http://openapi.baidu.com/public/2.0/translate/dict/simple?client_id=%s&q=%s&from=%s&to=%s"%(client_id,Q,From,To)).read()
    js = json.loads(resp,encoding = 'utf-8')
    return js

# alfred 的入口函数.
def query(query):
# 对query 作防御
    if(len(query)==0):
        rowList = [{'uid':'1',
                    'arg':'',
                    'autocomplete':'',
                    'icon':'icon.png',
                    'subtitle':'Please Input',
                    'title':'Please Input'}]
        element = alfredxml.genAlfredXML(rowList)
        print(element)
        return
# 判断是否为中文字符，若是 汉译英 若否 英译汉
    if(is_cn_char(unicode(query,"utf-8")[0] )):
        resp = bdDict("zh","en",query)
    else:
        resp = bdDict("en","zh",query)
    if(resp[u'errno'] == 0):  # 通过bdDict() 函数 ，调用百度词典HTTP接口.
        if(len(resp[u'data'])==0):
            return
        rowList = []
        subtitle = ''
        k = resp['data']['symbols'][0] # 解析JSON.
        uid = 1
        for i in k.keys():
            if(i.startswith("ph_")):
                subtitle +=i[3:]+'['+ resp['data']['symbols'][0][i] + ']'
        # 解析JSON, 生成rowList
        rowList.append({
                'uid':uid,
                'arg':query,
                'autocomplete':query,
                'icon':'icon.png',
                'subtitle':subtitle.encode("utf-8"),
                'title':'发音'})
        uid +=1


        for i in resp['data']['symbols'][0]['parts']:
            if(len(i['part'])>0):
                subtitle = reduce(lambda x,y:x+","+y, i['part'])
            else:
                subtitle = ''
            title = reduce(lambda x,y:x+","+y, i['means'])
            rowList.append({
                'uid':uid,
                'arg':query,
                'autocomplete':query,
                'icon':'icon.png',
                'subtitle':subtitle.encode("utf-8"),
                'title':title.encode("utf-8")})
            uid +=1


    else:
        print("err")
        pass
    #print(rowList)
    # 生成XML文件.
    print(alfredxml.genAlfredXML(rowList))
```


-------

其中， 生成XML的代码如下  alfredxml.py ：

``` python
# 生成XML文件
def genElement(lists):
    assert(len(lists)%3==0)
    name = lists[0]         # 节点名
    params = lists[1]      # 节点属性list
    content = lists[2]      # 节点内容 ， 可能是String 或者是包含其他Element.
    string = ''
    string +="<%s"%name + “ “  # 以下为解析XML List的过程

    for k,v in params.items():    # 枚举属性
        string+='%s = "%s" '%(k,v)

    if(isinstance(content, str)):   # 通过递归 解析子节点
        text = content
    else:
        text = genElement(content)
    string +=">" + text + "</%s>\n"%name

    if(len(lists)<=3):  # 通过递归， 解析同级节点
        return string
    else:
        return string + genElement(lists[3:])

def genAlfredXML(rowList):  # 生成alfred所需要的XML String.
    item = []
    for row in rowList:
        tsi = ['title',{},row['title'],'subtitle',{},row['subtitle'],'icon',{},row['icon']]
        item.extend(['item',{'uid':row['uid'],'arg':row['arg'],'autocomplete':row['autocomplete']},tsi])
    items = ['items',{},item]
    return genElement(items)

if __name__=='__main__':
    rowList = [{'uid':'123321','arg':'argsx','autocomplete':'autocompletex','icon':'icon','subtitle':'subtitle','title':'title'}]
    element = genAlfredXML(rowList)

    print(element)

```




###2.4.2 Output

output  是 alfred 中的『执行器』。

通过Script Filter， 我们已经得到了一个alfred 的结果列表， 通过点按回车键，我们可以执行相应的命令。
以百度词典这个workflow为例，我们打开了百度词典的网站。

执行的时候，我们打开了http://dict.baidu.com/s?wd={query}

注意，这里又出现了一个{query}

这个{query} 将承接 Scirpt Filter 中，被点击回车的那条XML 节点中的 arg参数 。


![image](/images/alfredtutorial/5.png)


实际上，我们不仅可以使用URL Open 作为 结果，也可以使用Terminal Command 作为结果执行。

使用Terminal ，可以令你更加随心所欲的控制你的MAC. 实现诸如Wifi切换、更改DNS服务器、启动某个服务端、甚至可以控制你家的空调或者咖啡机， 这以前只有emacs才能做到 ;)   

#3.参考文献


------

1.[Alfred - workflow  python dev lib](http://www.deanishe.net/alfred-workflow/)

2.[Tutorial on alfred workflow with python](http://www.deanishe.net/alfred-workflow/tutorial.html)



