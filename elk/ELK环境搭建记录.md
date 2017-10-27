# 安装

## ELK官网


https://www.elastic.co/cn/start?elektra=home&storm=banner

## 安装JDK
要求安装JDK1.8以上

## 创建系统用户
elasticsearch不允许以root用户启动，创建用户及用户组：yideb-u，用于运行elk。
```
# 使用adduser命令，根据提示创建yideb-u用户，会自动生成yideb-u的工作目录，即：/home/yideb-u/
adduser yideb-u

```
授权yideb-u数据目录权限：
```
chown -R yideb-u:yideb-u elk-data
```

## 下载和安装ELK组件
[ELK官网](https://www.elastic.co/cn/start?elektra=home&storm=banner)下载最新版本：5.5.1

下载路径：/home/yideb-u/software/download
```
elasticsearch-5.5.1.tar.gz
kibana-5.5.1-linux-x86_64.tar.gz
logstash-5.5.1.tar.gz
```
解压：
```
tar -zxvf elasticsearch-5.5.1.tar.gz
tar -zxvf kibana-5.5.1-linux-x86_64.tar.gz
tar -zxvf logstash-5.5.1.tar.gz
```

## 安装和配置ES
ES作为本地集群进行安装，安装目录：/home/yideb-u/software/es-cluster
```
cp -r elasticsearch-5.5.1 ../es-cluster/elasticsearch-5.5.1-1
cp -r elasticsearch-5.5.1 ../es-cluster/elasticsearch-5.5.1-2
cp -r elasticsearch-5.5.1 ../es-cluster/elasticsearch-5.5.1-3
```

修改配置文件：/home/yideb-u/software/es-cluster/elasticsearch-5.5.1-1/config/elasticsearch.yml
```
# 修改集群名称,其他节点如果集群名相同，会自动归为一个集群
cluster.name: es-cluster
# 修改节点名称。每个节点名称需要不一样
node.name: node-1
# 修改节点数据目录，该目录需要提前创建，当前系统用户具有读写权限
path.data: /alidata1/elk-data/es-data1
# 修改节点日志目录
path.logs: /alidata1/elk-data/logs/es-log/
# 修改节点绑定ip
network.host: 10.251.197.233
# 修改对外端口
http.port: 15001
# 新增自动创建索引名，可以使用*作为通配符，主要是为了logstash写入数据时自动创建索引库，否则会写入失败
# .security,.monitoring*,.watches,.triggered_watches,.watcher-history* 这些索引库是X-Pack所需要的，如果安装，可以删除
# log*是自定义索引库，都以log开头
action.auto_create_index: .security,.monitoring*,.watches,.triggered_watches,.watcher-history*,log*
```

复制elasticsearch-5.5.1-1/config/elasticsearch.yml到另外两个节点：
```
cp ./elasticsearch-5.5.1/config/elasticsearch.yml ./elasticsearch-5.5.1-2/config
cp ./elasticsearch-5.5.1/config/elasticsearch.yml ./elasticsearch-5.5.1-3/config
```
修改配置文件，主要修改下面的参数值：

elasticsearch-5.5.1-2：
```
node.name: node-2
path.data: /alidata1/elk-data/es-data2
path.logs: /alidata1/elk-data/logs/es-log/
http.port: 15002
```
elasticsearch-5.5.1-3:
```
node.name: node-3
path.data: /alidata1/elk-data/es-data3
path.logs: /alidata1/elk-data/logs/es-log/
http.port: 15003

```

启动：
```
./elasticsearch-5.5.1/bin/elastic -d
```
启动如果报错，参考[问题](#q)
## 安装和配置Kibana

安装目录：/home/yideb-u/software/kibana-5.5.1

```
cp -r kibana-5.5.1-linux-x86_64 ../kibana-5.5.1
```
修改配置文件：kibana.yml
```
# 对外端口
server.port: 15005
# 允许访问的ip地址，0.0.0.0不限制
server.host: "0.0.0.0"
# 使用代理访问时，指定代理url的基本路径
server.basePath: "/elk"
# 服务器名称
server.name: "ELK"
# es服务器地址
elasticsearch.url: "http://10.251.197.233:15001"
```

## 安装和配置Logstash
logstash作为日志收集服务器，收集从各个业务系统发送过来的日志数据。

安装目录：/home/yideb-u/software/logstash-5.5.1
```
cp -r logstash-5.5.1 ../logstash-5.5.1
```
根据服务器配置，修改logstash配置文件：logstash.yml
```
# 管道工作线程数，建议服务器cpu数
pipeline.workers: 8
pipeline.output.workers: 8
# 管道批量处理数，批量处理可以降低后端写入压力，降低CPU资源消耗
pipeline.batch.size: 10000
pipeline.batch.delay: 50

```
创建管道配置文件，用于收集日志：

```

input {
  tcp {
    host => "0.0.0.0"
    port => "15007"
    mode => "server"
    type => "log"
    add_field => {
      "log_env" => "test"
    }
    codec => multiline {
        pattern => "^\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}"
        negate => true
        what => "previous"
    }
  }
  tcp {
    host => "0.0.0.0"
    port => "15008"
    mode => "server"
    type => "log"
    add_field => {
      "log_env" => "prod"
    }
    codec => multiline {
        pattern => "^\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}"
        negate => true
        what => "previous"
    }
  }
}
filter {
   grok {
       match => [ "message", "%{NOTSPACE:day} %{NOTSPACE:time} %{LOGLEVEL:level} \s{0,1}%{NOTSPACE:thread-id} %{NOTSPACE:trace-id} %{NOTSPACE:appName} %{NOTSPACE:logType} %{NOTSPACE:bizType} %{GREEDYDATA:classinfo} - %{GREEDYDATA:msginfo}" ]
   }
   
   if "ERROR" == [level] {
       metrics {
        meter => "log_%{level}"
        add_tag => "monitor"
        ignore_older_than => 10
      }
	  if "monitor" in [tags] {
        ruby {
            code => "event.cancel if event.get('[log_ERROR][rate_5m]') < 100"
        }
      }
   }

   if "metric" == [bizType] {
        json {
            source => "msginfo"
            target => "metric"
			add_tag => "metric"
            remove_field => ["message"]
        } 
   }
}
output {
   # 如果event中标记为“monitor”，表示只发送计数的消息。
    if "monitor" in [tags] {
      elasticsearch {
        hosts => ["0.0.0.0:15001"]
        index => "log-monitor-error-%{+YYYY.MM.dd}"
        document_type => "log-monitor-error"
        flush_size => 2000
        idle_flush_time => 10
        sniffing => true
        template_overwrite => true
        user => elastic
        password => changeme
      }
	  # 内网环境下无法发送邮件，暂时注释
	  #email {
      #  to => "chensg@miyzh.com"
      #  from => "myzh@miyzh.com"
	  #	address => "smtp.qiye.163.com"
	  # 	username => "myzh@miyzh.com"
	  #	password => "myzh1234ydb"
      #  subject => "Warning: %{appName}-%{log_env} 最近1分钟内发生[metric][data][count]次以上错误，请关注"
      #   htmlbody => "%{appName}-%{log_env} 最近1分钟内发生%{[metric][data][count]}次以上错误，最近一次错误日志追踪id:%{trace-id}，错误日志内容：\r\n%{message}"
      #  body => "Tags: %{tags}\\n\\Content:\\n%{message}"
      #}
    } else if "metric" in [tags] {
      elasticsearch {
        hosts => ["0.0.0.0:15001"]
        index => "log-metric-%{+YYYY.MM.dd}"
        document_type => "log-metric"
        flush_size => 2000
        idle_flush_time => 10
        sniffing => true
        template_overwrite => true
        user => elastic
        password => changeme
      }
    } else if "trace" in [bizType] {
      elasticsearch {
        hosts => ["0.0.0.0:15001"]
        index => "log-trace-%{log_env}-%{+YYYY.MM.dd}"
        document_type => "log-trace"
        flush_size => 2000
        idle_flush_time => 10
        sniffing => true
        template_overwrite => true
        user => elastic
        password => changeme
      }
    }else {
      elasticsearch {
        hosts => ["0.0.0.0:15001"]
        index => "log-data-%{log_env}-%{logType}-%{appName}-%{+YYYY.MM.dd}"
        document_type => "log-data-%{log_env}-%{logType}-%{appName}"
        flush_size => 2000
        idle_flush_time => 10
        sniffing => true
        template_overwrite => true
        user => elastic
        password => changeme
      }
    }
}

```


##  问题汇总


### 1 logstash
##### 1) 新的out写入elasticsearch的index:log-monitor,但在kibana监控中发现未出现对应index
原因：elasticsreach配置文件中，需要配置自动创建index的配置，添加对应配置即可：

```xml
action.auto_create_index: .security,.monitoring*,.watches,.triggered_watches,.watcher-history*,log*
```

#### 2) Logstash向Elasticsearch插入数据报错

报错信息如下：
```xml
{:timestamp=>"2016-07-06T00:02:22.289000+0800", :message=>"retrying failed action with response code: 429 ({\"type\"=>\"es_rejected_execution_exception\", \"reason\"=>\"rejected execution of org.elasticsearch.transport.TransportService$4@74bf8f58 on EsThreadPoolExecutor[bulk, queue capacity = 50, org.elasticsearch.common.util.concurrent.EsThreadPoolExecutor@48d6d348[Running, pool size = 32, active threads = 32, queued tasks = 50, completed tasks = 787682]]\"})", :level=>:info}
```
Logstash从批量接收数据然后批量写入到elasticsearch，从报错信息来看初步判断是写入到elasticsearch的速度赶不上读取的速度

解决：调整logstash参数
```xml
# pipeline线程数，官方建议是等于CPU内核数
pipeline.workers: 24
# 实际output时的线程数
pipeline.output.workers: 24
# 每次发送的事件数
pipeline.batch.size: 5000
# 发送延时
pipeline.batch.delay: 5
```


参考：[ELK集群故障处理](http://john88wang.blog.51cto.com/2165294/1796125)
      [ELK的一次吞吐量优化](http://blog.csdn.net/ypc123ypc/article/details/69945031)

### 2 kibana

### 3 elasticsearch

#### X-pack5.5.1破解
参考：[ES X-Pack 5.4.3破解](https://my.oschina.net/guol/blog/1204258)

创建文件：LicenseVerifier.java
```java
package org.elasticsearch.license;

import java.nio.*;
import java.util.*;
import java.security.*;
import org.elasticsearch.common.xcontent.*;
import org.apache.lucene.util.*;
import org.elasticsearch.common.io.*;
import java.io.*;

public class LicenseVerifier
{
    public static boolean verifyLicense(final License license, final byte[] encryptedPublicKeyData) {
        return true;
    }

    public static boolean verifyLicense(final License license) {
        return true;
    }
}
```

编译class文件：
```
yideb-u@iZ256dqtgzoZ:~/software/download/track$ javac -cp "/home/yideb-u/software/es-cluster/elasticsearch-5.5.1-1/lib/elasticsearch-5.5.1.jar:/home/yideb-u/software/es-cluster/elasticsearch-5.5.1-1/lib/lucene-core-6.6.0.jar:/home/yideb-u/software/es-cluster/elasticsearch-5.5.1-1/plugins/x-pack/x-pack-5.5.1.jar" LicenseVerifier.java 
```
编译完成：
```
yideb-u@iZ256dqtgzoZ:ll
drwxrwxr-x 3 yideb-u yideb-u    4096 Aug 17 16:18 ./
drwxr-xr-x 3 yideb-u yideb-u    4096 Aug 17 16:06 ../
-rw-rw-r-- 1 yideb-u yideb-u     410 Aug 17 16:05 LicenseVerifier.class
-rw-rw-r-- 1 yideb-u yideb-u     488 Aug 17 16:03 LicenseVerifier.java
drwxrwxr-x 5 yideb-u yideb-u    4096 Aug 17 16:09 x-pack-5.5.1/
-rw-rw-r-- 1 yideb-u yideb-u 3656275 Aug 17 16:09 x-pack-5.5.1.jar
-rw-r--r-- 1 yideb-u yideb-u 3627700 Aug 17 16:10 x-pack-5.5.1.old.jar
-rw-rw-r-- 1 yideb-u yideb-u    1204 Aug 17 16:18 xpack-license.json
```

替换class文件：
 解压：
    
```
cp /home/yideb-u/software/es-cluster/elasticsearch-5.5.1-1/plugins/x-pack/x-pack-5.5.1.jar ./x-pack-5.5.1
cd x-pack-5.5.1/
jar xvf x-pack-5.5.1.jar
```
替换：
```
cp org/elasticsearch/license
rm -f LicenseVerifier.class
mv ~/software/download/track/LicenseVerifier.class .
```
打包：
```
jar cvf x-pack-5.5.1.jar .
mv x-pack-5.5.1.jar ../
```

申请license：    
```
#来此注册，并下载license文件
https://license.elastic.co/registration
```
修改license文件：
```
{"license":{"uid":"34713f0e-a7c1-4ab9-b39b-42455367efb7","type":"platinum","issue_date_in_millis":1499299200000,"expiry_date_in_millis":2524579200999,"max_nodes":1000,"issued_to":"chen chen (明医众禾)","issuer":"Web Form","signature":"AAAAAwAAAA0VQbrMMTR/N8EVvoB3AAABmC9ZN0hjZDBGYnVyRXpCOW5Bb3FjZDAxOWpSbTVoMVZwUzRxVk1PSmkxaktJRVl5MUYvUWh3bHZVUTllbXNPbzBUemtnbWpBbmlWRmRZb25KNFlBR2x0TXc2K2p1Y1VtMG1UQU9TRGZVSGRwaEJGUjE3bXd3LzRqZ05iLzRteWFNekdxRGpIYlFwYkJiNUs0U1hTVlJKNVlXekMrSlVUdFIvV0FNeWdOYnlESDc3MWhlY3hSQmdKSjJ2ZTcvYlBFOHhPQlV3ZHdDQ0tHcG5uOElCaDJ4K1hob29xSG85N0kvTWV3THhlQk9NL01VMFRjNDZpZEVXeUtUMXIyMlIveFpJUkk2WUdveEZaME9XWitGUi9WNTZVQW1FMG1DenhZU0ZmeXlZakVEMjZFT2NvOWxpZGlqVmlHNC8rWVVUYzMwRGVySHpIdURzKzFiRDl4TmM1TUp2VTBOUlJZUlAyV0ZVL2kvVk10L0NsbXNFYVZwT3NSU082dFNNa2prQ0ZsclZ4NTltbU1CVE5lR09Bck93V2J1Y3c9PQAAAQBc7LBljO0S3uI0ZQ3JD8QoR15H0ejVUc27jLD/rC67+MAfg+LPJwCAccU7QxzXQ+XZTLTSL8WV/riyuzDdtDGrYp/PaS8ybqOSpPLkwbHmB1ptMpekERV8WotHCq4kFFuZWurhVgOCAtwV3BACFnvg4ytGsN+SujZPo2Me6QQL9/mVrem2pNW3Y0aCd3LeURRoIyPT9IbxMcO0dlHAm/Gz4uYRUL2gQv49TN2ujke+p5KQDKL4YBH67Y0R8EHT+P3J16Me5wNPIILv7+7RZ46HncRxxe20X9fv5MVqO3loJVgMCtCJMUrPTmF5Jp10Hn5HKemFy8PRm1c2fYlr2hxg","start_date_in_millis":1502928000000}}
```
 ps：platinum表示白金版，可以使用所有功能。其他的如expiry_date_in_millis、max_nodes等根据自己需要修改即可。
 
 重启集群：
 ```
 elasticsearch
 kibina
 ```
 
注册新license：

root用户登录安装curl:
 ```
apt-get install curl
 ```
查看当前license：
 ```
yideb-u@iZ256dqtgzoZ:~/software/download/track$ curl -XGET -u elastic:changeme 'http://10.251.197.233:15001/_license'
{
  "license" : {
    "status" : "active",
    "uid" : "d83d804e-7821-4370-8a03-25e7cf5f4e10",
    "type" : "trial",
    "issue_date" : "2017-08-11T13:07:11.632Z",
    "issue_date_in_millis" : 1502456831632,
    "expiry_date" : "2017-09-10T13:07:11.632Z",
    "expiry_date_in_millis" : 1505048831632,
    "max_nodes" : 1000,
    "issued_to" : "es-cluster",
    "issuer" : "elasticsearch",
    "start_date_in_millis" : -1
  }
}
 
 ```
 
安装新license:
 ```
curl -XPUT -u elastic:changeme 'http://10.251.197.233:15001/_xpack/license?acknowledge=true' -d @guodalu.jso
 ```
 查看当前license：
 ```
yideb-u@iZ256dqtgzoZ:~/software/download/track$ curl -XGET -u elastic:changeme 'http://10.251.197.233:15001/_license'
 {
  "license" : {
    "status" : "active",
    "uid" : "34713f0e-a7c1-4ab9-b39b-42455367efb7",
    "type" : "platinum",
    "issue_date" : "2017-07-06T00:00:00.000Z",
    "issue_date_in_millis" : 1499299200000,
    "expiry_date" : "2049-12-31T16:00:00.999Z",
    "expiry_date_in_millis" : 2524579200999,
    "max_nodes" : 1000,
    "issued_to" : "chen chen (明医众禾)",
    "issuer" : "Web Form",
    "start_date_in_millis" : 1502928000000
  }
}
 
 ```


#### 1) 集群冷热数据分配：[elk之高性能集群搭建](http://blog.csdn.net/daiyudong2020/article/details/51195213)

#### 1) 集群冷热数据分配：[elk之高性能集群搭建](http://blog.csdn.net/daiyudong2020/article/details/51195213)