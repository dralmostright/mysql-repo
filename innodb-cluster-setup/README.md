## MySQL InnoDB Cluster 8.0 - Hands-On & Production Deployment

MySQL being lightweight and easy to use has become one of the most popular database choices among the developers. Starting with release 8 MySQL InnoDB Cluster has been providing out-of-the-box HA solution for MySQL. Here we will be doing a typical InnoDB cluster setup. We will be using MySQL Communitiy edition to confure the cluster.

<table>
<tr><th>Host Details </th><th>Cluster Architecture</th></tr>
<tr><td>

| HostName | Instance Type | Operating System & MySQL version |
| ----------- | ----------- | -----------------|
| mysqlvm1.localdomain| Primary Instance (R/W) | Oracle Linux 8.4, MySQL 8.0.35 |
| mysqlvm2.localdomain| Secondary Instance (R/O) | Oracle Linux 8.4, MySQL 8.0.35 |
| mysqlvm3.localdomain| Secondary Instance  (R/O)| Oracle Linux 8.4, MySQL 8.0.35 |
| mysqlvm1.localdomain| MySQL Router Instance| Oracle Linux 8.4 |

</td>
<td>

<img src="imgs/innodb-cluster.png" alt="Cluster Architecture"> 

</td>
</tr> </table>
<hr >

### MySQL InnoDB has three major components:

* MySQL group replication – It is a group of database servers. It replicates the MySQL databases across the multiple nodes, and it has fault tolerance. When changes in the data occur in the MySQL databases, it automatically replicates to the secondary nodes of the server.
* MySQL Router – When a failover occurs, the client application must be aware of the PRIMARY instance and cluster topology. This functionality is handled by the MySQL Router. It routes that redirects the data requests to the available MySQL Server instance. MySQL Router acts as a proxy that is used to hide the multiple MySQL database servers. The concept of the MySQL Router is similar to the Virtual network name of the Windows Server failover cluster
* MySQL Shell – It is a configuration tool that can be used to connect, deploy, and manage the MySQL InnoDB cluster. MySQL Shell contains an Admin API that has a dba global variable. The dba variable is used to deploy and manage the InnoDB cluster

### InnoDB Cluster Requirements

* InnoDB Cluster uses Group Replication and therefore your server instances must meet the same requirements. AdminAPI provides the dba.checkInstanceConfiguration() method to verify that an instance meets the Group Replication requirements, and the dba.configureInstance() method to configure an instance to meet the requirements.

* Data for use with InnoDB Cluster, must be stored in the InnoDB transactional storage engine. 
The Performance Schema must be enabled on any instance which you want to use with InnoDB Cluster.

* There are others too you can refer more on [MySQL Doc](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-innodb-cluster-requirements.html)

* SELINUX policies need to be set properly, but its better to disable it

<hr >

#### Softwares Required:
We will be installing the below rpms and we will download them using wget
```
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-server-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-client-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-devel-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-common-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-libs-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-icu-data-files-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-community-client-plugins-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-Router/mysql-router-community-8.0.35-1.el8.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-8.0.35-1.el8.x86_64.rpm
```

```
[root@mysqlvm1 ~]# ls -ltr | grep rpm
-rw-r--r--  1 root root 29226220 Oct 15 15:59 mysql-shell-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root 16724688 Oct 16 11:14 mysql-community-client-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root  3725096 Oct 16 11:14 mysql-community-client-plugins-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root   683572 Oct 16 11:14 mysql-community-common-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root  2305100 Oct 16 11:15 mysql-community-devel-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root  2347332 Oct 16 11:15 mysql-community-icu-data-files-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root  1564500 Oct 16 11:15 mysql-community-libs-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root 67554704 Oct 16 11:16 mysql-community-server-8.0.35-1.el8.x86_64.rpm
-rw-r--r--  1 root root  5372088 Oct 16 11:19 mysql-router-community-8.0.35-1.el8.x86_64.rpm
[root@mysqlvm1 ~]#
```

