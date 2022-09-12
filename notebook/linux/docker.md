10.2.76.21
ssh root@10.2.76.21 
123456

init 3
service docker start
systemctl stop firewalld.service
systemctl disable firewalld.service
systemctl restart docker

mysql:

docker volume create --name l_mysql_conf
docker volume create --name l_mysql_logs
docker volume create --name l_mysql_data

docker run --rm --name mysql-ldd -v l_mysql_conf:/etc/mysql/conf.d -v l_mysql_logs:/logs -v l_mysql_data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root@123 -p 13306:3306 -d mysql:5.7.31



zk:

单机模式
docker volume create --name l_zk_conf

cd /var/lib/docker/volumes/l_zk_conf/_data
vim zoo.cfg
clientPort=21811
dataDir=/data
dataLogDir=/data/log
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
server.1=127.0.0.1:28881:38881

docker volume create --name l_zk_data

cd /var/lib/docker/volumes/l_zk_data/_data
echo 1 > myid

docker volume create --name l_zk_log
docker volume create --name l_zk_datalog

docker run --rm --name zk-ldd --network host -v l_zk_data:/data -v l_zk_conf:/conf -v l_zk_log:/logs -v l_zk_datalog:/datalog -d zookeeper

docker exec -it zk-ldd /bin/bash
zkServer.sh status

三台集群模式：
docker volume create --name l_zk_conf1

clientPort=21811
dataDir=/data
dataLogDir=/data/log
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
server.1=192.168.194.10:28881:38881
server.2=192.168.194.10:28882:38882
server.3=192.168.194.10:28883:38883

docker volume create --name l_zk_data1

echo 1 > myid

docker volume create --name l_zk_log1
docker volume create --name l_zk_datalog1



docker volume create --name l_zk_conf2
docker volume create --name l_zk_data2
docker volume create --name l_zk_log2
docker volume create --name l_zk_datalog2



docker volume create --name l_zk_conf3
docker volume create --name l_zk_data3
docker volume create --name l_zk_log3
docker volume create --name l_zk_datalog3


docker run --rm --name zk-ldd-1 --network host -v l_zk_data1:/data -v l_zk_conf1:/conf -v l_zk_log1:/logs -v l_zk_datalog1:/datalog -d zookeeper
docker run --rm --name zk-ldd-2 --network host -v l_zk_data2:/data -v l_zk_conf2:/conf -v l_zk_log2:/logs -v l_zk_datalog2:/datalog -d zookeeper
docker run --rm --name zk-ldd-3 --network host -v l_zk_data3:/data -v l_zk_conf3:/conf -v l_zk_log3:/logs -v l_zk_datalog3:/datalog -d zookeeper


pg:

docker volume create --name l_pg_data
docker run --rm --name pg-ldd -v l_pg_data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=root@123 -p 15432:5432 -d postgres


自己创建镜像

docker pull centos:7
docker run -di --name ldd-centos centos:7
docker cp jdk-8u301-linux-x64.tar.gz ldd-centos:/root
docker exec -it ldd-centos bash //进docker执行相应操作（配置path等）
docker commit -a="ldd" -m="jdk8" ldd-centos ldd-centos:7
docker images
docker run --rm -di --name ldd-centos-7 ldd-centos:7
docker exec -it ldd-centos-7 bash

使用dockfile
cd /usr/local/dockfile/
vim Docifile
FROM centos:7
LABEL maintainer="ldd"
WORKDIR /usr/local
RUN mkdir -p /usr/local/java
ADD jdk/jdk-8u301-linux-x64.tar.gz /usr/local/java
EXPOSE 8080
ENV JAVA_HOME /usr/local/java/jdk1.8.0_301
ENV PATH $PATH:$JAVA_HOME/bin

docker ps
docker build -f Docifile -t ldd-centos-d:7 /mnt/hgfs/share
docker run --rm --name ldd-centos-d-1 -di ldd-centos-d:7
docker exec -it ldd-centos-d-1 bash





oracle

docker volume create --name l_oracle_data
docker pull registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
docker run -d -p 1521:1521 -v l_oracle_data:/home/oracle/app/oracle/oradata --name oracle11g registry.cn-hangzhou.aliyuncs.com/helowin/oracle_11g
root/helowin

docker stop oracle11g
docker start oracle11g
docker exec -it oracle11g bash
source /home/oracle/.bash_profile
sqlplus / as sysdba
alter user system identified by system;

alter user sys identified by sys;

也可以创建用户  create user test identified by test;

并给用户赋予权限  grant connect,resource,dba to test; 输入：alter database mount;

输入 ：alter database open;
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;

CREATE TABLESPACE "LDD_TS_DATA"  LOGGING DATAFILE '/home/oracle/app/oracle/oradata/helowin/h_test.dbf' SIZE 100M  AUTOEXTEND  ON NEXT  100M MAXSIZE  8000M EXTENT MANAGEMENT LOCAL SEGMENT  SPACE MANAGEMENT  AUTO;

