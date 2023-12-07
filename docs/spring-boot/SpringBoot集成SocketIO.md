## SpringBoot 集成SocketIO

SocketIO官网：https://socket.nodejs.cn/

博客的五花八门，Socket'IO官网写的还是比较的详细。

SpringBoot集成netty-socketio

1、安装依赖

```xml
		<!--SocketIO-->
        <dependency>
            <groupId>com.corundumstudio.socketio</groupId>
            <artifactId>netty-socketio</artifactId>
            <version>1.7.11</version>
        </dependency>
```



2、yml文件配置

```yml
socketio:
  allowCustomRequests: true
  bossCount: 1
  host: 0.0.0.0       # 注意：要配置成0.0.0.0本机和服务器都能访问，不然很容易造成本地测试没问题，发布到环境上不能用
  maxFramePayloadLength: 1048576
  maxHttpContentLength: 1048576
  pingInterval: 25000
  pingTimeout: 6000000
  port: 8099
  upgradeTimeout: 1000000
  workCount: 100
```

3、SocketIoConfig配置注册启动SocketIO服务端

