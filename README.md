# AWS EKS to Azure AKS Cluster Migration
## What is our purpose: 
Migrate AWS kubernetes cluster to Azure kubernetes cluster included persistent volumes with using Velero DR tool. It is effortless to migrate k8s cluster objects and data under same cloud provider because they are using same components(load balancer, block disks, file shares, container registries and .etc stuff ) but our case is non-identical. In this case we will face different issues(will present the steps) and by-default velero doesn't support different cloud provider migration specially in volume migration. Because cloud providers use different disk snapshot approach but velero also has  restic integration feature  and we can achieve data migration as files with using this feature. I do agree it works slowly if we compare it with disk snaphsot. [You can get in more detail info from this link about restic](https://velero.io/docs/main/restic/).  Restic doesn't support host volumes. You should care about it. 

## What will be our steps:
* Architecture&Prerequisites
* [Spin Up EKS Cluster in AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)           
* [Spin Up AKS Cluster in Azure](https://docs.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli)
* [Setup Azure Blob Storage with Access Keys](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-cli)
* [Enable EFS in AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
* Configure Velero with  EKS 
* Deploy Dummy  Services to EKS(Redis Cluster-StatefullSet, Nginx Deployment, Sample Pod)
* Configure Velero with AKS
* Test Backup and Restore  Operations
* Resources

## Architecture&Prerequisites
I assume that everything is clear when you look at architecture picture. One thing I would like to point out why I choose  Azure Storage Blob as backup location because of our restore endpoint is azure aks therefore restore time will be speed up.  

![Alt text](images/Architecture.png?raw=true "Architecture")
You should pay close attention following things when you start migrate process. 

As mentioned above cloud providers use different block/file storage class name, different container registry endpoints,  ingress rules annotations and .etc. It will cause to some problems when restore happened

### Storage Classes
You should create same storage class name in restore k8s cluster as backup AWS EKS cluster according to block and file share classes. For example you have one block and efs storage classes in EKS cluster name as  gp2 and aws-efs and then maybe these names can't be exist in AKS side that's why you should create these storage classes in AKS side with relevant names in advance migration but behind the scenes  it will point out AKS resources block and file shared resources. 

### Container Images
Perhaps you use ECR as  container registry  in  AWS EKS side and your aks cluster won't be connect AWS ECR registry if you don't make it up. In this scenario either you have to configure AKS with ECR or you import all AWS ECR images to Azure Container Registry and you must use kubectl patch command to update container images url  in your deployments after migrate.  This approach is almost same for ingress rule load balancer annotations.

Above things it is my considerations I payed attention for getting success in backup&restore operations. Maybe you will see different object differences between cloud providers k8s clusters if it is so you must also care about it.


Let's move forward with technical steps. I didn't touch 2,3,4 and 5 steps I put links how to spin  up clusters, [create blob storage and get access keys](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage?tabs=azure-portal) for velero backup location and configure efs in eks for demonstration purposes and will point out only important steps.

So for now we have two kind of cluster. One of them is AWS EKS and Azure AKS.

## Configure Velero with  EKS 
It is my variables to connect Velero in EKS side to Azure Storage Blob Container. [This is official documentation how to configure velero with Azure Storage](https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure).  I used Access key way because it is accessed by AWS EKS as external access and it doesn't support snapshot backup after all we don't need it because we used different cloud k8s clusters for backup/restore purposes. 
```
tenant_id=''
resource_g=''
subs_id=''
saccount_name=''
scontainer_name=''
AZURE_STORAGE_ACCOUNT_ACCESS_KEY='bla_bla_bla'
AZURE_CLOUD_NAME='AzurePublicCloud'

cat << EOF > ./credentials-velero
AZURE_STORAGE_ACCOUNT_ACCESS_KEY=bla_bla_bla
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

# Install Velero to EKS, I already install velero client to my laptop and authenticated to AWS EKS with kubeconfig.
velero install \
--provider azure --plugins velero/velero-plugin-for-microsoft-azure:v1.4.0 \
--bucket velero \
--secret-file ./credentials-velero \
--backup-location-config resourceGroup=$resource_g,storageAccount=$saccount_name,storageAccountKeyEnvVar=AZURE_STORAGE_ACCOUNT_ACCESS_KEY,subscriptionId=bla_bla \
--use-volume-snapshots=false \
--default-volumes-to-restic \
--use-restic true

# If we check velero pods status
kubectl get pods -n velero               
NAME                     READY   STATUS    RESTARTS   AGE
restic-7qwj4             1/1     Running   0          13d
restic-8btvc             1/1     Running   0          13d
restic-hjtq7             1/1     Running   0          13d
restic-hqx9z             1/1     Running   0          13d
restic-rk8v9             1/1     Running   0          13d
velero-c6666b5b4-64hl8   1/1     Running   0          13d
```

## Deploy Dummy  Services to AWS EKS 
I will deploy dummy services in EKS side for demonstration purposes:  
* Redis Cluster As StatefullSet for testing block volume based backup
* Nginx as a deployment  for testing block volume based backup
* Sample Pod for testing EFS volume based backup

### Redis Cluster Deploy
```
# Let's list our storage classes in EKS
kubectl get sc                              
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
aws-efs         example.com/aws-efs     Delete          Immediate              false                  7d4h
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  14d

# Add Help Repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Install Redis, It will use default eks block storage class
helm install redis bitnami/redis -n default

# Lets check our redis instances
kubectl get pods -l app.kubernetes.io/name=redis
NAME               READY   STATUS    RESTARTS   AGE
redis-master-0     1/1     Running   0          14d
redis-replicas-0   1/1     Running   0          14d
redis-replicas-1   1/1     Running   0          14d
redis-replicas-2   1/1     Running   0          14d

kubectl get pvc -l app.kubernetes.io/name=redis      
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
redis-data-redis-master-0     Bound    pvc-bdf9b183-aca4-4a61-bb0a-42ce5c63bd66   8Gi        RWO            gp2            14d
redis-data-redis-replicas-0   Bound    pvc-3e89048a-174c-4264-995c-4ece60af99a9   8Gi        RWO            gp2            14d
redis-data-redis-replicas-1   Bound    pvc-ecc6d427-c1f7-4e87-a08a-cd45197cc09d   8Gi        RWO            gp2            14d
redis-data-redis-replicas-2   Bound    pvc-24b1cb6b-67bf-456a-a9f8-e9cfaa8434b4   8Gi        RWO            gp2            14d

# Export Redis password for connect redis and write some data
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 --decode)

# SpinUp redis client pod to connect redis cluster
kubectl run --namespace default redis-client --restart='Never' --env REDIS_PASSWORD=$REDIS_PASSWORD --image docker.io/bitnami/redis:6.2.7-debian-10-r23 --command -- sleep infinity

# Exec redis client pod and connect to redis service
kubectl exec --tty -i redis-client --namespace default -- bash
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master

#Write some data to redis
redis-master:6379> set A b
redis-master:6379> get A
"b"
```
Everything looks fine, We deployed redis cluster and it used block volumes and we wrote data here. Let's go on deploy other services.

### Deploy Nginx in separate namespace
It will create pvc, deployment and load balancer service under nginx-example namespace and will write nginx access logs to volume mount.
``` 
kubectl apply -f nginx/deploy.yaml 
kubectl  get pods -n nginx-example -l app=nginx  
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-5df9976557-hsbt9   1/1     Running   0          14d

kubectl  get svc -n nginx-example -l app=nginx  
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
nginx-svc   LoadBalancer   10.100.91.209   ac4dfdcee35be4bb395112567d3218a0-100145987.us-east-1.elb.amazonaws.com   80:30048/TCP   14d

kubectl  get pvc -n nginx-example -l app=nginx  
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-logs   Bound    pvc-9ebf4de2-af31-468b-a6fa-e019f7035b33   5Gi        RWO            gp2            14d

# Exec pod and let's see our data in exist
kubectl -n nginx-example  exec -it nginx-deploy-5df9976557-hsbt9    bash 
root@nginx-deploy-5df9976557-hsbt9:/var/log/nginx# cd /var/log/nginx/
root@nginx-deploy-5df9976557-hsbt9:/var/log/nginx# ls
access.log  error.log  gentoo_root.img lost+found
root@nginx-deploy-5df9976557-hsbt9:/var/log/nginx# echo "Hello" >> access.log 
root@nginx-deploy-5df9976557-hsbt9:/var/log/nginx# tail -5 access.log 
192.168.33.159 - - [08/Jun/2022:09:30:03 +0000] "Gh0st\xAD\x00\x00\x00\xE0\x00\x00\x00x\x9CKS``\x98\xC3\xC0\xC0\xC0\x06\xC4\x8C@\xBCQ\x96\x81\x81\x09H\x07\xA7\x16\x95e&\xA7*\x04$&g+\x182\x94\xF6\xB000\xAC\xA8rc\x00\x01\x11\xA0\x82\x1F\x5C`&\x83\xC7K7\x86\x19\xE5n\x0C9\x95n\x0C;\x84\x0F3\xAC\xE8sch\xA8^\xCF4'J\x97\xA9\x82\xE30\xC3\x91h]&\x90\xF8\xCE\x97S\xCBA4L?2=\xE1\xC4\x92\x86\x0B@\xF5`\x0CT\x1F\xAE\xAF]" 400 157 "-" "-" "-"
192.168.59.178 - - [08/Jun/2022:09:39:35 +0000] "GET /?a=fetch&content=<php>die(@md5(HelloThinkCMF))</php> HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36" "-"
192.168.4.62 - - [08/Jun/2022:09:42:40 +0000] "GET /.env HTTP/1.1" 404 555 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36" "-"
192.168.10.61 - - [08/Jun/2022:09:42:42 +0000] "POST / HTTP/1.1" 405 559 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.129 Safari/537.36" "-"
Hello
192.168.10.61 - - [08/Jun/2022:09:42:42 +0000] "POST / HTTP/1.1" 405 559 "-" "Mozilla/5.0
``` 
### Deploy Sample pod which will use efs storage class
I already configured EFS in AWS and deploy efs driver to k8s with helm.
``` 
# Add helm repo and install efs driver, Before that you should configure efs related stuff in AWS. I put link in above
helm repo add isotoma https://isotoma.github.io/charts
helm install -f efs-sc/values.yaml my-efs-provisioner isotoma/efs-provisioner --version 0.13.3

# Apply sample pod , will use pvc whihc bind to our efs and will write data there.
kubectl apply -f efs-sc/efs-app.yaml

kubectl get pods -n default -l sample=pod                                    
NAME      READY   STATUS    RESTARTS   AGE
efs-app   1/1     Running   0          7d4h

# We see it is already bonded to EFS.
kubectl get pvc -n default | grep my-efs-vol-1
my-efs-vol-1                  Bound    pvc-a0fa51d9-2988-4af9-a53d-c5781bca1876   1Gi        RWX            aws-efs        7d4h

# Exec and check real data
kubectl  -n default exec -it efs-app    bash 

[root@efs-app /]# cd data/
[root@efs-app data]# tail -5 out.txt
Wed Jun 8 10:12:37 UTC 2022
Wed Jun 8 10:12:42 UTC 202
Wed Jun 8 10:12:47 UTC 2022
Wed Jun 8 10:12:52 UTC 2022
Wed Jun 8 10:12:57 UTC 2022
``` 

## Configure Velero with AKS
Indeed It is really same steps which we did in EKS therefore I won't do same steps. I  only different thing I  will do velero in aks as read only connection in interaction with Azure Blob Storage. Because It will only get files from there. it is just protection of backup files.
``` 
# Check pods and update connection to read-only state
kubeclt get pods -n velero                          
NAME                     READY   STATUS    RESTARTS        AGE
restic-jlcth             1/1     Running   2 (6d13h ago)   7d5h
restic-lvpn7             1/1     Running   0               7d5h
restic-smv8g             1/1     Running   0               7d5h
velero-c6666b5b4-rtl7v   1/1     Running   2 (6d13h ago)   7d5h

kubectl patch backupstoragelocation default -n velero --type merge --patch '{"spec":{"accessMode":"ReadOnly"}}'

# Create storage classes in AKS as exact name in EKS but it will point out Azure related resources.
Basically I exported exist SC from AKS and update name and apply again
kubectl apply -f aks/aks-block-sc.yaml
kubectl apply -f aks/aks-file-share-sc.yaml

# List Storage Classes in AKS side.
kubeclt get sc          
NAME                    PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
aws-efs                 file.csi.azure.com   Delete          Immediate              true                   7d4h
azurefile               file.csi.azure.com   Delete          Immediate              true                   13d
azurefile-csi           file.csi.azure.com   Delete          Immediate              true                   13d
azurefile-csi-premium   file.csi.azure.com   Delete          Immediate              true                   13d
azurefile-premium       file.csi.azure.com   Delete          Immediate              true                   13d
default (default)       disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
gp2 (default)           disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
managed                 disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
managed-csi             disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
managed-csi-premium     disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
managed-premium         disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   13d
```
Now Azure AKS side configured and looks fine.  Let's backup AWS EKS services to Azure Blob Storage and check backup finished.

## Test Backup and Restore  Operations
### Backup eks services in AWS side.
``` 
# Backup samplepod with label, nginx with full namespace and redis with labels(master and replica)
➜ aws-eks velero backup create sampleappwithefsworking --selector sample=pod
➜ aws-eks velero backup create nginxbackuplatest --include-namespaces nginx-example
➜ aws-eks velero backup create redisbackup --selector app.kubernetes.io/name=redis
# List backups and statuses
➜ aws-eks velero backup get
NAME STATUS ERRORS WARNINGS CREATED EXPIRES STORAGE LOCATION SELECTOR
redisbackup Completed 0 0 2022-06-08 14:35:56 +0400 +04 29d default app.kubernetes.io/name=redis
nginxbackuplatest Completed 0 0 2022-05-25 14:44:46 +0400 +04 16d default <none>
sampleappwithefsworking Completed 0 0 2022-06-01 09:41:01 +0400 +04 22d default sample=pod

# Describe one of the backup object in more details.If you look at result of command you will see which objects backup.
➜ aws-eks velero backup describe redisbackup --details
Name: redisbackup
Namespace: velero
Labels: velero.io/storage-location=default
Annotations: velero.io/source-cluster-k8s-gitversion=v1.21.9-eks-14c7a48
velero.io/source-cluster-k8s-major-version=1
velero.io/source-cluster-k8s-minor-version=21+

Phase: InProgress

Errors: 0
Warnings: 0

Namespaces:
Included: *
Excluded: <none>

Resources:
Included: *
Excluded: <none>
Cluster-scoped: auto
Label selector: app.kubernetes.io/name=redis
Storage Location: default
Velero-Native Snapshot PVs: auto
TTL: 720h0m0s
Hooks: <none>
Backup Format Version: 1.1.0

Started: 2022-06-08 14:35:56 +0400 +04
Completed: <n/a>

Expiration: 2022-07-08 14:35:56 +0400 +04

Estimated total items to be backed up: 32
Items backed up so far: 9

Resource List: <backup resource list not found>

Velero-Native Snapshots: <none included>

Restic Backups:
Completed:
default/redis-master-0: redis-data, redis-tmp-conf, tmp
default/redis-replicas-0: redis-data, redis-tmp-conf
default/redis-replicas-1: redis-data, redis-tmp-conf
In Progress:
default/redis-replicas-2: redis-data
New:
default/redis-replicas-2: redis-tmp-conf
```

Then you should verify Azure Blob Storage which backup files uploaded successfully.In my case everything looks fine as you can see.

![Alt text](images/blobstorage01.png?raw=true "BlobContent")
![Alt text](images/blobstorage02.png?raw=true "BlobContent")

### Let's restore in AKS side
``` 
➜ azure-aks velero restore create --from-backup sampleappwithefsworking
➜ azure-aks velero restore create --from-backup nginxbackuplatest
➜ azure-aks velero restore create --from-backup redisbackup
# List restore and look at status , you should see "Completed" Status
➜ azure-aks velero restore get
# Describe one of the restore
➜ azure-aks velero restore describe  sampleappwithefsworking-20220601094145  --details
Name:         sampleappwithefsworking-20220601094145
Namespace:    velero
Labels:       <none>
Annotations:  <none>

Phase:                       Completed
Total items to be restored:  3
Items restored:              3

Started:    2022-06-01 09:41:46 +0400 +04
Completed:  2022-06-01 09:41:54 +0400 +04

Backup:  sampleappwithefsworking

Namespaces:
  Included:  all namespaces found in the backup
  Excluded:  <none>
Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Restic Restores:
  Completed:
    default/efs-app: persistent-storage

Preserve Service NodePorts:  auto
```
Now If you list pods, pvc and  check data in volumes regarding AWS EKS , you will see backup data and that it created objects successfully. 


# Resources
https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure <br />
https://blog.kubernauts.io/backup-and-restore-pvcs-using-velero-with-restic-and-openebs-from-baremetal-cluster-to-aws-d3ac54386109 <br />
https://faun.pub/clone-migrate-data-between-kubernetes-clusters-with-velero-e298196ec3d8 <br />
https://www.qloudx.com/velero-for-kubernetes-backup-restore-stateful-workloads-with-restic-for-velero/ <br />
https://docs.microsoft.com/en-us/azure/aks/aks-migration<br />
https://nyadav251.medium.com/migrate-your-kubernetes-from-aks-to-eks-how-99da6b37be27 <br />
https://pumpingco.de/blog/backup-and-restore-a-kubernetes-cluster-with-state-using-velero/<br />
https://velero.io/docs/v1.1.0/azure-config/<br />
https://medium.com/egen/backing-up-aks-cluster-with-velero-b1cec289438f<br />
https://faun.pub/clone-migrate-data-between-kubernetes-clusters-with-velero-e298196ec3d8<br />

Have Fun :) 