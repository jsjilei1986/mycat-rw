部署mysql主从复制,本文介绍如何搭建`mycat中间件`,并用`mycat`来做`读写分离`.

配置文件以及文档地址:

系统环境
=======
* docker 1.12.3
* mysql5.7.17
* mycat1.6

要点说明
======
* 看上篇文章的详细介绍
* 暴露`mysql` `mycat`端口号,方便管理
* 本文直接从`docker-compose.yml`开始

Begin
======

### docker-compose.yml文件

为了看起来方便,咱还是一起都贴出来吧
```
version: '2'
services:
  m1:
    build: ./master
    container_name: m1
    volumes:
      - /home/ssab/config/mysql-master/:/etc/mysql/:ro
      - /etc/localtime:/etc/localtime:ro
      - /home/ssab/config/hosts:/etc/hosts:ro
    ports:
      - "3309:3306" #暴露mysql的端口
    networks:
      mysql:
        ipv4_address: 172.18.0.2
    ulimits:
      nproc: 65535
    hostname: m1
    mem_limit: 1024m
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: m1test
  s1:
      build: ./s1
      container_name: s1
      volumes:
        - /home/ssab/config/mysql-s1/:/etc/mysql/:ro
        - /etc/localtime:/etc/localtime:ro
        - /home/ssab/config/hosts:/etc/hosts:ro
      ports:
        - "3307:3306"
      networks:
        mysql:
          ipv4_address: 172.18.0.3
      links:
        - m1
      ulimits:
        nproc: 65535
      hostname: s1
      mem_limit: 1024m
      restart: always
      environment:
        MYSQL_ROOT_PASSWORD: s1test
  s2:
    build: ./s2
    container_name: s2
    volumes:
      - /home/ssab/config/mysql-s2/:/etc/mysql/:ro
      - /etc/localtime:/etc/localtime:ro
      - /home/ssab/config/hosts:/etc/hosts:ro
    ports:
      - "3308:3306"
    links:
      - m1
    networks:
      mysql:
        ipv4_address: 172.18.0.4
    ulimits:
      nproc: 65535
    hostname: s2
    mem_limit: 1024m
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: s2test
  mycat: # 设置mycat
    build: ./mycat
    container_name: mycat
    volumes:
      - /home/ssab/config/mycat/:/mycat/conf/:ro # mycat配置文件
      - /home/ssab/config/mycat-logs/:/mycat/logs/:rw # mycat日志文件
      - /etc/localtime:/etc/localtime:ro
      - /home/ssab/config/hosts:/etc/hosts:ro
    ports:
      - "8066:8066" # 暴露mycat服务端口
      - "9066:9066" # 暴露mycat管理端口
    links: # mycat可以连接m1 s1 s2
      - m1
      - s1
      - s2
    networks:
      mysql:
        ipv4_address: 172.18.0.5
    ulimits:
      nproc: 65535
    hostname: mycat
    mem_limit: 1024m
    restart: always
networks:
  mysql:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.18.0.0/24
        gateway: 172.18.0.1
```

### mycat 配置

这里只是说一个成功运行的配置,具体详细的配置规则请自己参考mycat权威指南.

#### schema.xml配置

```
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

	<schema name="mall" checkSQLschema="false" sqlMaxLimit="100" dataNode="mallDN">

	</schema>

	<dataNode name="mallDN" dataHost="mallDH" database="mall">

	</dataNode>

	<dataHost name="mallDH" maxCon="1000" minCon="10" balance="1"
			  writeType="0" dbType="mysql" dbDriver="native" switchType="-1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="m1" url="172.18.0.2:3306" user="root" password="m1test">
			<readHost host="s1" url="172.18.0.3:3306" user="root" password="s1test" />
			<readHost host="s2" url="172.18.0.4:3306" user="root" password="s2test" />
		</writeHost>

	</dataHost>

</mycat:schema>
```

#### server.xml配置

