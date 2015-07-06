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
  * Browse to http://localhost:9200/
  * Plugin Browse to http://localhost:9200/_plugin/head
  * Plugin Browse to http://localhost:9200/_plugin/bigdesk

#### Using tar.gz
* Untar it to /opt
```sh
$ tar -xzf elasticsearch-1.6.0.tar.gz -C /opt
```
* Configure: Restrict outside access
```sh
$ sudo nano /opt/elasticsearch-1.6.0/config/elasticsearch.yml
network.host: localhost
```
* Configure: Adding plugins
```sh
$ sudo /opt/elasticsearch-1.6.0/bin/plugin -install mobz/elasticsearch-head
$ sudo /opt/elasticsearch-1.6.0/bin/plugin -install lukas-vlcek/bigdesk
```
* Start
```sh
$ sudo nohup /opt/elasticsearch-1.6.0/bin/elasticsearch start
```
* Test ```curl -X GET http://localhost:9200```
  * Browse to http://localhost:9200/
  * Plugin Browse to http://localhost:9200/_plugin/head
  * Plugin Browse to http://localhost:9200/_plugin/bigdesk

### Installing Logstash
#### Using apt-get
* Adding source list
```sh
$ echo 'deb http://packages.elasticsearch.org/logstash/1.5/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash.list
$ sudo apt-get update
```
* Install
```sh
$ sudo apt-get install logstash
```
* Configure
```sh
$ sudo nano /etc/logstash/conf.d/logstash.conf
input {
  file {
    path => "/var/log/apache/access.log"
    type => "apache"
  }

  file {
    path => "/tmp/testing.log"
    type => "custom"
  }
}

output {
  elasticsearch {
    action => "index"
    host => "localhost"
    index => "logs"
    protocol => "http"
  }
}
```
* Start
```sh
$ sudo service logstash restart
```
#### Using tar.gz
* Extract
```sh
$ tar -xzf logstash-1.5.2.tar.gz -C /opt
```
* Configure
```sh
$ sudo nano /opt/logstash-1.5.2/bin/logstash.conf
input {
  file {
    path => "/var/log/apache/access.log"
    type => "apache"
  }

  file {
    path => "/tmp/testing.log"
    type => "custom"
  }
}

output {
  elasticsearch {
    action => "index"
    host => "localhost"
    index => "logs"
    protocol => "http"
  }
}
```
* Start
```sh
$ sudo /opt/logstash-1.5.2/bin/logstash -f logstash.conf
```

### Installing Kibana
* Download & Exract OS specifi Kibana to home directory
```sh
$ cd ~; wget https://download.elastic.co/kibana/kibana/kibana-4.1.1-linux-x64.tar.gz
$ tar -xzf kibana-*.tar.gz -C /opt
```
* Configure
```sh
$ sudo nano /opt/kibana-4*/config/kibana.yml
host: "localhost"
```
* Start
```sh
$ sudo /opt/kibana-4*/bin/kibana
```
* Run as a Service
```sh
$ cd /etc/init.d && sudo wget https://gist.githubusercontent.com/thisismitch/8b15ac909aed214ad04a/raw/bce61d85643c2dcdfbc2728c55a41dab444dca20/kibana4
$ sudo chmod +x /etc/init.d/kibana4
$ sudo update-rc.d kibana4 defaults 96 9
$ sudo service kibana4 start
```
===
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
____                                ^       ^            
|==|        +----+                   \     /              
|=_|__ +--> |LSF |      +-----+      ++---++      +-----+
|_|==|      +----+ +--> |     | +--> |     | +--> |     |
  |==| +--> +----+ +--> | LS  |      | ELB |      | KIB |
  |__|      |LSF |      +-----+      +-----+      +-----+
            +----+                                       
 LOGs     LogstashFwdr  Logstash    LoadBalancer   Kibana
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
* Add all the Elasticsearch nodes into Loadbalancer and get the IP of load balancer [>>**ELB_IP_URL**<<]
* Test ```curl -X GET http://ELB_IP_URL:9200```
  * Browse to http://ELB_IP_URL:9200/
  * Plugin Browse to http://ELB_IP_URL:9200/_plugin/head
  * Plugin Browse to http://ELB_IP_URL:9200/_plugin/bigdesk

