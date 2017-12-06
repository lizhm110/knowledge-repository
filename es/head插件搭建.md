# 安装Elasticsearch-head插件(下面简称ES-head)

## ES-head官网

https://github.com/mobz/elasticsearch-head

## 安装说明
便于es数据可视化

## 下载和安装ES-head
[ES-head下载地址](https://github.com/mobz/elasticsearch-head)

下载路径：/home/myzh/soft/es-cluster
下载命令：
```
git clone git://github.com/mobz/elasticsearch-head.git
```
安装：
```
cd elasticsearch-head
npm install
grunt server
```
启动：
单独启动
```
grunt server
```
## 配置ES
修改配置文件：/home/myzh/soft/es-cluster/elasticsearch-6.0.0-1/config/elasticsearch.yml
```
# 增加新的参数，这样head插件可以访问es
http.cors.enabled: true
http.cors.allow-origin: "*"
```

复制elasticsearch-6.0.0-1/config/elasticsearch.yml到另外两个节点：
```
cp ./elasticsearch-6.0.0-1/config/elasticsearch.yml ./elasticsearch-6.0.0-2/config
cp ./elasticsearch-6.0.0-1/config/elasticsearch.yml ./elasticsearch-6.0.0-3/config
```

启动：
直接启动ES就可以,可以不用执行启动head命令
```
./elasticsearch-6.0.0-x/bin/elasticsearch -d
```

