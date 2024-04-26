# ELK STACK SETUP IN EC2

## Introduction
The Elastic Stack — formerly known as the ELK Stack — is a collection of open-source software produced by Elastic which allows you to search, analyze, and visualize logs generated from any source in any format, a practice known as centralized logging. Centralized logging can be useful when attempting to identify problems with your servers or applications as it allows you to search through all of your logs in a single place. It’s also useful because it allows you to identify issues that span multiple servers by correlating their logs during a specific time frame.

The Elastic Stack has four main components:
- **Elasticsearch**: a distributed RESTful search engine which stores all of the collected data.
- **Logstash**: the data processing component of the Elastic Stack which sends incoming data to Elasticsearch.
- **Kibana**: a web interface for searching and visualizing logs.
- **Beats**: lightweight, single-purpose data shippers that can send data from hundreds or thousands of machines to either Logstash or Elasticsearch.

## Prerequisites
To complete this tutorial, you will need the following:
- An Ubuntu 22.04 server with 4GB RAM and 2 CPUs set up with a non-root sudo user. You can achieve this by following the [Initial Server Setup with Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04).
- OpenJDK 11 installed. See the section [Installing the Default JRE/JDK](https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-22-04) in our guide How To Install Java with Apt on Ubuntu 22.04 to set this up.
- Nginx installed on your server, which we will configure later in this guide as a reverse proxy for Kibana. Follow our guide on [How to Install Nginx on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04) to set this up.

## Step 1 — Installing and Configuring Elasticsearch

```bash
$ curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch |sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
$ echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
$ sudo apt update
$ sudo apt install elasticsearch
$ sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: localhost
```

```bash
$ sudo systemctl start elasticsearch
$ sudo systemctl enable elasticsearch
$ curl -X GET "localhost:9200"
```
```
Output
{
  "name" : "Elasticsearch",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "n8Qu5CjWSmyIXBzRXK-j4A",
  "version" : {
    "number" : "7.17.2",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "de7261de50d90919ae53b0eff9413fd7e5307301",
    "build_date" : "2022-03-28T15:12:21.446567561Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## Step 2 — Installing and Configuring the Kibana Dashboard

```bash
$ sudo apt install kibana
$ sudo systemctl enable kibana
$ sudo systemctl start kibana
$ echo "kibanaadmin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
```
upon above command it will ask for password here you have to password and then verify
## Installing Nginx

```bash
$ sudo apt update
$ sudo apt install nginx
```

## Adjusting the Firewall

```bash
$ sudo ufw app list
```
```
Output
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```
As demonstrated by the output, there are three profiles available for Nginx:
- **Nginx Full:** This profile opens both port 80 (normal, unencrypted web traffic) and port 443 (TLS/SSL encrypted traffic)
- **Nginx HTTP:** This profile opens only port 80 (normal, unencrypted web traffic)
- **Nginx HTTPS:** This profile opens only port 443 (TLS/SSL encrypted traffic)
It is recommended that you enable the most restrictive profile that will still allow the traffic you’ve configured. Right now, we will only need to allow traffic on port 80.
```bash
$ sudo ufw allow 'Nginx HTTP'
$ sudo ufw status
```
```
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx HTTP                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
```
## Checking your Web Server

```bash
$ systemctl status nginx
```
```
Output
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-04-20 16:08:19 UTC; 3 days ago
     Docs: man:nginx(8)
 Main PID: 2369 (nginx)
    Tasks: 2 (limit: 1153)
   Memory: 3.5M
   CGroup: /system.slice/nginx.service
           ├─2369 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─2380 nginx: worker process
```
When you have your server’s IP address, enter it into your browser’s address bar:

`http://<ec2-ip>`

You should receive the default Nginx landing page:
```
Welcome to nginx! If you see this page, the nginx web server is successfully installed and working. Further configuration is required. For online documentation and support please refer to nginx.org. Commercial support is available at nginx.com. Thank you for using nginx. 
```
If you are on this page, your server is running correctly and is ready to be managed.

**Note:** check security inbound rule of ec2 if not worked properly
```bash
$ sudo nano /etc/nginx/sites-available/<ec2-ip>
```
you may have already created this file and populated it with some content. In that case, delete all the existing content in the file before adding the following: 

