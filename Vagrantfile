
# Basic OS configuration
$sysconfig = <<SCRIPT
  # disable IPv6
  if [ "$(grep disable_ipv6 /etc/sysctl.conf | wc -l)" == "0" ]; then
    echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.conf \
      && echo "net.ipv6.conf.default.disable_ipv6=1" >> /etc/sysctl.conf \
      && sysctl -f /etc/sysctl.conf \
      && sysctl -p

    sysctl -w net.ipv6.conf.all.disable_ipv6=1
    sysctl -w net.ipv6.conf.default.disable_ipv6=1

  fi

  # this should be a persistent config
  ulimit -n 65536
  ulimit -s 10240
  ulimit -c unlimited

  systemctl disable firewalld && systemctl stop firewalld
  
  # Add entries to /etc/hosts
  ip=$(ip a s eth1 | awk '/inet/ {split($2, a,"/"); print a[1] }')
  host=$(hostname)
  echo "127.0.0.1 localhost" > /etc/hosts
  echo "$ip $host" >> /etc/hosts
  if [ "$(grep vm.swappiness /etc/sysctl.conf | wc -l)" == "0" ]; then
    echo "vm.swappiness=0" >> /etc/sysctl.conf && sysctl vm.swappiness=0
  fi

  # disable selinux 
  sudo setenforce 0
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

  # disable swap 
  swapoff -a

SCRIPT

# DNF configuration
$dnf_config = <<SCRIPT
  rpm -i https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm 2> /dev/null
  dnf -y remove java-1.6* > /dev/null
  dnf -y install wget curl curl-devel libxml2 libxml2-devel openssl openssl-devel java-latest-openjdk
SCRIPT


# Zookeeper installation & configuration
$zookeeper = <<SCRIPT

  ZOOKPR_LINK=/opt/zookeeper
  if [ ! -e ${ZOOKPR_LINK} ]; then
    echo "Zookeeper installation..."
    ZOOKPR_VER=zookeeper-3.5.6
    ZOOKPR_DAT=/opt/zookeeper-data
    adduser zookeeper
    wget http://mirrors.whoishostingthis.com/apache/zookeeper/${ZOOKPR_VER}/apache-${ZOOKPR_VER}-bin.tar.gz -q -P /tmp/ \
      && tar zxf /tmp/apache-${ZOOKPR_VER}-bin.tar.gz -C /opt/ \
      && ln -f -s /opt/apache-${ZOOKPR_VER}-bin ${ZOOKPR_LINK} \
      && echo "export ZOO_LOG_DIR=/opt/zookeeper/logs" > /etc/profile.d/zookeeper.sh \
      && echo "export PATH=\\${PATH}:${ZOOKPR_LINK}/bin" >> /etc/profile.d/zookeeper.sh \
      && mkdir -p ${ZOOKPR_DAT} \
      && mkdir -p ${ZOOKPR_LINK}/logs \
      && chown -R zookeeper:zookeeper ${ZOOKPR_DAT} \
      && chown -R zookeeper:zookeeper /opt/apache-${ZOOKPR_VER}-bin \
      && echo "dataDir=${ZOOKPR_DAT}" > ${ZOOKPR_LINK}/conf/zoo.cfg \
      && echo "maxClientCnxns=0" >> ${ZOOKPR_LINK}/conf/zoo.cfg \
      && echo "clientPort=2181" >> ${ZOOKPR_LINK}/conf/zoo.cfg \
      && echo "clientPortAddress=0.0.0.0" >> ${ZOOKPR_LINK}/conf/zoo.cfg
  fi

  cat << ZOOINITD > /etc/init.d/zookeeper
#! /bin/sh
# /etc/init.d/zookeeper: start the zookeeper daemon.
# chkconfig: 345 98 01
# description: zookeeper
ZOOKEEPER_HOME=/opt/zookeeper
ZOOKEEPER_CONFIG=\\${ZOOKEEPER_HOME}/conf/server.properties
USER=zookeeper

function start {
  sudo -E -u \\${USER} ZOO_LOG_DIR=${ZOOKPR_LINK}/logs \
    JVMFLAGS="-Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false" \
    \\${ZOOKEEPER_HOME}/bin/zkServer.sh start
}

function stop {
  sudo -E -u \\${USER} \\${ZOOKEEPER_HOME}/bin/zkServer.sh stop
}

case "\\$1" in
start)
  start
  ;;
stop)
  stop
  ;;
restart)
  stop
  start
  ;;
*)
  echo "Usage: \\$0 {start|stop|restart}"
  exit 1
esac
ZOOINITD

  chmod 755 /etc/init.d/zookeeper \
    && chkconfig zookeeper on

  if [ ! "$(ps aux | grep /opt/zookeeper | wc -l)" == "2" ]; then
    service zookeeper start
  fi

