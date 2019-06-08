---
title: Kafka安全认证
date: 2019-05-20 10:35:45
tags:
        - Kafka
        - Java
---

# Kafka安全认证整理

## Jaas文件说明
```
// sasl  
KafkaClient {  // 客户端需要的jaas   
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="yourname"  // 客户端登录的用户名和密码，该用户名和密码要在服务配置过
  password="yourpw";
};

KafkaServer { // 服务端需要的jaas
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"  // 服务端也需要一个账户进行登录
  password="admin"
  user_admin="admin"  // 配置admin账户
  user_myuser="mypw"; // 格式 user_<username>="<password>"
};
```

```
// kerberos
KafkaClient {   // 还有配置Client连接zk
  com.sun.security.auth.module.Krb5LoginModule required       
  useKeyTab=true
  keyTab="src/main/resources/user.keytab"        // 设置keytab所在的相对路径 
  principal="CORPLK50OPENAPI@HADOOP.COM"
  useTicketCache=false                           // 若为true时，可先用kinit命令获取凭证
  storeKey=true
  debug=true;
};
```
**最后一行和倒数第二行要以分号结尾**

----

## 运行配置
1. 以程序方式运行  

    程序运行时要将krb5和jaas文件的路径设置在系统的环境变量里

    ```
    System.setProperty("java.security.auth.login.config", "src/main/resources/jaas.conf");
    System.setProperty("java.security.krb5.conf", "src/main/resources/krb5.conf");
    ```
    或者
    ```
    props.put("sasl.jaas.config", PlainLoginModule.class.getName() + " required username=\"" + username + "\" password=\"" + password + "\";");
    ```

2. 以脚本方式运行

    ```
    -Djava.security.krb5.conf=./krb5.conf -Djava.security.auth.login.config=./jaas.conf
    ```

*sasl认证时不用配krb5*  
*kinit -kt ./user.keytab MYPRINCIPAL@HADOOP.COM*

----

## SASL + ACL
properties文件中配置
```
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

使用kafka-acls.sh脚本对用户权限进行授权


## Kerberos
properties文件中配置
```
security.protocol=SASL_PLAINTEXT
sasl.mechanism=GSSAPI
sasl.kerberos.service.name=kafka
```