---
layout: post
title:  "使用nolearn构建ANN神经网络完成数字识别挑战"
date:   2015-04-06 11:40:56
categories: 数据挖掘
tags: DM python ann
---


对于我这样的data science新手来说，kaggle是一个很好的练手平台，在kaggle的starter难度中，数字识别是一个比较有代表性的项目。
昨晚使用nolearn构建ANN神经网络，识别率可以达到到98%. 


( 做到100%的那位同学，为何你如此神猛...)

---
以下是代码:

``` python
import csv
import numpy as np
import sys
import pandas
from nolearn.dbn import DBN

testRest = 3000

# 导入csv文件并转换为array.
print("loading txt...")
fil = file('train.csv')
fil.readline()

txt = np.loadtxt(fil, delimiter=',',dtype=np.float)

data = txt

digits_label = np.array([i[0] for i in data])
digits_image = np.array([i[1:]/255.0 for i in data])

#digits_image = map(binaryzation, digits_image)

print("file loading ok , prepare to train model")

# object of ANN model. 
# Input 784 , Hidden 1000,500 Output 10.

clf = DBN(
    [digits_image.shape[1], 1000,500, 10],
    learn_rates=0.3,
    learn_rate_decays=0.9,
    epochs=10,
    verbose=1,
    )

clf.fit(digits_image[:-testRest],digits_label[:-testRest])

result = clf.predict(digits_image[-testRest:])

print("the err rate : %.2f %%" % (100*((result != digits_label[-testRest:]).sum())/float(testRest)) )


print("the result num %.2f "%(result!= digits_label[-testRest:]).sum() )

testTxt = file('test.csv')
testTxt.readline()

testData = np.loadtxt(testTxt,delimiter=',',dtype=np.int)
result2 = clf.predict(testData)
out = open('outputann.csv','wb')
count = 1
out.write(u'"ImageId","Label"\n')
for i in result2:
    out.write(repr(count) + ",")
    count +=1
    out.write(r'"'+repr(i)+r'"')
    out.write('\n')
out.close()
exit()
```
