## MySQL InnoDB Cluster 8.0 - Hands-On & Production Deployment

MySQL being lightweight and easy to use has become one of the most popular database choices among the developers. Starting with release 8 MySQL InnoDB Cluster has been providing out-of-the-box HA solution for MySQL. Here we will be doing a typical InnoDB cluster setup.

<table>
<tr><th>Host Details </th><th>Cluster Architecture</th></tr>
<tr><td>
| Parameter      | Value |
| ----------- | ----------- |
| mysqlvm1.localdomain      | Primary Instance  (R/W)     |
| mysqlvm2.localdomain      | Secondary Instance (R/O)     |
| mysqlvm3.localdomain      | Secondary Instance  (R/O)     |
</td><td>
<img src="imgs/innodb-cluster.png" alt="Cluster Architecture" height="400" style="display: block; margin-left: auto; margin-right: auto;">  
</td></tr> </table>



![Cluster Architecture](imgs/innodb-cluster.png)