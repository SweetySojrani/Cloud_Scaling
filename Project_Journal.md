# CMPE 281 - Personal NoSQL Project

## Weekly progress and challenge

### Week1: (13-Oct-18- 19-Oct-18)  
 
- Understanding the requirement of the project.
- Read on the Nosql db options for CP and AP and choose one for each
- Started with MongoDB for CP and created replication on Mongodb of 5 nodes on AWS.														


**Challenges: How does network partition work on MongoDB Replication.

### Week2: (19-Oct-18- 27-Oct-18) 

- Successfully created MongoDB replication and topologyof 5 nodes to test network partition in CP.

![mongodb_architecture](https://user-images.githubusercontent.com/42895383/49353574-5276f600-f673-11e8-9d34-beb9348180c3.png)

**Steps to create AP cluster of 5 nodes

**Setup MongoDB Cluster of 5 nodes**

Launch Ubuntu Server 16.04 LTS

1. AMI:             Ubuntu Server 16.04 LTS (HVM)
2. Instance Type:   t2.micro
3. VPC:             cmpe281
4. Network:         public subnet
5. Auto Public IP:  no
6. Security Group:  mongodb-cluster 
7. SG Open Ports:   22, 27017
8. Key Pair:        cmpe281-us-west-1

SSH into Mongo Instance
ssh -i <key>.pem ubuntu@<public ip>

**Install MongoDB**

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4

echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb.list

sudo apt update

sudo apt install mongodb-org=4.0.1 mongodb-org-server=4.0.1 mongodb-org-shell=4.0.1 mongodb-org-mongos=4.0.1 mongodb-org-tools=4.0.1

**MongoDB Install Verification**

sudo systemctl enable mongod
sudo systemctl start mongod 
sudo systemctl stop mongod
sudo systemctl restart mongod
mongod --version 
sudo systemctl enable mongod.service
sudo service mongod restart
sudo service mongod status


**MongoDB Keyfile**
openssl rand -base64 741 > keyFile
sudo mkdir -p /opt/mongodb
sudo cp keyFile /opt/mongodb
sudo chown mongodb:mongodb /opt/mongodb/keyFile
sudo chmod 0600 /opt/mongodb/keyFile

**Config mongod.conf**

sudo vi /etc/mongod.conf

1.  remove or comment out bindIp: 127.0.0.1
    replace with bindIp: 0.0.0.0 (binds on all ips) 

    #network interface
    net:
        port: 27017
        bindIp: 0.0.0.0

2. Uncomment security section & add key file

   security:
        keyFile: /opt/mongodb/keyFile

3. Uncomment Replication section. Name Replica Set = cmpe281

   replication:
     replSetName: cmpe281

4. Create mongod.service

   sudo vi /etc/systemd/system/mongod.service

    [Unit]
        Description=High-performance, schema-free document-oriented database
        After=network.target

    [Service]
        User=mongodb
        ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

    [Install]
        WantedBy=multi-user.target

5. Enable Mongo Service

   sudo systemctl enable mongod.service

6. Restart MongoDB to apply our changes

   sudo service mongod restart
   sudo service mongod status

   Save to AMI Image
   Save the EC2 instance AMI image and create 4 more EC2 instances of MongoDB


**Create Replication set**
1. Edit /etc/hosts in each EC2 Instance adding local host names for Public Ips.
   make the hostname as Primary, Secondary1,Secondary2 as per the topology

- sudo nano /etc/hosts

eg. <Public_ip_address> Primary
    <Public_ip_address> Secondary1

2. Update the hostname:
   - sudo hostnamectl set-hostname Primary
   - sudo hostname -f
   - sudo reboot

   Similarly update the hostname for all the secondary instances as well as Secondary1, Secondary2 and so forth

3. Create the replication set
   Login to Primary: >mongo

   - rs.initiate()

   Show the replication set details:  Below command will show primary added to it.
   - rs.status()


   Note: Make the node as Primary first and then create the user

   - use admin
   - db.createUser( {
        user: "admin",
        pwd: “xxxxxx",
        roles: [{ role: "root", db: "admin" }]
    });


   login using admin
   - db.auth(‘admin’,’xxxxx’);

   Add the secondary nodes to the replication set
   rs.add("Secondary1:27017");\
   rs.add("Secondary2:27017");\
   rs.add("Secondary3:27017");\
   rs.add("Secondary4:27017");

   Check the status of the replication set with members consisting of 1 primary and 4 secondary nodes as per the topology for    testing AP Partition

   rs.status();

**Next steps:** 
- Create the test cases to find the behaviour of the cluster in different scenarios.
- Create a network partition approach.

### Week3: (28-Oct-18- 03-Nov-18)

**Create Network Partition using iptables firewall**
1. Choose a secondary instance that needs to be partitioned from rest of the replication set nodes

2. List the existing iptables on the secondary2 instance using below command
   sudo iptables -S 
   
3. Add iptable firewall rules to the list using below command which will disallow the secondary2 instance from connecting any of the        instances in the replication set
   sudo iptables -A INPUT -s <ip-address> -j DROP
   Add 4 rules one for each of the instance's public ip address for primary, secondary1, secondary3, secondary4, secondary5
   
4. Once the rules are added, run below commands to check the replication set status in Primary node
   mongo
   use admin
   db.auth('admin','xxxxx')
   rs.status()
    
   This will show that the secondary2 node is not reachable/healthy in the replication set and hence not available.
    
   Now that the network partition is created, we can test the experiments on consistency of data.
   

### Week5: (04-Nov-18- 10-Nov-18)

1. Now, we have one primary and 4 secondary members in the replication set. 
  Create a document in the primary mongodb instance:
  >db.race.insert({Id: 1, RaceStatus: 'Start'})
  
  - Query db.race.find() in Primary instance and the 4 secondary instances as well. It is found that the consistent data is read from       all the instances. This is the normal mode of the system. 

2. Create the Network Partition
   Let's consider partition of Secondary2 instance from the system. For demonstration, I have achieved partition using firewall rules on    the mongodb iptables. Below commands will be run on Secondary2 instance which will drop the incoming traffic from Primary,              Secondary3, Secondary4, Secondary5 instances

   sudo iptables -A INPUT -s <ip address of Primary> -j DROP\
   sudo iptables -A INPUT -s <ip address of Secondary3> -j DROP\
   sudo iptables -A INPUT -s <ip address of Secondary4> -j DROP\
   sudo iptables -A INPUT -s <ip address of Secondary5> -j DROP

   Once the firewall rules are created. Run the below command to check the status of replication set using 
   - rs.status()
   The Secondary2 member is now not reachable to the other members of the replication set.So, the network partition is created.

   Create new documents in Primary:
   db.race.insert({ Id: 2, RaceStatus: 'Run' })

   Query the document in all the member instances of the system. It is found that the paritioned instance Secondary2 is still reading      the stale data.Since, secondary2 instance is a slave node and hence a read only node, so new data will not be created on this            instance. 
   To achieve Consistency, 

   Perform the below tests before, during and after partition

   Below are the results of test experiments:


   |Test No | Test Case        | Result                                                                                |
   |--------|------------------|---------------------------------------------------------------------------------------|
   |     1  | Before Partition |Data is consistent across all the members of the replication set                       |
   |     2  | After Partition  |Data is not consistent on the partitioned member                                       |
   |     3  | After Recovery   |Once the partitioned member was recovered, the data became consistent again            |


### Week4: (11-Nov-18- 17-Nov-18)

3. Recover from the Network Partition. Use the below commands to delete the iptables rules that were created in Step2 to create Network    partition. 

   sudo iptables -L --line-numbers\
   sudo iptables -D INPUT 1

   Once the partition is recovered , the Secondary2 instance will become available in the replication set again and read the new            document from Primary instance that was created after the Network Partition.
   
   
   ## AP System with Riak Database
   
   ### Week5: (18-Nov-18- 24-Nov-18)
   
   Riak is explicitly designed to expect server and network failure. Riak is a masterless system meaning any server can respond to read    or write requests. If one fails, others will continue to service client requests. Because Riak’s system allows for reads and writes      when multiple servers are offline or otherwise unreachable, data may not always be consistent across the environment (usually only      for a few milliseconds). However, through self-healing mechanisms like read repair and Active Anti-Entropy, all updates will            propagate to all servers making data eventually consistent. Riak is suitable for many use cases where high availability is more          important than strict consistency. (Referred from Basho.com)
     
   To benefit from the architectural principle that underpin Riak's availability, fault-tolerance, and scaling properties we should use    minimum of 5 nodes. So we have considered below topology with 5 Riak Nodes. 
   
   
   ![riak_cluster_architecture](https://user-images.githubusercontent.com/42895383/49471755-8ad91a00-f7c2-11e8-84dd-7c63c9e7ad04.png)
   
   Create Riak Cluster in AWS with below steps:
   
   In US West(N California) region
   1. Launch Riak AMI from AWS Marketplace
       - AMI:              Riak KV 2.2 Series
       - Instance Type:   t2.micro
       - VPC:            CMPE281_RiakKV
       - Network:         public subnet
       - Auto Public IP:  no
       - Security Group:  riak-cluster 
       - SG Open Ports:   (see below)
       - Key Pair:        cmpe281_KeySept4

   2. Assign below rules to the Security group
      Riak Cluster Security Group (Open Ports):
       - 22 (SSH)
       - 8087 (Riak Protocol Buffers Interface)
       - 8098 (Riak HTTP Interface)

      You will need to add additional rules within this security group to allow your Riak instances to communicate. For each port range       below, create a new Custom TCP rule with the source set to the current security group ID (found on the Details tab).
       - Port range: 4369
        - Port range: 6000-7999
        - Port range: 8099
        - Port range: 9080
   
   3. Create 2 instances in this VPC. Name them Riak1 and Riak2.\
      Similarly Create 3 instances in other region US West (Oregon). Name them Riak3, Riak4 and Riak5.
      Now, Riak1 is the admin of the cluster. SSh to Riak1 and Riak2.\
       
      ssh -i cmpe281_KeySept4.pem ec2-user@<Riak1_ip_address>\
      ssh -i cmpe281_KeySept4.pem ec2-user@<Riak2_ip_address>
       
      In Riak1: 
       - Start Riak
       sudo riak start
        
      In Riak2:
       - Start Riak\
       sudo riak start
       
       - Stage to the admin cluster on Riak1\
       sudo riak-admin cluster join riak@<Riak1_private_ip>
       - This was giving an error that the node is not reachable. On investigating found that below two lines needs to be added to                riak.conf in all the nodes which ressolved the error.\
       erlang.distribution.port_range.minimum = 6000\
       erlang.distribution.port_range.maximum = 7999
       
       
       In Riak1
       - Check the cluster plan which will show the staged changes with Riak2 available to join the admin cluster\
       sudo riak-admin cluster plan
       
       - the below command will show that Riak2 is available to join the cluster.\
       sudo riak-admin cluster status
   
       - this command will commit the joining Riak2 node to valid status\
       sudo riak-admin cluster commit
   
       With this Riak2 has joined the cluster with Riak1. 
     
   4. In order to add Riak3, Riak4 and Riak5 to the cluster, we need to perform below steps to perform VPC peering that will allow             communication between the subnet of VPC in California and the subnet of VPC in Oregon as per the topology created. \
      A VPC peering connection is a networking connection between two VPCs that enables you to route traffic between them using               private IPv4 addresses or IPv6 addresses. Instances in either VPC can communicate with each other as if they are within the same         network.
      - Create VPC peering connection in US West (Oregon) with 'VPC_Riak_Oregon' as source and target as Region: US West(N California)           VPC Id of 'CMPE281_RiakKV' . Note that the subnet group of both the VPC should be different. Otherwise the VPC peering will             fail due to subnet overlap error.
      - After submitting the VPC peering request in Oregon region. Switch to California region, review the VPC peering request and               accept it.
      - Once the VPC peering was established I tried to join Riak3, Riak4 and Riak5 from Oregon to Riak1 in California. However, the             node of Riak1 was still not reachable to them. On investigating I found that the route tables did not have an entry to send             data to the VPC through VPC Peering. Hence , I added route table mapping in both the VPCs. After which the network connection           was established successfully and Riak3, Riak4 and Riak5 were able to join the cluster with Riak1 and Riak2.
         
      Now, I will test the normal behaviour of the cluster before the network partition is established. 
       
      1. Create a key value pair in Riak1 and test if the pair is read from all the nodes of the cluster.
        
        - Below command will show a bucket in buckets in Riak1
          curl -i http://<public ip of Riak1>:8098/buckets?buckets=true
        
        - create a key in bucket
          curl -v -XPUT -d '{"db":"riak"}' \
          http://<public ip of Riak1>:8098/buckets/bucket/keys/key1?returnbody=true
          
          
         - Query the bucket in each node of Riak Cluster to find the value of key1. The value will be consistent across the nodes.
           curl -i http://Ip address of node:8098/buckets/bucket/keys/key1
       
       
  
      2. Let's create the network partition int the cluster. we will break the connection between both the VPC which will split the               Riak Cluster into two. 
      
      ![riak_network_partition](https://user-images.githubusercontent.com/42895383/49473635-751a2380-f7c7-11e8-9649-4fd7f11eccbe.png)
      
      
      
