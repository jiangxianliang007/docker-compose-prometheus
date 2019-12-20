# 使用docker-compose 快速部署 promethues、grafana

promethues配置文件中注释了 mysql、redis、nginx、elasticsearch 的exporter需要自行开启。 

## 截止2019-12-20 使用版本均为最新

```
prometheus:  v2.14.0
alertmanager:  v0.19.0
grafana:  v6.5.0
node-exporter:  v0.18.1
```

## 告警规则含有：
1、主机资源 内存、硬盘、CPU 使用率

2、redis 内存使用、连接数

3、mysql 存活、连接数、qps、主从同步、同步延迟

4、elasticsearch 集群状态、节点数量

## node-exporter
启动 node-exporter
```
docker run -d -it \
 -p 9100:9100 \
 -v "/proc:/host/proc" \
 -v "/sys:/host/sys" \
 -v "/:/rootfs" \
 --net="host" \
 --name=node-exporter prom/node-exporter:v0.18.1 \
 --path.procfs /host/proc  \
 --collector.filesystem.ignored-mount-points "^(/rootfs|/host|)/(sys|proc|dev|host|etc)($$|/)"
```
模板：
https://grafana.com/grafana/dashboards/11074
https://grafana.com/grafana/dashboards/1860

## mysqld-exporter
监控mysql 需要授权账号
```
CREATE USER 'prometheus'@'%' IDENTIFIED BY 'miyuankeji666' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'%';

docker run -d \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="prometheus:miyuankeji666@(172.19.233.184:3306)/" \
  prom/mysqld-exporter
```
参数解释：

DATA_SOURCE_NAME ：数据库的用户、密码、连接地址与端口

模板：https://grafana.com/grafana/dashboards/7362

## redis_exporter
启动redis_exporter
```
docker run -d \
 --name redis_exporter \
 -p 9121:9121 oliver006/redis_exporter \
 --redis.addr redis://172.19.233.184:6379 \
 --redis.password 'miyuankeji666'
```
参数解释：

--redis.addr :  redis连接地址与端口

--redis.password :  redis密码(无设置密码，不需要加该参数)

模板：https://grafana.com/grafana/dashboards/763

## elasticsearch_exporter
启动 elasticsearch_exporter
```
docker run -d \
--name elasticsearch_exporter \
-p 9114:9114 justwatch/elasticsearch_exporter:1.1.0 \
--es.uri http://172.19.233.184:9200
```
参数解释：

--es.uri： es 节点地址与端口

模板：https://grafana.com/dashboards/2322