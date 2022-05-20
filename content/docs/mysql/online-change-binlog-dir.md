---
title: "MySQL在线更改Binlog目录"
date: 2022-05-19 22:07:16
tags: ["MySQL"]
Categories: ["MySQL"]
description: |
  线上使用共享表空间，因共享表空间没办法释放，导致磁盘可用空间太小，所以新找几台机器，重新导出导入，将业务迁移到新机器上。
  因需要从备机导数据，需要保留binlog个数不固定，为避免导数机器被binlog撑满，将其他磁盘的目录挂载到binlog目录。
draft: false
---

## 背景

线上使用共享表空间，因共享表空间没办法释放，导致磁盘可用空间太小，所以新找几台机器，重新导出导入，将业务迁移到新机器上。
因需要从备机导数据，需要保留binlog个数不固定，为避免导数机器被binlog撑满，将其他磁盘的目录挂载到binlog目录。

## 原理

1. 将binlog目录挂载到其他分区
2. 重新生成binlog/relay log日志，使其在新目录生成
   
   > 注1：因为操作会进行`reset master`操作，所以不适用下游有空库或者其他拉取binlog的服务(如DM、Debezium等)
   > 注2：如果可以接受停机的话，直接挂载目录，将Binlog文件复制到新目录更简单，对下游也没有影响

## 操作

1. 将备机写入暂停，记录操作前GTID/slave信息
   
   ```mysql
   # 1. 停止复制线程，将备机数据固定
   stop slave;
   
   # 2. 记录当前GTID/Binlog/Slave信息
   show master status\G
   show slave status\G
   ```

2. 在另一分区创建新目录，并将新目录挂载到旧binlog目录
   
   ```shell
   mkdir ${new_binlog_dir}
   chown mysql:mysql ${new_binlog_dir}
   mount --bind ${new_binlog_dir} ${old_binlog_dir}
   ```

3. 重置binlog和relay log，使其在新目录重新生成
   
   ```mysql
   # 1. 清空master信息(会同时清空GTID/binlog)，重新生成binlog文件
   reset master;
   
   # 2. 清空slave信息，重新生成relay log文件
   reset slave;
   ```

4. 恢复复制关系，下面两种方式都可以恢复
   
   ```mysql
   # 1. 手动设置第一步保存的GTID信息，使用auto prosition自动找点
   set global gtid_purged='{old_gtid}';
   change master to master_auto_position=1;
   start slave;
   
   # 2. 根据第一步保存的binlog位置，手动设置binlog位置
   change master to master_log_file='{old_file}', master_log_pos={old_pos}, master_auto_position=0;
   start slave;
   ```
