
# MySQL High Availability Cluster with ProxySQL in Kubernetes

This document outlines the steps to deploy a MySQL cluster with a single Master, two Slaves, and ProxySQL for load balancing in Kubernetes. The setup uses `StatefulSet` for MySQL instances and a `Deployment` for ProxySQL, with replication configured using GTID (Global Transaction Identifiers).

### Prerequisites
- Kubernetes cluster (e.g., K3s, Minikube, or any managed service)
- `kubectl` configured to interact with the cluster
- StorageClass available (e.g., `local-path` for K3s/Minikube)
- Namespace `mysql-ha` created:
  ```bash
  kubectl create namespace mysql-ha
###Architecture
Master: mysql-master-0 (server-id=1)
Slaves: mysql-slave-0 (server-id=2), mysql-slave-1 (server-id=3)
ProxySQL: Load balancer distributing writes to Master and reads to Slaves
###Step 1: Deploy MySQL Master

```mysql-master.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-master
  namespace: mysql-ha
spec:
  serviceName: "mysql-master"
  replicas: 1
  selector:
    matchLabels:
      app: mysql-master
  template:
    metadata:
      labels:
        app: mysql-master
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpass"
        - name: MYSQL_REPLICATION_USER
          value: "repl_user"
        - name: MYSQL_REPLICATION_PASSWORD
          value: "repl_pass"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        args:
        - "--server-id=1"
        - "--log-bin=mysql-bin"
        - "--gtid-mode=ON"
        - "--enforce-gtid-consistency=ON"
        - "--innodb-buffer-pool-size=700M"
        - "--expire_logs_days=1"
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
          limits:
            memory: "1.5Gi"
            cpu: "2"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  namespace: mysql-ha
spec:
  clusterIP: None
  ports:
  - port: 3306
  selector:
    app: mysql-master
Apply:

bash


kubectl apply -f mysql-master.yml

