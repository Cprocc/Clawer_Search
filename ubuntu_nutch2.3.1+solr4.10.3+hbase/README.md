auther:赵新诚

---

# 本文介绍了在linux下，配置nutch抓取固定网页内容，放在hbase，并通过solr检索
## 1.运行环境
- ubuntu 18.04.1 / 运行于virtualbox虚拟机
- jdk1.8.0_181 / 
- hbase-0.98.8-hadoop2 / 
- apache-nutch-2.3.1 / 已经是最新的2.X版本了，依然发布于15年
- solr-4.10.3 / 为了与nutch版本对应，如果下载其他版本会导致实际运行crawler时出现长时间暂停

## 2.安装java，配置环境变量
1. 下载jdk。如果以经安装有jdk的话要进行检查，hbase要求适配的jdk版本为1.6及以后，官方下载
http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
2. 解压安装包。在ubuntu下，下载过程可以选择归档，然后选择提取文件到指定目录，即为解压到指定目录。当然也可以用 `tar xvf jdk-8u25-linux-x64.tar.gz`命令解压
3. 配置环境变量。ubuntu下配置环境变量有很多种方法，我们为当前用户配置全局变量 `sudo gedit ~/.bashrc`,sudo的作用是强行提升当前用户的权限，第一次需要键入用户密码。在文档末尾添加如下配置，JAVA_HOME=后应该跟解压的jdk的目录
```js
export JAVA_HOME=/usr/java/jdk1.8.0_181
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
`source ~/.bashrc` 使该配置立即生效，在修改时会出现终端会出现一系列warning，可以忽略。配置完成后输入 `java -version` / `javac -version`来验证环境变量配置是否成功

## 3.安装hbase 
1. 下载 hbase-0.98.8-hadoop2-bin.tar.gz ，官方地址 http://archive.apache.org/dist/hbase/hbase-0.98.8/hbase-0.98.8-hadoop2-bin.tar.gz

2. 解压安装包
3. 配置hbase环境变量，本例采用配置所有用户全局变量的方式`sudo gedit /etc/profile`,在文档末尾增加,HBASE_HOME为hbase解压地址。输入`hbase version`检查环境变量是否配置成功。

```js
export HBASE_HOME=/home/zxc/hbase-0.98.8-hadoop2
export PATH=$PATH:$HBASE_HOME/bin
```
4. 配置hbase，修改hbase-0.98.8-hadoop2/cnf/下的hbase-site.xml文件，增加如下代码，然后在hbase-0.98-8-hadoop2目录下 `./bin/start-hbase.sh`检测是否能正常启动。有时候会出现启动一会儿自动关闭的情况，多半是配置出了问题，此时我们进入`hbase shell`，`list`，或者打开浏览器进入localhost:60010,能成功加载就是没有问题的。

```xml
<configuration>
    <property>
        <name>hbase:rootdir</name>
        <value>file://home/zxc/hbase-0.98.8-hadoop2/data/hbase</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/home/zxc/hbase-0.98.8-hadoop2/data/zookeeper</value>
    </property>
</configuration>
```

## 4.Nutch的安装配置与编译
1. 下载 http://nutch.apache.org/downloads.html 后解压
2. 配置：
    - 进入ivy目录，修改ivy.xml文件，找到` <dependency org="apache.org.gora" name="gora.hbase" rev=0.6.1 conf="*->default"/> `，并取消注释，将Nutch的默认的结果存储方式变更为Hbase。 添加代码`<dependency org="org.apache.hbase" name="hbase-common" rev="0.98.8-hadoop2" conf="*->default" />`为nutch添加hbase相关的jar包。
    - 进入conf目录，修改nutch-site.xml文件，增加Nutch默认存储类型，爬虫名字，和插件库的位置三个属性值，如下

    ```xml
    <property>  
          <name>storage.data.store.class</name>  
          <value>org.apache.gora.hbase.store.HBaseStore</value>  
          <description>Default class for storing data</description>  
    </property>  
    <property>  
            <name>http.agent.name</name>  
            <value>My Nutch Spider</value>  
    </property>  
    <property>
    <name>plugin.includes</name>
    <value>protocol-httpclient|urlfilter-regex|index-(basic|more)|query-(basic|site|url|lang)|indexer-solr|nutch-extensionpoints|protocol-httpclient|urlfilter-regex|parse-(text|html|msexcel|msword|mspowerpoint|pdf)|summary-basic|scoring-opic|urlnormalizer-(pass|regex|basic)protocol-http|urlfilter-regex|parse-(html|tika|metatags)|index-(basic|anchor|more|metadata)</value>
    </property>
    ```

    - 修改conf文件夹下的regex-urlfilter.txt， ` # accept anything else
  #+.
  +^http://([a-z0-9]*\.)*nutch.apache.org/ ` ,设置抓取url的规则

   - 为了确保存储在hbase上，修改conf的gora.properties文件，增加配置 `gora.datastore.default=org.apache.gora.hbase.store.HBaseStore`
3. 编译,在nutch目录下 `ant runtime`,如果跳出没有ant命令，根据提示apt安装后再次运行，此步骤需要下载依赖，花费时间较长，一般卡住了，不要着急。
4. 在编译生成的runtime/local文件夹下，新建url文件夹，在该文件夹下新建seed.txt文件，并将要抓取的网址写在其中。
5. 在 runtime/local/conf下，修改nutch-site.xml文件，增加下属性，保证插件加载的正确

```xml
 <property>
      <name>plugin.folders</name>
     <value>plugins</value>
 </property>
 ```

## 5. Solr安装
1. 下载4.10.3 版本的solr  http://archive.apache.org/dist/lucene/solr/4.10.3 ，后解压
2. 结合nutch复制nutch/runtime/local/conf目录下的schema.xml到solr/example/solr/collection1/conf目录下，选择替换
3. 检测安装是否成功,solr-4.10.3/example/目录下运行 `java –jar start.jar`,打开浏览器localhost:8983,成功连接

## 6. 集成运行
1. 启动hbase
2. 启动solr
3. 进入Nutch安装目录的/runtime/local/子目录下，启动Nutch(seed.txt url种子文件夹，是爬取网页的起点，mycrawl 爬虫名称，http://localhost:8983/solr solr地址，2 爬取深度):
`./bin/crawl ./url/seed.txt mycrawl http://localhost:8983/solr 2`

## 值得注意
- 此过程没有太多意外性，环境变量配置要正确
- 环境变量配置知识https://blog.csdn.net/hipkai/article/details/41548677
- 在最后的运行过程中，会出现某些步骤时间较长，可能是因为版本问题，不过本例测试时时间正常，爬取过程https://www.cnblogs.com/lujinhong2/p/4637286.html这篇博客有详细介绍，可供参照
