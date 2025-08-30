# hadoop_setup
## 1) System prep & Java
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get -y install openjdk-11-jdk curl wget ssh pdsh
java -version

Set env vars (add to your shell):

echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> ~/.bashrc
source ~/.bashrc

export HADOOP_SHELL_EXECUTION_METHOD=ssh
sudo apt-get remove --purge -y pdsh

## 2) Passwordless SSH to localhost (Hadoop scripts use SSH)
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
First trust prompt:
ssh -o StrictHostKeyChecking=accept-new localhost 'echo ok'

## 3) Install Hadoop (e.g., 3.3.6)

You can swap the version if you prefer. 3.3.x works well with JDK 11.

HADOOP_VERSION=3.3.6
cd /tmp
wget https://downloads.apache.org/hadoop/common/hadoop-$HADOOP_VERSION/hadoop-$HADOOP_VERSION.tar.gz
sudo mkdir -p /opt
sudo tar -xzf hadoop-$HADOOP_VERSION.tar.gz -C /opt
sudo ln -s /opt/hadoop-$HADOOP_VERSION /opt/hadoop
sudo chown -R ubuntu:ubuntu /opt/hadoop-$HADOOP_VERSION /opt/hadoop

Add Hadoop to PATH:

cat <<'EOF' >> ~/.bashrc
export HADOOP_HOME=/opt/hadoop
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
EOF
source ~/.bashrc

Tell Hadoop where Java is:
sed -i 's|^# export JAVA_HOME=.*|export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64|' $HADOOP_HOME/etc/hadoop/hadoop-env.sh


## 4) Configure core HDFS/YARN/MapReduce

Create data dirs:

mkdir -p ~/hadoop-data/nn ~/hadoop-data/dn ~/hadoop-tmp
core-site.xml:
cat > $HADOOP_HOME/etc/hadoop/core-site.xml <<'XML'
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/home/ubuntu/hadoop-tmp</value>
  </property>
</configuration>
XML


hdfs-site.xml:
cat > $HADOOP_HOME/etc/hadoop/hdfs-site.xml <<'XML'
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/home/ubuntu/hadoop-data/nn</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/home/ubuntu/hadoop-data/dn</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address</name>
    <value>localhost:9000</value>
  </property>
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.namenode.http-address</name>
    <value>0.0.0.0:9870</value>
  </property>
  <property>
    <name>dfs.datanode.http.address</name>
    <value>0.0.0.0:9864</value>
  </property>
</configuration>
XML


mapred-site.xml:
cp $HADOOP_HOME/etc/hadoop/mapred-site.xml.template $HADOOP_HOME/etc/hadoop/mapred-site.xml
cat > $HADOOP_HOME/etc/hadoop/mapred-site.xml <<'XML'
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>localhost:10020</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>0.0.0.0:19888</value>
  </property>
</configuration>
XML


yarn-site.xml:
cat > $HADOOP_HOME/etc/hadoop/yarn-site.xml <<'XML'
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>localhost</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>0.0.0.0:8088</value>
  </property>
</configuration>
XML



## 5) Format HDFS & start daemons
hdfs namenode -format -force
start-dfs.sh
start-yarn.sh
(Optional) MapReduce HistoryServer:
mapred --daemon start historyserver

Verify processes:
jps

Create your HDFS user dir:
hdfs dfs -mkdir -p /user/ubuntu
hdfs dfs -chown ubuntu:ubuntu /user/ubuntu