Alternatively, we can download from below link too:
* [MySQL Communitiy Edition RPM's](https://dev.mysql.com/downloads/mysql/)
* [MySQL Router RPM](https://dev.mysql.com/downloads/router/)
* [MySQL Shell RPM](https://dev.mysql.com/downloads/shell/)

<hr >

#### Disable Selinux
Verify the status of Selinux

```
[root@mysqlvm1 ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      31
[root@mysqlvm1 ~]# cat /etc/sysconfig/selinux | grep -i Selinux | grep -v '#'
SELINUX=enforcing
SELINUXTYPE=targeted
[root@mysqlvm1 ~]#
```

Open the SELinux configuration file with a text editor.
```
[root@mysqlvm1 ~]# vi /etc/sysconfig/selinux
```
In the file, set SELINUX to disabled:
```
SELINUX=disabled
```
Save and exit the file.

Reboot the server to make your changes take effect and you need to do this in all servers.
```
[root@mysqlvm1 ~]# reboot
```
Once the server is up verify the status:
```
[root@mysqlvm1 ~]# sestatus
SELinux status:                 disabled
[root@mysqlvm1 ~]#
```
<hr >

### Disable Firewall on the server:
We will disable the firewall completely as in most situation we will have external firewall.
```
[root@mysqlvm1 ~]# systemctl stop firewalld
[root@mysqlvm1 ~]# systemctl disable firewalld
Removed /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@mysqlvm1 ~]#
```

### Configure Node reachability 
By editing and appending last three lines as below in /etc/hosts file the host name can be resolved without needing of dns, if in case you are using dns this is not required. This needs to be done in all three nodes.
```
[root@mysqlvm1 ~]# vi /etc/hosts
[root@mysqlvm1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.229.139 mysqlvm1.localdomain mysqlvm1
192.168.229.135 mysqlvm2.localdomain mysqlvm2
192.168.229.136 mysqlvm3.localdomain mysqlvm3
[root@mysqlvm1 ~]#
```
<hr >

### Test node reachability
* From mysqlvm1
```
[root@mysqlvm1 ~]# for i in 1 2 3; do ping mysqlvm$i -c 1; echo " "; done
PING mysqlvm1.localdomain (192.168.229.138) 56(84) bytes of data.
64 bytes from mysqlvm1.localdomain (192.168.229.138): icmp_seq=1 ttl=64 time=0.042 ms

--- mysqlvm1.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.042/0.042/0.042/0.000 ms

PING mysqlvm2.localdomain (192.168.229.135) 56(84) bytes of data.
64 bytes from mysqlvm2.localdomain (192.168.229.135): icmp_seq=1 ttl=64 time=0.754 ms

--- mysqlvm2.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.754/0.754/0.754/0.000 ms

PING mysqlvm3.localdomain (192.168.229.136) 56(84) bytes of data.
64 bytes from mysqlvm3.localdomain (192.168.229.136): icmp_seq=1 ttl=64 time=0.842 ms

--- mysqlvm3.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.842/0.842/0.842/0.000 ms

[root@mysqlvm1 ~]#
```
* From mysqlvm2
```
[root@mysqlvm2 ~]# for i in 1 2 3; do ping mysqlvm$i -c 1; echo " "; done
PING mysqlvm1.localdomain (192.168.229.138) 56(84) bytes of data.
64 bytes from mysqlvm1.localdomain (192.168.229.138): icmp_seq=1 ttl=64 time=1.99 ms

--- mysqlvm1.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.986/1.986/1.986/0.000 ms

PING mysqlvm2.localdomain (192.168.229.135) 56(84) bytes of data.
64 bytes from mysqlvm2.localdomain (192.168.229.135): icmp_seq=1 ttl=64 time=0.043 ms

--- mysqlvm2.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.043/0.043/0.043/0.000 ms

PING mysqlvm3.localdomain (192.168.229.136) 56(84) bytes of data.
64 bytes from mysqlvm3.localdomain (192.168.229.136): icmp_seq=1 ttl=64 time=1.17 ms

--- mysqlvm3.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.168/1.168/1.168/0.000 ms

[root@mysqlvm2 ~]#
```
* From mysqlvm3
```
[root@mysqlvm3 ~]# for i in 1 2 3; do ping mysqlvm$i -c 1; echo " "; done
PING mysqlvm1.localdomain (192.168.229.138) 56(84) bytes of data.
64 bytes from mysqlvm1.localdomain (192.168.229.138): icmp_seq=1 ttl=64 time=1.05 ms

--- mysqlvm1.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.046/1.046/1.046/0.000 ms

PING mysqlvm2.localdomain (192.168.229.135) 56(84) bytes of data.
64 bytes from mysqlvm2.localdomain (192.168.229.135): icmp_seq=1 ttl=64 time=0.920 ms

--- mysqlvm2.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.920/0.920/0.920/0.000 ms

PING mysqlvm3.localdomain (192.168.229.136) 56(84) bytes of data.
64 bytes from mysqlvm3.localdomain (192.168.229.136): icmp_seq=1 ttl=64 time=0.043 ms

--- mysqlvm3.localdomain ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.043/0.043/0.043/0.000 ms

[root@mysqlvm3 ~]#
```
<hr >

### Configure MySQL instance directories and parameters
We will not be using default directories to hold mysql configuration and data, hence we have created seperate paritions for data and binlog and this needs to be done on all mysql server instances.

```
[root@mysqlvm1 ~]# df -Ph | grep 'pgdata\|walarc'
/dev/mapper/ol-pgdata  5.0G   68M  5.0G   2% /pgdata
/dev/mapper/ol-walarc  2.0G   47M  2.0G   3% /walarc
[root@mysqlvm1 ~]#
```

Create respecitive directories and change permission and this needs to be done in all instances
```
[root@mysqlvm1 ~]# mkdir -pv /pgdata/mysql/
mkdir: created directory '/pgdata/mysql/'
[root@mysqlvm1 ~]# mkdir -pv /pgdata/mysql/data
mkdir: created directory '/pgdata/mysql/data'
[root@mysqlvm1 ~]# mkdir -pv /pgdata/mysql/log
mkdir: created directory '/pgdata/mysql/log'
[root@mysqlvm1 ~]# mkdir -pv /pgdata/mysqlrouter
mkdir: created directory '/pgdata/mysqlrouter'
[root@mysqlvm1 ~]# mkdir -pv /walarc/mysql/binlog/
mkdir: created directory '/walarc/mysql'
mkdir: created directory '/walarc/mysql/binlog/'
[root@mysqlvm1 ~]#
[root@mysqlvm1 ~]# chown -R mysql:mysql /pgdata/mysql
[root@mysqlvm1 ~]# chmod -R 775 /pgdata/mysql
[root@mysqlvm1 ~]# chown -R mysql:mysql /walarc/mysql/
[root@mysqlvm1 ~]# chmod -R 775 /walarc/mysql/
[root@mysqlvm1 ~]# chown -R mysqlrouter:mysqlrouter /pgdata/mysqlrouter/

```

Update the parameters to the directories created and copy this file in all instances
```
[root@mysqlvm1 ~]# vi /etc/my.cnf
```
Verify the changes
```
[root@mysqlvm1 ~]# cat /etc/my.cnf | grep -v '#'

[mysqld]

datadir=/pgdata/mysql/data
socket=/var/lib/mysql/mysql.sock
log-bin=/walarc/mysql/binlog
bind-address=0.0.0.0
log-error=/pgdata/mysql/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
[root@mysqlvm1 ~]#
```
Copy the update configuration in all nodes:
```
[root@mysqlvm1 ~]# scp /etc/my.cnf mysqlvm2://etc/my.cnf
root@mysqlvm2's password:
my.cnf                                                                     100% 1288   260.3KB/s   00:00
[root@mysqlvm1 ~]# scp /etc/my.cnf mysqlvm3://etc/my.cnf
root@mysqlvm3's password:
my.cnf                                                                     100% 1288   264.7KB/s   00:00
[root@mysqlvm1 ~]#
```
<hr >


### Install the MySQL RPM's and its suppliment packages in each server
```
[root@mysqlvm1 ~]# yum install mysql-community-client-8.0.35-1.el8.x86_64.rpm \
> mysql-community-client-plugins-8.0.35-1.el8.x86_64.rpm \
> mysql-community-common-8.0.35-1.el8.x86_64.rpm \
> mysql-community-devel-8.0.35-1.el8.x86_64.rpm \
> mysql-community-icu-data-files-8.0.35-1.el8.x86_64.rpm \
> mysql-community-libs-8.0.35-1.el8.x86_64.rpm \
> mysql-community-server-8.0.35-1.el8.x86_64.rpm
Last metadata expiration check: 0:01:39 ago on Sun 17 Dec 2023 05:45:28 PM +0545.
Dependencies resolved.
=============================================================================================================
 Package                               Architecture  Version                  Repository                Size
=============================================================================================================
Installing:
 mysql-community-client                x86_64        8.0.35-1.el8             @commandline              16 M
 mysql-community-client-plugins        x86_64        8.0.35-1.el8             @commandline             3.6 M
 mysql-community-common                x86_64        8.0.35-1.el8             @commandline             668 k
 mysql-community-devel                 x86_64        8.0.35-1.el8             @commandline             2.2 M
 mysql-community-icu-data-files        x86_64        8.0.35-1.el8             @commandline             2.2 M
 mysql-community-libs                  x86_64        8.0.35-1.el8             @commandline             1.5 M
 mysql-community-server                x86_64        8.0.35-1.el8             @commandline              64 M
Installing dependencies:
 keyutils-libs-devel                   x86_64        1.5.10-6.el8             ol8_baseos_latest         48 k
 krb5-devel                            x86_64        1.18.2-8.el8             ol8_baseos_latest        559 k
 libcom_err-devel                      x86_64        1.45.6-1.el8             ol8_baseos_latest         38 k
 libkadm5                              x86_64        1.18.2-8.el8             ol8_baseos_latest        186 k
 libselinux-devel                      x86_64        2.9-5.el8                ol8_baseos_latest        200 k
 libsepol-devel                        x86_64        2.9-2.el8                ol8_baseos_latest         86 k
 libverto-devel                        x86_64        0.3.0-5.el8              ol8_baseos_latest         18 k
 openssl-devel                         x86_64        1:1.1.1g-15.el8_3        ol8_baseos_latest        2.3 M
 pcre2-devel                           x86_64        10.32-2.el8              ol8_baseos_latest        605 k
 pcre2-utf16                           x86_64        10.32-2.el8              ol8_baseos_latest        229 k
 pcre2-utf32                           x86_64        10.32-2.el8              ol8_baseos_latest        220 k
 zlib-devel                            x86_64        1.2.11-17.el8            ol8_baseos_latest         57 k

Transaction Summary
=============================================================================================================
Install  19 Packages

Total size: 95 M
Total download size: 4.5 M
Installed size: 439 M
Is this ok [y/N]: Y
Downloading Packages:
(1/12): libcom_err-devel-1.45.6-1.el8.x86_64.rpm                             295 kB/s |  38 kB     00:00
(2/12): keyutils-libs-devel-1.5.10-6.el8.x86_64.rpm                          359 kB/s |  48 kB     00:00
(3/12): krb5-devel-1.18.2-8.el8.x86_64.rpm                                   3.0 MB/s | 559 kB     00:00
(4/12): libkadm5-1.18.2-8.el8.x86_64.rpm                                     3.6 MB/s | 186 kB     00:00
(5/12): libselinux-devel-2.9-5.el8.x86_64.rpm                                3.7 MB/s | 200 kB     00:00
(6/12): libverto-devel-0.3.0-5.el8.x86_64.rpm                                1.4 MB/s |  18 kB     00:00
(7/12): libsepol-devel-2.9-2.el8.x86_64.rpm                                  6.0 MB/s |  86 kB     00:00
(8/12): pcre2-utf16-10.32-2.el8.x86_64.rpm                                   7.1 MB/s | 229 kB     00:00
(9/12): pcre2-utf32-10.32-2.el8.x86_64.rpm                                   8.6 MB/s | 220 kB     00:00
(10/12): zlib-devel-1.2.11-17.el8.x86_64.rpm                                 2.9 MB/s |  57 kB     00:00
(11/12): pcre2-devel-10.32-2.el8.x86_64.rpm                                  6.3 MB/s | 605 kB     00:00
(12/12): openssl-devel-1.1.1g-15.el8_3.x86_64.rpm                             12 MB/s | 2.3 MB     00:00
-------------------------------------------------------------------------------------------------------------
Total                                                                         12 MB/s | 4.5 MB     00:00
warning: /var/cache/dnf/ol8_baseos_latest-e4c6155830ad002c/packages/keyutils-libs-devel-1.5.10-6.el8.x86_64.r                                                                                pm: Header V3 RSA/SHA256 Signature, key ID ad986da3: NOKEY
Oracle Linux 8 BaseOS Latest (x86_64)                                        3.0 MB/s | 3.1 kB     00:00
Importing GPG key 0xAD986DA3:
 Userid     : "Oracle OSS group (Open Source Software group) <build@oss.oracle.com>"
 Fingerprint: 76FD 3DB1 3AB6 7410 B89D B10E 8256 2EA9 AD98 6DA3
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
Is this ok [y/N]: y
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                     1/1
  Installing       : mysql-community-common-8.0.35-1.el8.x86_64                                         1/19
  Installing       : mysql-community-client-plugins-8.0.35-1.el8.x86_64                                 2/19
  Installing       : mysql-community-libs-8.0.35-1.el8.x86_64                                           3/19
  Running scriptlet: mysql-community-libs-8.0.35-1.el8.x86_64                                           3/19
/sbin/ldconfig: /etc/ld.so.conf.d/kernel-5.4.17-2102.201.3.el8uek.x86_64.conf:6: hwcap directive ignored

  Installing       : mysql-community-client-8.0.35-1.el8.x86_64                                         4/19
  Installing       : mysql-community-icu-data-files-8.0.35-1.el8.x86_64                                 5/19
  Installing       : zlib-devel-1.2.11-17.el8.x86_64                                                    6/19
  Installing       : pcre2-utf32-10.32-2.el8.x86_64                                                     7/19
  Installing       : pcre2-utf16-10.32-2.el8.x86_64                                                     8/19
  Installing       : pcre2-devel-10.32-2.el8.x86_64                                                     9/19
  Installing       : libverto-devel-0.3.0-5.el8.x86_64                                                 10/19
  Installing       : libsepol-devel-2.9-2.el8.x86_64                                                   11/19
  Installing       : libselinux-devel-2.9-5.el8.x86_64                                                 12/19
  Installing       : libkadm5-1.18.2-8.el8.x86_64                                                      13/19
  Installing       : libcom_err-devel-1.45.6-1.el8.x86_64                                              14/19
  Installing       : keyutils-libs-devel-1.5.10-6.el8.x86_64                                           15/19
  Installing       : krb5-devel-1.18.2-8.el8.x86_64                                                    16/19
  Installing       : openssl-devel-1:1.1.1g-15.el8_3.x86_64                                            17/19
  Installing       : mysql-community-devel-8.0.35-1.el8.x86_64                                         18/19
  Running scriptlet: mysql-community-server-8.0.35-1.el8.x86_64                                        19/19
  Installing       : mysql-community-server-8.0.35-1.el8.x86_64                                        19/19
  Running scriptlet: mysql-community-server-8.0.35-1.el8.x86_64                                        19/19
/sbin/ldconfig: /etc/ld.so.conf.d/kernel-5.4.17-2102.201.3.el8uek.x86_64.conf:6: hwcap directive ignored

  Verifying        : keyutils-libs-devel-1.5.10-6.el8.x86_64                                            1/19
  Verifying        : krb5-devel-1.18.2-8.el8.x86_64                                                     2/19
  Verifying        : libcom_err-devel-1.45.6-1.el8.x86_64                                               3/19
  Verifying        : libkadm5-1.18.2-8.el8.x86_64                                                       4/19
  Verifying        : libselinux-devel-2.9-5.el8.x86_64                                                  5/19
  Verifying        : libsepol-devel-2.9-2.el8.x86_64                                                    6/19
  Verifying        : libverto-devel-0.3.0-5.el8.x86_64                                                  7/19
  Verifying        : openssl-devel-1:1.1.1g-15.el8_3.x86_64                                             8/19
  Verifying        : pcre2-devel-10.32-2.el8.x86_64                                                     9/19
  Verifying        : pcre2-utf16-10.32-2.el8.x86_64                                                    10/19
  Verifying        : pcre2-utf32-10.32-2.el8.x86_64                                                    11/19
  Verifying        : zlib-devel-1.2.11-17.el8.x86_64                                                   12/19
  Verifying        : mysql-community-client-8.0.35-1.el8.x86_64                                        13/19
  Verifying        : mysql-community-client-plugins-8.0.35-1.el8.x86_64                                14/19
  Verifying        : mysql-community-common-8.0.35-1.el8.x86_64                                        15/19
  Verifying        : mysql-community-devel-8.0.35-1.el8.x86_64                                         16/19
  Verifying        : mysql-community-icu-data-files-8.0.35-1.el8.x86_64                                17/19
  Verifying        : mysql-community-libs-8.0.35-1.el8.x86_64                                          18/19
  Verifying        : mysql-community-server-8.0.35-1.el8.x86_64                                        19/19

Installed:
  keyutils-libs-devel-1.5.10-6.el8.x86_64               krb5-devel-1.18.2-8.el8.x86_64
  libcom_err-devel-1.45.6-1.el8.x86_64                  libkadm5-1.18.2-8.el8.x86_64
  libselinux-devel-2.9-5.el8.x86_64                     libsepol-devel-2.9-2.el8.x86_64
  libverto-devel-0.3.0-5.el8.x86_64                     mysql-community-client-8.0.35-1.el8.x86_64
  mysql-community-client-plugins-8.0.35-1.el8.x86_64    mysql-community-common-8.0.35-1.el8.x86_64
  mysql-community-devel-8.0.35-1.el8.x86_64             mysql-community-icu-data-files-8.0.35-1.el8.x86_64
  mysql-community-libs-8.0.35-1.el8.x86_64              mysql-community-server-8.0.35-1.el8.x86_64
  openssl-devel-1:1.1.1g-15.el8_3.x86_64                pcre2-devel-10.32-2.el8.x86_64
  pcre2-utf16-10.32-2.el8.x86_64                        pcre2-utf32-10.32-2.el8.x86_64
  zlib-devel-1.2.11-17.el8.x86_64

Complete!
[root@mysqlvm1 ~]#
```
I am using yum so that in case of any dependencies it will auto download the required rpm's. If you have any issues with installing rpm's i would suggest configuring public yum repo using the link [OEL 8 Public Yum Repo](https://yum.oracle.com/getting-started.html)

Seems we forgot to install mysqlshell and router rpms, we will install them too. These two can be installed in only one server and on rest they are optional.
```
[root@mysqlvm1 ~]# yum install mysql-router-community-8.0.35-1.el8.x86_64.rpm mysql-shell-8.0.35-1.el8.x86_64.rpm
Last metadata expiration check: 0:13:54 ago on Sun 17 Dec 2023 05:45:28 PM +0545.
Dependencies resolved.
=============================================================================================================================================================================================
 Package                                           Architecture                   Version                                                        Repository                             Size
=============================================================================================================================================================================================
Installing:
 mysql-router-community                            x86_64                         8.0.35-1.el8                                                   @commandline                          5.1 M
 mysql-shell                                       x86_64                         8.0.35-1.el8                                                   @commandline                           28 M
Installing dependencies:
 python39-libs                                     x86_64                         3.9.18-1.module+el8.9.0+90071+8dc52a4f                         ol8_appstream                         8.2 M
 python39-pip-wheel                                noarch                         20.2.4-8.module+el8.9.0+90016+9c2d6573                         ol8_appstream                         1.1 M
 python39-setuptools-wheel                         noarch                         50.3.2-4.module+el8.9.0+90016+9c2d6573                         ol8_appstream                         497 k
Enabling module streams:
 python39                                                                         3.9

Transaction Summary
=============================================================================================================================================================================================
Install  5 Packages

Total size: 43 M
Total download size: 9.8 M
Installed size: 259 M
Is this ok [y/N]: y
Downloading Packages:
(1/3): python39-setuptools-wheel-50.3.2-4.module+el8.9.0+90016+9c2d6573.noarch.rpm                                                                           1.9 MB/s | 497 kB     00:00
(2/3): python39-pip-wheel-20.2.4-8.module+el8.9.0+90016+9c2d6573.noarch.rpm                                                                                  2.5 MB/s | 1.1 MB     00:00
(3/3): python39-libs-3.9.18-1.module+el8.9.0+90071+8dc52a4f.x86_64.rpm                                                                                        12 MB/s | 8.2 MB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                         14 MB/s | 9.8 MB     00:00
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                     1/1
  Installing       : python39-setuptools-wheel-50.3.2-4.module+el8.9.0+90016+9c2d6573.noarch                                                                                             1/5
  Installing       : python39-pip-wheel-20.2.4-8.module+el8.9.0+90016+9c2d6573.noarch                                                                                                    2/5
  Installing       : python39-libs-3.9.18-1.module+el8.9.0+90071+8dc52a4f.x86_64                                                                                                         3/5
  Installing       : mysql-shell-8.0.35-1.el8.x86_64                                                                                                                                     4/5
  Running scriptlet: mysql-router-community-8.0.35-1.el8.x86_64                                                                                                                          5/5
  Installing       : mysql-router-community-8.0.35-1.el8.x86_64                                                                                                                          5/5
  Running scriptlet: mysql-router-community-8.0.35-1.el8.x86_64                                                                                                                          5/5
/sbin/ldconfig: /etc/ld.so.conf.d/kernel-5.4.17-2102.201.3.el8uek.x86_64.conf:6: hwcap directive ignored

/sbin/ldconfig: /etc/ld.so.conf.d/kernel-5.4.17-2102.201.3.el8uek.x86_64.conf:6: hwcap directive ignored

  Verifying        : python39-libs-3.9.18-1.module+el8.9.0+90071+8dc52a4f.x86_64                                                                                                         1/5
  Verifying        : python39-pip-wheel-20.2.4-8.module+el8.9.0+90016+9c2d6573.noarch                                                                                                    2/5
  Verifying        : python39-setuptools-wheel-50.3.2-4.module+el8.9.0+90016+9c2d6573.noarch                                                                                             3/5
  Verifying        : mysql-router-community-8.0.35-1.el8.x86_64                                                                                                                          4/5
  Verifying        : mysql-shell-8.0.35-1.el8.x86_64                                                                                                                                     5/5

Installed:
  mysql-router-community-8.0.35-1.el8.x86_64                                                       mysql-shell-8.0.35-1.el8.x86_64
  python39-libs-3.9.18-1.module+el8.9.0+90071+8dc52a4f.x86_64                                      python39-pip-wheel-20.2.4-8.module+el8.9.0+90016+9c2d6573.noarch
  python39-setuptools-wheel-50.3.2-4.module+el8.9.0+90016+9c2d6573.noarch

Complete!
[root@mysqlvm1 ~]#
```

```
[root@mysqlvm1 ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
[root@mysqlvm1 ~]# systemctl status mysqlrouter
● mysqlrouter.service - MySQL Router
   Loaded: loaded (/usr/lib/systemd/system/mysqlrouter.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@mysqlvm1 ~]#
```
<hr >

### Configure/Initialize MySQL Server Instance
After all parameters are set now initialize the mysql database, which creates all metadata and other necessary configurations. This needs to be done on all mysql server instances.
```
[root@mysqlvm1 ~]# mysqld --initialize
[root@mysqlvm1 ~]#
```
After initialization is completed it will generate a temporary root password and it can be view in the mysql logfile as below. Later we will change the password of the root.
```
[root@mysqlvm1 ~]# tail -5f /pgdata/mysql/log/mysqld.log
2023-12-17T13:18:54.719547Z 0 [System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.35)  MySQL Community Server - GPL.
2023-12-17T13:21:34.361935Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.35) initializing of server in progress as process 3626
2023-12-17T13:21:34.367465Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-12-17T13:21:34.595848Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-12-17T13:21:35.125972Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: BJ-!55HHhqC*
```

The initialization of mysql instance reverts the permission which we have set earlier hence we need to redo this, else during startup it will fail. This needs to be done on all mysql server instances.
```
[root@mysqlvm1 ~]# chown -R mysql:mysql /pgdata/mysql
[root@mysqlvm1 ~]# chown -R mysql:mysql /walarc/mysql
[root@mysqlvm1 ~]#
```
Once Done, finally startup the mysql instance and view the status. This needs to be done on all mysql server instances.
```
[root@mysqlvm1 ~]# systemctl start mysqld
[root@mysqlvm1 ~]# systemcl status mysqld
[root@mysqlvm1 ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-12-17 19:09:26 +0545; 27s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 3757 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 3784 (mysqld)
   Status: "Server is operational"
    Tasks: 38 (limit: 22960)
   Memory: 403.2M
   CGroup: /system.slice/mysqld.service
           └─3784 /usr/sbin/mysqld

Dec 17 19:09:25 mysqlvm1.localdomain systemd[1]: Starting MySQL Server...
Dec 17 19:09:26 mysqlvm1.localdomain systemd[1]: Started MySQL Server.
[root@mysqlvm1 ~]# systemctl enable mysqld
Created symlink /etc/systemd/system/multi-user.target.wants/mysqld.service → /usr/lib/systemd/system/mysqld.service.
[root@mysqlvm1 ~]#
```
* Configure MySQL Server instance on mysqlvm2
```
[root@mysqlvm2 data]# mysqld --initialize
[root@mysqlvm2 data]# chown -R mysql:mysql /pgdata/mysql/
[root@mysqlvm2 data]# chown -R mysql:mysql /walarc/mysql/
[root@mysqlvm2 data]# systemctl start mysqld
[root@mysqlvm2 data]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-12-17 19:27:17 +0545; 5s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 4193 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 4228 (mysqld)
   Status: "Server is operational"
    Tasks: 38 (limit: 22960)
   Memory: 429.8M
   CGroup: /system.slice/mysqld.service
           └─4228 /usr/sbin/mysqld

Dec 17 19:27:14 mysqlvm2.localdomain systemd[1]: Starting MySQL Server...
Dec 17 19:27:17 mysqlvm2.localdomain systemd[1]: Started MySQL Server.
[root@mysqlvm2 data]# systemctl enable mysqld
Created symlink /etc/systemd/system/multi-user.target.wants/mysqld.service → /usr/lib/systemd/system/mysqld.service.
[root@mysqlvm2 data]#
```
Note the temporary root password.
```
[root@mysqlvm2 ~]# tail -5f /pgdata/mysql/log/mysqld.log
2023-12-17T13:41:16.597596Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.35) initializing of server in progress as process 4126
2023-12-17T13:41:16.602728Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-12-17T13:41:16.852494Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-12-17T13:41:17.575635Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: oqVW0XeJh(,a
2023-12-17T13:42:16.957913Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.35) starting as process 4228
2023-12-17T13:42:16.974809Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-12-17T13:42:17.503610Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-12-17T13:42:17.845177Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2023-12-17T13:42:17.845225Z 0 [System] [MY-013602] [Server] Channel mysql_main configured to support TLS. Encrypted connections are now supported for this channel.
2023-12-17T13:42:17.858092Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
```
* Configure MySQL Server instance on mysqlvm3
```
[root@mysqlvm3 ~]# mysqld --initialize
[root@mysqlvm3 ~]# systemctl start mysqld
Job for mysqld.service failed because the control process exited with error code.
See "systemctl status mysqld.service" and "journalctl -xe" for details.
[root@mysqlvm3 ~]# chown -R mysql:mysql /pgdata/mysql
[root@mysqlvm3 ~]# chown -R mysql:mysql /walarc/mysql
[root@mysqlvm3 ~]# systemctl start mysqld
[root@mysqlvm3 ~]# systectl status mysqld
bash: systectl: command not found...
^C
[root@mysqlvm3 ~]# systemctl start mysqld
[root@mysqlvm3 ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-12-17 19:31:36 +0545; 19s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 4232 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 4260 (mysqld)
   Status: "Server is operational"
    Tasks: 38 (limit: 22960)
   Memory: 405.6M
   CGroup: /system.slice/mysqld.service
           └─4260 /usr/sbin/mysqld

Dec 17 19:31:34 mysqlvm3.localdomain systemd[1]: Starting MySQL Server...
Dec 17 19:31:36 mysqlvm3.localdomain systemd[1]: Started MySQL Server.
[root@mysqlvm3 ~]# systemctl enable mysqld
Created symlink /etc/systemd/system/multi-user.target.wants/mysqld.service → /usr/lib/systemd/system/mysqld.service.
[root@mysqlvm3 ~]#
```

Note the temporary root password.
```
[root@mysqlvm3 ~]# tail -5f /pgdata/mysql/log/mysqld.log
2023-12-17T13:45:20.522884Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.35) initializing of server in progress as process 4135
2023-12-17T13:45:20.530486Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-12-17T13:45:20.791820Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-12-17T13:45:21.525265Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: kls_3;!s*Qi!
2023-12-17T13:46:35.433256Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.35) starting as process 4260
2023-12-17T13:46:35.439177Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-12-17T13:46:36.089039Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-12-17T13:46:36.624662Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2023-12-17T13:46:36.624708Z 0 [System] [MY-013602] [Server] Channel mysql_main configured to support TLS. Encrypted connections are now supported for this channel.
2023-12-17T13:46:36.640527Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.35'  socket: '/pgdata/mysql/data/mysql.sock'  port: 3306  MySQL Community Server - GPL.
2023-12-17T13:46:36.640637Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
```
<hr >

### Update root password and create dedicated user for cluster replication.
* Update and create users on mysqlvm1
```
[root@mysqlvm1 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.35

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'admin123';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
[root@mysqlvm1 ~]#
```
```
[root@mysqlvm1 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'clusteradmin'@'%' identified by 'admin123';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL privileges on *.* to 'clusteradmin'@'%' with grant option;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql>
```

* Update and create users on mysqlvm2
```
[root@mysqlvm2 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.35

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'admin123';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
[root@mysqlvm2 ~]#
```
```
[root@mysqlvm2 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'clusteradmin'@'%' identified by 'admin123';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL privileges on *.* to 'clusteradmin'@'%' with grant option;
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql>
```

* Update and create users on mysqlvm3
```
[root@mysqlvm3 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.35

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'admin123';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
[root@mysqlvm3 ~]#
```
```
[root@mysqlvm3 ~]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'clusteradmin'@'%' identified by 'admin123';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL privileges on *.* to 'clusteradmin'@'%' with grant option;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql>
```
<hr >

### Verify the connectivity among the nodes
* from mysqlvm1 to other
```
[root@mysqlvm1 ~]# for i in 1 2 3; do mysql -uclusteradmin -padmin123 -hmysqlvm$i -P3306 -e 'select user(), @@hostname, @@read_only, @@super_read_only'; echo " "; done
mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm1.localdomain | mysqlvm1.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm1.localdomain | mysqlvm2.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm1.localdomain | mysqlvm3.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

[root@mysqlvm1 ~]#
```
* from mysqlvm2 to other
```
[root@mysqlvm2 ~]# for i in 1 2 3; do mysql -uclusteradmin -padmin123 -hmysqlvm$i -P3306 -e 'select user(), @@hostname, @@read_only, @@super_read_only'; echo " "; done
mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm2.localdomain | mysqlvm1.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm2.localdomain | mysqlvm2.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm2.localdomain | mysqlvm3.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

[root@mysqlvm2 ~]#
```
* from mysqlvm3 to other
```
[root@mysqlvm3 ~]# for i in 1 2 3; do mysql -uclusteradmin -padmin123 -hmysqlvm$i -P3306 -e 'select user(), @@hostname, @@read_only, @@super_read_only'; echo " "; done
mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm3.localdomain | mysqlvm1.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm3.localdomain | mysqlvm2.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------------------------+----------------------+-------------+-------------------+
| user()                            | @@hostname           | @@read_only | @@super_read_only |
+-----------------------------------+----------------------+-------------+-------------------+
| clusteradmin@mysqlvm3.localdomain | mysqlvm3.localdomain |           0 |                 0 |
+-----------------------------------+----------------------+-------------+-------------------+

[root@mysqlvm3 ~]#
```

<hr >

### Configure the cluster

Login to any instance using the clusteradmin user using the MySQL Shell,we are connecting to mysqlvm1.

```
[root@mysqlvm1 ~]# mysqlsh -uclusteradmin -padmin123
MySQL Shell 8.0.35

Copyright (c) 2016, 2023, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
WARNING: Using a password on the command line interface can be insecure.
Creating a session to 'clusteradmin@localhost'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 15 (X protocol)
Server version: 8.0.35 MySQL Community Server - GPL
No default schema selected; type /\use <schema> to set one.
```

The argument to checkInstanceConfiguration() is the connection data to a MySQL Server instance. The connection data may be specified in the following formats:
* A URI string
* A dictionary with the connection options
We'll use the URI string for this example.

* Check configuration for mysqlvm1 instance

```
 MySQL  localhost:33060+ ssl  JS > dba.checkInstanceConfiguration("clusteradmin@mysqlvm1:3306")
Please provide the password for 'clusteradmin@mysqlvm1:3306': ********
Save password for 'clusteradmin@mysqlvm1:3306'? [Y]es/[N]o/Ne[v]er (default No):
Validating local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm1.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update",
            "current": "COMMIT_ORDER",
            "option": "binlog_transaction_dependency_tracking",
            "required": "WRITESET"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "enforce_gtid_consistency",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "gtid_mode",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "1",
            "option": "server_id",
            "required": "<unique ID>"
        }
    ],
    "status": "error"
}
 MySQL  localhost:33060+ ssl  JS >
```
The command reports that the instance is not ready for InnoDB cluster usage since some settings are not valid and at this stage its ok to proceed we will later let Admin API to fix it as this is our new configuration.

* Check configuration for mysqlvm2 instance

```
 MySQL  localhost:33060+ ssl  JS > dba.checkInstanceConfiguration("clusteradmin@mysqlvm2:3306")
Please provide the password for 'clusteradmin@mysqlvm2:3306': ********
Save password for 'clusteradmin@mysqlvm2:3306'? [Y]es/[N]o/Ne[v]er (default No):
Validating MySQL instance at mysqlvm2.localdomain:3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm2.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update",
            "current": "COMMIT_ORDER",
            "option": "binlog_transaction_dependency_tracking",
            "required": "WRITESET"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "enforce_gtid_consistency",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "gtid_mode",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "1",
            "option": "server_id",
            "required": "<unique ID>"
        }
    ],
    "status": "error"
}
 MySQL  localhost:33060+ ssl  JS >
```

* Check configuration for mysqlvm3 instance
```
 MySQL  localhost:33060+ ssl  JS > dba.checkInstanceConfiguration("clusteradmin@mysqlvm3:3306")
Please provide the password for 'clusteradmin@mysqlvm3:3306': ********
Save password for 'clusteradmin@mysqlvm3:3306'? [Y]es/[N]o/Ne[v]er (default No):
Validating MySQL instance at mysqlvm3.localdomain:3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm3.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update",
            "current": "COMMIT_ORDER",
            "option": "binlog_transaction_dependency_tracking",
            "required": "WRITESET"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "enforce_gtid_consistency",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "OFF",
            "option": "gtid_mode",
            "required": "ON"
        },
        {
            "action": "server_update+restart",
            "current": "1",
            "option": "server_id",
            "required": "<unique ID>"
        }
    ],
    "status": "error"
}
 MySQL  localhost:33060+ ssl  JS >
```
The AdminAPI provides a command to automatically and remotely configure an instance for InnoDB cluster usage: dba.configureInstance(). So the next step is to use dba.configureInstance() on each target instance.

* dba.configureInstance() on mysqlvm1

```
 MySQL  localhost:33060+ ssl  JS > dba.configureInstance("clusteradmin@mysqlvm1:3306")
Please provide the password for 'clusteradmin@mysqlvm1:3306': ********
Save password for 'clusteradmin@mysqlvm1:3306'? [Y]es/[N]o/Ne[v]er (default No):
Configuring local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm1.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
Do you want to perform the required configuration changes? [y/n]: y
Do you want to restart the instance after configuring it? [y/n]: y
Configuring instance...

WARNING: '@@binlog_transaction_dependency_tracking' is deprecated and will be removed in a future release. (Code 1287).
The instance 'mysqlvm1.localdomain:3306' was configured to be used in an InnoDB cluster.
Restarting MySQL...
NOTE: MySQL server at mysqlvm1.localdomain:3306 was restarted.
 MySQL  localhost:33060+ ssl  JS >
```

* dba.configureInstance() on mysqlvm2
```
 MySQL  localhost:33060+ ssl  JS > dba.configureInstance("clusteradmin@mysqlvm2:3306")
Please provide the password for 'clusteradmin@mysqlvm2:3306': ********
Save password for 'clusteradmin@mysqlvm2:3306'? [Y]es/[N]o/Ne[v]er (default No):
Configuring MySQL instance at mysqlvm2.localdomain:3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm2.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
Do you want to perform the required configuration changes? [y/n]: y
Do you want to restart the instance after configuring it? [y/n]: y
Configuring instance...

WARNING: '@@binlog_transaction_dependency_tracking' is deprecated and will be removed in a future release. (Code 1287).
The instance 'mysqlvm2.localdomain:3306' was configured to be used in an InnoDB cluster.
Restarting MySQL...
NOTE: MySQL server at mysqlvm2.localdomain:3306 was restarted.
 MySQL  localhost:33060+ ssl  JS >
```

* dba.configureInstance() on mysqlvm3
```
 MySQL  localhost:33060+ ssl  JS > dba.configureInstance("clusteradmin@mysqlvm3:3306")
Please provide the password for 'clusteradmin@mysqlvm3:3306': ********
Save password for 'clusteradmin@mysqlvm3:3306'? [Y]es/[N]o/Ne[v]er (default No):
Configuring MySQL instance at mysqlvm3.localdomain:3306 for use in an InnoDB cluster...

This instance reports its own address as mysqlvm3.localdomain:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| enforce_gtid_consistency               | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                              | OFF           | ON             | Update read-only variable and restart the server |
| server_id                              | 1             | <unique ID>    | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
Do you want to perform the required configuration changes? [y/n]: y
Do you want to restart the instance after configuring it? [y/n]: y
Configuring instance...

WARNING: '@@binlog_transaction_dependency_tracking' is deprecated and will be removed in a future release. (Code 1287).
The instance 'mysqlvm3.localdomain:3306' was configured to be used in an InnoDB cluster.
Restarting MySQL...
NOTE: MySQL server at mysqlvm3.localdomain:3306 was restarted.
 MySQL  localhost:33060+ ssl  JS >
```
With this all the instances is now ready to be used in an InnoDB cluster!
<hr >

### Initializing the InnoDB Cluster

Next, we connect the Shell to one of the instances we just configured, which will be the seed instance. The seed instance is the one that would hold the initial state of the database, which will be replicated to the other instances as they’re added to the cluster. Seed instance we choose is mysqlvm1
```
 MySQL  localhost:33060+ ssl  JS > var cls = dba.createCluster("mysqlclus")
Dba.createCluster: MySQL server has gone away (MYSQLSH 2006)
The global session got disconnected..
Attempting to reconnect to 'mysqlx://clusteradmin@localhost:33060'..
The global session was successfully reconnected.
 MySQL  localhost:33060+ ssl  JS > var cls = dba.createCluster("mysqlclus")
A new InnoDB Cluster will be created on instance 'mysqlvm1.localdomain:3306'.

Validating instance configuration at localhost:3306...

This instance reports its own address as mysqlvm1.localdomain:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'mysqlvm1.localdomain:3306'. Use the localAddress option to override.

* Checking connectivity and SSL configuration...

Creating InnoDB Cluster 'mysqlclus' on 'mysqlvm1.localdomain:3306'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```

Verify the status of the cluster which we just initialized
```
 MySQL  localhost:33060+ ssl  JS >  cls.status()
{
    "clusterName": "mysqlclus",
    "defaultReplicaSet": {
        "name": "default",
        "primary": "mysqlvm1.localdomain:3306",
        "ssl": "REQUIRED",
        "status": "OK_NO_TOLERANCE",
        "statusText": "Cluster is NOT tolerant to any failures.",
        "topology": {
            "mysqlvm1.localdomain:3306": {
                "address": "mysqlvm1.localdomain:3306",
                "memberRole": "PRIMARY",
                "mode": "R/W",
                "readReplicas": {},
                "replicationLag": "applier_queue_applied",
                "role": "HA",
                "status": "ONLINE",
                "version": "8.0.35"
            }
        },
        "topologyMode": "Single-Primary"
    },
    "groupInformationSourceMember": "mysqlvm1.localdomain:3306"
}
 MySQL  localhost:33060+ ssl  JS >
```
<hr >

### Add Instances to InnoDB Cluster

Now, you need to add replicas to the InnoDB cluster. Often, when a new instance is added to a replica set in a cluster, they will be behind the rest of the ONLINE members and need to catch up to the current state of the seed instance. If the amount of pre-existing data in the seed instance is very large, you may want to clone it or copy that data through a fast method beforehand. Otherwise, Group Replication will perform a sync automatically (this step is called recovery), re-executing all transactions from the seed, as long as they’re in the MySQL binary log. Since the seed instance in this example has little to no data (ie. just the metadata schema and internal accounts) and have binary logging enabled from the beginning, there’s very little that new replicas need to catch up with. Any transactions that were executed in the seed instance will be re-executed in each added replica.

* Add mysqlvm2 
```
 MySQL  localhost:33060+ ssl  JS > cls.addInstance("clusteradmin@mysqlvm2:3306")

NOTE: The target instance 'mysqlvm2.localdomain:3306' has not been pre-provisioned (GTID set is empty). The Shell is unable to decide whether incremental state recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'mysqlvm2.localdomain:3306' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.


Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone): C
Validating instance configuration at mysqlvm2:3306...

This instance reports its own address as mysqlvm2.localdomain:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'mysqlvm2.localdomain:3306'. Use the localAddress option to override.

* Checking connectivity and SSL configuration...
A new instance will be added to the InnoDB Cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Clone based state recovery is now in progress.

NOTE: A server restart is expected to happen as part of the clone process. If the
server does not support the RESTART command or does not come back after a
while, you may need to manually start it back.

* Waiting for clone to finish...
NOTE: mysqlvm2.localdomain:3306 is being cloned from mysqlvm1.localdomain:3306
** Stage DROP DATA: Completed
** Clone Transfer
    FILE COPY  ############################################################  100%  Completed
    PAGE COPY  ############################################################  100%  Completed
    REDO COPY  ############################################################  100%  Completed

NOTE: mysqlvm2.localdomain:3306 is shutting down...

* Waiting for server restart... ready
* mysqlvm2.localdomain:3306 has restarted, waiting for clone to finish...
** Stage RESTART: Completed
* Clone process has finished: 76.82 MB transferred in about 1 second (~76.82 MB/s)

State recovery already finished for 'mysqlvm2.localdomain:3306'

The instance 'mysqlvm2.localdomain:3306' was successfully added to the cluster.

 MySQL  localhost:33060+ ssl  JS >
```

* Add mysqlvm23
```
 MySQL  localhost:33060+ ssl  JS > cls.addInstance("clusteradmin@mysqlvm3:3306")

NOTE: The target instance 'mysqlvm3.localdomain:3306' has not been pre-provisioned (GTID set is empty). The Shell is unable to decide whether incremental state recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'mysqlvm3.localdomain:3306' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.


Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone): C
Validating instance configuration at mysqlvm3:3306...

This instance reports its own address as mysqlvm3.localdomain:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'mysqlvm3.localdomain:3306'. Use the localAddress option to override.

* Checking connectivity and SSL configuration...
A new instance will be added to the InnoDB Cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Clone based state recovery is now in progress.

NOTE: A server restart is expected to happen as part of the clone process. If the
server does not support the RESTART command or does not come back after a
while, you may need to manually start it back.

* Waiting for clone to finish...
NOTE: mysqlvm3.localdomain:3306 is being cloned from mysqlvm2.localdomain:3306
** Stage DROP DATA: Completed
** Clone Transfer
    FILE COPY  ############################################################  100%  Completed
    PAGE COPY  ############################################################  100%  Completed
    REDO COPY  ############################################################  100%  Completed

NOTE: mysqlvm3.localdomain:3306 is shutting down...

* Waiting for server restart... ready
* mysqlvm3.localdomain:3306 has restarted, waiting for clone to finish...
** Stage RESTART: Completed
* Clone process has finished: 76.80 MB transferred in about 1 second (~76.80 MB/s)

State recovery already finished for 'mysqlvm3.localdomain:3306'

The instance 'mysqlvm3.localdomain:3306' was successfully added to the cluster.

 MySQL  localhost:33060+ ssl  JS >
```
### Verify the cluster 
MySQL shell API provides lots of flexibility to manage and view the InnoDB Cluster. We dba package comes with lots of functions which we can use to manage and view cluster status. The below command queries the current status of the InnoDB cluster and produces a report. The status field for each instance should show either ONLINE or RECOVERING. RECOVERING means that the instance is receiving updates from the seed instance and should eventually switch to ONLINE.
```
 MySQL  localhost:33060+ ssl  JS > cluster = dba.getCluster();
<Cluster:mysqlclus>
 MySQL  localhost:33060+ ssl  JS > cluster.status()
{
    "clusterName": "mysqlclus",
    "defaultReplicaSet": {
        "name": "default",
        "primary": "mysqlvm1.localdomain:3306",
        "ssl": "REQUIRED",
        "status": "OK",
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.",
        "topology": {
            "mysqlvm1.localdomain:3306": {
                "address": "mysqlvm1.localdomain:3306",
                "memberRole": "PRIMARY",
                "mode": "R/W",
                "readReplicas": {},
                "replicationLag": "applier_queue_applied",
                "role": "HA",
                "status": "ONLINE",
                "version": "8.0.35"
            },
            "mysqlvm2.localdomain:3306": {
                "address": "mysqlvm2.localdomain:3306",
                "memberRole": "SECONDARY",
                "mode": "R/O",
                "readReplicas": {},
                "replicationLag": "applier_queue_applied",
                "role": "HA",
                "status": "ONLINE",
                "version": "8.0.35"
            },
            "mysqlvm3.localdomain:3306": {
                "address": "mysqlvm3.localdomain:3306",
                "memberRole": "SECONDARY",
                "mode": "R/O",
                "readReplicas": {},
                "replicationLag": "applier_queue_applied",
                "role": "HA",
                "status": "ONLINE",
                "version": "8.0.35"
            }
        },
        "topologyMode": "Single-Primary"
    },
    "groupInformationSourceMember": "mysqlvm1.localdomain:3306"
}
 MySQL  localhost:33060+ ssl  JS >

```
```
 MySQL  localhost:33060+ ssl  JS > cls.describe()
{
    "clusterName": "mysqlclus",
    "defaultReplicaSet": {
        "name": "default",
        "topology": [
            {
                "address": "mysqlvm1.localdomain:3306",
                "label": "mysqlvm1.localdomain:3306",
                "role": "HA"
            },
            {
                "address": "mysqlvm2.localdomain:3306",
                "label": "mysqlvm2.localdomain:3306",
                "role": "HA"
            },
            {
                "address": "mysqlvm3.localdomain:3306",
                "label": "mysqlvm3.localdomain:3306",
                "role": "HA"
            }
        ],
        "topologyMode": "Single-Primary"
    }
}
 MySQL  localhost:33060+ ssl  JS >
```
<hr >

### Deploy MySQL Router
In order for applications to handle failover, they need to be aware of the topology of the InnoDB cluster. They also need to know, at any time, which of the instances is the PRIMARY. While it’s possible for applications to implement that logic by themselves, MySQL Router can do that for you, with minimal work and no code changes in applications.

When a failover occurs, the client application must be aware of the PRIMARY instance and cluster topology. This functionality is handled by the MySQL Router. It routes that redirects the data requests to the available MySQL Server instance. MySQL Router acts as a proxy that is used to hide the multiple MySQL database servers. The concept of the MySQL Router is similar to the Virtual network

The recommended deployment of MySQL Router is on the same host as the application. During bootstrap, MySQL Router needs to connect to the cluster and have privileges to query the performance_schema, mysql_innodb_cluster_metadata and create a restricted, read-only account to be used by itself during normal operation.

The prerequisite for MySQL router is we need to install the package, which we have already done in previous section. Now lets implement it.

Lets create the directory to hold mysqlrouter config. And the ownership should be mysqlrouter for simplicity and seggregation.
```
[root@mysqlvm1 pgdata]# pwd
/pgdata
[root@mysqlvm1 pgdata]# ls -ltr | grep mysqlrou
drwxr-xr-x 2 mysqlrouter mysqlrouter  6 Dec 17 18:27 mysqlrouter
[root@mysqlvm1 pgdata]#
```

Now lets bootstrap mysqlrouter with the metadata, which can be done by calling mysqlrouter with the following command line option from the system’s shell:
```
[root@mysqlvm1 pgdata]# mysqlrouter --bootstrap clusteradmin@mysqlvm1:3306 -d /pgdata/mysqlrouter --user=mysqlrouter
Please enter MySQL password for clusteradmin:
# Bootstrapping MySQL Router 8.0.35 (MySQL Community - GPL) instance at '/pgdata/mysqlrouter'...

- Creating account(s) (only those that are needed, if any)
- Verifying account (using it to run SQL queries that would be run by Router)
- Storing account in keyring
- Adjusting permissions of generated files
- Creating configuration /pgdata/mysqlrouter/mysqlrouter.conf

# MySQL Router configured for the InnoDB Cluster 'mysqlclus'

After this MySQL Router has been started with the generated configuration

    $ mysqlrouter -c /pgdata/mysqlrouter/mysqlrouter.conf

InnoDB Cluster 'mysqlclus' can be reached by connecting to:

## MySQL Classic protocol

- Read/Write Connections: localhost:6446
- Read/Only Connections:  localhost:6447

## MySQL X protocol

- Read/Write Connections: localhost:6448
- Read/Only Connections:  localhost:6449

[root@mysqlvm1 pgdata]#
```

We can below configuration will be created once the bootstrap is done:
```
[root@mysqlvm1 mysqlrouter]# ls -ltr
total 16
drwx------ 2 mysqlrouter mysqlrouter    6 Dec 23 11:37 run
-rw------- 1 mysqlrouter mysqlrouter   90 Dec 23 11:37 mysqlrouter.key
-rwx------ 1 mysqlrouter mysqlrouter  167 Dec 23 11:37 stop.sh
-rwx------ 1 mysqlrouter mysqlrouter  301 Dec 23 11:37 start.sh
-rw------- 1 mysqlrouter mysqlrouter 1950 Dec 23 11:37 mysqlrouter.conf
drwx------ 2 mysqlrouter mysqlrouter   29 Dec 23 11:37 log
drwx------ 2 mysqlrouter mysqlrouter  116 Dec 23 11:37 data
[root@mysqlvm1 mysqlrouter]#
```

Lets start the mysqlrouter service if not already started, using below command in shell.
```
[root@mysqlvm1 mysqlrouter]# ./start.sh
[root@mysqlvm1 mysqlrouter]# PID 2733 written to '/pgdata/mysqlrouter/mysqlrouter.pid'
stopping to log to the console. Continuing to log to filelog

[root@mysqlvm1 mysqlrouter]#
```
```
[root@mysqlvm1 mysqlrouter]# tail -f log/mysqlrouter.log
2023-12-23 11:45:26 routing INFO [7f5a197fa700] [routing:bootstrap_x_ro] started: routing strategy = round-robin-with-fallback
2023-12-23 11:45:26 routing INFO [7f5a197fa700] Start accepting connections for routing routing:bootstrap_x_ro listening on '0.0.0.0:6449'
2023-12-23 11:45:26 routing INFO [7f5a18ff9700] [routing:bootstrap_x_rw] started: routing strategy = first-available
2023-12-23 11:45:26 routing INFO [7f5a18ff9700] Start accepting connections for routing routing:bootstrap_x_rw listening on '0.0.0.0:6448'
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700] Connected with metadata server running on mysqlvm1.localdomain:3306
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700] Potential changes detected in cluster after metadata refresh (view_id=0)
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700] Metadata for cluster 'mysqlclus' has 3 member(s), single-primary:
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700]     mysqlvm1.localdomain:3306 / 33060 - mode=RW
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700]     mysqlvm2.localdomain:3306 / 33060 - mode=RO
2023-12-23 11:45:26 metadata_cache INFO [7f5a58475700]     mysqlvm3.localdomain:3306 / 33060 - mode=RO
```

Now letss careate user to test:
```
[root@mysqlvm1 mysqlrouter]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 971
Server version: 8.0.35 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'testusr'@'%' identified by 'admin123';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL privileges on *.* to 'testusr'@'%' with grant option;
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```

Finally lets verify the connection, all the R/W connection will be routed to Primary instance and all R/O will be routed to Secondary instance when used respective ports.
```
[root@mysqlvm1 mysqlrouter]# mysql -utestusr -p -hmysqlvm1 -P6446 -e 'select user(), @@hostname, @@read_only, @@super_read_only'
Enter password:
+------------------------------+----------------------+-------------+-------------------+
| user()                       | @@hostname           | @@read_only | @@super_read_only |
+------------------------------+----------------------+-------------+-------------------+
| testusr@mysqlvm1.localdomain | mysqlvm1.localdomain |           0 |                 0 |
+------------------------------+----------------------+-------------+-------------------+
[root@mysqlvm1 mysqlrouter]#


[root@mysqlvm1 mysqlrouter]# mysql -utestusr -p -hmysqlvm1 -P6447 -e 'select user(), @@hostname, @@read_only, @@super_read_only'
Enter password:
+------------------------------+----------------------+-------------+-------------------+
| user()                       | @@hostname           | @@read_only | @@super_read_only |
+------------------------------+----------------------+-------------+-------------------+
| testusr@mysqlvm1.localdomain | mysqlvm3.localdomain |           1 |                 1 |
+------------------------------+----------------------+-------------+-------------------+
[root@mysqlvm1 mysqlrouter]#


[root@mysqlvm1 mysqlrouter]# mysql -utestusr -p -hmysqlvm1 -P6447 -e 'select user(), @@hostname, @@read_only, @@super_read_only'
Enter password:
+------------------------------+----------------------+-------------+-------------------+
| user()                       | @@hostname           | @@read_only | @@super_read_only |
+------------------------------+----------------------+-------------+-------------------+
| testusr@mysqlvm1.localdomain | mysqlvm2.localdomain |           1 |                 1 |
+------------------------------+----------------------+-------------+-------------------+
[root@mysqlvm1 mysqlrouter]#
```

### Checking failover
Lets stop mysql service on primary and try to connect
```
[root@mysqlvm1 ~]# systemctl stop mysqld
[root@mysqlvm1 ~]#

[root@mysqlvm1 mysqlrouter]# mysql -utestusr -p -hmysqlvm1 -P6446 -e 'select user(), @@hostname, @@read_only, @@super_read_only'
Enter password:
+------------------------------+----------------------+-------------+-------------------+
| user()                       | @@hostname           | @@read_only | @@super_read_only |
+------------------------------+----------------------+-------------+-------------------+
| testusr@mysqlvm1.localdomain | mysqlvm3.localdomain |           0 |                 0 |
+------------------------------+----------------------+-------------+-------------------+
[root@mysqlvm1 mysqlrouter]#
```
As we can see the there is new instance for Primary role and this is completely transparent to applications, and starting the mysql on the stopped node will automatically join the cluster.

### Restart cluster from complete outage
However, when all the nodes have been shutdown and started nodes may not join the cluster and below errors may be noticed in mysql log:
```
2023-12-23T07:52:51.037345Z 0 [ERROR] [MY-013781] [Repl] Plugin group_replication reported: 'Failed to establish MySQL client connection in Group Replication. Error sending connection delegation command. Please refer to the manual to make sure that you configured Group Replication properly to work with MySQL Protocol connections.'
2023-12-23T07:52:51.038259Z 0 [ERROR] [MY-011735] [Repl] Plugin group_replication reported: '[GCS] Error on opening a connection to peer node mysqlvm2.localdomain:3306 when joining a group. My local port is: 3306.'
2023-12-23T07:52:51.038300Z 0 [ERROR] [MY-011735] [Repl] Plugin group_replication reported: '[GCS] Error connecting to all peers. Member join failed. Local port: 3306'
```

When you try to get the cluster it will throw errors like below:
```
[root@mysqlvm1 mysqlrouter]# mysqlsh -uclusteradmin
Please provide the password for 'clusteradmin@localhost': ********
Save password for 'clusteradmin@localhost'? [Y]es/[N]o/Ne[v]er (default No):
MySQL Shell 8.0.35

Copyright (c) 2016, 2023, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'clusteradmin@localhost'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 358 (X protocol)
Server version: 8.0.35 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
 MySQL  localhost:33060+ ssl  JS > cluster = dba.getCluster();
Dba.getCluster: This function is not available through a session to a standalone instance (metadata exists, instance belongs to that metadata, but GR is not active) (MYSQLSH 51314)
 MySQL  localhost:33060+ ssl  JS >
```

And in this case you may need to reboot the whole cluster as below:

```
 MySQL  localhost:33060+ ssl  JS > dba.rebootClusterFromCompleteOutage('mysqlclus');
Restoring the Cluster 'mysqlclus' from complete outage...

Cluster instances: 'mysqlvm1.localdomain:3306' (OFFLINE), 'mysqlvm2.localdomain:3306' (OFFLINE), 'mysqlvm3.localdomain:3306' (OFFLINE)
Waiting for instances to apply pending received transactions...
Validating instance configuration at localhost:3306...

This instance reports its own address as mysqlvm1.localdomain:3306

Instance configuration is suitable.
* Waiting for seed instance to become ONLINE...
mysqlvm1.localdomain:3306 was restored.
Validating instance configuration at mysqlvm2.localdomain:3306...

This instance reports its own address as mysqlvm2.localdomain:3306

Instance configuration is suitable.
Rejoining instance 'mysqlvm2.localdomain:3306' to cluster 'mysqlclus'...

Re-creating recovery account...
NOTE: User 'mysql_innodb_cluster_636761201'@'%' already existed at instance 'mysqlvm1.localdomain:3306'. It will be deleted and created again with a new password.

* Waiting for the Cluster to synchronize with the PRIMARY Cluster...
** Transactions replicated  #############################################  100%

The instance 'mysqlvm2.localdomain:3306' was successfully rejoined to the cluster.

Validating instance configuration at mysqlvm3.localdomain:3306...

This instance reports its own address as mysqlvm3.localdomain:3306

Instance configuration is suitable.
Rejoining instance 'mysqlvm3.localdomain:3306' to cluster 'mysqlclus'...

Re-creating recovery account...
NOTE: User 'mysql_innodb_cluster_2796149058'@'%' already existed at instance 'mysqlvm1.localdomain:3306'. It will be deleted and created again with a new password.

* Waiting for the Cluster to synchronize with the PRIMARY Cluster...
** Transactions replicated  #############################################  100%

The instance 'mysqlvm3.localdomain:3306' was successfully rejoined to the cluster.

The Cluster was successfully rebooted.

<Cluster:mysqlclus>
 MySQL  localhost:33060+ ssl  JS >
 
 MySQL  localhost:33060+ ssl  JS > cluster.describe()
{
    "clusterName": "mysqlclus",
    "defaultReplicaSet": {
        "name": "default",
        "topology": [
            {
                "address": "mysqlvm1.localdomain:3306",
                "label": "mysqlvm1.localdomain:3306",
                "role": "HA"
            },
            {
                "address": "mysqlvm2.localdomain:3306",
                "label": "mysqlvm2.localdomain:3306",
                "role": "HA"
            },
            {
                "address": "mysqlvm3.localdomain:3306",
                "label": "mysqlvm3.localdomain:3306",
                "role": "HA"
            }
        ],
        "topologyMode": "Single-Primary"
    }
}
 MySQL  localhost:33060+ ssl  JS >

```

### Some useful cmds:

```
shell.connect("clusteradmin@mysqlvm1:3306");
cluster = dba.getCluster();
select * from performance_schema.replication_group_members;
select cluster_name,primary_mode,description from mysql_innodb_cluster_metadata.clusters;
select instance_name, mysql_server_uuid, addresses from  mysql_innodb_cluster_metadata.instances;
dba.rebootClusterFromCompleteOutage('mysqlclus');
cluster.status()
cluster.describe()
```