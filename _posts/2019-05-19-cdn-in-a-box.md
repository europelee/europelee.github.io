# CDN in a Box(译)  
原文来自<https://traffic-control-cdn.readthedocs.io/en/latest/admin/quick_howto/ciab.html#cdn-in-a-box>  

"CDN in a Box"是一个按惯例起的名字，源于Traffic Control新的开发者或有意使用的用户试图建立一个他们自己的微型CDN来体验Traffic Control的各个组件是如何构建出一个CDN系统。在以前，这是一个噩梦，需要复杂的脚本和手工配置很多的网络设置。最近几年，很多人已经把项目的安装部署Docker化了, 简化网络配置，虽然目前还是有些过程比较有障碍。项目已经达到了一个可工作状态，并且mock/测试CDN的工作是一个相当简单的任务（虽然比较花时间）。  

## Getting Started  

预置条件：  
Docker version >= 17.05.0-ce  
Docker Compose[1] version >= 1.9.0  

## Building  

CDN in a Box目录在Traffic Control仓库的infrastructure/cdn-in-a-box下。CDN in a Box依赖下面的预编译Traffic Contorl组件rpm文件：  
Traffic Monitor - at infrastructure/cdn-in-a-box/traffic_monitor/traffic_monitor.rpm
Traffic Ops - at infrastructure/cdn-in-a-box/traffic_ops/traffic_ops.rpm
Traffic Portal - at infrastructure/cdn-in-a-box/traffic_portal/traffic_portal.rpm
Traffic Router - at infrastructure/cdn-in-a-box/traffic_router/traffic_router.rpm - also requires an Apache Tomcat RPM at infrastructure/cdn-in-a-box/traffic_router/tomcat.rpm  


这些rpm包可以通过那些编译Traffic Control或者外部源的步骤来获取。另外方式，infrastructure/cdn-in-a-box路径下的Makefile文件包含了所有这些编译需要，在目录infrastructure/cdn-in-a-box简单运行make命令既可，一旦所有依赖的RPM生成，在infrastructure/cdn-in-a-box目录下执行docker-compose build就可以构建出运行"CDN in a Box"所需的镜像。  


## Usage  
典型场景下，假如上述编译步骤有顺利执行完成，那么启动CDN in a Box仅需要在目录infrastructure/cdn-in-a-box执行一个操作 docker-compose up, -d参数是可选的，用于运行在后台。这个操作将启动所有stack,并且会做好所需的各种初始化配置。容器内的服务将通过指定端口暴露于本地。这些是在infrastructure/cdn-in-a-box/docker-compose.yml文件内配置的，但是默认端口显示在Service Info。某些服务需要证书关联，这些是配置在variables.env.  

Table 41 Service Info  
Service | Ports exposed and their usage| Username | Password
- | :- | :- | :- 
DNS | DNS name resolution on 9353	 | N/A | N/A
Edge Tier Cache|Apache Trafficserver HTTP caching reverse proxy on port 9000|N/A|N/A
Mid Tier Cache|Apache Trafficserver HTTP caching forward proxy on port 9100|N/A|N/A
Mock Origin Server|Example web page served on port 9200|N/A|N/A
Traffic Monitor|Web interface and API on port 80|N/A|N/A
Traffic Ops|Main API endpoints on port 6443, with a direct route to the Perl API on port 60443[3]|TO_ADMIN_USER in variables.env|TO_ADMIN_PASSWORD in variables.env
Traffic Ops PostgresQL Database|PostgresQL connections accepted on port 5432 (database name: DB_NAME in variables.env)|DB_USER in variables.env|DB_USER_PASS in variables.env
Traffic Portal|Web interface on 443 (Javascript required)|TO_ADMIN_USER in variables.env|TO_ADMIN_PASSWORD in variables.env
Traffic Router|Web interfaces on ports 3080 (HTTP) and 3443 (HTTPS), with a DNS service on 53 and an API on 3333|N/A|N/A
Traffic Vault|Riak key-value store on port 8010|TV_ADMIN_USER in variables.env|TV_ADMIN_PASSWORD in variables.env
Traffic Stats|N/A|N/A|N/A
Traffic Stats Influxdb|Influxdbd connections accepted on port 8086 (database name: cache_stats, daily_stats and deliveryservice_stats)|INFLUXDB_ADMIN_USER in variables.env|INFLUXDB_ADMIN_PASSWORD in variables.env  

当组件通过使用这些ports来互相交互，CDN的操作只能从Docker网络看到，为了看到CDN的实践，连接到CDN in a Box项目一个container, 使用cURL来请求URL http://video.demo1.mycdn.ciab.test, 会被DNS container解析到Traffic Router的ip，Traffic Router会提供302响应指向Edge-Tier cache.
一个典型的选择是"enroller"服务，它有curl命令安装的。对用户更友好的接口，参考VNC Server.  

