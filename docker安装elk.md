# docker的安装

## 1、删除旧版

```shell
 yum remove docker \                  
 docker-client \                  
 docker-client-latest \                  
 docker-common \                  
 docker-latest \                  
 docker-latest-logrotate \                  
 docker-logrotate \                  
 docker-engine           
```


## 2、设置仓库
```shell
 sudo yum install -y yum-utils  
 
 sudo yum-config-manager \    
 --add-repo \    
 https://download.docker.com/linux/centos/docker-ce.repo
 ```


## 3、安装

```
yum install docker-ce docker-ce-cli containerd.io     
```

上述两步由于网络问题可能会失败，如果失败多执行几次
## 4、检查是否安装成功

 ```
   docker -v
 ```


## ![docker-v](./images/docker-v.png)

## 5、设置docker开机自启

```
systemctl enable docker
```


## 6、启动命令：

```
   systemctl start docker
```

# Docker安装Elasticsearch

## 1. 下载镜像文件
```shell
docker pull elasticsearch:7.4.2
```

## 2. 配置挂载数据文件夹
```shell
# 创建配置文件目录
mkdir -p /mydata/elasticsearch/config
## 创建数据目录
mkdir -p /mydata/elasticsearch/data
# 将/mydata/elasticsearch/文件夹中文件都可读可写
chmod -R 777 /mydata/elasticsearch/
# 配置任意机器可以访问 elasticsearch
echo http.host: 0.0.0.0 >/mydata/elasticsearch/config/elasticsearch.yml
```
## 3. 启动Elasticsearch
```shell
# 命令后面的 \是换行符，注意前面有空格
   docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
   -e discovery.type=single-node \
   -e ES_JAVA_OPTS="-Xms64m -Xmx512m" \
   -v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
   -v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
   -v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
   -d elasticsearch:7.4.2
```

       -p 9200:9200 -p 9300:9300：向外暴露两个端口，9200用于HTTP REST API请求，9300 ES 在分布式集群状态下 ES 之间的通信端口；
       -e discovery.type=single-node：es 以单节点运行
       -e ES_JAVA_OPTS=-Xms64m -Xmx512m 设置启动占用内存，不设置可能会占用当前系统所有内存，生产中一般为32个G,可不写此配置
       -v：挂载容器中的配置文件、数据文件、插件数据到本机的文件夹；
       -d elasticsearch:7.6.2：指定要启动的镜像
如果启动不成功，通过 `docker logs -f -n 500 elasticsearch`查看日志，有问题一定要学会查看日志

## 4、访问 IP:9200 看到返回的 json 数据说明启动成功。
如果启动不成功可先在Linux上执行 `lsof -i:9200` 查看指定端口是否启动，如果启动则查看防火墙是否关闭或者端口是否开放
![es启动成功.jpg](images/es%E5%90%AF%E5%8A%A8%E6%88%90%E5%8A%9F.jpg)

## 4. 设置 Elasticsearch 随Docker启动
```shell
# 当前 Docker 开机自启，所以 ES 现在也是开机自启
docker update elasticsearch --restart=always
```
# docker 安装Kibana
## 1. 启动可视化Kibana
```shell
   docker run --name kibana \
   -e ELASTICSEARCH_HOSTS=http://60.205.122.253:9200 \
   -e "I18N_LOCALE=zh-CN" \
   -p 5601:5601 \
   -d kibana:7.4.2
```

-e ELASTICSEARCH_HOSTS=http://60.205.122.253:9200: 这里要设置成自己的服务器地址，不要输入127.0.0.1，因为docker容器会在容器中寻找
   
浏览器输入60.205.122.253:5601 测试：
![kibana启动成功.jpg](images/kibana%E5%90%AF%E5%8A%A8%E6%88%90%E5%8A%9F.jpg)

## 2. 设置 Kibana 随Docker启动
```shell
# 当前 Docker 开机自启，所以 kibana 现在也是开机自启
docker update kibana --restart=always
```
# docker安装logstash

## 1、创建目录
```shell
mkdir -p /mydata/logstash/config
```

## 2、创建 logstash.conf文件
```shell
tee > /mydata/logstash/config/logstash.conf <<-'EOF'
input {                                                                                                          
tcp {
#logstash的端口号
port => 5044     
#以json的形式存储                                                                                        
codec => "json"                                                                                          
}                                                                                                            
}

output {                                                                                                         
elasticsearch {
hosts => ["http://60.205.122.253:9200"] # 此处用docker部署不能填写127.0.0.1，只能填写docker能访问到的地址
index => "logstash-%{[server_name]}-%{+YYYY.MM.dd}"  #索引名
}                                                                   
stdout {
codec => rubydebug
}                                                                              
}
EOF
```
## 3、拉取镜像
```shell
docker pull logstash:7.4.2
```
## 4、启动镜像
```shell
docker run --name logstash -p 5044:5044  \
-v /mydata/logstash/config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
-d logstash:7.4.2
```
-v /mydata/logstash/config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf为挂载目录，可挂载在其他目录下，但是要存在该目录
## 5、随docker自启
```shell
docker update logstash --restart=always
```
# SpringBoot集成elk
1、导入依赖
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```


2、新增logback-spring.xml的配置文件
```shell
#---配置方式一
#spring配置方式
<?xml version="1.0" encoding="UTF-8"?>
 <configuration debug="false">
    <!--提取配置文件中的服务名-->
     <springProperty scope="context" name="springApplicationName" source="spring.application.name" />
     <springProperty scope="context" name="springProfilesActive" source="spring.profiles.active"/>
     <property name="LOG_HOME" value="logs/demo.log" />
     <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
         <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
             <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
         </encoder>
     </appender>

     <appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
         <destination>60.205.122.253:5044</destination>
         <encoder class="net.logstash.logback.encoder.LogstashEncoder" >
             <!--添加自定义属性 -->
             <customFields>{"server_name": "${springApplicationName}-${springProfilesActive}"}</customFields>
         </encoder>
     </appender>

     <springProfile name="dev">
        <root>
            <level value="INFO"/>
            <appender-ref ref="STDOUT"/>
            <appender-ref ref="FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="logstash"  />
        </root>
    </springProfile>
 </configuration>


---配置方式二
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
   <springProperty scope="context" name="springApplicationName" source="spring.application.name" />
   <property name="LOG_HOME" value="logs/demo.log" />
   <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
       <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
           <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
       </encoder>
   </appender>

   <!--DEBUG日志输出到LogStash-->
   <appender name="LOG_STASH_DEBUG" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
       <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
           <level>DEBUG</level>
       </filter>
       <destination>60.205.122.253:5044</destination>
       <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
           <providers>
               <timestamp>
                   <timeZone>Asia/Shanghai</timeZone>
               </timestamp>
               <!--自定义日志输出格式-->
               <pattern>
                   <pattern>
                       {
                       "project": "elk",
                       "level": "%level",
                       "service": "${springApplicationName:-}",
                       "pid": "${PID:-}",
                       "thread": "%thread",
                       "class": "%logger",
                       "message": "%message",
                       "stack_trace": "%exception"
                       }
                   </pattern>
               </pattern>
           </providers>
       </encoder>
   </appender>

   <root >
       <appender-ref ref="STDOUT" />
       <appender-ref ref="LOG_STASH_DEBUG" />
   </root>
</configuration>
```



