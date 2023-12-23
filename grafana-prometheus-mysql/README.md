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

### Install and configure Prometheus

To configure Prometheus we will need to download the [Prometheus Binaries](https://prometheus.io/download/). We will be using Linux compatible binaries.

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
Lets unzip the binaries.
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

1. Create a Prometheus System Group & User and directories
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

```
[root@watchsrv prometheus-2.45.2.linux-amd64]# pwd
/root/prometheus-2.45.2.linux-amd64
[root@watchsrv prometheus-2.45.2.linux-amd64]#
[root@watchsrv prometheus-2.45.2.linux-amd64]# ls
console_libraries  LICENSE  prometheus      promtool
consoles           NOTICE   prometheus.yml
[root@watchsrv prometheus-2.45.2.linux-amd64]# mv prometheus promtool /usr/local/bin/
[root@watchsrv prometheus-2.45.2.linux-amd64]# chown prometheus:prometheus /usr/local/bin/prometheu^C
[root@watchsrv prometheus-2.45.2.linux-amd64]# cd
[root@watchsrv ~]# chown prometheus:prometheus /usr/local/bin/prometheus
[root@watchsrv ~]# chown prometheus:prometheus /usr/local/bin/promtool
```

```
[root@watchsrv ~]# cp -r prometheus-2.45.2.linux-amd64/consoles /etc/prometheus
[root@watchsrv ~]# cp -r prometheus-2.45.2.linux-amd64/console_libraries /etc/prometheus
[root@watchsrv ~]# chown -R prometheus:prometheus /etc/prometheus/consoles
[root@watchsrv ~]# chown -R prometheus:prometheus /etc/prometheus/console_libraries
[root@watchsrv ~]#
```

```
[root@watchsrv ~]# cp prometheus-2.45.2.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml
[root@watchsrv ~]# chown prometheus:prometheus /etc/prometheus/prometheus.yml
[root@watchsrv ~]#
```

We will be using the temaplate as it is without any change if needed we will change it in future
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

<img src="imgs/prometheus-initial.png" alt="Grafana Initial User Interface"> 

### Installing & Configuring MySQL Prometheus Exporter

Prometheus requires an exporter for collecting MySQL server metrics. This exporter can be run centrally on the Prometheus server or locally on the MySQL database server. For further reading, refer to this [Prometheus documentation](https://prometheus.io/docs/instrumenting/writing_exporters/#deployment).

Follow the below steps to install and setup MySQL Prometheus Exporter on the central Prometheus host.