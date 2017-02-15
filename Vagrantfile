
# Basic OS configuration
$sysconfig = <<SCRIPT

  # disable IPv6
  echo "net.ipv6.conf.all.disable_ipv6=1" > /etc/sysctl.conf 
  echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
  sysctl -f /etc/sysctl.conf


  # this should be a persistent config
  ulimit -n 65536
  ulimit -s 10240
  ulimit -c unlimited

  service iptables stop && chkconfig iptables off

  # Add entries to /etc/hosts
  ip=$(ifconfig eth1 | awk -v host=$(hostname) '/inet addr/ {print substr($2,6)}')
  host=$(hostname)
  echo "127.0.0.1 localhost" > /etc/hosts
  echo "$ip $host" >> /etc/hosts

  PASSWORD=$(echo "$(date)$RANDOM" | md5sum | awk '{print $1}')
  KAFKA_USER=kafka
  if ! grep ${KAFKA_USER} /etc/passwd; then
    echo "Creating user ${KAFKA_USER}" \
        && useradd -p $(openssl passwd -1 ${PASSWORD}) ${KAFKA_USER}
  fi

  ZOOKPR_USER=zookeeper
  if ! grep ${ZOOKPR_USER} /etc/passwd; then
    echo "Creating user ${ZOOKPR_USER}" \
        && useradd -p $(openssl passwd -1 ${PASSWORD}) ${ZOOKPR_USER}
  fi

SCRIPT

$javajdk = <<SCRIPT
  wget -q --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/7u79-b15/jdk-7u79-linux-x64.rpm" \
    && yum -y remove java-1.6* \
    && rpm -i jdk-7u79-linux-x64.rpm

  echo "export JAVA_HOME=/usr/java/default" > /etc/profile.d/java.sh
SCRIPT

# Zookeeper installation & configuration
$zookeeper = <<SCRIPT

  ZOOKPR_LINK=/opt/zookeeper
  if [ ! -e ${ZOOKPR_LINK} ]; then
    echo "Zookeeper installation..."
    ZOOKPR_VER=zookeeper-3.4.9
    ZOOKPR_DAT=/opt/zookeeper-data
    wget http://mirrors.whoishostingthis.com/apache/zookeeper/${ZOOKPR_VER}/${ZOOKPR_VER}.tar.gz -q -P /tmp/ \
      && tar zxf /tmp/${ZOOKPR_VER}.tar.gz -C /opt/ \
      && ln -f -s /opt/${ZOOKPR_VER} ${ZOOKPR_LINK} \
      && echo "export ZOO_LOG_DIR=/opt/zookeeper/logs" > /etc/profile.d/zookeeper.sh \
      && echo "export PATH=\\${PATH}:${ZOOKPR_LINK}/bin" >> /etc/profile.d/zookeeper.sh \
      && mkdir -p ${ZOOKPR_DAT} \
      && mkdir -p ${ZOOKPR_LINK}/logs \
      && chown -R zookeeper:zookeeper ${ZOOKPR_DAT} \
      && chown -R zookeeper:zookeeper /opt/${ZOOKPR_VER} \
      && echo "dataDir=${ZOOKPR_DAT}" > ${ZOOKPR_LINK}/conf/zoo.cfg \
      && echo "maxClientCnxns=0" >> ${ZOOKPR_LINK}/conf/zoo.cfg \
      && echo "clientPort=2181" >> ${ZOOKPR_LINK}/conf/zoo.cfg
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
  if [ ! -e ${KAFKA_LINK} ]; then
    echo "Kafka installation..."
    KAFKA_VER=0.10.1.1
    wget http://ftp.heanet.ie/mirrors/www.apache.org/dist/kafka/${KAFKA_VER}/kafka_2.11-${KAFKA_VER}.tgz -q -P /tmp/ \
      && tar zxf /tmp/kafka_2.11-${KAFKA_VER}.tgz -C /opt/ \
      && ln -f -s /opt/kafka_2.11-${KAFKA_VER} ${KAFKA_LINK} \
      && echo "PATH=\\${PATH}:${KAFKA_LINK}/bin" > /etc/profile.d/kafka.sh \
      && mkdir -p ${KAFKA_LINK}/logs \
      && chown -R kafka:kafka /opt/kafka_2.11-${KAFKA_VER}
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
  sudo -E -u \\${USER} JMX_PORT=\\${KAFKA_JMX_PORT} \\${KAFKA_HOME}/bin/kafka-server-start.sh -daemon \\${KAFKA_CONFIG}
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
  for topic in clickstream transactions; do
    ${KAFKA_LINK}/bin/kafka-topics.sh \
      --create \
      --zookeeper localhost:2181 \
      --topic ${topic} \
      --partitions 1 \
      --replication-factor 1 &> /dev/null
  done

SCRIPT

# Info
$information = <<SCRIPT
    ip=$(ifconfig eth1 | awk -v host=$(hostname) '/inet addr/ {print substr($2,6)}')
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
    echo "kafka-console-consumer.sh --bootstrap-server localhost:9092 --blacklist none"
    echo "kafka-console-producer.sh --broker-list $ip:9092 --topic transactions"
    echo "> test event"
    echo " "
SCRIPT

Vagrant.configure(2) do |config|

  config.vm.box = "boxcutter/centos66"
  config.vm.hostname = "kafka.instance.com"
  config.vm.network :public_network, :mac => "0800DEADBEEF"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "kafka-node"
    vb.cpus = 2
    vb.memory = 4096
    vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "100"]
  end

  config.vm.provision :shell, :name => "sysconfig", :inline => $sysconfig
  config.vm.provision :shell, :name => "javajdk", :inline => $javajdk
  config.vm.provision :shell, :name => "zookeeper", :inline => $zookeeper
  config.vm.provision :shell, :name => "kafka", :inline => $kafka
  config.vm.provision :shell, :name => "information", :inline => $information

end
