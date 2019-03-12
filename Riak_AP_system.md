
 ## AP System with Riak Database
   
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
       - Start Riak \
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
           curl -i http://<Ip address of node:8098>/buckets/bucket/keys/key1
       
       
  
      2. Create the network partition int the cluster. we will break the connection between both the VPC which will split the                    Riak Cluster into two. 
      
      ![riak_network_partition](https://user-images.githubusercontent.com/42895383/49473635-751a2380-f7c7-11e8-9649-4fd7f11eccbe.png)
      
        - create a new key in Riak1
          curl -v -XPUT -d '{"Model":"Nosql"}' \
          http://<Ip address of node:8098>/buckets/bucket/keys/key2?returnbody=true
          
         - create a new key in Riak3
           curl -v -XPUT -d '{"Model":"Masterless"}' \
           http://<Ip address of node:8098>/buckets/bucket/keys/key2?returnbody=true
          
          
         - Query the bucket in each node of Riak Cluster to find the value of key1.
           curl -i http://<Ip address of node:8098>/buckets/bucket/keys/key2
           
           On querying, I found that the Riak1 and Riak2 split showed key2 value as '{"Model":"Nosql"}'
           and the Riak cluster split of Riak3, Riak4 and Riak5 showed key2 value as '{"Model":"Masterless"}'
           
           Since Riak is highly available. It accepts read and write requests from cluster nodes in both the splits inspite the splits              being inconsistent. Riak is an AP system which is why it compromises consistency over availability during a network                      partition.
        
      3. Recover the cluster from the network parition by entering the removed route table mapping of VPC Peering.
         Once the connection is established again, the new keys created propogate through the cluster.
           
         Riak KV uses logical clocks, called dotted version vectors (DVVs), to ensure data accuracy. As data is updated, DVVs provide            a causal history that makes it possible to determine the precise order of events. If concurrent writes cannot be resolved,              DVVs ensure that all writes are stored as siblings. These sibling versions provide information that allows for conflict                  resolution logic when reading data. DVVs play an important role in minimizing the need for client-side conflict resolution.              (referred from Basho.com)
           
         Since, both the splits of Riak Cluster has now two different values for Key2. Riak resolves this write conflict using DVV and            the write that was last made in the splits takes precedence.So, the key2 value of '{"Model":"Masterless"}' is propogated                across the Riak cluster nodes. So the Cluster achieves eventual consistency after the Network Parition was recovered.
