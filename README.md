# MariaDB server operator for Kubernetes

## Features
* Setup a MariaDB server. Database server version can be configured in the CR file.
* Creates a new custom Database along with a user credential set for the custom database.
* Operator uses Persistent Volume where MariaDB can write its data files.
* Seamless upgrades of MariaDB is possible without loosing data.
* Take database backup at defined intervals


## CRs

### MariaDB
```yaml
apiVersion: mariadb.persistentsys/v1alpha1
kind: MariaDB
metadata:
  name: mariadb
spec:
  # Keep this parameter value unchanged.
  size: 1
  
  # Root user password
  rootpwd: password

  # New Database name
  database: test-db
  # Database additional user details (base64 encoded)
  username: db-user 
  password: db-user 

  # Image name with version
  image: "mariadb/server:10.3"

  # Database storage Path
  dataStoragePath: "/mnt/data" 

  # Database storage Size (Ex. 1Gi, 100Mi)
  dataStorageSize: "1Gi"
```
This CR with create a database called `test-db`, along with user credentials.
The Server image name is mentioned in "image" parameter.
Database files will be stored at location: '/mnt/data'. This location should be created before applying the CR.

### MariaDB Backup CR
```yaml
apiVersion: mariadb.persistentsys/v1alpha1
kind: Backup
metadata:
  name: mariadb-backup
spec:
  # Backup Path
  backupPath: "/mnt/backup"

  # Backup Size (Ex. 1Gi, 100Mi)
  backupSize: "1Gi" 

  # Schedule period for the CronJob.
  # This spec allow you setup the backup frequency
  # Default: "0 0 * * *" # daily at 00:00
  schedule: "0 0 * * *"

```
This CR will schedule backup of MariaDB at defined schedule.
The Database backup files will be stored at location: '/mnt/backup'. This location should be created before applying the CR. 


## Setup Instructions
MariaDB Database uses external location on host to store all DB files. This location is default set to "/mnt/data" in CR file. 
Similarly Database backup files are stored at default location "/mnt/backup" and can be configured in Backup CR file.
Ensure that these paths exist and have all necessary permissions.

Check if there are no existing Persistent Volumes defined for same locations. If so, delete those PVs before applying CRs

### Start operator and create all resources
Run the following make command to start all resources:
```
# make install
```
By default, all resources will be created in a namespace called "mariadb"

### Verify Deployment
Verify list of pods. One Operator and One Server pod should be created.
```
# kubectl get pods -n mariadb
NAME                              READY   STATUS    RESTARTS   AGE
mariadb-operator-78c95468-m824g   1/1     Running   0          118s
mariadb-server-778b9b7cb5-nt6n5   1/1     Running   0          109s
```

Verify that "mariadb-service" is created.
```
# kubectl get svc -n mariadb
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
mariadb-backup-service     ClusterIP   10.96.69.127    <none>        3306/TCP            103s
mariadb-operator-metrics   ClusterIP   10.110.31.195   <none>        8383/TCP,8686/TCP   105s
mariadb-service            NodePort    10.102.17.13    <none>        80:30685/TCP        104s
```
Service "mariadb-service" is a NodePort Service that exposes mariadb service on port 3306
```
# kubectl describe service/mariadb-service -n mariadb
Name:                     mariadb-service
Namespace:                mariadb
Labels:                   MariaDB_cr=mariadb
                          app=MariaDB
                          tier=mariadb
Annotations:              <none>
Selector:                 MariaDB_cr=mariadb,app=MariaDB,tier=mariadb
Type:                     NodePort
IP:                       10.100.40.161
Port:                     <unset>  80/TCP
TargetPort:               3306/TCP
NodePort:                 <unset>  30685/TCP
Endpoints:                172.17.0.5:3306
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Test connectivity to MariaDB Server:
```
# mysql -h 192.168.29.217 -P 30685 -u db-user -pdb-user
```
where 192.168.29.217 is the minikube IP and 30685 is the configured NodePort.
If everything is correct, mysql prompt will be presented to the user.

Test Backup Service:
Ensure that cronjob is configured correctly
```
# kubectl get cronjob -n mariadb
NAME             SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
mariadb-backup   0 0 * * *   False     0        <none>          17m
```
At scheduled interval, a new Job will start to take database backup and create a backup file at defined location (default: /mnt/backup)


## Upgrade MariaDB server version
Mariadb server image is mentioned in CR key "image".
To upgrade the Server version, change value of "image" key and reapply the CR YAML file.
```
# kubectl apply -f deploy/crds/mariadb.persistentsys_v1alpha1_mariadb_cr.yaml
```


### Stop Operator and delete all resources
Run the command to delete all resources:
```
# make uninstall
```