#51 Example Command to See the CDN in Action  
```
sudo docker-compose exec enroller /usr/bin/curl -L "http://video.demo1.mycdn.ciab.test"  
```
使用docker-compose down -v关闭CDN, 这是因为下次启动时系统中共享卷的使用可能会和一个合适的初始化交互。  
**variables.env**  
```
TLD_DOMAIN=ciab.test
INFRA_SUBDOMAIN=infra
CDN_NAME=CDN-in-a-Box
CDN_SUBDOMAIN=mycdn
DS_HOSTS=demo1 demo2 demo3
X509_CA_NAME=CIAB-CA
X509_CA_COUNTRY=US
X509_CA_STATE=Colorado
X509_CA_CITY=Denver
X509_CA_COMPANY=NotComcast
X509_CA_ORG=CDN-in-a-Box
X509_CA_ORGUNIT=CDN-in-a-Box
X509_CA_EMAIL=no-reply@infra.ciab.test
X509_CA_DIGEST=sha256
X509_CA_DURATION_DAYS=365
X509_CA_KEYTYPE=rsa
X509_CA_KEYSIZE=4096
X509_CA_UMASK=0000
X509_CA_DIR=/shared/ssl
X509_CA_PERSIST_DIR=/ca
X509_CA_PERSIST_ENV_FILE=/ca/environment
X509_CA_ENV_FILE=/shared/ssl/environment
DB_NAME=traffic_ops
DB_PORT=5432
DB_SERVER=db
DB_USER=traffic_ops
DB_USER_PASS=twelve
DNS_SERVER=dns
DBIC_TRACE=0
ENROLLER_HOST=enroller
PGPASSWORD=twelve
POSTGRES_PASSWORD=twelve
EDGE_HOST=edge
INFLUXDB_HOST=influxdb
INFLUXDB_PORT=8086
INFLUXDB_ADMIN_USER=influxadmin
INFLUXDB_ADMIN_PASSWORD=influxadminpassword
GRAFANA_ADMIN_USER=grafanaadmin
GRAFANA_ADMIN_PASSWORD=grafanaadminpassword
GRAFANA_PORT=443
MID_HOST=mid
ORIGIN_HOST=origin
TM_HOST=trafficmonitor
TM_PORT=80
TM_EMAIL=tmonitor@cdn.example.com
TM_PASSWORD=jhdslvhdfsuklvfhsuvlhs
TM_USER=tmon
TO_ADMIN_PASSWORD=twelve
TO_ADMIN_USER=admin
TO_EMAIL=cdnadmin@example.com
TO_HOST=trafficops
TO_PORT=443
TO_PERL_HOST=trafficops-perl
TO_PERL_PORT=443
TO_SECRET=blahblah
TP_HOST=trafficportal
TP_EMAIL=tp@cdn.example.com
TR_HOST=trafficrouter
TR_DNS_PORT=53
TR_HTTP_PORT=80
TR_HTTPS_PORT=443
TR_API_PORT=3333
TP_PORT=443
TS_EMAIL=tstats@cdn.example.com
TS_HOST=trafficstats
TS_PASSWORD=trafficstatspassword
TS_USER=tstats
TV_HOST=trafficvault
TV_USER=tvault
TV_PASSWORD=mwL5GP6Ghu_uJpkfjfiBmii3l9vfgLl0
TV_EMAIL=tvault@cdn.example.com
TV_ADMIN_USER=admin
TV_ADMIN_PASSWORD=riakAdmin
TV_RIAK_USER=riakuser
TV_RIAK_PASSWORD=riakPassword
TV_INT_PORT=8087
TV_HTTP_PORT=8098
TV_HTTPS_PORT=8088
ENROLLER_DIR=/shared/enroller
AUTO_SNAPQUEUE_ENABLED=true
AUTO_SNAPQUEUE_SERVERS=trafficops,trafficops-perl,trafficmonitor,trafficrouter,trafficvault,edge,mid
AUTO_SNAPQUEUE_POLL_INTERVAL=2
AUTO_SNAPQUEUE_ACTION_WAIT=2
```  

## X.509 SSL/TLS Certificates  
All components in Apache Traffic Control utilize SSL/TLS secure communications by default. For SSL/TLS connections to properly validate within the “CDN in a Box” container network a shared self-signed X.509 Root CA is generated at the first initial startup. An X.509 Intermediate CA is also generated and signed by the Root CA. Additional “wildcard” certificates are generated/signed by the Intermediate CA for each container service and demo1, demo2, and demo3 Delivery Services. All certificates and keys are stored in the ca host volume which is located at infrastruture/cdn-in-a-box/traffic_ops/ca[4].  
Apache Traffic Control的所有组件默认是使用SSL/TLS保障安全通信。  

## Advanced Usage  
待续