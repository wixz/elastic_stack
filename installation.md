## Installation of elastic SIEM

Started by creating a minimal Ubuntu 18.04.3 server installation with 4 virtual cores and 8gb of memory. This guide covers the core installation. Remember to activate access controls for Kibana/Elastic, harden your OS as well as implement SSL/TLS for all relevant traffic. Since "nano" is installed bu default by Ubuntu Minimal this is what I use, if you prefer vi/vim or emacs, just add the package you want in the Pre-req section. All IP's in this guide and in the configuration files represent my RFC1918 local subnet so these settings will need to be changed for your own needs.

Once installed, the landscape client was configured:

Copy the certificate to the local machine
```
sudo nano /etc/landscape/server.pem
```

Run the client registration (change the hostname.domain to you spec)
```
sudo landscape-config --computer-title "hostname.domain" --account-name standalone  --url https://hostname.domain/message system --ping-url http://hostname.domain/ping --ssl-public-key /etc/landscape/server.pem
```

The client is registered in the Landscape asset server and updated.

## Pre-req

Since we have installed an Ubuntu Minimal Server installation we have to start by install all the prerequisites we need for the complete installation to work.

```
sudo apt-get install openjdk-11-jdk gnupg2 net-tools htop -y
```


## Install of ElasticSearch

Since I'm using Ubuntu 18.04 through my environment I use the deb variant of the elastic stack installation variants.


```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```

Now update the repository and install the elasticsearch package
```
sudo apt update && sudo apt install elasticsearch -y
```

After the installation is complete, go to the `/etc/elasticsearch` directory and edit the configuration file `elasticsearch.yml`
```
sudo -i
cd /etc/elasticsearch/
nano elasticsearch.yml
```

Uncomment and configure
```
cluster.name: elastic-cluster
node.name: esnode01
bootstrap.memory_lock: true
network.host: 172.20.20.51
http.port: 9200
cluster.initial_master_nodes: ["esnode01"]
```

Now start the elasticsearch service and enable it to launch every time on system boot.
```
exit
systemctl enable elasticsearch
```

The elasticsearch is now up and running, check open ports using netstat and elasticsearch status using curl:
```
netstat -plntu
curl -XGET 'localhost:9200/?pretty'
```

## Install and configure Kibana

Install Kibana dashboard using apt.
```
sudo apt install kibana -y
```

Now go to the `/etc/kibana` directory and edit the configuration file `kibana.yml`.
```
cd /etc/kibana/
nano kibana.yml
```

Configure at least these lines:
```
server.port: 5601
server.host: "172.20.20.51"
elasticsearch.url: "http://172.20.20.51:9200"
```

Enable the service at boot and start
```
sudo systemctl enable kibana && sudo systemctl start kibana
```

## Installation and configuration of logstash

Install logstash using the apt command below.
```
sudo apt install logstash -y
```

Copy the configuration present in `/logstash/` to your local `/etc/logstash/conf.d/` on the logstash host.

Enable and start logstash
```
sudo systemctl enable logstash
sudo systemctl start logstash
```

Check that all ports are open using netstat again
```
netstat -plntu
```

And that all services are running as expected
```
sudo systemctl status kibana elasticsearch logstash
```

## Install Filebeat

Install Filebeat as local logcollector
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
```
Add this repo if you have a support license for your elastic stack
```
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```
Add this repo if you want to play with the open source variant
```
echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```
Update apt and install Filebeat
```
sudo apt-get update && sudo apt-get install filebeat
```

Modify `/etc/filebeat/filebeat.yml` to set the connection information

Filebeat have a lot of different modules you can enable. Enabling the module `iptables` on the filebeat collector will enable filebeat to receive and parse log structured according to the iptales format, which includes log received from Ubiquiti Unifi USG :)

Enable module iptables
```
sudo filebeat modules enable iptables
```

The setup command loads the Kibana dashboards. If the dashboards are already set up, omit this command.
```
sudo filebeat setup
sudo service filebeat start
```
#### Install Filebeat as logsender on every node

In my environment, filebeats is installed on every node to collect logdata from system and auditd. Repeat the installation procedures above.

When configuring filebeat on endpoints, use the conf file in `/filebeat/` and enable:
```
sudo filebeat modules enable system && sudo filebeat modules enable auditd
```

Test filebeat connections
```
sudo filebeat test output
```

## Install Metricbeat (optional)

Download and install the Public Signing Key, ATH (if missing) and define repo:
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```

Run apt-get update, and the repository is ready for use. For example, you can install Metricbeat by running:
```
sudo apt-get update && sudo apt-get install metricbeat
sudo update-rc.d metricbeat defaults 95 10
```
