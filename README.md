Kafka - how to install and play with it
==========================================


```
https://github.com/christianb93/kafka

sudo apt update 
sudo apt install -y \
  libvirt-daemon \
  libvirt-clients \
  virt-manager \
  python3-pip \
  python3-libvirt \
  vagrant \
  vagrant-libvirt \
  git
sudo adduser $(id -un) libvirt
sudo adduser $(id -un) kvm

sudo apt install -y ansible aptitude dh-python

# python3.11-venv
sudo aptitude search python|cut -c4-|awk '{print $1}'|grep -E "^python.*venv$"|xargs sudo apt install -y

# openjdk
openjdk=$(sudo aptitude search openjdk|cut -c4-|awk '{print $1}'|grep -E "^openjdk.*jdk$"|head -1)
echo $openjdk
sudo apt install -y $openjdk 

# libvirt module
sudo aptitude search libvirt|cut -c4-|awk '{print $1}'|grep -E "^libvirt"|grep -Ev "(libvirt-daemon-system-systemd)"|xargs sudo apt install -y


python3 -m venv env
source ~/env/bin/activate
pip3 install ansible lxml pyopenssl

wget http://mirror.cc.columbia.edu/pub/software/apache/kafka/2.4.1/kafka_2.13-2.4.1.tgz
tar xvf kafka_2.13-2.4.1.tgz
mv kafka_2.13-2.4.1 kafka

git clone https://github.com/christianb93/kafka.git
cd kafka

kafkatgz=$(curl --silent https://kafka.apache.org/downloads | html2text |grep -E Scala.*kafka.*gz|tr ' ' "\n"|grep kafka|head -3|sort|tail -1)
echo $kafkatgz
URL=$(curl --silent https://kafka.apache.org/downloads|grep $kafkatgz|tr '"' "\n"|grep -E $kafkatgz$)
echo $URL
wget $URL
tar xzf $kafkatgz
mv ${kafkatgz%.tgz} kafka

virsh net-define kafka-private-network.xml
vagrant box list | cut -f 1 -d ' ' | xargs -L 1 vagrant box remove -f
vagrant up
source ~/env/bin/activate
pip install ansible libvirt-python
LC_ALL=C.UTF-8 LANG=C.UTF-8 ansible-playbook site.yaml

```


# Running the installation

First, you need to make sure that you have Ansible, Vagrant and the vagrant-libvirt plugin installed. Here are the instructions to do this on an Ubuntu based system.

```
sudo apt-get update 
sudo apt-get install \
  libvirt-daemon \
  libvirt-clients \
  virt-manager \
  python3-pip \
  python3-libvirt \
  vagrant \
  vagrant-libvirt \
  git \
  openjdk-8-jdk
sudo adduser $(id -un) libvirt
sudo adduser $(id -un) kvm
pip3 install ansible lxml pyopenssl
```

You also need to download the Kafka distribution to your local machine and unzip it there - first, to have a test client locally, and second because we use a Vagrant-synced folder to replicate this archive into the virtual machines, in order not to download it once for every broker

```
wget http://mirror.cc.columbia.edu/pub/software/apache/kafka/2.4.1/kafka_2.13-2.4.1.tgz
tar xvf kafka_2.13-2.4.1.tgz
mv kafka_2.13-2.4.1 kafka
```

Now you can start the installation simply by running

```
virsh net-define kafka-private-network.xml
vagrant up
ansible-playbook site.yaml
```

Note that we map the local working directory into the virtual machine, so that the installation can use the already downloaded Kafka tarball. This synchronization is done using rsync and apparently only once, so it is important that the downloaded tarball is in place **before** you run the playbook. 

Once the installation completes, it is time to run a few checks. First, let us verify that the ZooKeeper is running correctly on each node. For that purpose, SSH into the first node using `vagrant ssh broker1` and run

```
/usr/share/zookeeper/bin/zkServer.sh status
```

This should print out the configuration file used by ZooKeeper as well as the mode the node is in (follower or leader). Note that the ZooKeeper server is listening for client connections on port 2181 on all interfaces, i.e. if you should also be able to connect to the server from the lab PC as follows (keep this in mind if you are running in an exposed environment, you might want to setup a firewall if needed)

