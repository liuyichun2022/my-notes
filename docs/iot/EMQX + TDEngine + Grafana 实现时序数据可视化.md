EMQX + TDEngine + Grafana 实现时序数据可视化

## 2、TDEngine部署
### 2.1、TDEngine安装
```bash
docker pull tdengine/tdengine:latest

docker run -d -v ~/data/taos/dnode/data:/var/lib/taos \
  -v ~/data/taos/dnode/log:/var/log/taos \
  -p 6030:6030 -p 6041:6041 -p 6043-6049:6043-6049 -p 6043-6049:6043-6049/udp tdengine/tdengine

docker ps

docker exec -it 830fa78e7029 bash
```

### 2.2、 测试TDEngine写入性能
默认的测试工具：taosBenchmark
```bash
SELECT COUNT(*) FROM test.meters;

SELECT AVG(current), MAX(voltage), MIN(phase) FROM test.meters;

SELECT COUNT(*) FROM test.meters WHERE location = "California.SanFrancisco";

SELECT AVG(current), MAX(voltage), MIN(phase) FROM test.meters WHERE groupId = 10;

SELECT FIRST(ts), AVG(current), MAX(voltage), MIN(phase) FROM test.d10 INTERVAL(10s);

```


## 3、安装Grafana
```bash

docker pull grafana/grafana
mkdir -p /data/grafana/{data,plugins,config}
chmod -R 777 /data/grafana/data
chmod -R 777 /data/grafana/plugins
chmod -R 777 /data/grafana/config

# 先临时启动一个容器
docker run --name grafana-tmp -d -p 3000:3000 grafana/grafana
# 将容器中默认的配置文件拷贝到宿主机上
docker cp grafana:/etc/grafana/grafana.ini /data/grafana/config/grafana.ini
# 移除临时容器
docker stop grafana-tmp
docker rm grafana-tmp

# 修改配置文件（需要的话）
vim /data/grafana/config/grafana.ini



# 启动 grafana
# 环境变量GF_SECURITY_ADMIN_PASSWORD：指定admin的密码
# 环境变量GF_INSTALL_PLUGINS：指定启动时需要安装得插件
#         grafana-clock-panel代表时间插件
#         grafana-simple-json-datasource代表json数据源插件
#         grafana-piechart-panel代表饼图插件

docker run -d \
    -p 3000:3000 \
    --name=grafana \
    -v /etc/localtime:/etc/localtime:ro \
    -v /data/grafana/data:/var/lib/grafana \
    -v /data/grafana/plugins/:/var/lib/grafana/plugins \
    -v /data/grafana/config/grafana.ini:/etc/grafana/grafana.ini \
    -e "GF_SECURITY_ADMIN_PASSWORD=admin" \
    grafana/grafana:10.2.2

```

## 报错
```bash
root@dell:~# docker logs e4be980b813c
Error: ✗ failed to download plugin archive: Get "https://storage.googleapis.com/plugins-community/tdengine-datasource/release/3.5.0/tdengine-datasourread tcp 172.17.0.9:50750->142.251.43.27:443: read: connection reset by peer

```


## 创建测试数据
CREATE DATABASE power;
USE power;  
CREATE TABLE sensor (ts timestamp, temp float, humi float) TAGS (location binary(64), groupdId int);
CREATE TABLE d1001 USING sensor TAGS ("Beijing.Chaoyang", 2);
insert into d1001 values (now,12.01,34);





## EMQX 默认密码
http://192.168.100.122:18083/#/ admin/public

root@dell:~# curl 192.168.100.122:6041/rest/login/root/taosdata
{"code":0,"desc":"/KfeAzX/f9na8qdtNZmtONryp201ma04bEl8LcvLUd7a8qdtNZmtONryp201ma04"}root@dell:~#


curl -H 'Authorization: Taosd /KfeAzX/f9na8qdtNZmtONryp201ma04bEl8LcvLUd7a8qdtNZmtONryp201ma04' -d 'select * from power.d1001' 192.168.100.122:6041/rest/sql

root@dell:~# curl -H 'Authorization: Taosd /KfeAzX/f9na8qdtNZmtONryp201ma04bEl8LcvLUd7a8qdtNZmtONryp201ma04' -d 'select * from power.d1001' 192.168.100.122:6041/rest/sql
{"code":0,"column_meta":[["ts","TIMESTAMP",8],["temp","FLOAT",4],["humi","FLOAT",4]],"data":[["2024-04-16T05:33:49.444Z",12.01,34],["2024-04-16T05:40:47.885Z",12.01,34],["2024-04-16T05:40:52.693Z",12.02,34],["2024-04-16T05:40:55.923Z",12.03,34],["2024-04-16T05:40:58.572Z",12.03,34],["2024-04-16T05:41:02.294Z",12.03,32],["2024-04-16T05:41:05.907Z",12.04,32],["2024-04-16T05:41:10.118Z",12.05,32],["2024-04-16T05:41:15.019Z",12.08,32],["2024-04-16T05:41:48.098Z",12.07,32],["2024-04-16T05:41:54.716Z",12.06,32],["2024-04-16T05:41:57.598Z",12.06,32],["2024-04-16T05:42:00.284Z",12.06,32],["2024-04-16T05:42:01.945Z",12.06,32],["2024-04-16T05:42:06.562Z",12.07,32],["2024-04-16T05:42:09.188Z",12.07,32],["2024-04-16T05:42:11.995Z",12.07,32],["2024-04-16T05:42:13.553Z",12.07,32],["2024-04-16T05:42:14.874Z",12.07,32],["2024-04-16T05:42:16.152Z",12.07,32],["2024-04-16T05:42:19.663Z",12.06,32],["2024-04-16T05:42:21.119Z",12.06,32],["2024-04-16T05:42:24.345Z",12.06,32],["2024-04-16T05:42:25.717Z",12.06,32],["2024-04-16T05:42:27.777Z",12.06,32],["2024-04-16T05:42:44.065Z",12.06,32],["2024-04-16T05:42:48.591Z",12.06,32],["2024-04-16T05:42:50.818Z",12.06,32],["2024-04-16T05:42:52.611Z",12.06,32],["2024-04-16T05:42:54.918Z",12.06,32],["2024-04-16T05:42:57.069Z",12.06,32],["2024-04-16T05:42:58.599Z",12.06,32],["2024-04-16T05:43:00.444Z",12.06,32],["2024-04-16T05:43:09.248Z",12.06,32],["2024-04-16T05:43:13.334Z",12.06,32],["2024-04-16T05:43:15.691Z",12.06,32],["2024-04-16T05:43:16.933Z",12.06,32]],"rows":37}root@dell:~#


## 参考文档
https://blog.csdn.net/u013810234/article/details/120432721

https://www.emqx.com/zh/blog/emqx-tdengine-grafana
