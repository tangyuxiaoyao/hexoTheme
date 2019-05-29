---
title: docker use
date: 2019-01-15 19:16:06
tags: [docker]
---


----------

docker的使用总结

----------

<!--more-->


+ 指定镜像启动容器

``` shell
docker run -P tomcat //随机端口访问 将tomcat容器的8080端口映射到宿主机的随机端口
docker run -p 80:8080 tomcat //指定端口访问 将tomcat容器的8080端口映射到宿主机的80端口
```
 
+ 编辑正在访问的容器

``` shell
docker exec -it CONTAINER ID bash //进入容器的命令行式交互式界面
```

+ 停止正在运行的容器

``` shell
docker kill cid/cname
docker stop cid/cname
```

+ 继承并定制化自己风格的容器

``` shell
mkdir empty
cd empty
vi Dockerfile
FROM tomcat
RUN echo '<h1>Hello, Docker!</h1>' > /usr/local/tomcat/webapps/ROOT/index.jsp

docker build -t tomcat:hello .

~/docker/testDocker$ docker images
REPOSITORY   TAG                 IMAGE ID            CREATED              SIZE
tomcat       hello               a5ed8e716cce        About a minute ago   462MB
```
+ 本地文件打包部署到容器

``` shell
TODO
```


+ 清除掉归档状态的容器(status=Exited)


``` shell
docker container prune //删除全部
docker rm cid/cname //删除指定的容器
```

+ 使用数据卷实现宿主项目挂载到容器


``` shell
docker run -p 8081:8080  -v /home/user/docker/testDocker/vtest:/usr/local/tomcat/webapps/ROOT --name tomcat2 -d tomcat
```

+ 部署mysql指定版本容器


``` shell
docker run -p 3306:3306 --name mysql \
-v $PWD/conf:/etc/mysql/conf.d \
-v $PWD/logs:/logs \
-v $PWD/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7.22
```