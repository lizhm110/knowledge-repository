# 安装说明
用两台虚拟机模拟6个节点，一台机器3个节点，创建出3 master、3 salve 环境。

redis 采用 redis-4.0.2 版本。

两台虚拟机都是 ubuntu 16.04，一台IP:192.168.1.211，一台IP:192.168.1.212。

redis规划端口：212机器上是8001、8002、8003，211机器上是8004、8005、8806。

# 安装过程
## 1.下载和解压
```
cd /root/soft
wget http://download.redis.io/releases/redis-4.0.2.tar.gz
tar -zxvf redis-4.0.2.tar.gz
```
## 2. 编译和安装
```
cd redis-4.0.2
make && make install
```
## 3.创建执行目录
```
mkdir bin
cd src
mv redis-cli redis-server redis-benchmark redis-sentinel redis-trib.rb ../bin
```
## 4. 创建redis节点
- 创建配置目录conf，并将redis.conf拷贝到这个目录中
```
cd /root/soft/redis-4.0.2/
mkdir conf
cp redis.conf ./conf/redis-8001.conf
cp redis.conf ./conf/redis-8002.conf
cp redis.conf ./conf/redis-8003.conf
```
- 分别修改这三个配置文件，修改如下内容

```
port  8001                                        //端口8001,8002,8003        
bind 本机ip                                       //默认ip为127.0.0.1 需要改为其他节点机器可访问的ip 否则创建集群时无法访问对应的端口，无法创建集群
daemonize    yes                               //redis后台运行
pidfile  /var/run/redis_8001.pid          //pidfile文件对应8001,8002,8003
cluster-enabled  yes                           //开启集群  把注释#去掉
cluster-config-file  nodes_7000.conf   //集群的配置  配置文件首次启动自动生成 8001,8002,8003
cluster-node-timeout  15000                //请求超时  默认15秒，可自行设置
appendonly  yes                           //aof日志开启  有需要就开启，它会每次写操作都记录一条日志
logfile "/root/soft/redis-4.0.2/logs/redis-8001.log"     //日志保存路径 
```
- 在目录bin中生成redis-server.sh文件,内容如下：

```
#!/bin/sh
/root/soft/redis-4.0.2/bin/redis-server /root/soft/redis-4.0.2/conf/redis-8001.conf
/root/soft/redis-4.0.2/bin/redis-server /root/soft/redis-4.0.2/conf/redis-8002.conf
/root/soft/redis-4.0.2/bin/redis-server /root/soft/redis-4.0.2/conf/redis-8003.conf
exit 0
```
- 在文件/etc/rc.local中添加如下代码，系统启动时自启动redis

```
/root/soft/redis-4.0.2/bin/redis-server.sh
```
- 接着在另外一台机器（192.168.1.211）上的的操作重复以上三步，把对应的配置文件也按照这个规则修改即可。
## 5. 启动各个节点
- 启动212机器上的redis
```
/root/soft/redis-4.0.2/bin/redis-server.sh
```
- 启动211机器上的redis
```
/root/soft/redis-4.0.2/bin/redis-server.sh
```
也可以重启两台机器，使redis重启。
## 6. 检查redis启动情况
```
root@cp212:~# ps -aux|grep redis
root      1105  0.1  0.0  41720  3700 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.212:8001 [cluster]
root      1115  0.0  0.0  41720  3628 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.212:8002 [cluster]
root      1125  0.1  0.0  41720  3716 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.212:8003 [cluster]
root      1281  0.0  0.0  14228   968 pts/0    S+   23:04   0:00 grep --color=auto redis
```

```
root@host-base-image:~# ps -aux|grep redis
root      1104  0.1  0.0  41720  3584 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.211:8004 [cluster]
root      1121  0.1  0.0  41720  3592 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.211:8005 [cluster]
root      1126  0.0  0.0  41720  3704 ?        Ssl  23:00   0:00 /root/soft/redis-4.0.2/bin/redis-server 192.168.1.211:8006 [cluster]
root      1237  0.0  0.0  14228  1092 pts/0    S+   23:03   0:00 grep --color=auto redis
```
## 7. 创建集群

```
./redis-trib.rb create --replicas 1 192.168.1.212:8001 192.168.1.212:8002  192.168.1.212:8003 192.168.1.211:8004  192.168.1.211:8005  192.168.1.211:8006
```
其中，前三个 ip:port 为第一台机器的节点，剩下三个为第二台机器。

选项–replicas 1 表示我们希望为集群中的每个主节点创建一个从节点

运行 redis-trib.rb 命令，会出现如下提示：

