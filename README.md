# HA HBase Deployment

## Preparation Before Deploy

### Setting up python 2.7

```commandline
wget https://www.python.org/ftp/python/2.7.9/Python-2.7.9.tar.xz
tar xf Python-2.7.9.tar.xz 
cd Python-2.7.9
./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
cd /usr/bin
ls python*
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2.7 1
python --version
```
### Install pip
```commandline
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py 
python get-pip.py --user 
```

### Installing Ansible
```commandline
wget https://releases.ansible.com/ansible/ansible-2.9.27.tar.gz
tar -xf ansible-2.9.27.tar.gz 
pushd ansible-2.9.27/ 
python setup.py build(use python 2)
sudo apt install python3-pip(we need pip3)
sudo python3 setup.py install
ansible --version
```

### Preparing 4 instances and setting up ssh connections

My 4 instances:
- 10.182.0.24
- 10.182.0.25
- 10.182.0.26
- 10.182.0.27

Setting up connections:
```commandline
ssh-keygen -t rsa -b 3072 
cat ~/.ssh/id_rsa.pub
cat <<EOF >> ~/.ssh/authorized_keys 
>ssh-rsa xyangshen99@10.182.0.24
>ssh-rsa xyangshen99@10.182.0.25
>ssh-rsa xyangshen99@10.182.0.26
>ssh-rsa xyangshen99@10.182.0.27
>EOF
```
Then go to Google Cloud -> Settings -> Metadata, paste what you get under “cat ~/.ssh/id_rsa.pub” to SSH KEYS.

Then at all instances and ansible host:
```commandline
sudo nano /etc/ssh/ssh_config
```
Change 
```commandline
Host * StrictHostKeyChecking no
```
You can verify connections by:
```commandline
ssh xyangshen99@10.182.0.24
ssh xyangshen99@10.182.0.25
ssh xyangshen99@10.182.0.26
ssh xyangshen99@10.182.0.27
```

### Getting Packages ready
- Go to each instance and download java
 ```commandline
ssh xyangshen99@10.182.0.24
ssh xyangshen99@10.182.0.25
ssh xyangshen99@10.182.0.26
ssh xyangshen99@10.182.0.27
```
At each instance download jdk8 and jdk17 and put them in the right place:
 ```commandline
cd /opt
sudo apt update
sudo apt-get install openjdk-8-jdk
sudo mv /usr/lib/jvm/java-8-openjdk-amd64 /opt
sudo mv java-8-openjdk-amd64 jdk8

sudo apt-get install openjdk-17-jdk
sudo mv /usr/lib/jvm/java-17-openjdk-amd64 /opt
sudo mv java-17-openjdk-amd64 jdk17

sudo apt-get install libssl-dev
```

Then go back to ansible host, get packages:
 ```commandline
mkdir repo/hadoop

# Zookeeper:
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz

# HBase:
wget http://archive.apache.org/dist/hbase/2.4.11/hbase-2.4.11-bin.tar.gz

# HBase server: 
wget https://repo1.maven.org/maven2/org/apache/hbase/hbase-server/2.4.11/hbase-server-2.4.11.jar

# Hadoop:
wget https://downloads.apache.org/hadoop/core/hadoop-3.2.3/hadoop-3.2.3.tar.gz
```

