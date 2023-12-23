## Install/Configure Monitoring of MySQL Using Gafana & Prometheus

Being a DBA its crucial to have monitoring tool to see whats happening in the database. There are various monitoring tools which are open source and paid. But here we will be explicitly using Grafana and Prometheus to monitoring MySQL.

Grafana labs give more performance visibility in comparison to others. It is an open-source metric analytics and visualization tool that enables developers to write plugins from scratch to integrate with several data sources.

Even if there is a failure, we can easily troubleshoot the issue with the available stats, like database connections, number of containers running and performance, number of bytes written and read, etc. This blog throws light on how to monitor a MySQL database using Grafana and Prometheus.

Steps to Enable MySQL Database Monitoring Using Prometheus and Grafana
* Install and configure Grafana
* Install and configure Prometheus
* Install a database exporter

<hr >

### Install and configure Grafana

The Grafana binaries are publicly available to download from [Grafana Binaries](https://grafana.com/grafana/download). We are using the latest binaries and directly install it. You can download locally first using wget and later install too.
```
[root@watchsrv ~]# yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-10.2.3-1.x86_64.rpm
Oracle Linux 8 BaseOS Latest (x86_64)           626 kB/s |  67 MB     01:49
Oracle Linux 8 Application Stream (x86_64)      1.1 MB/s |  53 MB     00:48
Latest Unbreakable Enterprise Kernel Release 6  2.2 MB/s |  83 MB     00:38
Last metadata expiration check: 0:00:09 ago on Sat 23 Dec 2023 05:39:08 PM +0545.
grafana-enterprise-10.2.3-1.x86_64.rpm          450 kB/s | 103 MB     03:55
Dependencies resolved.
================================================================================
 Package                  Architecture Version         Repository          Size
================================================================================
Installing:
 grafana-enterprise       x86_64       10.2.3-1        @commandline       103 M

Transaction Summary
================================================================================
Install  1 Package

Total size: 103 M
Installed size: 382 M
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                        1/1
  Installing       : grafana-enterprise-10.2.3-1.x86_64                     1/1
  Running scriptlet: grafana-enterprise-10.2.3-1.x86_64                     1/1
### NOT starting on installation, please execute the following statements to configure grafana to start automatically using systemd
 sudo /bin/systemctl daemon-reload
 sudo /bin/systemctl enable grafana-server.service
### You can start grafana-server by executing
 sudo /bin/systemctl start grafana-server.service

POSTTRANS: Running script

/sbin/ldconfig: /etc/ld.so.conf.d/kernel-5.4.17-2102.201.3.el8uek.x86_64.conf:6: hwcap directive ignored

  Verifying        : grafana-enterprise-10.2.3-1.x86_64                     1/1

Installed:
  grafana-enterprise-10.2.3-1.x86_64

Complete!
[root@watchsrv ~]#
```

With the installation of binaries, the grafana services won't have started as reported in log during rpm installation. Next we will do as instructed. 
```
[root@watchsrv ~]# /bin/systemctl daemon-reload
[root@watchsrv ~]# /bin/systemctl enable grafana-server.service
Synchronizing state of grafana-server.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable grafana-server
Created symlink /etc/systemd/system/multi-user.target.wants/grafana-server.service → /usr/lib/systemd/system/grafana-server.service.
[root@watchsrv ~]# /bin/systemctl start grafana-server.service
[root@watchsrv ~]# systemctl status grafana-server
● grafana-server.service - Grafana instance
   Loaded: loaded (/usr/lib/systemd/system/grafana-server.service; enabled; ven>
   Active: active (running) since Sat 2023-12-23 17:45:14 +0545; 9s ago
     Docs: http://docs.grafana.org
 Main PID: 3489 (grafana)
    Tasks: 7 (limit: 22960)
   Memory: 86.3M
   CGroup: /system.slice/grafana-server.service
           └─3489 /usr/share/grafana/bin/grafana server --config=/etc/grafana/g>

Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=http.server t=2023-1>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=ngalert.state.manage>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=ngalert.state.manage>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=ngalert.scheduler t=>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=ticker t=2023-12-23T>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=report t=2023-12-23T>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=ngalert.multiorg.ale>
Dec 23 17:45:14 watchsrv.localdomain grafana[3489]: logger=sqlstore.transaction>
Dec 23 17:45:16 watchsrv.localdomain grafana[3489]: logger=grafana.update.check>
Dec 23 17:45:17 watchsrv.localdomain grafana[3489]: logger=plugins.update.check>
[root@watchsrv ~]#
```

Now lets test if its running. Grafana uses 3000 port so obtaining the server ip we will hith the url as below. And the Default login and password of Grafana are: <strong>admin/admin</strong>, default location Grafana will log into: /var/log/Grafana.
<img src="imgs/grafana-initial.png" alt="Grafana Initial User Interface"> 

When you provide default password it may request you to change the password, which you can or skip it but i have changed it in my case and the initial dashboard look like below:
<img src="imgs/grafana-login.png" alt="Grafana Initial Dashboard"> 

<hr >

### Install and configure Prometheus

> - To configure Prometheus we will need to download the [Prometheus Binaries](https://prometheus.io/download/). We will be using Linux compatible binaries.

```
[root@watchsrv ~]# wget https://github.com/prometheus/prometheus/releases/download/v2.45.2/prometheus-2.45.2.linux-amd64.tar.gz
--2023-12-23 17:59:37--  https://github.com/prometheus/prometheus/releases/download/v2.45.2/prometheus-2.45.2.linux-amd64.tar.gz
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/6838921/6c7e9d0f-0deb-4961-b7b5-7b65f10dfa37?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20231223%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231223T121438Z&X-Amz-Expires=300&X-Amz-Signature=e440ccd24e594c05f5a913ddc24ebabd66561c6208c2f906b813505dc00bf02b&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=6838921&response-content-disposition=attachment%3B%20filename%3Dprometheus-2.45.2.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream [following]
--2023-12-23 17:59:38--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/6838921/6c7e9d0f-0deb-4961-b7b5-7b65f10dfa37?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20231223%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231223T121438Z&X-Amz-Expires=300&X-Amz-Signature=e440ccd24e594c05f5a913ddc24ebabd66561c6208c2f906b813505dc00bf02b&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=6838921&response-content-disposition=attachment%3B%20filename%3Dprometheus-2.45.2.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.111.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 92582580 (88M) [application/octet-stream]
Saving to: ‘prometheus-2.45.2.linux-amd64.tar.gz’

prometheus-2.45.2.l 100%[===================>]  88.29M   206KB/s    in 2m 10s

2023-12-23 18:01:50 (694 KB/s) - ‘prometheus-2.45.2.linux-amd64.tar.gz’ saved [92582580/92582580]

[root@watchsrv ~]#
```
> - Once the binaries are downloaded lets unzip it.

```
[root@watchsrv ~]# tar -xvf prometheus-2.45.2.linux-amd64.tar.gz
prometheus-2.45.2.linux-amd64/
prometheus-2.45.2.linux-amd64/NOTICE
prometheus-2.45.2.linux-amd64/LICENSE
prometheus-2.45.2.linux-amd64/prometheus
prometheus-2.45.2.linux-amd64/consoles/
prometheus-2.45.2.linux-amd64/consoles/prometheus-overview.html
prometheus-2.45.2.linux-amd64/consoles/node-cpu.html
prometheus-2.45.2.linux-amd64/consoles/node.html
prometheus-2.45.2.linux-amd64/consoles/prometheus.html
prometheus-2.45.2.linux-amd64/consoles/index.html.example
prometheus-2.45.2.linux-amd64/consoles/node-overview.html
prometheus-2.45.2.linux-amd64/consoles/node-disk.html
prometheus-2.45.2.linux-amd64/prometheus.yml
prometheus-2.45.2.linux-amd64/promtool
prometheus-2.45.2.linux-amd64/console_libraries/
prometheus-2.45.2.linux-amd64/console_libraries/menu.lib
prometheus-2.45.2.linux-amd64/console_libraries/prom.lib
[root@watchsrv ~]# ls -ltr | grep prome
drwxr-xr-x  4 1001  127      132 Dec 19 20:19 prometheus-2.45.2.linux-amd64
-rw-r--r--  1 root root 92582580 Dec 19 20:23 prometheus-2.45.2.linux-amd64.tar.gz
[root@watchsrv ~]#
```

> - Create a Prometheus System Group & User and directories
```
[root@watchsrv ~]# groupadd --system prometheus
[root@watchsrv ~]# useradd -s /sbin/nologin --system -g prometheus prometheus
[root@watchsrv ~]#
[root@watchsrv ~]# mkdir -pv /var/lib/prometheus
mkdir: created directory '/var/lib/prometheus'
[root@watchsrv ~]# mkdir -pv /etc/prometheus
mkdir: created directory '/etc/prometheus'
[root@watchsrv ~]# chown -R prometheus:prometheus /etc/prometheus
[root@watchsrv ~]# chown -R prometheus:prometheus /var/lib/prometheus
[root@watchsrv ~]#
```

> - Copy configuration files and set permission accordingly

```
[root@watchsrv prometheus-2.45.2.linux-amd64]# pwd
/root/prometheus-2.45.2.linux-amd64
[root@watchsrv prometheus-2.45.2.linux-amd64]#
[root@watchsrv prometheus-2.45.2.linux-amd64]# ls
console_libraries  LICENSE  prometheus      promtool
consoles           NOTICE   prometheus.yml
[root@watchsrv prometheus-2.45.2.linux-amd64]# mv prometheus promtool /usr/local/bin/
[root@watchsrv prometheus-2.45.2.linux-amd64]# cd
[root@watchsrv ~]#
[root@watchsrv ~]# chown prometheus:prometheus /usr/local/bin/prometheus
[root@watchsrv ~]# chown prometheus:prometheus /usr/local/bin/promtool
[root@watchsrv ~]# cp -r prometheus-2.45.2.linux-amd64/consoles /etc/prometheus
[root@watchsrv ~]# cp -r prometheus-2.45.2.linux-amd64/console_libraries /etc/prometheus
[root@watchsrv ~]# chown -R prometheus:prometheus /etc/prometheus/consoles
[root@watchsrv ~]# chown -R prometheus:prometheus /etc/prometheus/console_libraries
[root@watchsrv ~]#
```

> - For now copy the default configuration file for Prometheus
```
[root@watchsrv ~]# cp prometheus-2.45.2.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml
[root@watchsrv ~]# chown prometheus:prometheus /etc/prometheus/prometheus.yml
[root@watchsrv ~]#
```

> - We will be using the temaplate as it is without any change if needed we will change it in future
```
[root@watchsrv ~]# cat /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
[root@watchsrv ~]#
```

> - Create a Prometheus systemd Service Unit File

```
[root@watchsrv ~]# vi /etc/systemd/system/prometheus.service
[root@watchsrv ~]# cat /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Environment="GOMAXPROCS=1"
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
[root@watchsrv ~]#
```

> - Reload systemd Daemon & Start the Service
```
[root@watchsrv ~]# systemctl daemon-reload
[root@watchsrv ~]# systemctl status prometheus
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor pre>
   Active: inactive (dead)
     Docs: https://prometheus.io/docs/introduction/overview/
[root@watchsrv ~]# systemctl start prometheus
[root@watchsrv ~]# systemctl status prometheus
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor pre>
   Active: active (running) since Sat 2023-12-23 18:26:28 +0545; 6s ago
     Docs: https://prometheus.io/docs/introduction/overview/
 Main PID: 4661 (prometheus)
    Tasks: 6 (limit: 22960)
   Memory: 15.6M
   CGroup: /system.slice/prometheus.service
           └─4661 /usr/local/bin/prometheus --config.file=/etc/prometheus/prome>

Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.2>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.4>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.4>
Dec 23 18:26:28 watchsrv.localdomain prometheus[4661]: ts=2023-12-23T12:41:28.4>
[root@watchsrv ~]#
```
> - Once the setup is complete, you can access the Prometheus UI by logging in to http:/<Server Name/Server IP>/:9090, as below.
<img src="imgs/prometheus-initial.png" alt="Grafana Initial User Interface"> 

<hr >

### Installing & Configuring MySQL Prometheus Exporter

Prometheus requires an exporter for collecting MySQL server metrics. This exporter can be run centrally on the Prometheus server or locally on the MySQL database server. For further reading, refer to this [Prometheus documentation](https://prometheus.io/docs/instrumenting/writing_exporters/#deployment).

Follow the below steps to install and setup MySQL Prometheus Exporter on the central Prometheus host.

> - To configure Prometheus we will need to download the [Prometheus Binaries](https://prometheus.io/download/). We will be using Linux compatible binaries.
```
[root@watchsrv ~]# wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
--2023-12-23 18:33:37--  https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/32075541/ab36c0ef-84bc-41a2-a1f6-f2545b8d4054?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20231223%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231223T124838Z&X-Amz-Expires=300&X-Amz-Signature=a0fef14c42b3876b7cd41f9fcfb1afb638b3a438810a29d3ccef22186c5709a8&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=32075541&response-content-disposition=attachment%3B%20filename%3Dmysqld_exporter-0.15.1.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream [following]
--2023-12-23 18:33:38--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/32075541/ab36c0ef-84bc-41a2-a1f6-f2545b8d4054?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20231223%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20231223T124838Z&X-Amz-Expires=300&X-Amz-Signature=a0fef14c42b3876b7cd41f9fcfb1afb638b3a438810a29d3ccef22186c5709a8&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=32075541&response-content-disposition=attachment%3B%20filename%3Dmysqld_exporter-0.15.1.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.109.133, 185.199.110.133, 185.199.111.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.109.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8287769 (7.9M) [application/octet-stream]
Saving to: ‘mysqld_exporter-0.15.1.linux-amd64.tar.gz’

mysqld_exporter-0.1 100%[===================>]   7.90M  2.51MB/s    in 3.2s

2023-12-23 18:33:43 (2.51 MB/s) - ‘mysqld_exporter-0.15.1.linux-amd64.tar.gz’ saved [8287769/8287769]

[root@watchsrv ~]#
```

> - Once its downloaded lets unzip it and move to /usr/local/bin

```
[root@watchsrv ~]# tar -xvf mysqld_exporter-0.15.1.linux-amd64.tar.gz
mysqld_exporter-0.15.1.linux-amd64/
mysqld_exporter-0.15.1.linux-amd64/LICENSE
mysqld_exporter-0.15.1.linux-amd64/mysqld_exporter
mysqld_exporter-0.15.1.linux-amd64/NOTICE
[root@watchsrv ~]# mv mysqld_exporter-0.15.1.linux-amd64 mysqld_exporter
[root@watchsrv ~]# cd mysqld_exporter/
[root@watchsrv mysqld_exporter]# cp mysqld_exporter /usr/local/bin/
[root@watchsrv mysqld_exporter]# chmod +x /usr/local/bin/mysqld_exporter
[root@watchsrv mysqld_exporter]#
```

> - Create the user in mysql dedicated for monitoring.

```
[root@mysqlvm1 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8571
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'mysqldexp'@'%' IDENTIFIED BY 'admin123' WITH MAX_USER_CONNECTIONS 2;
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqldexp'@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
[root@mysqlvm1 ~]#
```

> - Once the user is created, lets configure the Database Credentials as we are monitoring 3 servers we need to create 3 seperate files.

```
[root@watchsrv ~]# vi /etc/mysqldmymysqlvm1.cnf
[root@watchsrv ~]#
[root@watchsrv ~]# cp /etc/mysqldmymysqlvm1.cnf /etc/mysqldmymysqlvm2.cnf
[root@watchsrv ~]# cp /etc/mysqldmymysqlvm1.cnf /etc/mysqldmymysqlvm3.cnf
[root@watchsrv ~]# vi /etc/mysqldmymysqlvm2.cnf
[root@watchsrv ~]# vi /etc/mysqldmymysqlvm3.
[root@watchsrv ~]#
[root@watchsrv ~]# chown root:prometheus /etc/mysqldmymysqlvm1.cnf
[root@watchsrv ~]# chown root:prometheus /etc/mysqldmymysqlvm2.cnf
[root@watchsrv ~]# chown root:prometheus /etc/mysqldmymysqlvm3.cnf
[root@watchsrv ~]#
[root@watchsrv ~]# cat /etc/mysqldmymysqlvm1.cnf
[client]
user=mysqldexp
password=admin123
host=mysqlvm1.
[root@watchsrv ~]#
[root@watchsrv ~]# cat /etc/mysqldmymysqlvm2.cnf
[client]
user=mysqldexp
password=admin123
host=mysqlvm2.localdomain
[root@watchsrv ~]#
[root@watchsrv ~]# cat /etc/mysqldmymysqlvm3.cnf
[client]
user=mysqldexp
password=admin123
host=mysqlvm3.localdomain
[root@watchsrv ~]#
```

> - Create systemd Unit File for each servers
```
[root@watchsrv ~]# vi /etc/systemd/system/mysqldmymysqlvm1.service
[root@watchsrv ~]# cp /etc/systemd/system/mysqldmymysqlvm1.service /etc/systemd/system/mysqldmymysqlvm2.service
[root@watchsrv ~]# cp /etc/systemd/system/mysqldmymysqlvm1.service /etc/systemd/system/mysqldmymysqlvm3.service
[root@watchsrv ~]# 
[root@watchsrv ~]# vi /etc/systemd/system/mysqldmymysqlvm2.service
[root@watchsrv ~]# vi /etc/systemd/system/mysqldmymysqlvm3.service
[root@watchsrv ~]# 
[root@watchsrv ~]# cat /etc/systemd/system/mysqldmymysqlvm1.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/mysqldmymysqlvm1.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target
[root@watchsrv ~]# 
[root@watchsrv ~]# 
[root@watchsrv ~]# cat /etc/systemd/system/mysqldmymysqlvm2.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/mysqldmymysqlvm2.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9105

[Install]
WantedBy=multi-user.target
[root@watchsrv ~]# 
[root@watchsrv ~]# 
[root@watchsrv ~]# cat /etc/systemd/system/mysqldmymysqlvm3.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/mysqldmymysqlvm3.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9106

[Install]
WantedBy=multi-user.target
[root@watchsrv ~]#


```

> - When done, reload systemd and start mysql_exporter service each

```
[root@watchsrv ~]# systemctl daemon-reload
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl enable mysqldmymysqlvm1
Created symlink /etc/systemd/system/multi-user.target.wants/mysqldmymysqlvm1.service → /etc/systemd/system/mysqldmymysqlvm1.service.
[root@watchsrv ~]# systemctl enable mysqldmymysqlvm2
Created symlink /etc/systemd/system/multi-user.target.wants/mysqldmymysqlvm2.service → /etc/systemd/system/mysqldmymysqlvm2.service.
[root@watchsrv ~]# systemctl enable mysqldmymysqlvm3
Created symlink /etc/systemd/system/multi-user.target.wants/mysqldmymysqlvm3.service → /etc/systemd/system/mysqldmymysqlvm3.service.
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl start mysqldmymysqlvm1
[root@watchsrv ~]# systemctl start mysqldmymysqlvm2
[root@watchsrv ~]# systemctl start mysqldmymysqlvm3
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl status mysqldmymysqlvm1
● mysqldmymysqlvm1.service - Prometheus MySQL Exporter
   Loaded: loaded (/etc/systemd/system/mysqldmymysqlvm1.service; disabled; vend>
   Active: active (running) since Sat 2023-12-23 19:27:15 +0545; 13s ago
 Main PID: 5936 (mysqld_exporter)
    Tasks: 4 (limit: 22960)
   Memory: 2.6M
   CGroup: /system.slice/mysqldmymysqlvm1.service
           └─5936 /usr/local/bin/mysqld_exporter --config.my-cnf /etc/mysqldmym>

Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
Dec 23 19:27:15 watchsrv.localdomain mysqld_exporter[5936]: ts=2023-12-23T13:42>
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl status mysqldmymysqlvm2
● mysqldmymysqlvm2.service - Prometheus MySQL Exporter
   Loaded: loaded (/etc/systemd/system/mysqldmymysqlvm2.service; disabled; vend>
   Active: active (running) since Sat 2023-12-23 19:27:17 +0545; 14s ago
 Main PID: 5942 (mysqld_exporter)
    Tasks: 4 (limit: 22960)
   Memory: 2.6M
   CGroup: /system.slice/mysqldmymysqlvm2.service
           └─5942 /usr/local/bin/mysqld_exporter --config.my-cnf /etc/mysqldmym>

Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
Dec 23 19:27:17 watchsrv.localdomain mysqld_exporter[5942]: ts=2023-12-23T13:42>
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl status mysqldmymysqlvm3
● mysqldmymysqlvm3.service - Prometheus MySQL Exporter
   Loaded: loaded (/etc/systemd/system/mysqldmymysqlvm3.service; disabled; vend>
   Active: active (running) since Sat 2023-12-23 19:27:19 +0545; 16s ago
 Main PID: 5948 (mysqld_exporter)
    Tasks: 4 (limit: 22960)
   Memory: 2.6M
   CGroup: /system.slice/mysqldmymysqlvm3.service
           └─5948 /usr/local/bin/mysqld_exporter --config.my-cnf /etc/mysqldmym>

Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
Dec 23 19:27:19 watchsrv.localdomain mysqld_exporter[5948]: ts=2023-12-23T13:42>
[root@watchsrv ~]#
```

> - Configure MySQL Endpoint to be Scraped by Prometheus and restart Prometheus
```
[root@watchsrv ~]# vi /etc/prometheus/prometheus.yml
[root@watchsrv ~]# cat /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "mysqlvm1"
    static_configs:
      - targets: ["localhost:9104"]
  - job_name: "mysqlvm2"
    static_configs:
      - targets: ['localhost:9105']
  - job_name: "mysqlvm3"
    static_configs:
      - targets: ['localhost:9106']
[root@watchsrv ~]#
[root@watchsrv ~]# systemctl stop prometheus
[root@watchsrv ~]# systemctl start prometheus
[root@watchsrv ~]# systemctl status prometheus
● prometheus.service - Prometheus
   Loaded: loaded (/etc/systemd/system/prometheus.service; disabled; vendor pre>
   Active: active (running) since Sat 2023-12-23 20:44:50 +0545; 11s ago
     Docs: https://prometheus.io/docs/introduction/overview/
 Main PID: 7474 (prometheus)
    Tasks: 7 (limit: 22960)
   Memory: 47.6M
   CGroup: /system.slice/prometheus.service
           └─7474 /usr/local/bin/prometheus --config.file=/etc/prometheus/prome>

Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
Dec 23 20:44:50 watchsrv.localdomain prometheus[7474]: ts=2023-12-23T14:59:50.7>
[root@watchsrv ~]#

```

> - View the endpoints in Prometheus
<img src="imgs/prometheus-targets.png" alt="Grafana Prometheus Targets"> 

> - Verify its working by clicking Graph and typing mysql and choose any
<img src="imgs/prometheus-query.png" alt="Grafana Prometheus Targets"> 

<hr >

### Adding a Data Source in Gafana

> - To add a Data Source go to <strong> Home | Data Sources </strong> as in below:
<img src="imgs/datasource-1.png" alt="Data Source 1"> 

> - Data Source tab opens then click on Add data source as below:
<img src="imgs/datasource-2.png" alt="Grafana Initial Dashboard"> 

> - Click on Prometheus as below:
<img src="imgs/datasource-3.png" alt="Grafana Initial Dashboard"> 

> - Provide name and connection details as below and click on <strong> Save & Test </strong> scrolling:
<img src="imgs/datasource-4.png" alt="Grafana Initial Dashboard"> 

Thats it data source addition to Grafana is completed.

<hr >

### Build MySQL Dashboard

> - Login to Grafana and click on dashboards as below:
<img src="imgs/dashboard-2.png" alt="Data Source 1"> 

> - Click on Import Dashboard:
<img src="imgs/dashboard-3.png" alt="Data Source 1"> 

> - We will used the default one provided by Grafana. Hence go to Grafana site for [Mysql Overview Dashboard](https://grafana.com/grafana/dashboards/7362-mysql-overview/?tab=revisions) and click on Copy ID to Clipboard as i have already clicked so it's showing copied.
<img src="imgs/dashboard-1.png" alt="Data Source 1"> 

> - Paste the copied id as below:
<img src="imgs/dashboard-4.png" alt="Data Source 1"> 

> - Select Prometheus and click on import
<img src="imgs/dashboard-5.png" alt="Data Source 1">

> - Your Dashboard will load as below:
<img src="imgs/dashboard-6.png" alt="Data Source 1">
