### Docker主从方式部署PostgreSql


####1. 运行PostgreSQL
 
1.1 主库


    $docker run --name pgsmaster -p 5500:5432 -e POSTGRES_PASSWORD=pgsmaster -v $(pwd)/pgsmaster:/var/lib/postgresql/data -d postgres

1.2 从库

    $docker run --name pgsslave -p 5501:5432 -e POSTGRES_PASSWORD=pgsslave -v $(pwd)/pgsslave:/var/lib/postgresql/data -d postgres
    

进入以上主、从库对应的实际挂载目录执行下面的操作

#### 2. 配置master（主库）

2.1 编辑pg_hba.conf，在最下面添加如下：


    // replication_username: 复制账号; slave_ip: 从库所在的服务器ip
    host    replication     <replication_username>      <slave_ip>/32          md5

2.2 编辑postgresql.conf（亲测，非必须），更改如下：


    synchronous_standby_names = '*'




2.3 进入容器，登录PostgreSQL，创建复制账号并验证：


		# 1.进入容器
        docker exec -it pgsmaster bash
		# 2.连接PostgreSQL
        psql -U postgres
		# 3.创建用户
        set synchronous_commit =off;
        // replication_username: 对应上面设置的复制账号; replication_username_password: 认证密码
        create role <replication_username> login replication encrypted password '<replication_username_password>';  
		# 4.验证用户
        \du



#### 3. 配置Slave（从库）

3.1 编辑postgresql.conf（亲测，非必须），更改如下：

    $hot_standby_feedback = on


3.2 新建recovery.conf，添加如下内容：

    standby_mode = 'on'
    // replication_username: 复制账号(同主库); master_ip: 主库所在的服务器ip; master_port: 主库端口; replication_username_password: 认证密码
    primary_conninfo = 'host=<master_ip> port=<master_port> user=<replication_username> password=<replication_username_password>'

#### 4. 同步主从库数据及测试

4.1 停止PostgreSQL

    docker stop pgsmaster 
    docker stop pgsslave


4.2 同步主从库数据(必须)

方法1：rsync

    // 1.1 已ssh认证，请将$(pwd)更改为实际的路径
    rsync -cva --inplace --exclude=*pg_xlog* $(pwd)/pgsmaster/ <slave_ip>:$(pwd)/pgsslave/
    // 1.2 无ssh认证，请将$(pwd)更改为实际的路径
    rsync -cva --inplace --exclude=*pg_xlog* $(pwd)/pgsmaster/ ssh root@<slave_ip>:$(pwd)/pgsslave/


方法2：pg_basebackup(自行谷歌)

4.3 先后启动主库、从库服务


    $docker start pgsmaster 
    $docker start pgsslave

4.4 连接测试

    // 进入主库容器
    docker exec -it pgsmaster bash
    // 查看复制状态
    psql -U postgres -x -c "select * from pg_stat_replication;"


进入容器执行命令：

    $psql -U postgres -x -c "select * from pg_stat_replication;"

###### 参考链接：
link:https://blog.csdn.net/qq_28804275/article/details/80891936