### Installing Logstash
* Install same as single node
* Configure for communication to elasticsearch
```sh
...
output {
  elasticsearch {
    action => "index"
    host => ">>ELB_IP_URL<<"
    index => "logs"
    protocol => "http"
  }
}
```
* Configuring SSL
  * Generate SSL certs Option 1: (Hostname FQDN)

    Before creating a certificate, make sure you have A record for logstash server; ensure that client servers are able to resolve the hostname of the logstash server. If you do not have DNS, kindly add the host entry for logstash server; where 10.0.0.26 is the ip address of logstash server and mylogstash is the hostname of your logstash server.
 
    ```sh
    $ nano /etc/hosts
    10.0.0.26 mylogstash
    ```

    Create a SSl certificate.  
    Goto OpenSSL directory.  

    ```sh
    $ cd /etc/pki/tls
    ```

    Execute the following command to create a SSL certificate, replace with your real logstash server.  

    ```sh
    $ openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt -subj /CN=mylogstash
    ```
  * Generate SSL certs Option 2: (IP Address)

    Before creating a SSL certificate, we would require to an add ip address of logstash server to SubjectAltName in the OpenSSL config file.

    Goto “[ v3_ca ]” section and replace with your logstash server ip.  

    ```sh
    $ nano /etc/pki/tls/openssl.cnf
    subjectAltName = IP:10.0.0.26
    ```

    Goto OpenSSL directory.  

    ```sh
    $ cd /etc/pki/tls
    ```

    Execute the following command to create a SSL certificate.  

    ```sh
    $ openssl req -x509 -days 365 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
    ```
  * Add SSL to config

    ```sh
    $ sudo nano /etc/logstash/conf.d/logstash.conf
    input {
      lumberjack {
        port => 5050
        type => "logs"
        ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
        ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
      }
    }
    ```

### Installing Logstash-Forwarder
#### Using apt-get
* Ading source list
```sh
$ echo 'deb http://packages.elasticsearch.org/logstashforwarder/debian stable main' | sudo tee /etc/apt/sources.list.d/logstashforwarder.list
$ wget -O - http://packages.elasticsearch.org/GPG-KEY-elasticsearch | sudo apt-key add -
$ sudo apt-get update
```
* Install
```sh
$ sudo apt-get install logstash-forwarder
```
#### Using binaries
* Download OS specific binaries
* Install it

#### Configure the SSL

Logstash-forwader uses SSL certificate for validating logstash server identity, so copy the logstash-forwarder.crt that we created earlier from the logstash server to the client.
  
In the “network” section, mention the logstash server with port number and path to the logstash-forwarder certificate that you copied from logstash server. This section defines the logstash-forwarder to send a logs to logstash server “mylogstash” on port 5050 and client validates the server identity with the help of SSL certificate.
  
```sh
$ nano /etc/logstash-forwarder.conf
{
  "network": {
    "servers": [ "mylogstash:5050" ],
    "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt",
    "timeout": 15
  },
  "files": [
    {
      "paths": [
        "/var/log/syslog",
        "/var/log/auth.log"
      ],
      "fields": { "type": "syslog" }
    }, {
      "paths": [
        "/var/log/apache/httpd-*.log"
      ],
      "fields": { "type": "apache" }
    }
  ]
}
```

Restart service
```sh
$ sudo service logstash-forwarder restart
```
### Installing Kibana
* Install just like single node
* Configure with ELB_IP_URL
* Add Reverse proxy to Kibana

## References
* https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04
* http://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-install-elasticsearch-logstash-and-kibana-4-on-centos-7-rhel-7.html
[Elasticsearch]:https://www.elastic.co/products/elasticsearch
[Logstash]:https://www.elastic.co/products/logstash
[Kibana]:https://www.elastic.co/products/kibana
