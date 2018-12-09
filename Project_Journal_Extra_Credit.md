# MongoDB Sharding

Below is the architecture that I have implemented for testing MongoDB sharding. This architecture consists of replication of config server and sharded cluster in order to provide high availability to the cluster. This is inspired by a concept of combination of replication and sharding from Chapter 4 in the book 'NoSQL Distilled'

![screen shot 2018-12-08 at 10 27 24 pm](https://user-images.githubusercontent.com/42895383/49694121-82713e00-fb38-11e8-85c3-b6c71364ee36.png)

Steps followed:

1. Create Config Server Replica Sets:

Create AWS instance: config1 and config2
AMI: Amazon Linux AMI 2018.03.0 (HVM), SSD Volume Type
Instance Type: t2.micro
VPC: cmpe281
Network: public subnet
Auto Public IP: enable
Security Group: mongodb-cluster
SG Open Ports: 22, 27019
Key Pair: cmpe281-mongo_config


- ssh to config server1
- sudo vi /etc/yum.repos.d/mongodb-org-4.0.repo
Add  below lines in the file:

[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2013.03/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc

- sudo yum install -y mongodb-org
- Create directory to store configurations and metadata\
  sudo mkdir -p /data/db

- Set the ownership of this folder. Mongod runs in mongod group\
   sudo chown -R mongod:mongod /data/db

- Edit the config file with config server replica set configurations:\
  sudo vi /etc/mongod.conf
- Edit the db path\
   dbPath: /data/db
 - Edit Port\
   port: 27019
- Edit the bind ip line\
   bindip: 0.0.0.0
- Uncomment the replication line  and add value\
   replSetName: crs
-Uncomment the sharding line and add value\
 clusterRole: configsvr

- Run mongo after making config changes:
sudo mongod --config /etc/mongod.conf --logpath /var/log/mongodb/mongod.log

- This will create mongo fork process at port 27019

- check that mongod service is listening on port 27019\
  sudo lsof -iTCP -sTCP:LISTEN | grep mongo

-connect to mongo using port\
 mongo -port 27019

- Repeat the above step 1 for config2 

2. Create the config server replica set\
   Enter mongo shell in config1 and add replica set with config1 as primary and config2 as secondary

rs.initiate(
 {
   _id:"crs",
   configsvr: true,
   members:[
     { _id : 0, host:"config1:27019"},
     { _id : 1, host:"config2:27019"}
    ]
 }


3. Create replica sets of Sharded cluster rs0 and rs1
   Create AMI image from Step1 config1 and edit the mongo config file.
   
  - ssh to shard1 instance
   sudo vi /etc/mongod.conf
- Edit the db path\
   dbPath: /data/db
 - Edit Port\
   port: 27018
- Edit the bind ip line\
   bindip: 0.0.0.0
- Uncomment the replication line  and add value\
   replSetName: rs0
-Uncomment the sharding line and add value\
 clusterRole: shardsvr

- Run mongo after making config changes:
sudo mongod --config /etc/mongod.conf --logpath /var/log/mongodb/mongod.log

- This will create mongo fork process at port 27018

- check that mongod service is listening on port 27018\
  sudo lsof -iTCP -sTCP:LISTEN | grep mongo

-connect to mongo using port\
 mongo -port 27018

- Repeat the above step3 for shard2 as well

4. Create rs0 replica set for Shard cluster 

- ssh to shard1
- mongo -port 27018
rs.initiate(
 {
   _id:"rs0",
   members:[
     { _id : 0, host:"shard1:27018"},
     { _id : 1, host:"shard2:27018"}
    ]
  }
)

![screen shot 2018-12-08 at 7 12 45 pm](https://user-images.githubusercontent.com/42895383/49694060-82bd0980-fb37-11e8-986c-bd72abb51570.png)

5. Perform the step 3 and Step 4 for creating rs1 replica set with shard3 and shard4 cluster instances.
Now we have 1 replica set of 2 config servers and 2 replica sets rs0 and rs1 of the sharded clusters.

We neeed a mongo router to route the mongo requests to shard clusters through config servers.

- Create Mongo router instance from the AMI that we created after step1. 
 
 sudo vi /etc/mongod.conf
- Comment the complete Storage section
- Edit Port\
   port: 27017
- Edit the bind ip line\
   bindip: 0.0.0.0
- Uncomment the sharding line and add value\
   configDB: crs/config1:27019,config2:27019

- Start the mongo process using\
  sudo mongos --config /etc/mongod.conf --fork --logpath /var/log/mongodb/mongod.log

- Run mongo shell\
   mongo -port 27017


- Add the sharded clusters to the router\
  sh.addShard("rs0/shard1:27018,shard2:27018");\
  sh.addShard("rs1/shard3:27018,shard4:27018");
  
 - use testdb
  
  
  

Now we will enable the sharding and perform the test cases.

- Create testdb database\
  use testdb
  
- Use admin to enable sharding on testdb\
  use admin\
  db.runCommand({enablesharding: "testdb"})

- Ensure Hashed index on _id in testdb\
  use testdb\
  db.bios.ensureIndex( { _id: "hashed" } )

-  Create the shard in bios collection on hashed _id
   use admin
   sh.shardCollection( "testdb.bios", { _id: "hashed" } )

- In testdb insert data from the link https://github.com/paulnguyen/cmpe281/blob/master/labs/lab4/bios.js \
  use testdb
  insert data

- Check the distribution of the sharded data in sharded replica sets\
  db.bios.getShardDistribution()


![screen shot 2018-12-08 at 10 19 19 pm](https://user-images.githubusercontent.com/42895383/49694056-6d47df80-fb37-11e8-8c72-6cfae69ce5df.png)

With this I have demonstrated the Sharded Cluster replica set in MongoDB which is a combination of replication and sharding. This combination provides high availability along with z axis scaling in AKF model.


# Kubernetes in AWS with Kops

This section is for implementing Kubernetes in AWS EC2 instances using kops. It is inspired from this link
https://hackernoon.com/7-ways-to-do-containers-on-aws-532f812196f1

Installer-based vanilla Kubernetes on EC2
Below are the steps that I performed:

- brew install python
- pip3 install awscli
- aws configure
- brew update && brew install kops kubectl
- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1
(This gives a location constraint error because the bucket needs location constraint if we use any other region than us-east-1) Below command resolved the error

- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1

![bucket_created](https://user-images.githubusercontent.com/42895383/49691654-2d173b80-fafc-11e8-8177-8c261611b1fd.png)

- The Bucket is created in S3 with this command.

![screen shot 2018-12-08 at 5 10 25 pm](https://user-images.githubusercontent.com/42895383/49692315-44f6bb80-fb0c-11e8-8705-2bf55ec19be6.png)

- aws s3api put-bucket-versioning --bucket payment-kops-state-store  --versioning-configuration Status=Enabled

- sudo vi ~/.bash_profile 
(Add below two environment variable in the file which will be used by kubectl while creating cluster)
export KOPS_CLUSTER_NAME=payment.k8s.local
export KOPS_STATE_STORE=s3://payment-kops-state-store

(Generate Cluster Configuration)
- kops create cluster --node-count=2 --node-size=t2.micro --zones=us-west-1 --name payment.k8s.local
This gives an error: Name: Invalid value: "payment": Cluster Name must be a fully-qualified DNS name (e.g. --name=mycluster.myzone.com)

To resolve this error a secret key rsa is needed which will have the fingerprint of the AWS that we have configured in the first step.
create rsa ssh secret key to create cluster  
- ssh-keygen -t rsa


use this secret key to create custer in AWS
- kops create secret --name payment.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub

(Build the Cluster)
- kops update cluster --name ${KOPS_CLUSTER_NAME} --yes

(Once the Cluster is build check the status of cluster using below command)
- kops validate cluster

![clustercreated](https://user-images.githubusercontent.com/42895383/49691796-2a6a1580-faff-11e8-963a-e4d2bd8c93fb.png)

- kops validate cluster

![kops_validate_cluster_run](https://user-images.githubusercontent.com/42895383/49691831-f3e0ca80-faff-11e8-84b9-259046df42ab.png)

(Check the cluster nodes running as instances in AWS)
- kubectl cluster-info
![kubectl_cluster_info](https://user-images.githubusercontent.com/42895383/49691844-14a92000-fb00-11e8-9737-2cea5abc972c.png)


(Now the Cluster is ready and we can deploy the kubernetes dashboard)
- kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

( This will create the Kubernetes dashboard which can be accessed using below link:)
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

( This link will show the dashboard with nodes from AWS)
![screen shot 2018-12-08 at 3 55 32 pm](https://user-images.githubusercontent.com/42895383/49691911-bd0bb400-fb01-11e8-9633-86ee14fed1d6.png)


- Now, we can create deployment and service using yaml files in the Kubernetes cluster created on AWS.\
 Create Deployment of an api\
kubectl create -f payment-deployment.yaml

![screen shot 2018-12-08 at 4 07 42 pm](https://user-images.githubusercontent.com/42895383/49691985-7454fa80-fb03-11e8-9662-bfc6460b8a6a.png)

- Create Service of the api\
 kubectl create -f payment-service.yaml
 
 ![screen shot 2018-12-08 at 4 08 57 pm](https://user-images.githubusercontent.com/42895383/49691987-9cdcf480-fb03-11e8-8dd4-ca8cb87d1c91.png)
 
 - Running pods from the Payment API deployment 
 
 ![screen shot 2018-12-08 at 4 09 39 pm](https://user-images.githubusercontent.com/42895383/49691990-b1b98800-fb03-11e8-888a-b28a7f358db3.png)



- Kubernetes created its inbuilt load balancer and auto scale groups in the AWS along with a master and 2 worker nodes EC2 instances.

Kubernetes deployed in AWS using kops is highly reliable since it will autoscale the application which are running as containers in the Kubernetes instance and also provide the Kubernetes dashboard. 

