---
layout: post
title: "plyvel抛出IOError的处理以及后续思考"
date: 2021-07-29 14:13:35 +0800
categories: python
tags: python
---

# 缘由

在疫情爆发之后学校便要求每日健康打卡，每天需要重复登录 → 点击打卡链接 → 输入体温等信息 → 打卡并截图一系列步骤，遂有编写自动打卡脚本的想法。在自动打卡的逻辑代码编写好后，又萌生了分享给好友共用的愿望。由于最初的想法非常简单并且并不涉及级联关系等设计，于是便选择了 kv 数据库—leveldb。

# 异常溯源

由于服务器框架选用 flask+gunicorn+gevent 并使用了多 worker，初衷是想更高的并发，谁知 plyvel 的坑来了。

在测试服务器功能时可能会时不时抛出异常，大意为无法获得目标数据库的锁

```python
plyvel._plyvel.IOError: b'IO error: lock /dakala/data/db/LOCK: Resource temporarily unavailable'
```

在 plyvel 的 github issues 中搜索后得到

[IOError: IO error: lock · Issue #13 · wbolster/plyvel](https://github.com/wbolster/plyvel/issues/13)

从中我们得知 plyvel 只允许一个线程连接并操作数据库。而正是多个 worker 造成了 db 被多个线程尝试连接进而抛出无法获得锁的异常

# worker 的原理

gunicorn 的一个 worker 对应一个进程，每个进程的内存不共享。造成了数据库被多个进程尝试打开的情况

# 解决思路

查阅后发现 gunicorn 有 preload_app 的配置选项，大概作用是代码会在运行时初始化一次之后再生成 worker，每个 worker 会拿到初始化后的内存的副本，并在副本上工作。故思路 1 产生了。

全局创建数据库连接然后不关闭了，让连接对象被 worker 拷贝。

虽然这个方式使用后异常没有抛出了，但是由于多个 worker 的数据库客户端并不是同一内存对象，在修改数据后造成了各 worker 间获得的数据不一致的情况（怀疑连接底层是有 cache 的）

思路 2，在该问题的 issues 下 plyvel 的作者推荐了将 leveldb 单独拿出作为一项服务运行并通过接口调用（例如 restapi，rpc...）的方式并推荐了 Elevator 这个库。这是一个比较完美的方案。但我想到这只是一个简单的打卡脚本便另寻他法。

思路 3，对 db 创建全局访问锁，只有拥有锁才能创建连接并操作数据库，但是，worker 之间内存不共享的情况会导致各个 worker 都会拥有一把锁，故单纯使用多线程库的锁来实现控制访问是不可行的。plyvel 作者的在前面提到 issues 的回答最下方说到他并不推荐外部锁定的方式来处理，因为这涉及了格外的锁文件，等待时间和重试等问题，容易出错。这激发了我的兴趣。在一番搜索之后，这个问题进入了我的视野

[Sharing a lock between gunicorn workers](https://stackoverflow.com/questions/18213619/sharing-a-lock-between-gunicorn-workers)

其中，问题也提到了如果想在多 worker 之间分享锁，都需要通过第三方来实现。而下面这个库帮我们实现了一把基于文件的锁。

[NamedAtomicLock](https://pypi.org/project/NamedAtomicLock/)

于是，思路 3 就是在获得这个由外部文件所做的锁之后就创建连接，并在 db 使用完成之后关闭数据库并释放锁。这样便实现了多个 worker 之间共享锁来控制数据库连接。

# 总结

虽然锁的实现会对性能大打折扣，但考虑到代码规模只是一个自动打卡的简单系统，故还是决定使用文件锁，若对数据库的并发有要求，需考虑将 leveldb 单独作为服务并提供远程调用的方式使用

# 吐槽

gunicorn 这个 worker 内存不共享实在是对代码的编写有巨大的限制。