```
server {
    listen 80;

    server_name your_domain;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
```bash
$ sudo ln -s /etc/nginx/sites-available/<ec2-ip> /etc/nginx/sites-enabled/<ec2-ip>
$ sudo nginx -t
$ sudo systemctl reload nginx
$ sudo ufw allow 'Nginx Full'
```
**Note:** If you followed the prerequisite Nginx tutorial, you may have created a UFW rule allowing the Nginx HTTP profile through the firewall. Because the Nginx Full profile allows both HTTP and HTTPS traffic through the firewall, you can safely delete the rule you created in the prerequisite tutorial. Do so with the following command:

```bash
$ sudo ufw delete allow 'Nginx HTTP'
```
Kibana is now accessible via your FQDN or the public IP address of your Elastic Stack server. You can check the Kibana server’s status page by navigating to the following address and entering your login credentials when prompted:
```
http://<ec2-ip>/status
```
## Step 3 — Installing and Configuring Logstash

Although it’s possible for Beats to send data directly to the Elasticsearch database, it is common to use Logstash to process the data. This will allow you more flexibility to collect data from different sources, transform it into a common format, and export it to another database.

```bash
$ sudo apt install logstash
$ sudo nano /etc/logstash/conf.d/02-beats-input.conf
```

```conf
input {
  beats {
    port => 5044
  }
}
```

```bash
$ sudo nano /etc/logstash/conf.d/30-elasticsearch-output.conf
```

```conf
output {
  if [@metadata][pipeline] {
        elasticsearch {
        hosts => ["localhost:9200"]
        manage_template => false
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        pipeline => "%{[@metadata][pipeline]}"
        }
  } else {
        elasticsearch {
        hosts => ["localhost:9200"]
        manage_template => false
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        }
  }
}
```

```bash
$ sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t

$ sudo systemctl start logstash

$ sudo systemctl enable logstash
```
## Step 4 — Installing and Configuring Filebeat

The Elastic Stack uses several lightweight data shippers called Beats to collect data from various sources and transport them to Logstash or Elasticsearch. Here are the Beats that are currently available from Elastic:
- **Filebeat:** collects and ships log files.
- **Metricbeat:** collects metrics from your systems and services.
- **Packetbeat:** collects and analyzes network data.
- **Winlogbeat:** collects Windows event logs.
- **Auditbeat:** collects Linux audit framework data and monitors file integrity.
- **Heartbeat:** monitors services for their availability with active probing.

```bash
$ sudo apt install filebeat
$ sudo nano /etc/filebeat/filebeat.yml
```
Filebeat supports numerous outputs, but you’ll usually only send events directly to Elasticsearch or to Logstash for additional processing. In this tutorial, we’ll use Logstash to perform additional processing on the data collected by Filebeat. Filebeat will not need to send any data directly to Elasticsearch, so let’s disable that output. To do so, find the output.elasticsearch section and comment out the following lines by preceding them with a `#`:
```
...
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]
...
```
Then, configure the output.logstash section. Uncomment the lines output. 
- logstash: and hosts: `["localhost:5044"]` 
by removing the `#`. This will configure Filebeat to connect to Logstash on your Elastic Stack server at port `5044`, the port for which we specified a Logstash input earlier:

```yaml
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
```

```bash
$ sudo filebeat modules enable system
$ sudo filebeat modules list
```
```bash
Output
Enabled:
system

Disabled:
apache2
auditd
elasticsearch
icinga
iis
kafka
kibana
logstash
mongodb
mysql
nginx
osquery
postgresql
redis
traefik
...
```
```bash
$ sudo filebeat setup --pipelines --modules system
$ sudo filebeat setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'
```
```
Output
Index setup finished.
```
```bash
$ sudo filebeat setup -E output.logstash.enabled=false -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601
```
After a few minutes, you should receive output similar to this:
```
Output
Overwriting ILM policy is disabled. Set `setup.ilm.overwrite:true` for enabling.

Index setup finished.
Loading dashboards (Kibana must be running and reachable)
Loaded dashboards
Setting up ML using setup --machine-learning is going to be removed in 8.0.0. Please use the ML app instead.
See more: https://www.elastic.co/guide/en/elastic-stack-overview/current/xpack-ml.html
Loaded machine learning job configurations
Loaded Ingest pipelines
```
```bash
$ sudo systemctl start filebeat
$ sudo systemctl enable filebeat
$ curl -XGET 'http://localhost:9200/filebeat-*/_search?pretty'
```
```Output
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4040,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "filebeat-7.17.2-2022.04.18",
        "_type" : "_doc",
        "_id" : "YhwePoAB2RlwU5YB6yfP",
        "_score" : 1.0,
        "_source" : {
          "cloud" : {
            "instance" : {
              "id" : "294355569"
            },
            "provider" : "digitalocean",
            "service" : {
              "name" : "Droplets"
            },
            "region" : "tor1"
          },
          "@timestamp" : "2022-04-17T04:42:06.000Z",
          "agent" : {
            "hostname" : "elasticsearch",
            "name" : "elasticsearch",
            "id" : "b47ca399-e6ed-40fb-ae81-a2f2d36461e6",
            "ephemeral_id" : "af206986-f3e3-4b65-b058-7455434f0cac",
            "type" : "filebeat",
            "version" : "7.17.2"
          },
```

Now you can access the dashboard on URL `<ec2-ip>`