CREATE USER lddtest IDENTIFIED BY lddtest DEFAULT TABLESPACE "LDD_TS_DATA" TEMPORARY TABLESPACE "TEMP";
GRANT "CONNECT" TO lddtest;
GRANT "RESOURCE" TO lddtest;
GRANT create view TO lddtest;
GRANT create session TO lddtest;
ALTER USER lddtest DEFAULT ROLE "RESOURCE";


重启镜像
docker stop oracle11g
docker start oracle11g
docker exec -it oracle11g bash -c 'source /home/oracle/.bash_profile'



es

docker volume create --name l_es_conf

cd /var/lib/docker/volumes/l_es_conf/_data

vim config/elasticsearch.yml

cluster.name: ldd-es-cluster
node.name: ldd-es-node1
network.bind_host: 0.0.0.0
network.publish_host: 10.2.76.21
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true 
node.data: true  
discovery.zen.ping.unicast.hosts: ["10.2.76.21:9300"]
discovery.zen.minimum_master_nodes: 1
cluster.initial_master_nodes: ldd-es-node1

#添加配置
cluster.name: "docker-cluster"
network.host: 0.0.0.0
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]

docker volume create --name l_es_data
docker volume create --name l_es_logs

sysctl -w vm.max_map_count=262144

docker network create esnet

docker run --rm -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 --network esnet -v l_es_conf:/usr/share/elasticsearch/config -v l_es_logs:/usr/share/elasticsearch/logs -v l_es_data:/usr/share/elasticsearch/data --name ldd-es elasticsearch:7.7.0



docker pull mobz/elasticsearch-head:5
docker run --rm -d -p 19100:9100  --name ldd-es-head mobz/elasticsearch-head:5

#创建容器
docker create --name elasticsearch-head -p 9100:9100 mobz/elasticsearch-head:5
docker start mobz/elasticsearch-head:5
http://10.2.76.21:19100/



atlas

docker pull sburn/apache-atlas

docker volume create --name l_atlas_data
docker volume create --name l_atlas_conf
docker volume create --name l_atlas_logs
/var/lib/docker/volumes/l_atlas_data/_data/kafka
rm -rf *
docker run --rm  -d \
    -p 21000:21000 \
    -v l_atlas_logs:/opt/apache-atlas-2.1.0/logs \
    -v l_atlas_conf:/opt/apache-atlas-2.1.0/conf \
    -v l_atlas_data:/opt/apache-atlas-2.1.0/data \
    --name atlas \
    sburn/apache-atlas \
    /opt/apache-atlas-2.1.0/bin/atlas_start.py
docker exec -ti atlas tail -f -n 1000 /opt/apache-atlas-2.1.0/logs/application.log


mongo

docker pull mongo:latest
docker volume create --name l_mongo_data
docker volume create --name l_mongo_conf
docker run --rm -itd --name lddmongo --privileged=true -p 27017:27017 -v l_mongo_data:/data/db -v l_mongo_conf:/data/configdb docker.io/mongo:latest --auth

#进入容器
docker exec -it lddmongo mongo admin

# 切换到admin用户数据库
use admin

# 创建一个用户，分配好权限
db.createUser({
  user : 'root',
  pwd : 'root',
  roles : [
    'clusterAdmin',
    'dbAdminAnyDatabase',
    'userAdminAnyDatabase',
    'readWriteAnyDatabase'
  ]
});



# 测试是否创建成功（显示1则成功）
db.auth('root', 'root')


# 再同上，创建一个需要使用的数据库和用户

# 切换到mongo数据库，没有会自己创建
use mongo	
db.createUser({ user: 'mongo', pwd: 'mongo', roles: [{ role: 'readWrite', db: 'database' }] })
db.auth('mongo', 'mongo')

# exit退出

rabbitmq

docker volume create --name l_rabbitmq_data
# 查看镜像
docker images
# 拉取镜像到本地仓库，这里是直接安装最新的，
# 如果需要安装其他版本在rabbitmq后面跟上版本号即可
# docker pull rabbitmq
# 启动rabbitMq
docker run --rm -d -v l_rabbitmq_data:/var/lib/rabbitmq -p 5672:5672 -p 15672:15672 --name rabbitmq --hostname myRabbit rabbitmq:3-management

# 启动rabbitmq_management, rabbitmq 为容器的名称，使用id也可以
docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_management







hadoop


yum install -y java-1.8.0-openjdk-devel.x86_64export 
HADOOP_HOME=/usr/local/hadoop/hadoop-3.1.4
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
./$HADOOP_HOME/bin/start-all.sh


golang学习
# 拉镜像

docker pull hunterhug/gotourzh
# 后台运行
docker run -d -p 9999:9999 hunterhug/gotourzh
http://10.2.76.21:9999