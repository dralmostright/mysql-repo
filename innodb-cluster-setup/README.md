## MySQL InnoDB Cluster 8.0 - Hands-On & Production Deployment

MySQL being lightweight and easy to use has become one of the most popular database choices among the developers. Starting with release 8 MySQL InnoDB Cluster has been providing out-of-the-box HA solution for MySQL. Here we will be doing a typical InnoDB cluster setup.


| Parameter      | Value |
| ----------- | ----------- |
| mysqlvm1.localdomain      | Primary Instance  (R/W)     |
| mysqlvm2.localdomain      | Secondary Instance (R/O)     |
| mysqlvm3.localdomain      | Secondary Instance  (R/O)     |

![Cluster Architecture](imgs/innodb-cluster.png)