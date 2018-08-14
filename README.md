# Clawer_Search
DF-team
# 本文介绍了在windows下，配置nutch抓取固定网页内容，放在本地，并通过tomcat部署nutch进行爬取内容检索的过程
## 1.写在前面
- nutch官方发布的有两条产品线，1.x版本是基于Hadoop架构的，底层存储使用的是HDFS，而2.x通过使用Apache Gora，使得Nutch可以访问HBase、Accumulo、Cassandra、MySQL、DataFileAvroStore、AvroStore等NoSQL
- 官方从1.7版本开始不再提供完整的部署文件，需要手动编译，而windows下的手动编译又需要额外的插件，所以本文选择了较为原始的1.1版本，仅作为自建搜索引擎的初次尝试。其他版本的使用将在后续进行。
## 2.配置nutch-1.1
- 需要jDK环境
- 在windows端有两种配置方法，1.Eclipse中配置；2.用cygwin在windows下模拟Linux环境配置
- 鉴于我自己的电脑上已安装过Eclipse，所以本教程采用在Eclipse中配置
1. 下载对应的nutch版本  http://archive.apache.org/dist/nutch/ 
2. 在Eclipse中新建java project，project name假设为nutch-1.1-test
3.  - 将nutch-1.1\src\java目录下的org文件夹整个复制到新建Java项目nutch-1.1-test的src包下
    - 将nutch-1.1目录下的conf、lib、plugins复制到与src同级目录
    - 在conf目录上单击右键→BuildPath→UseasSourceFolder，将配置文件conf加到path中   
    - 在Nutch-1.1-test目录上单击右键→BuildPath→ConfigureBuildPath…，然后将lib中所有的jar包添加到libraries里面
4. 修改Eclipse下文件的配置文件:  
    conf下的nutch-site.xml, 

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>

<property>
<name>http.agent.name</name>
<value>*</value>
</property>

</configuration> 
```

---
crawl-urlfilter.txt,

将 `# accept hosts in MY.DOMAIN.NAME  +^http://([a-z0-9]*\.)*MY.DOMAIN.NAME/`替换成 `# accept hosts in MY.DOMAIN.NAME  +^http://([a-z0-9]*\.)*`

5. 在根目录新建urls文件夹，并在其中新建一个urls.txt文件，是用来存放要爬取的url测试Crawl类。本例中urls中的内容 
   
   ```
   http://www.baidu.com
   http://www.pku.edu.cn
   ```

6. 在MyEclipse中，展开src目录，找到org/apache/nutch/crawl包下的Crawl.java类，双击打开。右击打开RunConfigurations配置一下运行参数：Run as →  Run Configuration → Arguments 

    Program arguments输入：`crawl urls -dir out -threads 20 -depth 4`

    crawl:是nutch的爬虫命令
    urls:是新建的urls文件夹，用来读取要爬取得网址
    -dir out：是爬去结果的输出路径、我们制定爬取的结果放置在与项目同路径的out（可自己取名）文件夹下，运行结束后在本地out文件夹下会生成5个文件夹：crawldb、index、indexes、linkdb、segments其中，linkdb是存放URL的互联关系的，是所有页面下载完成后所分析的来；indexes为每次下载的索引目录；index为Lucene格式的索引目录，为indexes中的索引合并后的完整索引，这一点从控制台的最后输出也可以看出来；segments存放抓取的页面信息，抓取多少层数的页面，就会有几个子文件夹，本文抓取层数为5，所以该文件下有5个子文件，每个子文件中又有一些子目录：其中，context为下载的页面内容；crawl_fetch为下载URL的状态；crawl_generate为待下载的URL集合信息；crawl_parse为外部链接库；parse_data为下载URL解析的外部链接及其他数据；parse_text为URL解析的文本内容
    -threads 20:是开启的进程数
    -depth 2:是要爬取得深度（可以自己试着调节，从小到大）
    -topN 10:是显示前10
    VM arguments输入：-Xms32m -Xmx800m 这是设置内存大小，如果不设置会导致内存溢出异常
## 3.将搜索布置在tomcat上

1. 首先配置tomcat，有很多相关教程，此处省略
2. 在下载的nutch-1.1中找到 nutch-1.1.war文件，复制至tomcat文件夹下的webapps目录下
3. 启动tomcat这时候会自动解压nutch-1.1.war，关闭tomcat
4. webapps\nutch-1.1\WEB-INF\classes目录下的nutch-site.xml文件，修改其中的属性，添加：

   ```xml
    <property>
    <name>searcher.dir</name>
    <value>上文中out的绝对路径</value>
    </property>

    ```
5. Tomcat添加中文字符支持。因为tomcat默认不支持中文字符集，所以还需要添加中文字符支持,找到tomcat根目录下的conf中的server.xml，修改其中的Connector port属性，在属性的最后增加 
    ` URIEncoding="UTF-8" useBodyEncodingForURI="true"` 如下：

```xml 
    <Connector port="8080" protocol="HTTP/1.1"
                ...
                URIEncoding="UTF-8"
                useBodyEncodingForURI="true"/>
```

## 值得注意
- 要严格注意nutch的下载版本
- 有些网站爬不下来数据，目前还不知道原因如: www.dlut.edu.cn