```
ip=$(virsh domifaddr kafka_broker1 \
  | grep "ipv4" \
  | awk '{ print $4 }' \
  | sed 's/\/24//')
echo srvr | nc $ip 2181
```

or, via the private network 

```
for i in {1..3}; do
    echo srvr | nc 10.100.0.1$i 2181
done
```

On each node, we should be able to reach each other node via its hostname on port 2181, i.e. on each node, you should be able to run

```
for i in {1..3}; do
    echo srvr | nc broker$i 2181
done
```

with the same results.  


Now let us see whether Kafka is running on each node. First, of course, you should check the status using `systemctl status kafka`. Then, we can see whether all brokers have registered themselves with ZooKeeper. To do this, run

```
sudo /usr/share/zookeeper/bin/zkCli.sh -server broker1:2181 ls /brokers/ids
```

on any of the broker nodes. You should get a list with the broker ids of the cluster, i.e. usually `[1,2,3]`. Next, log into one of the brokers and try to create a topic.

```
/opt/kafka/kafka_2.13-2.4.1/bin/kafka-topics.sh \
  --create \
  --bootstrap-server broker1:9092 \
  --replication-factor 3 \
  --partitions 2 \
  --topic test
```

This will create a topic called test with two partitions and a replication factor of three, i.e. each broker node will hold a copy of the log. When this command completes, you can check that corresponding directories (one for each partition) have been created in */tmp/kafka-logs* on every node.

Let us now try to write a message into this topic. Again, we run this on the broker:

```
/opt/kafka/kafka_2.13-2.4.1/bin/kafka-console-producer.sh \
  --broker-list broker1:9092 \
  --topic test
```

Enter some text and hit Ctrl-D. Then, on some other node, run

```
/opt/kafka/kafka_2.13-2.4.1/bin/kafka-console-consumer.sh \
  --bootstrap-server broker1:9092 \
  --from-beginning \
  --topic test
```

You should now see what you typed.

# Securing Kafka broker via TLS

In our setup, each Kafka broker will listen on the private interface with a PLAINTEXT listener, i.e. an unsecured listener. Of course, we can also add an additional listener on the public interface, so that we can reach the Kafka broker from a public network (in our setup using KVM, the "public" network is of course technically also just a local Linux bridge, but in a cloud setup, this will be different). To achieve this, we need to 

* create a public / private key pair for each broker
* create a certificate for each broker and sign it
* create keystore for the broker, holding the signed certificate and the keys
* create a truststore for the client, containing the CA used to sign the server certificate
* create a key pair and certificate for the client, bundle this in a keystore and make it available to the client
* create a properties file for the client containing the TLS configuration

Once this has been done, you can now run the consumer locally and connect via the SSL listener on the public interface:

```
kafka/bin/kafka-console-consumer.sh   \
  --bootstrap-server $(./python/getBrokerURL.py)   \
  --consumer.config .state/client_ssl_config.properties \
  --from-beginning \
  --topic test 
```

# Using the Python client

To be able to use the Python Kafka client, you will have to install the required packages on your local PC. Assuming that you have a working Python3 installation including pip3, simply run

```
pip3 install kafka-python
```

on your lab PC. For these notes, I have used version 2.0.1, so in case you want to use the exact same version, replace this by

```
pip3 install kafka-python==2.0.1
```

You can check which version you have by running `cat ~/.local/lib/python3*/site-packages/kafka/version.py`. Now you should be able to run the scripts in the python subdirectory. To simply create 10 messages for the test topic that we have created above, run (from the root of the repository)

```
python3 python/producer.py
```

On the broker node, you can now verify that this works by running

```
/opt/kafka/kafka_2.13-2.4.1/bin/kafka-dump-log.sh \
  --print-data-log \
  --files /opt/kafka/logs/test-0/00000000000000000000.log
```

which will print the first segment of partition 0 of the test partition. Finally, you can test our consumer by running

```
python3 python/consumer.py
```

The consumer has a few flags which are useful for testing.

* *--no_commit* - do not commit any messages, neither manually nor automatically
* *--disable_auto_commit* - disable automated commits and commit explictly after each batch of messages
* *--reset* - do not read any messages, but seek all partitions of the test topic to the beginning and commit these offsets
* *--max_poll_records* - number of records in a batch, default is one