###Step 2: Deploy MySQL Slaves
```mysql-slave.yml
yaml


apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-slave
  namespace: mysql-ha
spec:
  serviceName: "mysql-slave"
  replicas: 2
  selector:
    matchLabels:
      app: mysql-slave
  template:
    metadata:
      labels:
        app: mysql-slave
    spec:
      initContainers:
      - name: set-server-id
        image: busybox
        command: ["sh", "-c", "POD_ORDINAL=${POD_NAME##*-}; echo $((POD_ORDINAL + 2)) > /tmp/server-id"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: server-id
          mountPath: /tmp
      - name: init-mysql
        image: mysql:8.0
        command: ["sh", "-c"]
        args:
        - |
          if [ ! -d /var/lib/mysql/mysql ]; then
            mysqld --initialize-insecure --datadir=/var/lib/mysql
          fi
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpass"
        - name: MYSQL_REPLICATION_USER
          value: "repl_user"
        - name: MYSQL_REPLICATION_PASSWORD
          value: "repl_pass"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: server-id
          mountPath: /tmp
        command: ["sh", "-c"]
        args:
        - "SERVER_ID=$(cat /tmp/server-id); mysqld --server-id=$SERVER_ID --log-bin=mysql-bin --gtid-mode=ON --enforce-gtid-consistency=ON --read-only=1 --innodb-buffer-pool-size=700M --expire_logs_days=1"
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
          limits:
            memory: "1.5Gi"
            cpu: "2"
      volumes:
      - name: server-id
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-slave
  namespace: mysql-ha
spec:
  clusterIP: None
  ports:
  - port: 3306
  selector:
    app: mysql-slave
Apply:

bash


kubectl apply -f mysql-slave.yml
Step 3: Deploy ProxySQL
proxysql.yml
yaml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: proxysql
  namespace: mysql-ha
spec:
  replicas: 1
  selector:
    matchLabels:
      app: proxysql
  template:
    metadata:
      labels:
        app: proxysql
    spec:
      containers:
      - name: proxysql
        image: proxysql/proxysql:2.5
        ports:
        - containerPort: 6033
---
apiVersion: v1
kind: Service
metadata:
  name: proxysql
  namespace: mysql-ha
spec:
  ports:
  - port: 3306
    targetPort: 6033
  selector:
    app: proxysql
Apply:

bash


kubectl apply -f proxysql.yml
Step 4: Configure Replication
Set up replication user on Master:
bash


kubectl exec -it -n mysql-ha mysql-master-0 -- mysql -u root -prootpass
sql


CREATE USER 'repl_user'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'repl_pass';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
Configure Slaves:
For mysql-slave-0:
bash


kubectl exec -it -n mysql-ha mysql-slave-0 -- mysql -u root -prootpass
sql


CHANGE MASTER TO 
    MASTER_HOST = 'mysql-master-0.mysql-master.mysql-ha.svc.cluster.local',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'repl_pass',
    MASTER_AUTO_POSITION = 1;
START SLAVE;
SHOW SLAVE STATUS\G
For mysql-slave-1:
bash


kubectl exec -it -n mysql-ha mysql-slave-1 -- mysql -u root -prootpass
sql


CHANGE MASTER TO 
    MASTER_HOST = 'mysql-master-0.mysql-master.mysql-ha.svc.cluster.local',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'repl_pass',
    MASTER_AUTO_POSITION = 1;
START SLAVE;
SHOW SLAVE STATUS\G
Fix root password on Slaves (if initialized without password):
bash


kubectl exec -it -n mysql-ha mysql-slave-0 -- mysql -u root
sql


ALTER USER 'root'@'localhost' IDENTIFIED BY 'rootpass';
FLUSH PRIVILEGES;
Repeat for mysql-slave-1.
Step 5: Configure ProxySQL
Access ProxySQL:
bash


kubectl exec -it -n mysql-ha $(kubectl get pod -n mysql-ha -l app=proxysql -o jsonpath="{.items[0].metadata.name}") -- mysql -u admin -padmin -h 127.0.0.1 -P 6032
Add servers:
sql


INSERT INTO mysql_servers (hostgroup_id, hostname, port) 
VALUES (10, 'mysql-master-0.mysql-master.mysql-ha.svc.cluster.local', 3306),
       (20, 'mysql-slave-0.mysql-slave.mysql-ha.svc.cluster.local', 3306),
       (20, 'mysql-slave-1.mysql-slave.mysql-ha.svc.cluster.local', 3306);
Set replication hostgroups:
sql


INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup, comment) 
VALUES (10, 20, 'mysql-replication');
Add user:
sql


INSERT INTO mysql_users (username, password, default_hostgroup) 
VALUES ('root', 'rootpass', 10);
Create monitor user on MySQL servers: On Master and Slaves:
sql


CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';
GRANT SELECT, REPLICATION CLIENT ON *.* TO 'monitor'@'%';
FLUSH PRIVILEGES;
Apply changes:
sql


LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL USERS TO DISK;
Step 6: Test the Setup
Test replication: On Master:
bash


kubectl exec -it -n mysql-ha mysql-master-0 -- mysql -u root -prootpass
sql


CREATE DATABASE test_db;
USE test_db;
CREATE TABLE test_table (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test_table (name) VALUES ('test1');
Check on Slaves:
bash


kubectl exec -it -n mysql-ha mysql-slave-0 -- mysql -u root -prootpass -e "SELECT * FROM test_db.test_table"
Test ProxySQL:
bash


kubectl exec -it -n mysql-ha mysql-slave-0 -- mysql -u root -prootpass -h proxysql.mysql-ha.svc.cluster.local -P 3306 -e "SELECT * FROM test_db.test_table"
Troubleshooting
CrashLoopBackOff: Check logs with kubectl logs -n mysql-ha <pod-name>.
Replication errors: Use SHOW SLAVE STATUS\G on Slaves.
ProxySQL timeouts: Verify mysql_servers and stats_mysql_connection_pool:
sql


SELECT * FROM mysql_servers;
SELECT * FROM stats_mysql_connection_pool;
Next Steps
Security: Use Kubernetes Secrets for passwords.
Monitoring: Deploy Prometheus and Grafana with MySQL Exporter.
Backup: Implement a backup strategy.
