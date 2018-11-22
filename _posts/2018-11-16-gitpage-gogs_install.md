---
title: docker安装gogs\备份\字符集折腾心路历程
tags: PM git docker 
author: O0lele0O
catalog: true
---



# 开始

网上随便挡了个docker-compos.yml，开启了docker乱整模式。在centos7.2无网模式下各种乱下包服务器安装了docker环境，拷贝了mysql镜像和gogs镜像，搞起 docker-compose up -d。

```yaml
version: '3.3'
services:
  gogsdb:
    image: mysql:latest
    container_name: gogsdb
#1#    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --init-connect='SET NAMES utf8mb4;' --innodb-flush-log-at-trx-commit=0
    restart: always
    environment:
      MYSQL_DATABASE: gogs
      MYSQL_ROOT_PASSWORD: gogs
      MYSQL_USER: gogs
      MYSQL_PASSWORD: gogs
    volumes:
      - db_data:/var/lib/mysql_gogs
#2#      - /Users/lele/docker_project/mysql/conf.d:/etc/mysql/conf.d
    ports:
      - "13306:3306"
    networks:
      - gogs-network

  gogsapp:
    depends_on:
      - gogsdb
    image: gogs/gogs
    container_name: gogsapp
    restart: always
    ports:
      - "322:22"
#3#   - "3038：3038"  
      - "3000:3000"
    volumes:
      - app_data:/data
      - /Users/lele/Downloads:/tmp
    networks:
      - gogs-network
volumes:
  db_data:
  app_data:
networks:
  gogs-network:
    driver: bridge
```



- 开始没有考虑mysql配置和volume的模式  没有用那两个***#***注释配置项，从此为使用bug埋下了伏笔。

***题外***：服务器上面稳定的跑了几个月，暗暗夸赞稳定性不错

---

# 问题

## Q1 安装页面要求输入mysql配置地址 



![gogs_install](/img/gogs_install.jpg)

  <!-- 因为是docker环境，按照配置gogsapp是一个容器，gogsdb是一个容器，这里有两个选择 1 要不填主机IP+port 2 要不填gogsdb容器IP+port -->  

## A1：使用docker(自带DNS功能? ❌)  

1. gogsapp怎么访问主机IP？可以查看主机docker内部网络IP输入，但是启动关闭容器都可能导致ip变化，不靠谱放弃。
2. 学习docker网络，发现docker自带DNS功能命名 从gogsapp容器ping gogsdb完美通畅(目前没搞明白，桥接网络可能在配置了hosts，没确认)，那就填gogsdb:3306。

![gogs_pingdb](/img/gogs_pingdb.png)

​        

- 带来疑惑，如果需要配置主机ip，比如邮箱服务，有没有主机名可用？这个似乎就是docker网络管理的东东了，k8s派上用场？打通各种容器网络全部集合之？㊙️

## Q2： 头像显示失败

## A2：这个得瞄代码 目前更新下头像可以解决 



## Q3：建仓库添加汉字描述 500错误

![gogs_500bug](/img/gogs_500bug.png)

## A3: 程序员都会遇到的字符集问题😹

说说折腾历程，一眼看错误大概就是数据库字符集不兼容，嗯~~~

果断去看mysql的字符集设置,度娘了个基本概念，觉得应该记录下来拷贝如下：

----



> **基本概念**

