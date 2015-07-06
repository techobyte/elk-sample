# Tutorial for Elasticsearch, Logstash & Kibana
A step-by-step setup instructions [Elasticsearch], [Logstash] & [Kibana].

## Single Server Stack/Architecture
### Pre-requisite
* AWS a/c
* JAVA 7
* Ubuntu 14.04
* 4 GB RAM
* 2 CPUs

### Installing Elastic Search
#### Using apt-get
* Add Elasticsearch sourcelist
```sh
$ wget -O - http://packages.elasticsearch.org/GPG-KEY-elasticsearch | sudo apt-key add -
$ echo 'deb http://packages.elasticsearch.org/elasticsearch/1.6/debian stable main' | sudo tee /etc/apt/sources.list.d/elasticsearch.list
$ sudo apt-get update
```
* Install
```sh
$ sudo apt-get -y install elasticsearch=1.6.0
```
* Configure: Restrict outside access
```sh
$ sudo nano /etc/elasticsearch/elasticsearch.yml
network.host: localhost
```
* Start
```sh
$ sudo service elasticsearch restart
```
* Start on boot
```sh
$ sudo update-rc.d elasticsearch defaults 95 10
```
* Test ```curl -X GET http://localhost:9200```

#### Using tar.gz
* Untar it to /opt
```sh
$ tar -xzf elasticsearch-1.6.0.tar.gz -O /opt
```
* Configure: Restrict outside access
```sh
$ sudo nano /opt/elasticsearch-1.6.0/config/elasticsearch.yml
network.host: localhost
```
* Configure: Adding plugins
```sh
$ sudo . /opt/elasticsearch-1.6.0/bin/plugin -install mobz/elasticsearch-head
$ sudo . /opt/elasticsearch-1.6.0/bin/plugin -install lukas-vlcek/bigdesk
```
* Start
```sh
$ sudo nohup . /opt/elasticsearch-1.6.0/bin/elasticsearch start
```

## Multi Server Stack/Architecture
### Pre-requisite
* AWS a/c
* 4 EC2 Instances with Ubuntu 14.04
* JAVA 7 on all
* Load Balancer

### Architecture
```sh
                     +-----+ +-----+         
                     |     | |     |         
                     | ES1 | | ES2 |         
                     +-----+ +-----+  ....     
____                    ^       ^            
|==|                     \     /              
|=_|__      +-----+      ++---++      +-----+
|_|==| +--> |     | +--> |     | +--> |     |
  |==|      | LS  |      | ELB |      | KIB |
  |__|      +-----+      +-----+      +-----+
 LOGs       Logstash    LoadBalancer   Kibana
```

### Installing Elasticsearch
* Install same as single node
* Configure for inter-communication
```sh
$ sudo . /opt/elasticsearch-1.6.0/bin/plugin install elasticsearch/elasticsearch-cloud-aws/2.6.0
$ sudo nano /opt/elasticsearch-1.6.0/config/elasticsearch.yml
cluster.name: elk-sample
cloud.aws.access_key: ACCESS_KEY_HERE
cloud.aws.secret_key: SECRET_KEY_HERE
cloud.aws.region: us-east-1
discovery.type: ec2
discovery.ec2.tag.Name: "elk-sample - Elasticsearch"
http.cors.enabled: true
http.cors.allow-origin: "*"
```
* Add all the Elasticsearch nodes into Loadbalancer and get the IP of load balancer [ELB_IP]

[Elasticsearch]:https://www.elastic.co/products/elasticsearch
[Logstash]:https://www.elastic.co/products/logstash
[Kibana]:https://www.elastic.co/products/kibana
