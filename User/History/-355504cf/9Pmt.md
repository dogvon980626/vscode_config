# 第1章 传统日志分析需求

## 1.为什么需要分析日志

## 2.日志分析的需求

```
1.找出访问排名前十的IP,URL
2.找出10点到12点之间排名前十的IP,URL
3.对比昨天这个时间段访问情况有什么变化
4.对比上个星期同一天同一时间段的访问变化
5.找出搜索引擎访问的次数和每个搜索引擎各访问了多少次
6.指定域名的关键链接访问次数,响应时间
7.网站HTTP状态码情况
8.找出攻击者的IP地址,这个IP访问了什么页面,这个IP什么时候来的,什么时候走的,共访问了多少次
9.5分钟内告诉为结果
```

# 第2章 EBLK介绍

## 1.EBLK的功能介绍

```
E	  Elasticsearch	 java
B  	Filebeat		   Go
L	  Logstash		   java
K  	Kibana			   java
```

## 2.EBLK日志收集流程

![image-20201218172401565](https://vim30.oss-accelerate.aliyuncs.com/img/image-20201218172401565.png)

# 第3章 实验环境配置

## 1.Elasticsearch单节点安装部署

```
rpm -ivh elasticsearch-7.9.1-x86_64.rpm
cat > /etc/elasticsearch/elasticsearch.yml << 'EOF'    
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 127.0.0.1,10.0.0.51
http.port: 9200
discovery.seed_hosts: ["10.0.0.51"]
cluster.initial_master_nodes: ["10.0.0.51"]
EOF
systemctl daemon-reload
systemctl start elasticsearch.service
netstat -lntup|grep 9200
curl 127.0.0.1:9200
```

## 2.kibana安装部署

```
rpm -ivh kibana-7.9.1-x86_64.rpm
cat > /etc/kibana/kibana.yml << 'EOF'
server.port: 5601
server.host: "10.0.0.51"
elasticsearch.hosts: ["http://10.0.0.51:9200"]
kibana.index: ".kibana"
EOF
systemctl start kibana
```

## 3.Elasticsearch-head安装部署

```
google浏览器-->更多工具-->拓展程序-->开发者模式-->选择解压缩后的插件目录
```

# 第4章 收集普通格式的nginx日志

## 1.安装配置nginx

```
cat > /etc/yum.repos.d/nginx.repo <<'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=0
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
EOF

yum makecache fast
yum install nginx -y
systemctl start nginx
```

## 2.安装配置filebeat

```
rpm -ivh filebeat-7.9.1-x86_64.rpm
cp  /etc/filebeat/filebeat.yml /opt/
cat > /etc/filebeat/filebeat.yml << EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
EOF
```

## 3.启动并检查

```
systemctl start filebeat
tail -f /var/log/filebeat/filebeat
```

## 4.生成nginx访问日志

```
curl 127.0.0.1
```

## 5.检查收集效果



# 第5章 收集Json格式的Nginx日志

## 1.当前日志收集方案的不足

```
所有日志都存储在message的value里,不能拆分单独显示
要想单独显示，就得想办法把日志字段拆分开，变成json格式
```

## 2.我们期望的日志收集效果

```
可以把日志所有字段拆分出来
{
	$remote_addr : 192.168.12.254
	- : -
	$remote_user : -
	[$time_local]: [10/Sep/2019:10:52:08 +0800]
	$request: GET /jhdgsjfgjhshj HTTP/1.0
	$status : 404
	$body_bytes_sent : 153
	$http_referer : -
	$http_user_agent :ApacheBench/2.3
	$http_x_forwarded_for:-
}
```

## 3.修改Nginx配置文件

```
log_format json '{ "time_local": "$time_local", '
                          '"remote_addr": "$remote_addr", '
                          '"referer": "$http_referer", '
                          '"request": "$request", '
                          '"status": $status, '
                          '"bytes": $body_bytes_sent, '
                          '"user_agent": "$http_user_agent", '
                          '"x_forwarded": "$http_x_forwarded_for", '
                          '"up_addr": "$upstream_addr",'
                          '"up_host": "$upstream_http_host",'
                          '"upstream_time": "$upstream_response_time",'
                          '"request_time": "$request_time"'
    ' }';
    access_log  /var/log/nginx/access.log  json;
```

## 4.清空旧的日志文件

```
> /var/log/nginx/access.log
```

## 5.检查并重启nginx

```
nginx -t
systemctl restart nginx
```

## 6.检查Nginx日志是否为json格式

```
curl 127.0.0.1
cat /var/log/nginx/access.log
```

## 7.修改filebeat配置文件

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
```

## 8.重启filebeat

```
systemctl restart filebeat
```

## 8.创造访问日志并检查收集结果

```

```

## 9.kibana重建索引匹配

![image-20220214151233952](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151233952.png)

![image-20220214151331705](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151331705.png)

![image-20220214151349407](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151349407.png)

![image-20220214151445501](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151445501.png)

![image-20220214151500589](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151500589.png)

![image-20220214151529415](https://vim30.oss-accelerate.aliyuncs.com/img/image-20220214151529415.png)

# 第6章 自定义Elasticsearch索引名称

## 1.当前日志收集方案的不足

```
虽然日志可以拆分了，但是索引名称还是默认的，根据索引名称并不能看出来收集的是什么日志。
```

## 2.我们期望的日志收集效果

```

```

## 3.修改filebeat配置文件

```

```

## 4.创造访问日志并检查收集结果

```

```

# 第7章 按日志类型定义索引名称

## 1.当前日志收集方案的不足

```

```

## 2.我们期望的效果

```

```

## 3.修改filebeat配置文件

```

```

## 4.创造访问日志并检查收集结果

```

```

# 第8章 使用ES预处理节点转换Nginx普通日志

## 1.什么是预处理节点

```

```

## 2.预处理节点处理日志流程

```

```

## 3.配置pipeline规则

```

```

## 4.修改nginx配置文件 

```

```

## 5.修改filebeat配置文件

```

```

## 6.创造访问日志并检查收集结果

```

```

# 第9章 使用filebeat模块收集Nginx普通日志

## 1.什么是filebeat模块

## 2.filebeat模块收集日志流程

## 3.修改filebeat配置文件

## 4.激活以及配置filebeat模块

## 5.创造访问日志并检查收集结果

# 第10章 使用模块收集MySQL慢日志

## 1.mysql日志介绍

## 2.二进制安装配置MySQL 5.7

## 3.修改filebeat配置文件

## 4.激活以及配置filebeat模块

## 5.创造mysql慢日志并检查收集结果

# 第11章 收集tomcat的json日志

## 1.安装部署tomcat

## 2.tomcat日志介绍

## 3.修改tomcat配置文件

## 4.修改filebeat配置文件

## 5.创造访问日志并检查收集结果

# 第12章 收集Java多行日志

## 1.java日志特点介绍

## 2.filebeat多行匹配模式介绍

## 3.修改filebeat配置文件

## 4.创造访问日志并检查收集结果

# 第13章 使用Redis作为缓存

## 1.为什么需要使用redis作为缓存

## 2.使用redis作为缓存的日志收集流程

## 3.安装配置redis

## 4.修改filebeat配置文件

## 5.安装配置logstash

## 6.创造访问日志并检查收集结果

# 第14章 使用kafka作为缓存

## 1.为什么需要使用redis作为缓存

## 2.使用kafka作为缓存的日志收集流程

## 3.安装配置kafka

## 4.修改filebeat配置文件

## 5.安装配置logstash

## 6.创造访问日志并检查收集结果

# 第15章 EBLK安全认证配置

## 1.EBLK认证介绍

## 2.Elasticsearch配置账号密码

## 3.修改filebeat配置文件

## 4.修改logstash配置文件

## 5.修改kibana配置文件

## 6.通过kibana设置不同用户的权限

## 7.检查实验效果是否符合期望效果

# 第16章 使用EBK收集k8s的pod日志

## 1.k8s日志收集流程介绍

## 2.k8s环境快速搭建部署

## 3.k8s交付EBK

## 4.k8s交付边车模式的NginxPOD

## 5.创造访问日志并检查收集结果

# 第X章 kibana画图

