```
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License");
	- you may not use this file except in compliance with the License. - You
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0
	- - Unless required by applicable law or agreed to in writing, software -
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the
	License for the specific language governing permissions and - limitations
	under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
	<system>
	<property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
	<property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->

		<property name="sequnceHandlerType">2</property>
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
	<!-- <property name="processorBufferChunk">40960</property> -->
	<!--
	<property name="processors">1</property>
	<property name="processorExecutor">32</property>
	 -->
		<!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena-->
		<property name="processorBufferPoolType">0</property>
		<!--默认是65535 64K 用于sql解析时最大文本长度 -->
		<!--<property name="maxStringLiteralLength">65535</property>-->
		<!--<property name="sequnceHandlerType">0</property>-->
		<!--<property name="backSocketNoDelay">1</property>-->
		<!--<property name="frontSocketNoDelay">1</property>-->
		<!--<property name="processorExecutor">16</property>-->
		<!--
			<property name="serverPort">8066</property> <property name="managerPort">9066</property>
			<property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
			<property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
		<!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
		<property name="handleDistributedTransactions">0</property>

			<!--
			off heap for merge/order/group/limit      1开启   0关闭
		-->
		<property name="useOffHeapForMerge">1</property>

		<!--
			单位为m
		-->
		<property name="memoryPageSize">1m</property>

		<!--
			单位为k
		-->
		<property name="spillsFileBufferSize">1k</property>

		<property name="useStreamOutput">0</property>

		<!--
			单位为m
		-->
		<property name="systemReserveMemorySize">384m</property>


		<!--是否采用zookeeper协调切换  -->
		<property name="useZKSwitch">true</property>


	</system>

	<!-- 全局SQL防火墙设置
	<firewall>
	   <whitehost>
	      <host host="172.18.0.2" user="root"/>
	      <host host="172.18.0.3" user="root"/>
				<host host="172.18.0.4" user="root"/>
	   </whitehost>
       <blacklist check="false">
       </blacklist>
	</firewall>-->

	<user name="root">
		<property name="password">jiabin</property>
		<property name="schemas">mall</property>

		<!-- 表级 DML 权限设置 -->
		<!--
		<privileges check="false">
			<schema name="TESTDB" dml="0110" >
				<table name="tb01" dml="0000"></table>
				<table name="tb02" dml="1111"></table>
			</schema>
		</privileges>
		 -->
	</user>

</mycat:server>
```

#### log4j2.xml配置

这个把日志级别更改为debug,方便我们观察测试.

#### mycat的Dockerfile

```
FROM java:8-jre
MAINTAINER <ssab work_wjj@163.com>
LABEL Description="使用mycat做mysql数据库的读写分离"
ENV mycat-version Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
USER root
COPY ./Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz /
RUN tar -zxf /Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
ENV MYCAT_HOME=/mycat
ENV PATH=$PATH:$MYCAT_HOME/bin
WORKDIR $MYCAT_HOME/bin
RUN chmod u+x ./mycat
EXPOSE 8066 9066
CMD ["./mycat","console"]
```

### 启动

在`docker-compose.yml`文件目录下运行
```shell
  docker-compose up -d
```

如果没有容器对应的镜像文件,则`docker-compose`会自动构建镜像.

使用`docker-compose`手动构建镜像的命令:`docker-compose build mycat`

命令成功执行,则容器mycat,m1,s1,s2都已经启动成功.

我们用`docker ps -a`来看一下.![mycat](https://github.com/sunshineasbefore/resource/blob/master/mycat.png?raw=true)

### 测试

#### 进入mycat客户端

```
mysql -u root -p -P 8066 -h 127.0.0.1
```

#### 执行select语句

因为在上一篇文章中已经做过主从复制的测试,所以这个地方我们就不再重复了,我们直接执行`select`语句,看是否已经实现了读写分离.

```
mysql> select * from salesman limit 0,10;
```

结果集:
![222](https://github.com/sunshineasbefore/resource/blob/master/mycat-salesman.png?raw=true)

然后我们打开mycat的日志mycat.log看一下
![log](https://github.com/sunshineasbefore/resource/blob/master/mycatlog.png?raw=true)

注意看图中标记出来的地方.

好吧,从日志中我们看出我们执行的`select`语句是走从库s1执行的.

#### 执行insert语句

```
mysql> insert into salesman (id,user_num,true_name,address,mobile,disabled) values('30769','33333','ssab','山东省','33333321',0);
```

打开mycat的日志mycat.log看一下

![insert](https://github.com/sunshineasbefore/resource/blob/master/mycatinsert.png?raw=true)

这次我们发现,执行`insert`语句走的是主库m1.

总结
======

简单来讲,一个使用`mycat中间件`搭建mysql 1主2从 主从复制 读写分离的实例就完成了.

要说为什么使用`mycat数据库中间件`,很简单啊,就是因为它对开发人员基本没有影响,不会侵入到代码中.