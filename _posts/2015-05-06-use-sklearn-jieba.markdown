---
layout: post
title:  "使用sklearn + jieba中文分词构建文本分类器"
date:   2015-05-06 10:44:56
categories: 数据挖掘
tags: python 文本分类 数据挖掘
---

jieba是一个优秀的中文分词模块，使用python编写，并在Github上开源。
使用jieba分词可以将一整串的中文句式切分为独立的语言元素。

------------

**sklearn**也是由python编写的机器学习算法库，其实现了许多有用的算法，对于文本分类来说，使用sclera分类模型所需要的向量形式。使用scikit learn 的 naive bayes 算法库 可以快速构建一个朴素贝叶斯模型。对于文本分类这种动辄上万的维度的问题来说，朴素贝叶斯有以下特点：

1. 模型简单 
2. 训练速度快 
3. 易于理解

------------

结合以上两个工具的优秀特性，我们可以很轻松的构建一个文本分类器。使用jieba对文本进行分词，再使用sklearn的feature extraction模块对词袋进行Hashing Trick 得到向量形式，然后再训练bayes分类器，最后使用bayes分类器对未知的文本进行分类。

------------
对于一个文本分类的问题来说，一共有以下几个阶段：


1. 数据准备 ： 要得到一个准确率高、效果好的模型，大量的数据是必须的。只有数据的量达到一定程度，模型的正确率才能达到一个可以接受的水平，数据的量与模型复杂度、置信率的关系，参见：[VC维]()
 
2. 数据清洗： 大量的数据中必然存在错误数据，我们要先尽可能的把不靠谱的数据剔除出去，不然就会出现”吃进去的是杂草，拉出来的是屎“ 这种情况。

3. 特征提取： 对于文本分类问题来说，数据的特征可以是：词频特征、TF-IDF特征等.  由于词袋的规模非常的大，我们可以使用Hashing Trick 技巧来进行降维操作。当然，Hashing Trick的参数选择非常重要，太大了会导致运行缓慢甚至内存溢出，太低了会导致碰撞的几率加大，降低模型准确性。[HashingTrick](http://www.cnblogs.com/kemaswill/p/3903099.html)

4. 训练模型：接下来就是将进行特征提取之后的特征向量喂给分类器。 通常，这类数据的数据结构是sklearn内建的spread矩阵或者是numpy的array. 

5. 交叉验证： 使用cross validate模块可以验证不同的模型对同一个问题的正确率差异。 便于我们选择更好的模型（其实这个我也没试过呢）

代码如下 :

``` python
import jieba
import csv
import sklearn.feature_extraction
import sklearn.naive_bayes as nb
import sklearn.externals.joblib as jl
import sys
def predict(txt):
    kv = [t for t in jieba.cut(txt)]
    mt = fh.transform([kv])
    num =  gnb.predict(mt)
    for (k,v) in catedict.viewitems():
        if(v==num):
            print(" do you mean...%s" % k )
def clipper(txt):
    return jieba.cut(txt)
if __name__=='__main__':
    # Load file.
    reader = csv.reader(open('catedict.csv'))
    catedict = {}
    catelist = []
    memolist = []
    for i in reader :
        catelist +=[  [i[0],int(i[1]) ]  ] 
    catedict = dict(catelist)
    #init vars
    reader = open('finished.csv','r')
    ctr = 0
    gnb = nb.MultinomialNB(alpha = 0.01)
    fh = sklearn.feature_extraction.FeatureHasher(n_features=15000,non_negative=True,input_type='string')
    kvlist = []
    targetlist = []
    for col in reader:
        line = col.split(',')
        if(len(line) == 2):
            line[1].replace('\n','')
            kvlist += [  [ i for i in clipper(line[0]) ] ]
            targetlist += [int(line[1])]
            ctr+=1
            sys.stdout.write('\r' + repr(ctr) + ' rows has been read   ')
            sys.stdout.flush()
        if(ctr%100000==0):
            print("\npartial fitting...")
            X = fh.fit_transform(kvlist)
            gnb.partial_fit(X,targetlist,classes = [i for i in catedict.viewvalues()])
            # clean context
            result = gnb.predict(X)
            rate = (result == targetlist).sum()*100.0 / len(result)*1.0
            print("rate %.2f %% " % rate)
            targetlist = []
            kvlist = []

"""
    kvlist = []
    targetlist = []
    for i in memolist:
        if(catedict.has_key(i[1])):
            kvlist += [ [t for t in clipper(i[0])] ]
            targetlist += [ catedict[i[1]] ]
        print(cnt)
        cnt+=1
    fh = sklearn.feature_extraction.FeatureHasher(n_features=15000,non_negative=True,input_type='string')


    X = fh.fit_transform(kvlist)
    

    gnb = nb.MultinomialNB()
    gnb.fit(X,targetlist)
    result = gnb.predict(X)
    accuracy = ((result == targetlist).sum() ) / float(len(targetlist))
    print("accuracy : %.2f" % accuracy)
    """


```