- 字符(Character)是指人类语言中最小的表义符号。例如’A'、’B'等；
- 给定一系列字符，对每个字符赋予一个数值，用数值来代表对应的字符，这一数值就是字符的编码(Encoding)。例如，我们给字符’A'赋予数值0，给字符’B'赋予数值1，则0就是字符’A'的编码；
- 给定一系列字符并赋予对应的编码后，所有这些字符和编码对组成的集合就是字符集(Character Set)。例如，给定字符列表为{’A',’B'}时，{’A'=>0, ‘B’=>1}就是一个字符集；
- 字符序(Collation)是指在同一字符集内字符之间的比较规则；
- 确定字符序后，才能在一个字符集上定义什么是等价的字符，以及字符之间的大小关系；
- 每个字符序唯一对应一种字符集，但一个字符集可以对应多种字符序，其中有一个是默认字符序(Default Collation)；
- MySQL中的字符序名称遵从命名惯例：以字符序对应的字符集名称开头；以_ci(表示大小写不敏感)、_cs(表示大小写敏感)或_bin(表示按编码值比较)结尾。例如：在字符序“utf8_general_ci”下，字符“a”和“A”是等价的；

> **MySQL字符集设置**

**系统变量**：

-    character_set_server：默认的内部操作字符集
-  character_set_client：客户端来源数据使用的字符集
-  character_set_connection：连接层字符集
-  character_set_results：查询结果字符集
-  character_set_database：当前选中数据库的默认字符集
-  character_set_system：系统元数据(字段名等)字符集
- 还有以collation_开头的同上面对应的变量，用来描述字符序。

​    用introducer指定文本字符串的字符集：

- 格式为：[_charset] ’string’ [COLLATE collation]
- 例如：

​       SELECT _latin1 ’string’;
       SELECT _utf8 ‘你好’ COLLATE utf8_general_ci;

- 由introducer修饰的文本字符串在请求过程中不经过多余的转码，直接转换为内部字符集处理。

> **MySQL中的字符集转换过程**

1. MySQL Server收到请求时将请求数据从character_set_client转换为character_set_connection；
2. 进行内部操作前将请求数据从character_set_connection转换为内部操作字符集，其确定方法如下：
   - 使用每个数据字段的CHARACTER SET设定值；
   - 若上述值不存在，则使用对应数据表的DEFAULT CHARACTER SET设定值(MySQL扩展，非SQL标准)；
   - 若上述值不存在，则使用对应数据库的DEFAULT CHARACTER SET设定值；
   - 若上述值不存在，则使用character_set_server设定值。
3. 将操作结果从内部操作字符集转换为character_set_results。

 我们现在回过头来分析下我们产生的乱码问题：

- [ ]  我们的字段没有设置字符集，因此使用表的数据集
- [ ]  我们的表没有指定字符集，默认使用数据库存的字符集
- [ ]  我们的数据库在创建的时候没有指定字符集，因此使用character_set_server设定值
- [ ]  我们没有特意去修改character_set_server的指定字符集，因此使用mysql默认
- [ ]  mysql默认的字符集是latin1，因此，我们使用了latin1字符集，而我们character_set_connection的字符集是UTF-8，插入中文乱码也再所难免了。

**常见问题解析**

1. FAQ-1 向默认字符集为utf8的数据表插入utf8编码的数据前没有设置连接字符集，查询时设置连接字符集为utf8
   - 插入时根据MySQL服务器的默认设置，character_set_client、character_set_connection和character_set_results均为latin1；
   - 插入操作的数据将经过latin1=>latin1=>utf8的字符集转换过程，这一过程中每个插入的汉字都会从原始的3个字节变成6个字节保存；
   - 查询时的结果将经过utf8=>utf8的字符集转换过程，将保存的6个字节原封不动返回，产生乱码。

2. 向默认字符集为latin1的数据表插入utf8编码的数据前设置了连接字符集为utf8（**遇到的错误大概就是属于这一种**）
   - 插入时根据连接字符集设置，character_set_client、character_set_connection和character_set_results均为utf8；
   - 插入数据将经过utf8=>utf8=>latin1的字符集转换，若原始数据中含有\u0000~\u00ff范围以外的Unicode字符，会因为无法在latin1字符集中表示而被转换为“?”(0×3F)符号，以后查询时不管连接字符集设置如何都无法恢复其内容了。

**检测字符集问题的一些手段**

- SHOW CHARACTER SET;
-  SHOW COLLATION;
-  SHOW VARIABLES LIKE ‘character%’;
-  SHOW VARIABLES LIKE ‘collation%’;
- SQL函数HEX、LENGTH、CHAR_LENGTH
- SQL函数CHARSET、COLLATION

**使用MySQL字符集时的建议**

- 建立数据库/表和进行数据库操作时尽量显式指出使用的字符集，而不是依赖于MySQL的默认设置，否则MySQL升级时可能带来很大困扰；

- 数据库和连接字符集都使用latin1时，虽然大部分情况下都可以解决乱码问题，但缺点是无法以字符为单位来进行SQL操作，一般情况下将数据库和连接字符集都置为utf8是较好的选择；

- 使用mysql CAPI（mysql提供C语言操作的API）时，初始化数据库句柄后马上用mysql_options设定MYSQL_SET_CHARSET_NAME属性为utf8，这样就不用显式地用SET NAMES语句指定连接字符集，且用mysql_ping重连断开的长连接时也会把连接字符集重置为utf8；

-  对于mysql PHP API，一般页面级的PHP程序总运行时间较短，在连接到数据库以后显式用SET NAMES语句设置一次连接字符集即可；但当使用长连接时，请注意保持连接通畅并在断开重连后用SET NAMES语句显式重置连接字符集。

  **其他注意事项**

  - my.cnf中的default_character_set设置只影响mysql命令连接服务器时的连接字符集，不会对使用libmysqlclient库的应用程序产生任何作用！
  - 对字段进行的SQL函数操作通常都是以内部操作字符集进行的，不受连接字符集设置的影响。
  -  SQL语句中的裸字符串会受到连接字符集或introducer设置的影响，对于比较之类的操作可能产生完全不同的结果，需要小心！

  **总结**

  - 根据上面的分析和建议，我们解决我们遇到问题应该使用什么方法大家心里应该比较清楚了。对，就是在创建database的时候指定字符集，不要去通过修改默认配置来达到目的，当然你也可以采用指定表的字符集的形式，但很容易出现遗漏，特别是在很多人都参与设计的时候，更容易纰漏。
  - 虽然不提倡通过修改mysql的默认字符集来解决，但对于如何去修改默认字符集，我这里还是给出一些方法，仅供大家参考。

  **MySQL默认字符集**

  MySQL对于字符集的指定可以细化到一个数据库，一张表，一列.传统的程序在创建数据库和数据表时并没有使用那么复杂的配置，它们用的是默认的配置.

  1. 编译MySQL 时，指定了一个默认的字符集，这个字符集是 latin1；
  2. 安装MySQL 时，可以在配置文件 (my.ini) 中指定一个默认的的字符集，如果没指定，这个值继承自编译时指定的；
  3. 启动mysqld 时，可以在命令行参数中指定一个默认的的字符集，如果没指定，这个值继承自配置文件中的配置,此时 character_set_server 被设定为这个默认的字符集；
  4. 安装 MySQL选择多语言支持，安装程序会自动在配置文件中把default_character_set 设置为 UTF-8，保证缺省情况下所有的数据库所有表的所有列的都用 UTF-8 存储。

  **查看默认字符集**

    (默认情况下，mysql的字符集是latin1(ISO_8859_1)，如何查看在上面我们已经给出了相关命令

- **修改默认字符集**

  1. 最简单的修改方法，就是修改mysql的my.ini文件中的字符集键值，如  default-character-set = utf8、character_set_server =  utf8 ，修改完后，重启mysql的服务

  2. 还有一种修改字符集的方法，就是使用mysql的命令

     ```mysql
      mysql> SET character_set_client = utf8 ;
      mysql> SET character_set_connection = utf8 ;
      mysql> SET character_set_database = utf8 ;
      mysql> SET character_set_results = utf8 ;
      mysql> SET character_set_server = utf8 ;
      mysql> SET collation_connection = utf8 ;
      mysql> SET collation_database = utf8 ;
      mysql> SET collation_server = utf8 ;
     
     ```

     

  设置了表的默认字符集为utf8并且通过UTF-8编码发送查询，存入数据库的仍然是乱码。那connection连接层上可能出了问题。解决方法是在发送查询前执行一下下面这句： SET NAMES 'utf8';它相当于下面的三句指令：

  ```mysql
  - SET character_set_client = utf8;
  - SET character_set_results = utf8;
  - SET character_set_connection = utf8;
  ```

  

## A3 go on： **回到正题**

去数据库看了一眼，果然字符集是 latin1；想想办法，镜像里面直接改配置？mysql终端直接set？嗯~~~试过了，都不行，原因很简单，前面的坑，mysql的配置没做volume配置，恶心了 ，只要重新启动容器就都不行了。



好吧，尝试修改compose文件 添加mysql配置volume 重新生成容器，想着应该没啥，

```shell
# vim docker-compose.yml  //添加#2#配置 

# docker stop gogsdb；
# docker rm gogsdb;  //不删除db_data的volume 保持原有数据不丢失
# docker-compose up -d 
```

```mysql
#附上配置修改项 /Users/lele/docker_project/mysql/conf.d/docker.cnf
[client]
default-character-set=utf8mb4


[mysqld]
skip-host-cache
skip-name-resolve


character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
init-connect='SET NAMES utf8mb4;'
```

测试，还是报错，嗯······ 换种修改思路 

```shell
# vim docker-compose.yml  //添加#1#配置 
# docker-compose up -d
Recreating gogsdb ... done
Recreating gogsapp ... done
```

测试，还是报错，嗯.....看来想保存历史数据行不通了.....继续折腾

```shell
//查询该命令 删除容器、删除volume 这下清净了
#docker-compose down
#docker-compose up -d
```

测试，一下回到解放前注册用户全没了。。。但是字符集问题搞定了，反思怎么样能保存原来的数据呢？线上使用的gogs可不能这么整哈。

嗯，备份数据库**mysqldump**备份

```mysql
#docker exec -it gogsdb /bin/bash
#mysqldump gogs -uroot -p > gogs.sql
//偷瞄一下
#cat gogs.sql
-- MySQL dump 10.13  Distrib 5.7.21, for Linux (x86_64)
--
-- Host: localhost    Database: gogs
-- ------------------------------------------------------
-- Server version	5.7.21

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `access`
--

DROP TABLE IF EXISTS `access`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `access` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL,
  `repo_id` bigint(20) DEFAULT NULL,
  `mode` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UQE_access_s` (`user_id`,`repo_id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 ROW_FORMAT=DYNAMIC;
/*!40101 SET character_set_client = @saved_cs_client */;
```

**嗯，找到问题根，gogs初始化库表已经设置好了每个表的CHARSET，所以不重新建是搞不定的了**

好吧，那看来迁移历史数据只有一个办法了，**只备份mysql数据不备份表结构**，重新彻底生成容器恢复 

```shell
#docker exec -it gogsdb /bin/bash
#mysqldump -t gogs -uroot -p > gogs_data.sql
#exit
#docker cp gogsdb:/gogs_data.sql .
#docker-compose down
#docker-compose up -d
#docker cp gogs_data.sql gogsdb:/gogs_data.sql 
#docker exec -it gogsdb /bin/bash
#mysql -uroot -p
mysql> use gogs;
mysql> source gogs_data.sql
```

这个只在本机测试通过，理论可行。下面自己决定拿个在线库玩一票



## Q4: 离线环境如何迁移备份？

## A4：有了上面的经验就简单多了

1. 在原服务器 docker dump 最新的镜像   mysql  gogs
2. 在原服务器备份 docker-compose.yml (开放#1 #2 )、mysql dump gogs数据库文件 、备份gogs-repositories数据目录和配置文件 app.ini
3. 在新服务器 docker load mysql gogs镜像
4. docker-compose up -d
5. 去mysql容器恢复gogs数据库，去gogsapp容器恢复仓库数据文件和配置app.ini
6. docker-compose restart 一切如初。



## Q5: 如何开启个人密钥免密码远端推送？

## A5: ssh协议可加密钥，http方式只能在客户端实现

先说下折腾的情况，按照上面docker-compose.yml默认建的gogs配置并没有开启sshserver端的服务，但是可以在页面上传个人公钥，而使用ssh协议克隆版本库是失败的（之前忽视一直用的是http）,用苹果笔记本，默默的拉取推送，在第一次送后需要输入用户和密码，后面就不需要输入了，还以为密钥配置成功，哎，学习理解不到位的bug，然后在linux/win系统就是不行。仔细查阅资料才大概了解清楚。

[git凭证工具](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage)

看明白了上面的官方说明git tools下的凭证说明，如果还是想使用ssh就修改下面的配置：

#### gogs开启 ssh服务

```shell
[server]
DOMAIN           = localhost
HTTP_PORT        = 3000
ROOT_URL         = http://localhost:3000/
DISABLE_SSH      = false
SSH_PORT         = 3038    ## docker安装下22端口会冲突  建议修改为其他端口同时不变映射到主机端口docker-compose.yml  开启#3#
START_SSH_SERVER = true   ## 这个配置为true
OFFLINE_MODE     = false
```

然后重启gogs服务，上传自己的公钥，使用ssh方式克隆拉取推送就可以免密玩耍了。



# 总结

docker的优势在于封装环境，抛弃服务器环境依赖，但是可变数据集volume怎么配置，网络配置也带来了一定的复杂度。字符集是大家都绕不过去的坑❌。

维护不易 且行且记录。

