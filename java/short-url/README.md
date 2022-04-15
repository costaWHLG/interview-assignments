# 1、需求

### 实现短域名服务（细节可以百度/谷歌）

撰写两个 API 接口:
- 短域名存储接口：接受长域名信息，返回短域名信息
- 短域名读取接口：接受短域名信息，返回长域名信息。

限制：
- 短域名长度最大为 8 个字符
- 采用SpringBoot，集成Swagger API文档；
- JUnit编写单元测试, 使用Jacoco生成测试报告(测试报告提交截图)；
- 映射数据存储在JVM内存即可，防止内存溢出；

# 2、设计思路

### 需求分析

##### (1) 长域名转短域名，并存储映射(短域名<key>,长域名<value>)

本质上就是通过某一个规则或算法生成长域名的唯一Key,且要求key的长度小于8（需要考虑冲突的可能性和处理方法）。

##### (2) 映射数据存储在JVM内存即可，防止内存溢出

实际业务场景，长域名转短域名的请求量会比较大，存在jvm且防止内存溢出需要考虑限制映射数据的容量，并有合理的自动回收机制。

##### (3) 其他需求

采用SpringBoot等技术性限制不做分析。实际业务场景涉及高并发场景，需要注意操作映射数据存储时的线程安全问题。

# 3、技术选型

### 需求（1）  
长域名转短域名需要保证唯一性，UUID,雪花算法都太长了，查阅资料得知普遍是两种算法：
>**自增序列算法**  
> 短址的长度一般设为 8 位，而每一位是由 [a - z, A - Z, 0 - 9] 总共 62 个字母组成的，所以8位的话，总共会有 62^8 ~= 218万亿种组合，一般肯定是够用了。 

>**摘要算法**  
> 1.将原网址 md5 生成 32 位签名串,分为 4 段, 每段 8 个字节  
> 2.对这四段循环处理, 取8个字节,将他看成16进制串与0x3fffffff(30位1)与操作,即超过30位的忽略处理这30位分成6段,每5位的数字作为字母表的索引取得特定字符,依次进行获得6位字符串  
> 3.总的 md5 串可以获得4个6位串,取里面的任意一个就可作为这个长 url 的短url地址这种算法,虽然会生成4个,但是仍然存在重复几率

这两种算法各有优劣，第一次容易被猜到规律，就算打乱[a - z, A - Z, 0 - 9]的顺序 ，如果有较多的连续的样本数据也是可以猜到的，存在安全问题。第二种有较
低的重复风险。考虑到本示例存储的映射是有限的，重复风险进一步降低，故选用第二种。

### 需求（2）和需求（3）
主要是涉及到映射数据存储的选型。  
首先想到的是spring cache，默认是ConcurrentHashMap,虽然线程安全但是不支持自定义容量，最大容量1<<30=2^30=1073741824, 如果不手动清理会占用太
多内存，不合理。找了下其他支持配置容量的缓存，先找到了google的Guava,配置的时候发现spring5+已经不支持这个了，现在改用
[caffeine](https://github.com/ben-manes/caffeine)
时间比较紧急，先看下基本配置满足要求，就是你了！

# 4、架构图

本工程正常是一个业务微服务，提供短域名能力，只是一个节点，加上jvm存映射数据，不支持一般的负载均衡(ip_hash好像可以)，感觉没啥好画的。。。

# 5、基本流程

本需求是个底层服务，只暴露两个接口，说白了就是两个流程：

>拿到长域名 -> 通过摘要算法转换 -> 判断缓存中是否存在，存在的话比对长域名,一样直接返回,不一样的话再MD5一遍并转换,还是冲突就返回错误（理论上概率很低）
> -> 成功拿到短域名，映射关系put进缓存并返回

> 接口拿到短域名-> 直接去缓存get -> 有值成功，没有拿到返回失效或不存在错误
>>**可以加上层服务用redis做布隆过滤器做短域名的有效性过滤，本服务负责添加数据，暂不实现**

# 6、测试

### 功能测试
功能测试比较简单，使用Swagger可以测试，最近在用apifox，感觉也不错。
启动程序后
[Swagger地址](http://127.0.0.1:8080/swagger-ui/index.html#/)

### 单元测试
以往写单元测试较少，本次也简单学习了下，使用的Junit5+Mockito，本次需求除了算法逻辑比较简单，用例也写得不复杂。

### 压力测试

之前没怎么用过jmeter，大概学习了下基本用法，加上都是再一台机器上跑的，测试可能不太规范。。。
使用了mock.js模拟url参数
```
{
   "url": "${__Mock(@url(http))}"
}
```

***PC配置***
```
处理器	Intel(R) Core(TM) i5-4460  CPU @ 3.20GHz   3.20 GHz
机带 RAM	8.00 GB
```
*家里电脑比较旧。。。*

***jvm参数***
```
-Xms4G
-Xmx4G
-server
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```
**结果分析**  
缓存最大容量设置为100000L，过期时间为7天。实际值需要根据现实场景来计算，在容量和有限时间内做平衡。如果容量太小，不太高的并发都会短时间撑爆缓存容量，
旧的短链可能几分钟被抹去失效。如果容量太大，会造成单个缓存查找时间过长，影响接口性能。

***生成短域名接口***：本机测试qps为1000多点，压测了大概3分钟，请求200000次，没有异常，短链冲突导致的错误返回 大概15次，错误率大约为0.0075%，
目前只做了一次冲突处理，多做几次理论上可以降低短链冲突，或者优化下算法。个人算法不强，暂时放弃。。。

***获取长域名接口***：此接口比较简单，就是通过短链从缓存拿长链,本机配置测试qps为8000多点，没有异常或错误返回。

具体结果截图见/screenshot文件夹

# 7、总结
        本次需求内容看起来比较简单，但是如果要达到正式商用的标准还是需要更多的打磨。比如引入分布式缓存储映射，优化算法和短链冲突的处理逻辑等，
     对一些如过期时间的关键参数的设置也主要结合实际场景多测试才能确定。或者可以试下自增序列算法，结合发牌器，提前生产出短链，可以有效降低实时
     编码的损耗，通过分布式缓存和每个节点不一样的字符序列（洗牌器实现），保证每个节点产生的短链没有明显规律。等等，技术细节还有很多值得细细推
     敲的地方
        同时这次小作业也是收获不少。像caffeine是第一次使用，感觉性能还不错，之前项目时间紧，对开发测试要求不高，jacoco、Mockito
    、Jmeter都是有所了解，但是没使用过，在这两天的时间，自己也是慢慢学着用起来，虽然很多地方不太规范，但是学习带来的满足感还是让我十分开心，
    也感谢贵公司给了的这样一个机会。