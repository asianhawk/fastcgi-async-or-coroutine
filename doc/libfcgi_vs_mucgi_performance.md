## 测试目的
发现以及分析NGINX -> fastcgi -> ice调用之间存在的性能瓶颈。
## 压测方案
* 找出nginx性能极限
* 找出nginx -> fastcgi性能极限。比较同步cgi和异步cgi的差异。
* 找出nginx -> fastcgi -> ice性能极限。比较同步cgi和异步cgi的差异。

#### 代码
异步fastcgi的[源码](../mucgi "悬停显示")  
同步fastcgi的[源码](https://fastcgi-archives.github.io/ "悬停显示")  

#### 压测方法
![展示](/doc/image/image001.png)  
 
#### 服务器状态
    10服务器，实体机
    8核(Intel(R) Xeon(R) CPUE5504  @ 2.00GHz)
    带宽1000Mbit
    硬盘7200转
    内存32GB

## 压测流程
### 1. nginx压测
![展示](/doc/image/image002.png)  
![展示](/doc/image/image003.png)   
    CPU使用：784  
    达到平均处理数：96.5Ktps  
> CPU资源差不多用完，所以nginx性能瓶颈在CPU
---
### 2. NGINX->CGI压力测试
#### a. 长链异步fastcgi
![展示](/doc/image/image004.png)  
![展示](/doc/image/image005.png)  
    CPU使用：770(370+400)  
    达到平均处理数：71.3Ktps  
> CPU资源差不多用完，所以nginx->cgi异步模型瓶颈也在CPU
#### b. 短链同步fastcgi
* i.使用nginx瓶颈时的weighttp并发数  
![展示](/doc/image/image006.png)  
![展示](/doc/image/image007.png)  
    CPU使用：672 (336+336)  
    达到平均处理数：21.2Ktps  
> CPU资源有剩余,性能瓶颈不在CPU
* ii.加大并发数  
![展示](/doc/image/image008.png)  
![展示](/doc/image/image009.png)  
    CPU使用：740 (353+390)  
    达到平均处理数：27.2Ktps  
> 加大并发数，性能有所提高但和预期不符合。 

>nginx->cgi同步模型瓶颈是由于nginx　upstream模块和cgi之间使用的是短链接，当net.ipv4.tcp_tw_recycle = 0的时候，压测过程中发现time_wait状态的端口达到接近：65536个。由于socket是四元组，所以我们通过增加一个fastcgi监听端口来优化这个模型。  
* iii.增加一个fastcgi监听端口  
![展示](/doc/image/image010.png)  
![展示](/doc/image/image011.png)  
![展示](/doc/image/image012.png)  
    CPU使用：790  
    达到平均处理数：28.8Ktps  
> 增加一个fastcgi监听端口来压测，从测试结果上看，资源使用开销上去了，但tps并无明显增加。
---
### 3. NGINX->CGI->PROXY压力测试
* a. 长链异步CGI  
![展示](/doc/image/image013.png)  
![展示](/doc/image/image014.png)  
    CPU使用：550 (120+300+130)  
    达到平均处理数：12Ktps  
>  CPU资源有剩余,性能瓶颈不在CPU
* b. 短链同步CGI  
![展示](/doc/image/image015.png)  
![展示](/doc/image/image016.png)  
    CPU使用：670(360+190+120)  
    达到平均处理数：10.8Ktps  
>  CPU资源有剩余,性能瓶颈不在CPU
---
### 4.优化NGINX +CGI +PROXY模型
__NGINX + MUCGI + PROXY异步模型有可优化空间。这个的瓶颈在ice单进程交易率的限制。所以将nginx反代到两个mcgi->两个proxy后。机器性能能完全发挥出来。__
  
![展示](/doc/image/image017.png)  
![展示](/doc/image/image018.png)  
    CPU使用：788(177+418+193)  
    达到平均处理数：19.6Ktps  
> CPU资源差不多用完，所以瓶颈也在CPU.
## 结论  
* 这台服务器上面单独压测nginx能到达到96500tps.  
* nginx->fastcgi在资源占用差不多的情况下，`mucgi`每秒处理71300次echo请求,`libfcgi`每秒处理21200次echo请求.  
* nginx->fastcgi->ice_proxy 资源占用差不多的情况下，`mucgi`每秒处理10800次echo请求,`libfcgi`每秒处理19600次echo请求.  

![展示](/doc/image/image019.png)  



