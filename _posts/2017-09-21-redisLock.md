---
layout: post
keywords: Start
description: redis加锁的几种实现
title: redis加锁的几种实现
categories: [PHP]
tags: [PHP]
group: archive
icon: globe
---



## 1. redis加锁分类
1. redis能用的的加锁命令分表是`INCR`、`SETNX`、`SET`

## 2. 第一种锁命令`INCR`
这种加锁的思路是， key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 INCR 操作进行加一。  
然后其它用户在执行 INCR 操作进行加一时，如果返回的数大于 1 ，说明这个锁正在被使用当中。  

```

    1、 客户端A请求服务器获取key的值为1表示获取了锁
    2、 客户端B也去请求服务器获取key的值为2表示获取锁失败
    3、 客户端A执行代码完成，删除锁
    4、 客户端B在等待一段时间后在去请求的时候获取key的值为1表示获取锁成功
    5、 客户端B执行代码完成，删除锁
 
    $redis->incr($key);
    $redis->expire($key, $ttl); //设置生成时间为1秒
    
```


## 3. 第二种锁`SETNX`
这种加锁的思路是，如果 key 不存在，将 key 设置为 value  
如果 key 已存在，则 `SETNX` 不做任何动作

```
    1、 客户端A请求服务器设置key的值，如果设置成功就表示加锁成功
    2、 客户端B也去请求服务器设置key的值，如果返回失败，那么就代表加锁失败
    3、 客户端A执行代码完成，删除锁
    4、 客户端B在等待一段时间后在去请求设置key的值，设置成功
    5、 客户端B执行代码完成，删除锁
    
    $redis->setNX($key, $value);
    $redis->expire($key, $ttl);
   
```

## 4. 第三种锁`SET`

上面两种方法都有一个问题，会发现，都需要设置 key 过期。那么为什么要设置key过期呢？如果请求执行因为某些原因意外退出了，导致创建了锁但是没有删除锁，那么这个锁将一直存在，以至于以后缓存再也得不到更新。于是乎我们需要给锁加一个过期时间以防不测。  
但是借助 Expire 来设置就不是原子性操作了。所以还可以通过事务来确保原子性，但是还是有些问题，所以官方就引用了另外一个，使用 `SET` 命令本身已经从版本 2.6.12 开始包含了设置过期时间的功能。

```
    1、 客户端A请求服务器设置key的值，如果设置成功就表示加锁成功
    2、 客户端B也去请求服务器设置key的值，如果返回失败，那么就代表加锁失败
    3、 客户端A执行代码完成，删除锁
    4、 客户端B在等待一段时间后在去请求设置key的值，设置成功
    5、 客户端B执行代码完成，删除锁
        
    $redis->set($key, $value, array('nx', 'ex' => $ttl));  //ex表示秒
    
```

## 5. 其它问题

虽然上面一步已经满足了我们的需求，但是还是要考虑其它问题？  
    1、 redis发现锁失败了要怎么办？中断请求还是循环请求？
    2、 循环请求的话，如果有一个获取了锁，其它的在去获取锁的时候，是不是容易发生抢锁的可能？
    3、 锁提前过期后，客户端A还没执行完，然后客户端B获取到了锁，这时候客户端A执行完了，会不会在删锁的时候把B的锁给删掉？


## 6. 解决办法

针对问题1：使用循环请求，循环请求去获取锁
针对问题2：针对第二个问题，在循环请求获取锁的时候，加入睡眠功能，等待几毫秒在执行循环
针对问题3：在加锁的时候存入的key是随机的。这样的话，每次在删除key的时候判断下存入的key里的value和自己存的是否一样

```
        do {  //针对问题1，使用循环
            $timeout = 10;
            $roomid = 10001;
            $key = 'room_lock';
            $value = 'room_'.$roomid;  //分配一个随机的值针对问题3
            $isLock = Redis::set($key, $value, 'ex', $timeout, 'nx');//ex 秒
            if ($isLock) {
                if (Redis::get($key) == $value) {  //防止提前过期，误删其它请求创建的锁
                    //执行内部代码
                    Redis::del($key);
                    continue;//执行成功删除key并跳出循环
                }
            } else {
                usleep(5000); //睡眠，降低抢锁频率，缓解redis压力，针对问题2
            }
        } while(!$isLock);

```

## 7. 另外一个锁
以上的锁完全满足了需求，但是官方另外还提供了一套加锁的算法，这里以PHP为例

```

    $servers = [
        ['127.0.0.1', 6379, 0.01],
        ['127.0.0.1', 6389, 0.01],
        ['127.0.0.1', 6399, 0.01],
    ];
    
    $redLock = new RedLock($servers);
    
    //加锁
    $lock = $redLock->lock('my_resource_name', 1000);
    
    //删除锁
    $redLock->unlock($lock)


```
上面是官方提供的一个加锁方法，就是和第6的大体方法一样，只不过官方写的更健壮。所以可以直接使用官方提供写好的类方法进行调用。官方提供了各种语言如何实现锁。

原文链接：[Dennis`s blog](http://ukagaka.github.io/php/2017/09/21/redisLock.html)  

[官方提供分布式redis锁说明](https://redis.io/topics/distlock)  
[PHPreids分布式锁](https://github.com/ronnylt/redlock-php)
[谈谈Redis的SETNX](https://huoding.com/2015/09/14/463)  
[redis五种常见使用场景下PHP实现](https://segmentfault.com/a/1190000008404117?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)