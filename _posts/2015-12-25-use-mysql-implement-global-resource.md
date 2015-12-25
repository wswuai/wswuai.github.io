---
layout: post
title:  "使用MySQL实现分布式资源调度"
date:   2015-12-25 21:00:56
categories: 分布式
tags: MySQL 分布式
---

一般来讲， 全局资源调度可以利用Zookeeper、Redis进行。 在全局资源量比较小的情况下，也可以利用MySQL的ACID特性来进行资源调度。

### 一、应用场景

有些时候，我们会面临分布式资源分配的问题。

举个例子，全局只有40个代理服务器资源，爬虫程序需要占用一个代理服务器资源来进行工作。 并且爬虫任务分布在不同的服务器上，我们要设计一个资源的调度方案来使每一个爬虫都能拿到资源并且不会将一个资源重复分配给两个爬虫。 在爬虫任务退出或挂掉的情况下，我们还希望资源可以静默的被释放，不会造成资源的浪费。

### 二、实现原理

MySQL是OLTP数据库，会保证数据的一致性。

UPDATE语句会返回影响数据行数的值。

执行 UPDATE some_table SET a = 1 WHERE a=2 时，如果有两个相同的语句被发送到MySQL服务器上，MySQL会保证只有一个线程得到1，其他的请求只能返回0，根据这个特点，我们可以设计出一个利用MySQL来进行分布式资源调度的系统。

###三、示例代码

``` python
class Resource(threading.Thread):
    search_Sql = None
    lock_Sql = None
    occupation_delta = None
    occupation_interval = None
    sql_date_format = "%Y-%m-%d %H:%M:%S"
    model = None

    def __init__(self):
        super(Resource, self).__init__()
        self._run = True
        self.initResource()

    def initResource(self):
        cur = django.db.connection.cursor()
        while True:
            now = chop_microseconds(datetime.datetime.now())
            res = self.model.objects.raw(self.search_Sql % (now.strftime(self.sql_date_format)))
            res = [i for i in res]  # if i.is_effective == 2]
            if len(res) == 0:
                time.sleep(10)
                logger.info("not enough Resource to get, wait for 3 seconds ...")
                continue
            else:
                r = res[0]
                if r.busy_before is None:
                    r.busy_before = datetime.datetime(2001, 01, 01)

                effect_lines = cur.execute(self.lock_Sql % (
                    (now + self.occupation_delta).strftime(self.sql_date_format), r.id,
                    r.busy_before.strftime(self.sql_date_format)))

                if effect_lines == 0:  # Race condition
                    logger.info("Race Condition appear when init resource.")
                    time.sleep(2)
                    continue
                else:
                    r.busy_before = now + self.occupation_delta
                    self.id = r.id
                    self.o = r
                    if not self.is_valid():
                        self.o.busy_before = self.o.busy_before + datetime.timedelta(days=1)
                        self.o.save()
                        logger.info("this resource is not valid, try to get another.")
                        continue
                    else:
                        logger.info("resource " + str(self) + "is OK. ")
                        print (str(os.getpid()) + " : resource " + str(self) + "is OK. ")
                        cur.close()
                        break

    def keepOccupation(self):
        if self.o.busy_before == datetime.datetime(2000, 01, 01):
            return
        now = chop_microseconds(datetime.datetime.now())
        cur = django.db.connection.cursor()
        el = cur.execute(self.lock_Sql % ((now + self.occupation_delta).strftime(self.sql_date_format), self.o.id,
                                          self.o.busy_before))
        self.o.busy_before = (now + self.occupation_delta).strftime(self.sql_date_format)
        cur.close()

        if el < 1:
            raise RuntimeError("Cannot keep Occupation.")

    def run(self):
        while self._run:
            time.sleep(self.occupation_interval)
            logger.info("Keep" + str(self) + "occupied.")
            try:
                self.keepOccupation()
            except RuntimeError, e:
                logger.info("Resource Occupation Wrong, init .")
                self.initResource()

    def close(self):
        logger.info("closing Resource " + str(self))
        self.o.busy_before = datetime.datetime(2000, 01, 01)
        self.o.save()

    def is_valid(self):
        raise NotImplementedError

    def fix(self):
        if self.is_valid():
            logger.info(str(self) + "is okay, no need to fix")
            print(str(self) + "is okay, no need to fix")
            return
        else:
            self.close()
            logger.info(str(self) + "is not valid , try to fix")
            print(str(os.getpid()) + " : " + str(self) + "is not valid , try to fix")
            self.initResource()

    def terminate(self):
        self.close()
        self._run = False

```

以上代码实现了一个使用MySQL来进行分布式调度的资源抽象类。

search_Sql 一般是形如 :

``` python

    search_Sql = 'SELECT * FROM some_table WHERE busy_before < "%s"'

```
作用是，搜索资源表有没有空闲的资源.

lock_Sql 一般是形如 :

``` python

lock_Sql = 'UPDATE some_table SET busy_before = "%s" WHERE id=%s AND busy_before = "%s"'

```

作用是，将busy_before字段置位当前时间后推几分钟（自定），代表该资源已被占用。

这样设计的原因是，不单纯使用一个bit来表示当前资源的状态，而是使用一个不断更新的busy_before字段，

这样，当占用该资源的线程死锁或硬退出后，这个全局资源不会一直处于占用状态，而是在几分钟之后静默的转换为未占用状态。


