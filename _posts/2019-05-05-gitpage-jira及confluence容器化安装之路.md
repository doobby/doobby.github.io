---
title: jira及confluence容器化安装之路
tags: 项目管理  docker
author: O0lele0O
date: 2019-05-05
catalog: true
---



# jira及confluence容器化安装之路

## 为何选择你
看到别人在用，官网了解一下，这个东东不错o，宣传视频给力，觉得可以试试。就是破解不出钱有点不合适 

## 选择docker容器化安装
之前一直认为docker搬迁方便快捷，搭建快速方便，所以趟趟坑，哪知水有点深啊，不过这次文档记录下来，以后就方便多了

## 遗留问题
docker可以快速化搭建，但是已有数据系统的迁移未探索过，所以jira的备份、confluence备份都应该在后续做出来 

## 正题

## 前提 

**系统已经安装docker环境**  

备注：
   rhel7.2系统安装最新docker CE环境的时候，可能系统对应依赖要升级libseccomp-2.3.1-3.el7.x86_64.rpm  否则启动docker程序可能会报oci类型错误 


### 安装准备

#### 镜像相关

- jira破解的安装包   

```dockerfile
FROM blacklabelops/jira:latest

USER jira

将代理破解包加入容器

COPY "atlassian-agent.jar" /opt/jira/

设置启动加载代理包

RUN echo 'export CATALINA_OPTS="-javaagent:/opt/jira/atlassian-agent.jar ${CATALINA_OPTS}"' >> /opt/jira/bin/setenv.sh
```

编译命令：

`docker build -t lele/jira:v7.13.0_crack  .`  

- confluence破解的安装包 dockerfile： 

```dockerfile
FROM blacklabelops/confluence:latest

USER confluence

将代理破解包加入容器

COPY "atlassian-agent.jar" /opt/atlassian/confluence/

设置启动加载代理包

RUN echo 'export CATALINA_OPTS="-javaagent:/opt/atlassian/confluence/atlassian-agent.jar ${CATALINA_OPTS}"' >> /opt/atlassian/confluence/bin/setenv.sh

```


编译命令：

`docker build -t lele/confluence:v6.13.1_crack  .`