SCRIPT

# Kafka installation & configuration
$kafka = <<SCRIPT

  KAFKA_LINK=/opt/kafka
  KAFKA_HOST=$(ip a s eth1 | awk '/inet/ {split($2, a,"/"); print a[1]}')

  if [ ! -e ${KAFKA_LINK} ]; then
    echo "Kafka installation..."
    KAFKA_VER=2.4.0
    adduser kafka
    wget https://www-eu.apache.org/dist/kafka/${KAFKA_VER}/kafka_2.12-${KAFKA_VER}.tgz -q -P /tmp/ \
      && tar zxf /tmp/kafka_2.12-${KAFKA_VER}.tgz -C /opt/ \
      && ln -f -s /opt/kafka_2.12-${KAFKA_VER} ${KAFKA_LINK} \
      && echo "PATH=\\${PATH}:${KAFKA_LINK}/bin" > /etc/profile.d/kafka.sh \
      && mkdir -p ${KAFKA_LINK}/logs \
      && chown -R kafka:kafka /opt/kafka_2.12-${KAFKA_VER} \
      && echo "advertised.host.name=${KAFKA_HOST}" >> ${KAFKA_LINK}/config/server.properties \
      && sed -i "s/localhost/${KAFKA_HOST}/g" ${KAFKA_LINK}/config/producer.properties
  fi

  cat << KFINITD > /etc/init.d/kafka
#! /bin/sh
# /etc/init.d/kafka: start the kafka daemon.
# chkconfig: 345 99 01
# description: kafka
KAFKA_HOME=/opt/kafka
KAFKA_CONFIG=\\${KAFKA_HOME}/config/server.properties
USER=kafka
KAFKA_JMX_PORT=9666

function start {
  sudo -E -u \\${USER} JMX_PORT=\\${KAFKA_JMX_PORT} LOG_DIR=/tmp/kafka/log \\${KAFKA_HOME}/bin/kafka-server-start.sh -daemon \\${KAFKA_CONFIG}
}

function stop {
  sudo -u \\${USER} \\${KAFKA_HOME}/bin/kafka-server-stop.sh
}

case "\\$1" in
start)
  start
  ;;
stop)
  stop
  ;;
restart)
  stop
  start
  ;;
*)
  echo "Usage: \\$0 {start|stop|restart}"
  exit 1
esac

KFINITD

  chmod 755 /etc/init.d/kafka \
    && chkconfig kafka on

  if [ ! "$(ps aux | grep /opt/kafka | wc -l)" == "2" ]; then
    service kafka start
  fi

  # create sample topics
  ${KAFKA_LINK}/bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic clickstream --partitions 1 --replication-factor 1 &> /dev/null
  ${KAFKA_LINK}/bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic transactions --partitions 1 --replication-factor 1 &> /dev/null

SCRIPT

# Info
$information = <<SCRIPT
    ip=$(ip a s eth1 | awk '/inet/ {split($2, a,"/"); print a[1]}')
    echo ""
    echo "VM's IP address: $ip"
    echo ""
    echo "Start Zookeeper with the below command:"
    echo "sudo service zookeeper start"
    echo "logs avail. under /opt/zookeeper/logs"
    echo "JMX Endpoint available at: $ip:9999"
    echo ""
    echo "Start Kafka with the below command:"
    echo "sudo service kafka start"
    echo "logs avail. under /opt/kafka/logs"
    echo "JMX Endpoint available at: $ip:9666"
    echo "--> Kafka broker available at: $ip:9092 <--"
    echo ""
    echo "Available (sample) Kafka topics:"
    kafka-topics.sh --list --zookeeper localhost:2181
    echo ""
    echo "Test commands:"
    echo "kafka-topics.sh --zookeeper localhost:2181 --list"
    echo "kafka-topics.sh --zookeeper localhost:2181 --create --topic <topic_name> --partitions 1 --replication-factor 1"
    echo "kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic clickstream"
    echo "kafka-console-producer.sh --broker-list $ip:9092 --topic clickstream"
    echo "> test event"
    echo " "
SCRIPT

Vagrant.configure(2) do |config|

  config.vm.box = "centos/8"
  config.vm.hostname = "kafka.instance.com"
  config.vm.network :public_network

  config.vm.provider "virtualbox" do |vb|
    vb.name = "kafka-node"
    vb.cpus = 2
    vb.memory = 4096
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "100"]
  end

  config.vm.provision :shell, :name => "sysconfig", :inline => $sysconfig
  config.vm.provision :shell, :name => "dnf_config", :inline => $dnf_config
  config.vm.provision :shell, :name => "zookeeper", :inline => $zookeeper
  config.vm.provision :shell, :name => "kafka", :inline => $kafka
  config.vm.provision :shell, :name => "information", :inline => $information

end
