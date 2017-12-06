# 安装Elasticsearch-analysis-ik分词器(下面简称IK)

## IK官网

https://github.com/medcl/elasticsearch-analysis-ik

## 安装说明
Elasticsearch中，内置了很多分词器（analyzers），
例如standard （标准分词器）、english （英文分词）和chinese （中文分词）。
其中standard 就是无脑的一个一个词（汉字）切分，所以适用范围广，但是精准度很低；所以我们需要安装ik。

## 下载和配置IK
[IK下载地址](https://github.com/medcl/elasticsearch-analysis-ik)下载和ES匹配的版本：6.0.0
git上有详细的安装说明 可以参考git上面的文档,在这里只提供一种手动打包安装的方式：
手动下载命令：
```
https://codeload.github.com/medcl/elasticsearch-analysis-ik/zip/master
```
解压打包：
```
unzip elasticsearch-analysis-ik-master.zip
mvn clean package
```
将elasticsearch-analysis-ik-master\target\releases下的zip文件拷贝到服务器/home/myzh/soft/download路径下
解压：
```
unzip elasticsearch-analysis-ik-6.0.0.zip
cd elasticsearch
```
配置：
将解压后的文件除config文件的内容都拷贝到/home/myzh/soft/es-cluster/elasticsearch-6.0.0-1/plugins/analysis-ik
```
cp -r ./* ../es-cluster/elasticsearch-6.0.0-1/plugins/analysis-ik
cd /home/myzh/soft/es-cluster/elasticsearch-6.0.0-1/plugins/analysis-ik
rm -r config
```
将config里面的文件拷贝到/home/myzh/soft/es-cluster/elasticsearch-6.0.0-1/config/analysis-ik
```
cd /home/myzh/soft/download/elasticsearch/config
cp -r ./* ../es-cluster/elasticsearch-6.0.0-1/config/analysis-ik
```
elasticsearch-6.0.0-2和elasticsearch-6.0.0-3节点也做同样的配置


## 配置IK热词分词字典
java 程序写个定时任务将你要扩展的自定义分词信息保存到tomcat/webApp/ikdic/IKDicHotRefresh.txt文件中,
可以让程序定时替换文件内容。然后找到%JAVA_HOME%/jre/lib/security/java.policy 文件,添加如下内容:
```
permission java.net.SocketPermission "127.0.0.1:8889","accept";
permission java.net.SocketPermission "127.0.0.1:8889","listen";
permission java.net.SocketPermission "127.0.0.1:8889","resolve";
permission java.net.SocketPermission "127.0.0.1:8889","connect";
```

启动java程序同时更改elasticsearch-6.0.0-1/config/analysis-ik/IKAnalyzer.cfg.xml配置如下
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict"></entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<entry key="remote_ext_dict">http://127.0.0.1:8889/ikdic/IKDicHotRefresh.txt</entry>

	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

同样修改elasticsearch-6.0.0-2/config/analysis-ik/IKAnalyzer.cfg.xml
和elasticsearch-6.0.0-3/config/analysis-ik/IKAnalyzer.cfg.xml的内容。

启动es集群，完成ik分词字典自定义配置