```
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
192.168.1.212:8001
192.168.1.211:8004
192.168.1.212:8002
Adding replica 192.168.1.211:8005 to 192.168.1.212:8001
Adding replica 192.168.1.212:8003 to 192.168.1.211:8004
Adding replica 192.168.1.211:8006 to 192.168.1.212:8002
M: a95bb6fbe18763f3155c52605adf16808cd777b7 192.168.1.212:8001
   slots:0-5460 (5461 slots) master
M: 1b2f7085f900834b8056034ee396af76ae670040 192.168.1.212:8002
   slots:10923-16383 (5461 slots) master
S: 94e0611827611ff593340426222c373c87c04aa0 192.168.1.212:8003
   replicates e317b01991d5077622ea4ed387ae84545a4cdc53
M: e317b01991d5077622ea4ed387ae84545a4cdc53 192.168.1.211:8004
   slots:5461-10922 (5462 slots) master
S: 81c550b3256d4a9d09ad373b6e8ffa4d70fbc770 192.168.1.211:8005
   replicates a95bb6fbe18763f3155c52605adf16808cd777b7
S: a58e4383a78bc39a3aaae767631df1c143d29196 192.168.1.211:8006
   replicates 1b2f7085f900834b8056034ee396af76ae670040
Can I set the above configuration? (type 'yes' to accept):
```
输入 yes 即可，然后出现如下内容，说明安装成功。

```
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join....
>>> Performing Cluster Check (using node 192.168.1.212:8001)
M: a95bb6fbe18763f3155c52605adf16808cd777b7 192.168.1.212:8001
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: a58e4383a78bc39a3aaae767631df1c143d29196 192.168.1.211:8006
   slots: (0 slots) slave
   replicates 1b2f7085f900834b8056034ee396af76ae670040
M: 1b2f7085f900834b8056034ee396af76ae670040 192.168.1.212:8002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 81c550b3256d4a9d09ad373b6e8ffa4d70fbc770 192.168.1.211:8005
   slots: (0 slots) slave
   replicates a95bb6fbe18763f3155c52605adf16808cd777b7
S: 94e0611827611ff593340426222c373c87c04aa0 192.168.1.212:8003
   slots: (0 slots) slave
   replicates e317b01991d5077622ea4ed387ae84545a4cdc53
M: e317b01991d5077622ea4ed387ae84545a4cdc53 192.168.1.211:8004
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
## 8. 集群验证
在212机器上连接集群的8001端口的节点，在211连接8005节点，连接方式为 redis-cli -h 192.168.1.212 -c -p 8001  ,加参数 -C 可连接到集群，因为上面 redis.conf 将 bind 改为了ip地址，所以 -h 参数不可以省略。

在212机器上执行 set key1 value1，执行结果如下:

```
192.168.1.212:8001> set key1 value1
-> Redirected to slot [9189] located at 192.168.1.211:8004
OK
192.168.1.211:8004> 
```
在211机器上执行 get key1，执行结果如下:

```
192.168.1.211:8005> get key1
-> Redirected to slot [9189] located at 192.168.1.211:8004
"value1"
```
说明集群运作正常。


# ==安装过程中遇到的问题==
## 1. 没有安装make和gcc

```
apt-get install make
apt-get install gcc
```
## 2. 编译redis报错/deps/hiredis/libhiredis.a
- 在编译redis时报错
```
cd src && make all
make[1]: Entering directory '/root/soft/redis-4.0.2/src'
    LINK redis-server
cc: error: ../deps/hiredis/libhiredis.a: No such file or directory
cc: error: ../deps/lua/src/liblua.a: No such file or directory
Makefile:199: recipe for target 'redis-server' failed
make[1]: *** [redis-server] Error 1
make[1]: Leaving directory '/root/soft/redis-4.0.2/src'
Makefile:6: recipe for target 'all' failed
make: *** [all] Error 2
```
- 解决办法

进入源码包目录下的deps目录中执行
```
cd /root/soft/redis-4.0.2/deps
make geohash-int hiredis jemalloc linenoise lua
```
## 3. make test报错

```
zmalloc.h:50:31: error: jemalloc/jemalloc.h: No such file or directory
zmalloc.h:55:2: error: #error "Newer version of jemalloc required"
make[1]: *** [adlist.o] Error 1
make[1]: Leaving directory `/data0/src/redis-2.6.2/src'
make: *** [all] Error 2
```
解决办法

```
make MALLOC=libc
```
## 4. Redis need tcl 8.5 or newer
解决方法

```
wget http://downloads.sourceforge.net/tcl/tcl8.6.1-src.tar.gz  
tar xzvf tcl8.6.1-src.tar.gz  -C /usr/local/  
cd  /usr/local/tcl8.6.1/unix/  
./configure  
make  
make install 
```
## 5. 安装ruby和redis库

```
apt-get install ruby
gem install redis
```