[github破解来源](https://github.com/pengzhile/atlassian-agent)

- postgres安装包  暂时使用的是已修改的官方镜像   

[postgres镜像](https://hub.docker.com/r/blacklabelops/postgres)

- docker-compose 文件内容:

```dockerfile
version: '3'

services:
  jira:
    depends_on:
      - postgresql
    image: lele/jira:v7.13.0_crack
    container_name: jira
    hostname: jira
    networks:
        jiranet:
           ipv4_address: 172.23.0.3
    restart: always
    volumes:
        - jiradata:/var/atlassian/jira
    ports:
        - '8080:8080'
    environment:
        - 'JIRA_DATABASE_URL=postgresql://jira@postgresql/jiradb'
        - 'JIRA_DB_PASSWORD=jellydzwl'
        - 'SETENV_JVM_MINIMUM_MEMORY=1g'
        - 'SETENV_JVM_MAXIMUM_MEMORY=4g'
        - 'JIRA_PROXY_NAME='
        - 'JIRA_PROXY_PORT='
        - 'JIRA_PROXY_SCHEME='
    logging:
      # limit logs retained on host to 25MB
      driver: "json-file"
      options:
        max-size: "500k"
        max-file: "50"
    labels:
      com.blacklabelops.description: "Atlassian Jira"
      com.blacklabelops.service: "jira"

  confluence:
    depends_on:
      - postgresql
    image: lele/confluence:v6.13.1_crack
    container_name: confluence
    hostname: confluence
    restart: always
	# because database locked by ip ,we should use static ip 
    networks:
        jiranet:
            ipv4_address: 172.23.0.2      
    volumes:
      - confluencedata:/var/atlassian/confluence
    ports:
      - '8090:8090'
      - '8091:8091'
    environment:
      - 'CATALINA_OPTS= -Xms1g -Xmx4g'
      - 'CONFLUENCE_PROXY_NAME='
      - 'CONFLUENCE_PROXY_PORT='
      - 'CONFLUENCE_PROXY_SCHEME='
      - 'CONFLUENCE_DELAYED_START='
    labels:
      com.blacklabelops.description: "Atlassian Confluence"
      com.blacklabelops.service: "confluence"

  postgresql:
    image: blacklabelops/postgres
    networks:
       jiranet:
          ipv4_address: 172.23.0.5  
    volumes:
      - postgresqldata:/var/lib/postgresql/data
    ports:
      - 5432:5432
    restart: always
    environment:
      - 'POSTGRES_USER=jira'
      # CHANGE THE PASSWORD!
      - 'POSTGRES_PASSWORD=jellydzwl'
      - 'POSTGRES_DB=jiradb'
      - 'POSTGRES_ENCODING=UNICODE'
      - 'POSTGRES_COLLATE=C'
      - 'POSTGRES_COLLATE_TYPE=C'
    logging:
      # limit logs retained on host to 25MB
      driver: "json-file"
      options:
        max-size: "500k"
        max-file: "50"
    labels:
      com.blacklabelops.description: "PostgreSQL Database Server"
      com.blacklabelops.service: "postgresql"

volumes:
  confluencedata:
     external: false
  jiradata:
     external: false
  postgresqldata:
     external: false

networks:
  jiranet:
     driver: bridge
     ##下面参数未测试
     ipam:
          config:
          - subnet: 172.23.0.0/16
              gateway: 172.23.0.1
```



### 安装步骤

#### 镜像导出及导入

由于可能是非互联网环境，需要将镜像导入对应服务器。


- 导出
`docker save  镜像名称：版本号 > xxxx.tar`
- 导入
`docker load < xxxx.tar`


#### docker-compose 启动

在compose文件对应目录启动  
`docker-compose up -d`

#### 添加数据库用户及库表

**注意**：上述compose文件配置中数据库只建立了jira用户相关的权限及库表，还需要手动添加confluence的用户权限及库，当然如果你会更改dockerfile打包自己的镜像直接建立也挺好

’‘’
填写 postgres 添加用户和库方法


‘’‘

#### jira页面安装

插图 

注意：
​	-  jira镜像默认已经配置了postgres数据库
​	-  破解的镜像已将破解包拷贝到镜像内部
​	-  容器间通讯配置使用服务名，不建议使用具体ip

破解：  
​	1. 页面拷贝系统生成的序列号
​        2. 执行提供的jar包破解生成序列号
​        3. 页面填入即可

```bash
java -jar atlassian-agent.jar -d -m lele@bm182.shs.com -n jira -p jira -o http://10.4.34.182 -s BF50-VOI4-F1KO-HZQY
```




#### confluence页面安装

插图
注意：
​	- confluence镜像需要配置了postgres数据库（上面已经建了用户和库）
​	- 破解的镜像已将破解包拷贝到镜像内部
​	- 容器间通讯配置使用服务名，不建议使用具体ip

破解：  
​	1. 页面拷贝系统生成的序列号
​        2. 执行提供的jar包破解生成序列号
​        3. 页面填入即可

```bash
java -jar atlassian-agent.jar -d -m lele@bm182.shs.com -n jira -p jira -o http://10.4.34.182 -s BF50-VOI4-F1KO-HZQY
```

与jira公用账户：
​	confluence初次安装会提示是否与jira共用账户，配置服务名不能配置ip
​	- jira的baseurl http://jira:8080
​	- confluence的baseurl  http://confluence:8090

后续jira和confluence建立应用程序链接 基本url系统会自动提示设置为容器宿主机IP，这样并不影响跳转失败的问题 

#### jira与confluence互相建立应用程序链接 保证相互认证通讯


插入图片说明 （应用程序url填写服务名加端口，显示url填宿主机ip加端口）

建议使用oauth认证


#### jira与confluece 配置邮件服务器

两者配置方式一样，用jira举例说明

##### 网络打通 
mail邮件服务其依然使用的是docker安装，快捷的方式是需要将mail容器链接到jiranet网络，否则配置宿主机ip加端口无法测试通过（原理上jira是一个虚拟机，要通主机需要通过网管IP 配置网关IP加端口依然不通，可能和其docker防火墙也有关系）

**注意**：
请固定mail的IP保证不冲突，否则可能会引发jira等容器ip占用问题 

```bash
docker network connect  docker_jira_confluence_jiranet mail --ip 172.23.0.4
```



##### TLS证书认证问题
自己搭建的mail是自签名的证书 所以存在认证问题

需要将mail容器使用的TLS证书导入到jira和confluence容器里面，这个其实是由容器里面JDK添加配置的

1.导出mail认证证书 使用openssl程序查出对应证书

```bash
# openssl s_client -connect localhost:465

CONNECTED(00000003)
depth=1 C = CN, ST = shannxi, O = shs, OU = dzwl, CN = shs.org, emailAddress = dzwl@bm182.shs.org
verify error:num=19:self signed certificate in certificate chain
verify return:0
---
Certificate chain
 0 s:/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=bm182.shs.org/emailAddress=dzwl@bm182.shs.org
   i:/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=shs.org/emailAddress=dzwl@bm182.shs.org
 1 s:/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=shs.org/emailAddress=dzwl@bm182.shs.org
   i:/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=shs.org/emailAddress=dzwl@bm182.shs.org
---
Server certificate
-----BEGIN CERTIFICATE-----
MIID5jCCAs6gAwIBAgIJANWid/y+7v2KMA0GCSqGSIb3DQEBCwUAMHExCzAJBgNV
BAYTAkNOMRAwDgYDVQQIDAdzaGFubnhpMQwwCgYDVQQKDANzaHMxDTALBgNVBAsM
BGR6d2wxEDAOBgNVBAMMB3Nocy5vcmcxITAfBgkqhkiG9w0BCQEWEmR6d2xAYm0x
ODIuc2hzLm9yZzAeFw0xOTAxMTUwMjQzMjJaFw0yMDAxMTUwMjQzMjJaMHcxCzAJ
BgNVBAYTAkNOMRAwDgYDVQQIDAdzaGFubnhpMQwwCgYDVQQKDANzaHMxDTALBgNV
BAsMBGR6d2wxFjAUBgNVBAMMDWJtMTgyLnNocy5vcmcxITAfBgkqhkiG9w0BCQEW
EmR6d2xAYm0xODIuc2hzLm9yZzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoC
ggEBALGxB9oaaPeCyTtDWBtBYIs5YDmWfB/3RmCXC7qrfnvRH5o4cMsArX/zn7lq
U7ytbS3eJL/MMVzfk2ZLHjfRPBTwvZ/E343qRcviH6IlUC/rM/wkYQa4iSL4RrDA
/EKNcfExPjaGNsGbAtO0Uq74f/vpdhLkn76uLD6biB+SxChiq5gM33AMUSb4sexe
M9WcdzenAk+ydB9ZqpiIRc3hf47aqrmFuDDUnwcpQ81mJjoIkOyZ2wm6nHbXXlXV
R3bxOo1/2p1KGUfwSkurvwVvB21ERUzH1VZCYSs2ToKs/dXi8JlssMsMYNtgHse/
PaPf3wQpsXUTPCMZxDaxkaDKPgcCAwEAAaN7MHkwCQYDVR0TBAIwADAsBglghkgB
hvhCAQ0EHxYdT3BlblNTTCBHZW5lcmF0ZWQgQ2VydGlmaWNhdGUwHQYDVR0OBBYE
FOhKUnRIzN1HiHLX/eJuejSS/DvBMB8GA1UdIwQYMBaAFKC/4P7gPB7aP8wZ05HT
JMIf4BiPMA0GCSqGSIb3DQEBCwUAA4IBAQDCBc9SYozJvqmEhM/vFgZNqMXOO8Rx
fmP+9Pbu3nUJY2L+JguAl8yR8xRDPox6Z9p1GhzReNJoieISLU96SrLbfWzUj3J2
OIVl5dw+bQpeAH6c7vBVga5hRQZirsD3XQMqrm9ek1iwPcOyU4ZgOI+LE7spukal
F2qYzVJGkgTzOymikfzEb8uBjM8AF84Z1saQcdvKLBEVE01PV5RrwWtmUR68DH5/
tZikTc4AIyYcgxUZEx4Ir0lkmqsDqnAU9B74ANGsIrQX+8ljLKaioeQblnsuEJoJ
U6KhXKsBo9wfDzXO2nRzLhSO1dg09pquZFuRk3s1srDyeJTlpdMOxi/F
-----END CERTIFICATE-----
subject=/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=bm182.shs.org/emailAddress=dzwl@bm182.shs.org
issuer=/C=CN/ST=shannxi/O=shs/OU=dzwl/CN=shs.org/emailAddress=dzwl@bm182.shs.org
---
No client certificate CA names sent
Server Temp Key: ECDH, prime256v1, 256 bits
---
SSL handshake has read 2615 bytes and written 375 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: C1494A253BA694907665BE9321E7866C9903598B090028B442657A1586B8F2D1
    Session-ID-ctx: 
    Master-Key: A8B53EC8B59ACD2060AC8B163A860ED167A2C85CEB7ED1A45E6F6241DE1747BB8BDBF51D48C274BFAC2AABEF263EAA22
    Key-Arg   : None
    Krb5 Principal: None
    PSK identity: None
    PSK identity hint: None
    TLS session ticket lifetime hint: 7200 (seconds)
    TLS session ticket:
    0000 - 2e 40 9d 2e b4 3a 84 ee-fe 20 51 a5 47 13 7a f0   .@...:... Q.G.z.
    0010 - 9b 08 b5 56 be 9b 38 ec-6c 57 ab bc db bb 68 59   ...V..8.lW....hY
    0020 - b5 ab 79 69 06 53 02 4a-92 f3 06 49 d6 76 c3 07   ..yi.S.J...I.v..
    0030 - 19 bb 34 da 0e ff 18 24-6d a5 f7 a5 0d 87 eb 08   ..4....$m.......
    0040 - ed 95 9d 12 bf 35 02 73-2d b2 e6 fd 80 35 31 72   .....5.s-....51r
    0050 - bf 79 55 d0 f9 49 c0 a0-e1 38 ea da 34 72 d2 f6   .yU..I...8..4r..
    0060 - cb f2 c3 d7 48 c2 65 14-bb f8 08 8a d6 f9 c3 3c   ....H.e........<
    0070 - 57 33 89 ca 0d ed e1 78-93 fb a7 e3 58 8e d5 6e   W3.....x....X..n
    0080 - f5 53 6d 79 45 f0 83 79-89 ac dc 74 ef de 8f 33   .SmyE..y...t...3
    0090 - 89 e6 6f 3c 15 04 1a 7f-c2 41 de 1e 1f 9f 11 cc   ..o<.....A......

    Start Time: 1550818773
    Timeout   : 300 (sec)
    Verify return code: 19 (self signed certificate in certificate chain)
---
220 bm182.shs.org ESMTP Postfix (Debian)


```

拷贝 begin到end行保存为  public.cert文件

2. 在容器中导入该证书

  ```bash
   # 找到jira及confluence使用的jdk所在目录，找到keytool工具导入
  keytool -import -alias dzwl -keystore ../lib/security/cacerts -file /tmp/public.crt
  ```
3. 重启容器



##### 页面设置

插图

系统--外发邮件--配置smtp电邮服务器

使用tls配置 使用587端口  使用mail服务名

测试即可



## 待实践
jira用户及数据的迁移，查询资料合成 
















​    








