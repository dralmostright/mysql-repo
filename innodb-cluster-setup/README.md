## MySQL InnoDB Cluster 8.0 - Hands-On & Production Deployment

MySQL being lightweight and easy to use has become one of the most popular database choices among the developers. Starting with release 8 MySQL InnoDB Cluster has been providing out-of-the-box HA solution for MySQL. Here we will be doing a typical InnoDB cluster setup.

<table>
<tr><th>Host Details </th><th>Cluster Architecture</th></tr>
<tr><td>

| Parameter | Value |
| ----------- | ----------- |
| mysqlvm1.localdomain| Primary Instance  |
| mysqlvm2.localdomain| Secondary Instance  |
| mysqlvm3.localdomain| Secondary Instance  |

</td>
<td>

<img src="imgs/innodb-cluster.png" alt="Cluster Architecture" height="400" style="display: block; margin-left: auto; margin-right: auto;"> 

</td>
</tr> </table>

<table>
<tr><th>Primary </th><th>Standby</th></tr>
<tr><td>

| Parameter      | Value |
| ----------- | ----------- |
| Version      | 19.3       |
| Database Type   | Standalone        |
| ASM | NO |
|Source host | mysqlvm1.localdomain|
|Source database |orcl|
|Source unique name |orcl|

</td><td>

| Parameter      | Value |
| ----------- | ----------- |
| Version      | 19.3       |
| Database Type   | Standalone        |
|ASM | NO|
|Source host | mysqlvm4.localdomain|
|Source database |orcl|
|Source unique name |orcldr|

</td></tr> </table>

<img src="imgs/innodb-cluster.png" alt="Cluster Architecture" height="400" style="display: block; margin-left: auto; margin-right: auto;">  

