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
```

Alternatively, we can download from below link too:
* [MySQL Communitiy Edition RPM's](https://dev.mysql.com/downloads/mysql/)
* [MySQL Router RPM](https://dev.mysql.com/downloads/router/)

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

