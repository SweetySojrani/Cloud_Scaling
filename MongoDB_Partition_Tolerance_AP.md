
## Test MongoDB network partition tolerance in a cluster as CP system

![mongodb_architecture](https://user-images.githubusercontent.com/42895383/49353574-5276f600-f673-11e8-9d34-beb9348180c3.png)

**Steps to create AP cluster of 5 nodes**

**1. Setup MongoDB Cluster of 5 nodes**

Launch Ubuntu Server 16.04 LTS

- AMI:             Ubuntu Server 16.04 LTS (HVM)
- Instance Type:   t2.micro
- VPC:             cmpe281
- Network:         public subnet
- Auto Public IP:  no
- Security Group:  mongodb-cluster 
- SG Open Ports:   22, 27017
- Key Pair:        cmpe281-us-west-1

SSH into Mongo Instance
ssh -i <key>.pem ubuntu@<public ip>

**Install MongoDB**

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4

echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb.list

sudo apt update

sudo apt install mongodb-org=4.0.1 mongodb-org-server=4.0.1 mongodb-org-shell=4.0.1 mongodb-org-mongos=4.0.1 mongodb-org-tools=4.0.1

**MongoDB Install Verification**

sudo systemctl enable mongod \
sudo systemctl start mongod \
sudo systemctl stop mongod \
sudo systemctl restart mongod \
mongod --version \
sudo systemctl enable mongod.service \
sudo service mongod restart \
sudo service mongod status


**MongoDB Keyfile**
openssl rand -base64 741 > keyFile \
sudo mkdir -p /opt/mongodb \
sudo cp keyFile /opt/mongodb \
sudo chown mongodb:mongodb /opt/mongodb/keyFile \
sudo chmod 0600 /opt/mongodb/keyFile 

**Config mongod.conf**

i. sudo vi /etc/mongod.conf
-  remove or comment out bindIp: 127.0.0.1 \
   replace with bindIp: 0.0.0.0 (binds on all ips) 

   #network interface \
   net: \
        port: 27017 \
        bindIp: 0.0.0.0

-  Uncomment security section & add key file

   security: \
        keyFile: /opt/mongodb/keyFile

-  Uncomment Replication section. Name Replica Set = cmpe281

   replication: \
     replSetName: cmpe281

ii.Create mongod.service
   sudo vi /etc/systemd/system/mongod.service

    [Unit] \
        Description=High-performance, schema-free document-oriented database \
        After=network.target

    [Service] \
        User=mongodb \
        ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

    [Install] \
        WantedBy=multi-user.target

iii.Enable Mongo Service

    sudo systemctl enable mongod.service

-  Restart MongoDB to apply our changes

   sudo service mongod restart \
   sudo service mongod status

   Save to AMI Image \
   Save the EC2 instance AMI image and create 4 more EC2 instances of MongoDB


**2. Create Replication set**
1. Edit /etc/hosts in each EC2 Instance adding local host names for Public Ips. \
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

**3. Create Network Partition using iptables firewall**
1. Choose a secondary instance that needs to be partitioned from rest of the replication set nodes

2. List the existing iptables on the secondary2 instance using below command
   sudo iptables -S 
   
3. Add iptable firewall rules to the list using below command which will disallow the secondary2 instance from connecting any of the        instances in the replication set
   sudo iptables -A INPUT -s <ip-address> -j DROP
   Add 4 rules one for each of the instance's public ip address for primary, secondary1, secondary3, secondary4, secondary5
   
4. Once the rules are added, run below commands to check the replication set status in Primary node
   mongo
   use admin \
   db.auth('admin','xxxxx') \
   rs.status()
    
   This will show that the secondary2 node is not reachable/healthy in the replication set and hence not available.
    
   Now that the network partition is created, we can test the experiments on consistency of data.
   
**4. Test the behaviour of the cluster after partition.**

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


3. Recover from the Network Partition. Use the below commands to delete the iptables rules that were created in Step2 to create Network    partition. 

   sudo iptables -L --line-numbers\
   sudo iptables -D INPUT 1

   Once the partition is recovered , the Secondary2 instance will become available in the replication set again and read the new            document from Primary instance that was created after the Network Partition.
   
   
  
           
           
        
