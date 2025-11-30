# Deploy-WordPress-site-with-my-SQL-database-on-aws
DEPI Project 




1- Create User with permissions (EBS, EFS):

create user with permissions ("AmazonEBSCSIDriverPolicy", "AmazonElasticFileSystemFullAccess")


2- create access key for user:


3- create role and Add permissions (EBS, EFS):
create role and Add permissions "AmazonEBSCSIDriverPolicy" and ...

---
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:DescribeFileSystems",
                "elasticfilesystem:DescribeMountTargets",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:CreateAccessPoint"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:RequestTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:TagResource"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": "elasticfilesystem:DeleteAccessPoint",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        }
    ]
}
---


ubuntu@master:~$ kubectl get csidriver
NAME              ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
ebs.csi.aws.com   true             false            false             <unset>         false               Persistent   34d
efs.csi.aws.com   false            false            false             <unset>         false               Persistent   27h


ubuntu@master:~$ kubectl get csinode
NAME     DRIVERS   AGE
master   2         69d
node1    2         69d
node2    2         69d


ubuntu@master:~$ kubectl create secret generic onlineshop-project \
     --namespace kube-system \
     --from-literal "key_id=AK************" \
     --from-literal "access_key=ie**********"
secret/dolfined-project created



#############
MySQL - EBS
#############

ubuntu@master:~$ echo -n 'myrootpassword' | openssl base64
bXlyb290cGFzc3dvcmQ=

## create a folder for project


vim secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: bXlzcWwtcGFzc3dvcmQ=
---

ubuntu@master:~/wordpress-project$ kubectl apply -f secret.yaml
secret/mysql-pass created

ubuntu@master:~/wordpress-project$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-mrvgj   kubernetes.io/service-account-token   3      69d
mysql-pass            Opaque                                1      20s


vim mysql-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
---


ubuntu@master:~/wordpress-project$ kubectl apply -f mysql-sc.yaml
storageclass.storage.k8s.io/mysql-sc created
 
ubuntu@master:~/wordpress-project$ kubectl get sc
NAME       PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
mysql-sc   ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  6s



vim mysql-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-sc
  resources:
    requests:
      storage: 5Gi
---

ubuntu@master:~/wordpress-project$ kubectl apply -f mysql-pvc.yaml
persistentvolumeclaim/mysql-pvc created

ubuntu@master:~/wordpress-project$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Pending                                      mysql-sc       20s
## Pending

vim mysql-app.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-app
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---


ubuntu@master:~/wordpress-project$ kubectl apply -f mysql-app.yaml
deployment.apps/wordpress-mysql created

ubuntu@master:~/wordpress-project$ kubectl get po
NAME                            READY   STATUS              RESTARTS   AGE
wordpress-mysql-dffcfcb-6cwd4   0/1     ContainerCreating   0          5s
# waiting for creating volume ebs on aws

ubuntu@master:~/wordpress-project$ kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
wordpress-mysql-dffcfcb-6cwd4   1/1     Running   0          47s


### Open AWS to show ebs volume ###

# Open KubeView #


ubuntu@master:~/wordpress-project$ kubectl get sc
NAME       PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
mysql-sc   ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  7m57s

ubuntu@master:~/wordpress-project$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound    pvc-67a5cf68-3568-4235-b552-c6b42228805a   5Gi        RWO            mysql-sc       6m3s

ubuntu@master:~/wordpress-project$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-67a5cf68-3568-4235-b552-c6b42228805a   5Gi        RWO            Delete           Bound    default/mysql-pvc   mysql-sc                2m25s


vim mysql-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
---

ubuntu@master:~/wordpress-project$ kubectl apply -f mysql-svc.yaml
service/wordpress-mysql created


ubuntu@master:~/wordpress-project$ kubectl get svc
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP    35d
wordpress-mysql   ClusterIP   10.101.144.152   <none>        3306/TCP   5s

# Open KubeView #



################
wordPress - EFS
################


1. Creste filesystem for "wordpress"
2. create "access point" Select your file system ID  for "wordpress" file system
3. Enter this values:
Root directory path: /wordpress

Posix User ID: 1000

Posix Group ID: 1000

Root Owner user ID: 1000

Root group ID: 1000

POSIX permissions: 777

Select Create access point

4. change security group for filesystem



vim wordpress-sc.yaml
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
---

ubuntu@master:~/wordpress-project$ kubectl get sc
NAME       PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
efs-sc     efs.csi.aws.com   Delete          Immediate              false                  9s
mysql-sc   ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  16m


vim wordpress-pv.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-079c8df4fe1f84069::fsap-039e81d78cbb409da
---


ubuntu@master:~/wordpress-project$ kubectl apply -f wordpress-pv.yaml
persistentvolume/wordpress-efs-pv created

ubuntu@master:~/wordpress-project$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
pvc-67a5cf68-3568-4235-b552-c6b42228805a   5Gi        RWO            Delete           Bound       default/mysql-pvc   mysql-sc                13m
wordpress-efs-pv                           5Gi        RWX            Retain           Available                       efs-sc                  5s
### status: Available


vim wordpress-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-efs-pvc
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---

ubuntu@master:~/wordpress-project$ kubectl apply -f wordpress-pvc.yaml
persistentvolumeclaim/wordpress-efs-pvc created

ubuntu@master:~/wordpress-project$ kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc           Bound    pvc-67a5cf68-3568-4235-b552-c6b42228805a   5Gi        RWO            mysql-sc       18m
wordpress-efs-pvc   Bound    wordpress-efs-pv                           5Gi        RWX            efs-sc         5s

ubuntu@master:~/wordpress-project$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   REASON   AGE
pvc-67a5cf68-3568-4235-b552-c6b42228805a   5Gi        RWO            Delete           Bound    default/mysql-pvc           mysql-sc                15m
wordpress-efs-pv                           5Gi        RWX            Retain           Bound    default/wordpress-efs-pvc   efs-sc                  111s


vim wordpress-app.yaml
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:php7.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-svc
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-efs-pvc
---

ubuntu@master:~/wordpress-project$ kubectl apply -f wordpress-app.yaml
deployment.apps/wordpress created


ubuntu@master:~/wordpress-project$ kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
wordpress-66ffbf64df-qdj5m      1/1     Running   0          6s
wordpress-mysql-dffcfcb-6cwd4   1/1     Running   0          19m

ubuntu@master:~/wordpress-project$ kubectl get deploy
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
wordpress         1/1     1            1           25s
wordpress-mysql   1/1     1            1           19m

ubuntu@master:~/wordpress-project$ kubectl scale deploy wordpress --replicas=2
deployment.apps/wordpress scaled

ubuntu@master:~/wordpress-project$ kubectl get deploy
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
wordpress         1/2     2            1           104s
wordpress-mysql   1/1     1            1           21m

ubuntu@master:~/wordpress-project$ kubectl get deploy
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
wordpress         2/2     2            2           107s
wordpress-mysql   1/1     1            1           21m



vim wordpress-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---

ubuntu@master:~/wordpress-project$ kubectl apply -f wordpress-svc.yaml
service/wordpress created

ubuntu@master:~/wordpress-project$ kubectl get svc
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP      10.96.0.1        <none>        443/TCP        35d
wordpress         LoadBalancer   10.96.200.129    <pending>     80:32298/TCP   6s
wordpress-mysql   ClusterIP      10.101.144.152   <none>        3306/TCP       20m



>>> try to access wordpress app <Node-ip>:<svc-port>


---
