# prometheus、grafana 监控部署

服务端通过 docker-compose 方式部署

```
git clone https://github.com/jiangxianliang007/docker-compose-prometheus.git
cd docker-compose-prometheus
docker-compose up -d
```
浏览器访问 服务器IP:3000 User:admin Password:admin  导入node-exporter 的 Dashboards 图表就显示服务端的数据了

exporter 均使用 docker 方式运行

包含 node、mysql、redis、nginx、elasticsearch、rabbitmq、jvm 的 exporter 运行； 


## 截止2019-12-20 使用版本均为最新

```
prometheus:  v2.14.0
alertmanager:  v0.19.0
grafana:  v6.5.0
node-exporter:  v0.18.1
```

## 告警规则含有：
1、主机资源 内存、硬盘、CPU 网卡进出口带宽

2、redis 内存使用、连接数

3、mysql 存活、连接数、QPS、主从同步、同步延迟

4、elasticsearch 集群状态、节点数量

5、nginx 5xx、4xx 状态码 

6、rabbitmq 存活、连接数

7、prometheus 动态更新配置是否生效

8、jvm 堆栈使用率、GC 时间

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
监控 mysql 需要授权账号
```
CREATE USER 'prometheus'@'%' IDENTIFIED BY 'miyuankeji666' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus'@'%';
```
启动 mysqld-exporter
```
docker run -d \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="prometheus:miyuankeji666@(172.19.233.184:3306)/" \
  prom/mysqld-exporter
```
参数解释：

DATA_SOURCE_NAME ：数据库的用户、密码、连接地址与端口

模板：https://grafana.com/grafana/dashboards/7362

## redis_exporter
启动 redis_exporter
```
docker run -d \
 --name redis_exporter \
 -p 9121:9121 oliver006/redis_exporter \
 --redis.addr redis://172.19.233.184:6379 \
 --redis.password 'miyuankeji666'
```
参数解释：

--redis.addr : redis连接地址与端口

--redis.password : redis密码(无设置密码，不需要加该参数)

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

--es.uri：es 节点地址与端口

模板：https://grafana.com/dashboards/2322

## nginx_exporter
开启 nginx 监控需要重新编译加入 nginx-vts-exporter 模块

在 nginx.conf 配置文件 http 段增加：
```
vhost_traffic_status_zone;
```
在虚拟主机 server 段增加：
```
    location /vts_status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
    }
```
通过 http://192.168.1.100/vts_status/format/json 可查看监控数据
启动 nginx_exporter
```
docker run -d \
 -p 9113:9913 \
 --name nginx_exporter \
 --env NGINX_STATUS="http://172.19.233.184/vts_status/format/json" \
 sophos/nginx-vts-exporter
```
参数解释：

NGINX_STATUS：nginx 数据采集地址

模板：https://grafana.com/grafana/dashboards/2949

## rabbitmq-exporter
启动 rabbitmq-exporter
```
docker run -d \
 -p 9119:9419 \
 -e RABBIT_URL="http://172.19.233.184:15672" \
 --name=rabbitmq-exporter \
 kbudde/rabbitmq-exporter 
```
参数解释：

RABBIT_URL：rabbitmq 连接地址与端口

模板：https://grafana.com/grafana/dashboards/4371


## process-exporter
以监控nginx 进程为例, 先创建配置文件
```
vim process-name.yaml
process_names:
  - name: "{{.Matches}}"
    cmdline:
    - 'nginx'
```
启动 process-exporter
```
docker run -d \
 -p 9156:9256 \
 --privileged \
 -v /proc:/host/proc \
 -v `pwd`:/config \
 --name=process-exporter  \
 ncabatoff/process-exporter \
 --procfs /host/proc -config.path /config/process-name.yaml
```

模板：https://grafana.com/grafana/dashboards/4202

## jmx_exporter
以监控tomcat为例

下载 jmx_exporter 
```
wget -P /usr/local/src/  https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.12.0/jmx_prometheus_javaagent-0.12.0.jar
```
在 tomcat 的 catalina.sh  脚本中加上如下配置，再启动tomcat

```
vim apache-tomcat-8.5.29/bin/catalina.sh

JAVA_OPTS="-javaagent:/usr/local/src/jmx_prometheus_javaagent-0.12.0.jar=9116:/usr/local/src/tomcat.yml"
```
参数解释：

javaagent：jmx_exporter jar 包路径、监听端口、配置文件（在configs可以获取）

模板：https://grafana.com/grafana/dashboards/8563