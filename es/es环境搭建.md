# 安装Elasticsearch(下面简称ES)

## ES官网

https://www.elastic.co/cn/products/elasticsearch

## 安装JDK
要求安装JDK1.8以上

## 创建系统用户
ES不允许以root用户启动，创建用户及用户组：myzh，用于运行elk。
```
# 使用adduser命令，根据提示创建myzh用户，会自动生成myzh的工作目录，即：/home/myzh/
adduser myzh

```
授权myzh数据目录权限：
```
chown -R myzh:myzh es-data
```

## 下载和安装ES
[ES下载地址](https://www.elastic.co/cn/downloads/elasticsearch)下载最新版本：6.0.0

下载路径：/home/myzh/soft/download
下载命令：
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.0.0.tar.gz
```
解压：
```
tar -zxvf elasticsearch-6.0.0.tar.gz
```
## 安装和配置ES
ES作为本地集群进行安装，安装目录：/home/myzh/soft/es-cluster
```
cp -r elasticsearch-6.0.0 ../es-cluster/elasticsearch-6.0.0-1
cp -r elasticsearch-6.0.0 ../es-cluster/elasticsearch-6.0.0-2
cp -r elasticsearch-6.0.0 ../es-cluster/elasticsearch-6.0.0-3
```

修改配置文件：/home/myzh/soft/es-cluster/elasticsearch-6.0.0-1/config/elasticsearch.yml
```
# 修改集群名称,其他节点如果集群名相同，会自动归为一个集群
cluster.name: es-cluster
# 修改节点名称。每个节点名称需要不一样
node.name: node-1
# 修改节点数据目录，该目录需要提前创建，当前系统用户具有读写权限
path.data: /alidata1/es-data/es-data1
# 修改节点日志目录
path.logs: /alidata1/es-data/logs/es-log/
# 修改节点绑定ip
network.host: 192.168.1.211
# 设置对外服务的http端口，默认为9200。
# 该端口是用来让HTTP REST API来访问ElasticSearch（如浏览器访问）
http.port: 9201
# 设置节点之间交互的tcp端口，默认是9300。
# 该端口是传输层监听的默认端口，主要用来程序访问数据传输的时候访问
transport.tcp.port: 9301
# 每个节点这个参数一定要配置，只有配置了，节点之间才能互通，并且保证每个节点访问都能正确匹配到数据，同时还能保证节点创建索引之后不报yellow状态。
discovery.zen.ping.unicast.hosts: ["192.168.1.211:9302", "192.168.1.211:9303"]
```

复制elasticsearch-6.0.0-1/config/elasticsearch.yml到另外两个节点：
```
cp ./elasticsearch-6.0.0-1/config/elasticsearch.yml ./elasticsearch-6.0.0-2/config
cp ./elasticsearch-6.0.0-1/config/elasticsearch.yml ./elasticsearch-6.0.0-3/config
```
修改配置文件，主要修改下面的参数值：

elasticsearch-6.0.0-2：
```
node.name: node-2
path.data: /alidata1/es-data/es-data2
path.logs: /alidata1/es-data/logs/es-log/
http.port: 9202
```
elasticsearch-6.0.0-3:
```
node.name: node-3
path.data: /alidata1/es-data/es-data3
path.logs: /alidata1/es-data/logs/es-log/
http.port: 9203

```

启动：
```
./elasticsearch-6.0.0-x/bin/elasticsearch -d
```
启动如果报错，参考[问题](#q)