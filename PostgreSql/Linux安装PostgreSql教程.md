## Docker install PostgreSql

### 1 下载

    $docker pull postgres:9.4
### 2 创建

    $docker run --name postgres1 -e POSTGRES_PASSWORD=password -p 54321:5432 -d postgres:9.4

解释： 
run，创建并运行一个容器； 
--name，指定创建的容器的名字； 
-e POSTGRES_PASSWORD=password，设置环境变量，指定数据库的登录口令为password； 
-p 54321:5432，端口映射将容器的5432端口映射到外部机器的54321端口； 
-d postgres:9.4，指定使用postgres:9.4作为镜像。

### 连接数据库

之前的准备工作都已完成，下一步就是从外部访问数据库了。 
这一步就很常规了：


    $psql -U postgres -h 192.168.100.172 -p 54321

注意： 
postgres镜像默认的用户名为postgres， 
登陆口令为创建容器是指定的值。


参考链接:https://blog.csdn.net/liuyueyi1995/article/details/61204205