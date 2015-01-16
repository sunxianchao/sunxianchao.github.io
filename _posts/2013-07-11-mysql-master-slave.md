---
layout: post
title: 记录一下mysql 主从复制的配置过程  
category : 技术分享
tagline: "Supporting tagline"
tags : [mysql]
---


首先在机器上安装mysql，两台机器ip如下：
    
　　master库ip地址是：192.168.78.134

　　slave库ip抵制是：192.168.78.1
　　
　　
   
### 1. 在官网下载mysql并进行安装，步骤如下：     
    
    {% highlight perl %}
    wget http://dev.mysql.com/get/Downloads/MySQL-5.6/MySQL-5.6.16-1.el6.x86_64.rpm-bundle.tar
    
    tar -xvf MySQL-5.6.16-1.el6.x86_64.rpm-bundle.tar
    
    yum install MySQL-*
    {% endhighlight %}   


   安装完毕  
   
    
    
###  2. 从库安装同理就不说了，如果你用的是mac的话下载一个dmg文件进行安装就好了，启动命令：  

    
    {% highlight perl %}
    sudo /Library/StartupItems/MySQLCOM/MySQLCOM start|stop|restart
    {% endhighlight %}  
    
    
    
### 3. 在master上编辑vi /etc/my.cnf 内容如下：  

    {% highlight perl %}
    [mysqld]
    user=mysql
    log-bin=/var/lib/mysql/mysql-bin
    max_binlog_size=4096
    binlog_format=row
    socket=/var/lib/mysql/mysql.sock
    server-id=1
    binlog_do_db=test
    binlog-ignore-db=mysql
    
    [client]
    socket=/var/lib/mysql/mysql.sock

    [mysqld_safe]
    err-log=/var/log/mysqld-err.log
    
    {% endhighlight %}
    
    
    ** 启动mysql：/etc/init.d/mysqld start **
 
    
### 4. 在master上创建用户从库同步数据的帐号：  

    {% highlight sql %}  
    grant replication slave on *.* to 'slave_user'@'192.168.78.1' identified by '123123';
    {% endhighlight %}  
    
    
    检查master状态
     
    
    {% highlight sql %}  
    mysql> show master status\G;
             File: mysql-bin.000001
         Position: 120
     Binlog_Do_DB: test
 Binlog_Ignore_DB: mysql
1 row in set (0.00 sec)
    {% endhighlight %}  
    
    
 
### 5. 切换到从库并编辑my.cny设置serverid  
    
    {% highlight perl %}  
    [mysqld]  
      user=mysql  
      log_bin=mysql-bin  
      server-id=1  
      binlog-do-db=test  
      binlog-ignore-db=mysql  
      socket=/var/run/mysql/mysql.sock  
      datadir=/usr/local/mysql/data  
      long_query_time=2  
      character-set-server=utf8  
     
     [mysqld_safe]  
     log-error=/var/log/mysqld.log  
     pid-file=/var/run/mysqld/mysqld.pid  
     long_query_time=2
     
    {% endhighlight %}  
    
    
    启动从库mysql服务，进入mysql检查slave_user是否能够连接master库
    
    
    {% highlight sql%}   
    mysql -uslave_user -h192.168.78.134 -p123123 
    {% endhighlight %}  
    
    
   能够进入就说明帐号权限配置没问题



### 6. 使用root帐号登录进去设置master同步信息执行代码如下：  

    {% highlight sql%}   
    CHANGE MASTER TO MASTER_HOST='192.168.78.134',
       MASTER_USER='slave_user',
       MASTER_PASSWORD='123123',
       MASTER_PORT=3306,
       MASTER_LOG_FILE='mysql-bin.000001',
       MASTER_LOG_POS=120,
       MASTER_CONNECT_RETRY=10;
    {% endhighlight %} 
    
    
 其中的信息是从主库中show master status中获取的
 这些信息将记录到master.info中
    
    
 启动从库同步,没有错误的话执行下面的语句查看状态.
    
    {% highlight sql%} 
    start slave;
    
    show slave status\G;  
    Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.78.134
                  Master_User: slave_user
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 120
               Relay_Log_File: SunxcMacBook-Pro-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 120
              Relay_Log_Space: 467
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: bc0f2f6f-9451-11e3-8a06-000c29fcefbe
             Master_Info_File: /usr/local/mysql-5.6.16-osx10.7-x86_64/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
    {% endhighlight %} 
    
    
    
 其中下面两个信息都要是yes 说明配置没问题  
     Slave_IO_Running: Yes  
     Slave_SQL_Running: Yes  
     Seconds_Behind_Master: 0  这个是从库延时的时间 一般0是我们最渴望的预期值。  
     ** 从库需要监控的也主要是这几个变量值 **  
     

   
### 7. 测试下切换到主库执行语句，再切换到从库看是否同步

     
     
    
    
    
    
