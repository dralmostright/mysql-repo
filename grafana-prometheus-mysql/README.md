## Install/Configure Monitoring of MySQL Using Gafana & Prometheus

Being a DBA its crucial to have monitoring tool to see whats happening in the database. There are various monitoring tools which are open source and paid. But here we will be explicitly using Grafana and Prometheus to monitoring MySQL.

Grafana labs give more performance visibility in comparison to others. It is an open-source metric analytics and visualization tool that enables developers to write plugins from scratch to integrate with several data sources.

Even if there is a failure, we can easily troubleshoot the issue with the available stats, like database connections, number of containers running and performance, number of bytes written and read, etc. This blog throws light on how to monitor a MySQL database using Grafana and Prometheus.

Steps to Enable MySQL Database Monitoring Using Prometheus and Grafana
* Install and configure Grafana
* Install and configure Prometheus
* Install a database exporter

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

Now lets test if its running. Grafana uses 3000 port so obtaining the server ip we will hith the url as below:
<img src="imgs/grafana-initial.png" alt="Grafana Initial User Interface"> 


yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-10.2.3-1.x86_64.rpm

### Install and configure Prometheus

To configure Prometheus we will need to download the [Prometheus Binaries](https://prometheus.io/download/). We will be using Linux compatible binaries.
 