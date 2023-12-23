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



yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-10.2.3-1.x86_64.rpm

### Install and configure Prometheus