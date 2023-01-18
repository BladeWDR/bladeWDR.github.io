+++
title = "How to install Graylog on Debian 11 (single node)"
date = "2023-01-17T23:00:46-05:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["debian", "ubuntu","graylog","logging","linux","tutorial"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

This was tested on 1/17/2023 on Debian 11 and Graylog 5.0
Using OpenSearch instead of ElasticSearch which is deprecated now.
This is for a single node... multiple nodes is something that I have yet to look into.


### Install prerequisites

#### MongoDB 

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv f5679a222c647c87527c2f8cb00a0bd1e2c63c11
echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/5.0 multiverse" |
sudo tee /etc/apt/sources.list.d/mongodb-org-5.x.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

##### Enable MongoDB on system startup

```
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl --type=service --state=active | grep mongod
```

#### OpenSearch
```
wget https://artifacts.opensearch.org/releases/bundle/opensearch/1.3.4/opensearch-1.3.4-linux-x64.tar.gz
```

Disable memory paging and swapping to improve performance.

```
sudo swapoff -a
```

Increase the number of memory maps available to OpenSearch.

```
# Edit the sysctl config file
sudo vi /etc/sysctl.conf

# Add a line to define the desired value
# or change the value if the key exists,
# and then save your changes.
vm.max_map_count=262144

# Reload the kernel parameters using sysctl
sudo sysctl -p

# Verify that the change was applied by checking the value
cat /proc/sys/vm/max_map_count
```

#### Create OpenSearch User
```
sudo adduser --system --disabled-password --disabled-login --home /var/empty --no-create-home --quiet --force-badname --group opensearch
```

#### Create directories and extract the OpenSearch archive you downloaded earlier

```
sudo mkdir -p /graylog/opensearch/data
sudo mkdir /var/log/opensearch

tar -xzvf opensearch-1.3.4-linux-x64.tar.gz
sudo mv opensearch-1.3.4/* /graylog/opensearch


#Set Permissions
sudo chown -R opensearch:opensearch /graylog/opensearch/
sudo chown -R opensearch:opensearch /var/log/opensearch
sudo chmod -R 2750 /graylog/opensearch/
sudo chmod -R 2750 /var/log/opensearch

#Create empty log file
sudo -u opensearch touch /var/log/opensearch/graylog.log
```

#### Create a system service for opensearch


```

sudo su

cat > /etc/systemd/system/opensearch.service <<EOF
[Unit]
Description=Opensearch
Documentation=https://opensearch.org/docs/latest
Requires=network.target remote-fs.target
After=network.target remote-fs.target
ConditionPathExists=/graylog/opensearch
ConditionPathExists=/graylog/opensearch/data
[Service]
Environment=OPENSEARCH_HOME=/graylog/opensearch
Environment=OPENSEARCH_PATH_CONF=/graylog/opensearch/config
ReadWritePaths=/var/log/opensearch
User=opensearch
Group=opensearch
WorkingDirectory=/graylog/opensearch
ExecStart=/graylog/opensearch/bin/opensearch
# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65535
# Specifies the maximum number of processes
LimitNPROC=4096
# Specifies the maximum size of virtual memory
LimitAS=infinity
# Specifies the maximum file size
LimitFSIZE=infinity
# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0
# SIGTERM signal is used to stop the Java process
KillSignal=SIGTERM
# Send the signal only to the JVM rather than its control group
KillMode=process
# Java process is never killed
SendSIGKILL=no
# When a JVM receives a SIGTERM signal it exits with code 143
SuccessExitStatus=143
# Allow a slow startup before the systemd notifier module kicks in to extend the timeout
TimeoutStartSec=180
[Install]
WantedBy=multi-user.target
EOF

```

#### Create opensearch configuration file

```
nano /graylog/opensearch/config/opensearch.yml
```

```
cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /graylog/opensearch/data
path.logs: /var/log/opensearch
network.host: ${HOSTNAME}
discovery.seed_hosts: ["SERVERNAME01", "SERVERNAME02", "SERVERNAME03"]
cluster.initial_master_nodes: ["SERVERNAME01", "SERVERNAME02", "SERVERNAME03"]
action.auto_create_index: false
plugins.security.disabled: true
```




#### Enable OpenSearch system service

```
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
```

### Install Graylog


```
wget https://packages.graylog2.org/repo/packages/graylog-5.0-repository_latest.deb
sudo dpkg -i graylog-5.0-repository_latest.deb
sudo apt-get update && sudo apt-get install graylog-server 
```

##### Generate a password secret and a root password.o

Keep these handy as we'll use them in the next step.

##### Password Secret
```
mkpassword -m sha-512 <password>
```

##### root password

```
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

### Edit the configuration file for Graylog

Enter the password secret and the root password into the relevant fields `/etc/graylog/server/server.conf`

Change the `http_bind_address` to the ip address or domain name of the server.


#### Enable Graylog to run at startup.

```
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
sudo systemctl --type=service --state=active | grep graylog
```

#### Connect to Graylog web interface

```
<ip-address>:9000
```

