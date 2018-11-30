**CMPE 281 - Personal NoSQL Project**

# Weekly progress and challenge

# Week1: (13-Oct-18- 19-Oct-18)  
 
- Understanding the requirement of the project.
- Read on the Nosql db options for CP and AP and choose one for each
- Started with MongoDB for CP and created replication on Mongodb of 5 nodes on AWS.														


**Challenges: How does network partition work on MongoDB Replication.

# Week2: (19-Oct-18- 27-Oct-18) 

- Successfully created MongoDB replication and topologyof 5 nodes to test network partition in CP.

# Steps to create AP cluster of 5 nodes
Project Journal

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
rs.add("Secondary1:27017");
rs.add("Secondary2:27017");
rs.add("Secondary3:27017");
rs.add("Secondary4:27017");

Check the status of the replication set with members consisting of 1 primary and 4 secondary nodes as per the topology for testing AP Partition

rs.status();

**Next steps:** 
- Create the test cases to find the behaviour of the cluster in different scenarios.
- Create a network partition approach.

# Week3: (28-Oct-18- 03-Nov-18)

**Create Network Partition using iptables firewall**
1. Choose a secondary instance that needs to be partitioned from rest of the replication set nodes

2. List the existing iptables on the secondary2 instance using below command
   sudo iptables -S 
   
3. Add iptable firewall rules to the list using below command which will disallow the secondary2 instance from connecting      any of the instances in the replication set. 
   sudo iptables -A INPUT -s <ip-address> -j DROP
   Add 4 rules one for each of the instance's public ip address for primary, secondary1, secondary3, secondary4, secondary5
   
4. Once the rules are added, run below commands to check the replication set status in Primary node
    mongo
    use admin
    db.auth('admin','xxxxx')
    rs.status()
    
    This will show that the secondary2 node is not reachable/healthy in the replication set and hence not available.
    
   Now that the network partition is created, we can test the experiments on consistency of data.
   

# Week4: (04-Nov-18- 10-Nov-18)



