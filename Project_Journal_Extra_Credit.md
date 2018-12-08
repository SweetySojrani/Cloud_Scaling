
Installer-based vanilla Kubernetes on EC2

- brew install python
- pip3 install awscli
- aws configure
- brew update && brew install kops kubectl
- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1
(This gives a location constraint error because the bucket needs location constraint if we use any other region than us-east-1) Below command resolved the error

- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1

![bucket_created](https://user-images.githubusercontent.com/42895383/49691654-2d173b80-fafc-11e8-8177-8c261611b1fd.png)

- aws s3api put-bucket-versioning --bucket payment-kops-state-store  --versioning-configuration Status=Enabled

- sudo vi ~/.bash_profile 
(Add below two environment variable in the file which will be used by kubectl while creating cluster)
export KOPS_CLUSTER_NAME=payment.k8s.local
export KOPS_STATE_STORE=s3://payment-kops-state-store

- kops create cluster --node-count=2 --node-size=t2.micro --zones=us-west-1 --name payment.k8s.local
This gives an error: Name: Invalid value: "payment": Cluster Name must be a fully-qualified DNS name (e.g. --name=mycluster.myzone.com)

To resolve this error a secret key rsa is needed which will have the fingerprint of the AWS that we have configured in the first step.
create rsa ssh secret key to create cluster  
- ssh-keygen -t rsa


use this secret key to create custer in AWS
- kops create secret --name payment.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub


kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