## Deploy HBase
### 1. Sync-hosts:
 ```commandline
ansible-playbook book/sync-host.yml
```
### 2. Install hadoop:
 ```commandline
sudo apt-get update
sudo apt-get install python3-jmespath

ansible-playbook book/install-hadoop.yml
```
### 3. Config and start zookeeper:
After successfully run install-hadoop.yml, go to hosts, clear all newborn nodes.
 ```commandline
ansible-playbook book/config-zk.yml
```
Zookeepers will be running, verify by:
 ```commandline
ansible 'hadoop_nodes' -m shell -a 'sudo -- bash -c ". /etc/profile && jps"'
```
You can also go to one zookeeper instance to check if 2181 is listening
 ```commandline
sudo lsof -i -P -n | grep LISTEN
```
### 4. Config and start hadoop:
 ```commandline
ansible-playbook book/config-hadoop.yml
```
First go to each journalnode instances and start journalnodes:
 ```commandline
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs --daemon start journalnode"
```
Go to the first namenodes and start namenode:
 ```commandline
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs namenode -format"
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs --daemon start namenode"
```
Then format zookeeper(at the first namenode)
 ```commandline
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs zkfc -formatZK"
```
Confirm above steps by:
 ```commandline
sudo -- bash -c ". /etc/profile && jps"
```
Then go to the second namenode: hadoop2, start this namenode in standby mode
 ```commandline
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs namenode -bootstrapStandby"
sudo -- bash -c ". /etc/profile && /opt/hadoop/bin/hdfs --daemon start namenode"
sudo -- bash -c ". /etc/profile && jps"
```
Then go back the the first namenode, change the first namenode to active mode
```commandline
sudo -- bash -c ". /etc/profile && hdfs haadmin -transitionToActive nn1 --forcemanual"
sudo -- bash -c ". /etc/profile && hdfs haadmin -getServiceState nn1"
sudo -- bash -c ". /etc/profile && hdfs haadmin -getServiceState nn2"
```
After starting journalnodes and namenodes, go to ansible host and start datanodes:
```commandline
ansible datanodes -m shell -a 'sudo -- bash -c ". /etc/profile && nohup hdfs --daemon start datanode"' 
ansible datanodes -m shell -a 'sudo -- bash -c ". /etc/profile && jps | grep DataNode"'
```
Then try to restart the whole system:
```commandline
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile && export HDFS_NAMENODE_USER=xyangshen99 && export HDFS_DATANODE_USER=xyangshen99 && export HDFS_JOURNALNODE_USER=xyangshen99 && export HDFS_ZKFC_USER=xyangshen99 && stop-dfs.sh"'
```
Then go to each hadoop nodes to delete directory /data/hadoop/journal/my-hdfs

Then establish connection between nn1 and nn2

At nn1: 
```commandline
ssh-copy-id xyangshen99@10.182.0.25 # (my nn2 instance)
```
At nn2
```commandline
ssh-copy-id xyangshen99@10.182.0.24 # (my nn1 instance)
```
Then start 
```commandline
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile && export HDFS_NAMENODE_USER=xyangshen99 && export HDFS_DATANODE_USER=xyangshen99 && export HDFS_JOURNALNODE_USER=xyangshen99 && export HDFS_ZKFC_USER=xyangshen99 && start-dfs.sh"' 
```
Check if namenode active, here the namenodes will automatically one in active one in standby, verify by:
```commandline
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile && hdfs haadmin -getServiceState nn1"' 
ansible 'namenodes[1:]' -m shell -a 'sudo -- bash -c ". /etc/profile && hdfs haadmin -getServiceState nn2"'
```
### 5. Start Yarn:
```commandline
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile && export YARN_RESOURCEMANAGER_USER=xyangshen99 && export YARN_NODEMANAGER_USER=xyangshen99 && start-yarn.sh"'
ansible 'hadoop_nodes' -m shell -a 'sudo -- bash -c ". /etc/profile && jps | grep Manager"'
```
- if you ever encountered error like can’t write to a directory, just ssh to that instance and change writing permission by
```commandline
sudo chmod 777 /dir
```
Finally, check all node status:
```commandline
ansible 'hadoop_nodes' -m shell -a 'sudo -- bash -c ". /etc/profile && jps"'
```
### 5. Config and start Hbase:
First go to each Hbase instance
```commandline
cd /opt/hbase/conf
nano hbase-env.sh
```
We unmark 
```commandline
export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP="true"
```
Go to ansible host:
```commandline
ansible-playbook book/config-hbase.yml
```
Then at Ansible host, establish all connections
```commandline
ssh-copy-id xyangshen99@10.182.0.24
ssh-copy-id xyangshen99@10.182.0.25
ssh-copy-id xyangshen99@10.182.0.26
ssh-copy-id xyangshen99@10.182.0.27
```
Then go to HMaster
```commandline
sudo -- bash -c ". /etc/profile &&nohup start-hbase.sh"
```
Then we have our HBase started. Then
```commandline
cd /opt/hadoop/bin
./hadoop fs -ls /hbase
```
By this you can see that HBase has its data written into HDFS.