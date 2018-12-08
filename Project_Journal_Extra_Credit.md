
Installer-based vanilla Kubernetes on EC2

- brew install python
- pip3 install awscli
- aws configure
- brew update && brew install kops kubectl
- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1
(This gives a location constraint error because the bucket needs location constraint if we use any other region than us-east-1) Below command resolved the error

- aws s3api create-bucket --bucket payment-kops-state-store --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1



aws s3api put-bucket-versioning --bucket payment-kops-state-store  --versioning-configuration Status=Enabled


export KOPS_CLUSTER_NAME=payment.k8s.local
export KOPS_STATE_STORE=s3://payment-kops-state-store



kops create cluster --node-count=2 --node-size=t2.micro --zones=us-west-1 --name payment.k8s.local

Error: Name: Invalid value: "payment": Cluster Name must be a fully-qualified DNS name (e.g. --name=mycluster.myzone.com)


--create rsa ssh secret key to create cluster  
ssh-keygen -t dsa


-- use this secret key to create custer in AWS
kops create secret --name payment.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub


kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
