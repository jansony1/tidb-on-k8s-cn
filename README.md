
#	How to setup TiDB on kubernetes cluster by kops in AWS China 

This document provides guides to deploy TiDB in k8s in AWS China using [kops-cn](https://github.com/nwcdlabs/kops-cn). 

##	Pre-requisites
*	Have a AWS China account. If you don't have one, [click here to register](https://www.amazonaws.cn/en/sign-up/)

## Step 1: Setup a kubernete cluster uing kops-cn

For dev and test, refer to https://github.com/nwcdlabs/kops-cn (Chinese version only). 
if you know Chinese, just to [kops-cn](https://github.com/nwcdlabs/kops-cn) and skip step1.
if you don't know Chinese, below is a quick guide to install kubernetes by kops:

1. launch an EC2 (select Amazon Linux 2 AMI) as a kubernetes management machine, and configure it as
    1. Login AWS China console and goto Services->IAM->Group, add a new group (e.g kubernetes) with “AdministratorAccess” attached (for test purpose)

    1. Goto Services->IAM->Users, add a new user (e.g. kops) with “Access type” of “Programmatic access” and add it into the new created group (e.g. kubernetes). After it, you will get the “Access key ID” and “Secret access key” and write them down for future use;

    1. Login the EC2, and perform the “aws configure”. It will ask you to input the following information. cn-north-1 is the code number for Beijing region, for Ningxia, it's cn-northwest-1
        - AWS Access key ID: <created above>
        - AWS Secret access key: <created above>
        - Default Region name: cn-north-1  
        - Default output format: none

1. Install the kops and kubectl by following the instructions in https://github.com/nwcdlabs/kops-cn, or using the script of install-tools.sh in the package;

1. By following the instructs in above kops-cn website, download kops-cn.zip and unzip it, and then make some changes to the kops-cn-master/Makefile as below:
    * modify the S3 bucket name (Note: S3 bucket is to store the kubernetes cluster state)
        - Create a S3 bucket through AWS China console in the Beijing region, for example, the S3 bucket name is "example-of-kops-state";
        - replace the value of “KOPS_STATE_STORE” with your S3 bucket name of "example-of-kops-state”;
    * modify SSH key pair
        - Generate a SSH key-pair including .pem and .pub, for example ec2kp_nx.pem and ec2kp_nx.pub (you can generate key pair by following https://github.com/nwcdlabs/kops-cn/issues/68#issuecomment-483879369);
        - Replace the value of “SSH_PUBLIC_KEY” with the full path of new generated public key of “ec2kp_nx.pub”;
    * Modify the VPCID
        - Create a new VPC (e.g. 10.32.0.0/16) in Beijing region through AWS China console. For example, the new VPC id is vpc-bb3e99d2;
        - Create a new internet gateway through AWS China console, and attach it to the new created VPC;
        - Replace the value of ‘VPCID’ with the new create VPC (e.g. vpc-bb3e99d2)

    * Install the kubernetes cluster by following the instructions of kops-cn
        - $ cd kops-cn-master
        - $ make create-cluster
        - $ make edit-cluster # follow the instructions in kops-cn website to copy the content from spec.yml
        - $ make update-cluster # will take 10-15 mins
        - $ make validate-cluster # or you can start to use kubectl to operate on the kubernetes cluster
        - In order to work around the ICP recordal for the public web service which is required by China government, you still need to perform the following steps:
	    - Login AWS China console and go to Services->EC2->Load Balancer, and select the the load balancer according to the .kube/config and make some changes:
	        - add a rule into the security group to allow 8443 TCP traffic from 0.0.0.0/0 source;
	        - Click on the “Listeners” tab and change the “Load Balancer Port” from 443 to 8444
	    - Edit .kube/config by appending “:8443” at end of line “server: https://xxxx”
	- And now, the kubernetes cluster is ready, and you can operate it by kubectl

## Step 2: install TiDB on kubernete cluster

1. Ssh to the bastion server. If you are using Amazon AMI,  ssh -i < name-of-your-private-key >.pem  ec2-user@< ip-address > . For other linux type, try centos or ubuntu for the username.

1. Run the following command
```

helm search | grep tidb

# fetch Tidb cluster package  
helm fetch pingcap/tidb-cluster   

# unzip it to get the config file
tar -zxvf tidb-cluster-v1.0.0.tgz

cd tidb-cluster
```

1. There is a values.yml file in this folder which contains the configuration set for components like TiKV, pd, TiDB etc. **Custom your own configuration by revising the yml file** before you install TiDB, examples of changes includes revise replica set, set a ELB, change pvReclaimPolicy or revise the storageClassName etc.    

- Changing storageclass to gp2, AWS EBS volume General SSD. For more choices, click [EBS Volume type](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)   

![](img/storage-class.png)

- changing component replica number   
![](img/replica-number.png)

1. After everything is set, run the following command.

```
helm install pingcap/tidb-cluster --name=tidb-cluster-test --namespace=tidbtest -f values.yml 
```

1. Check the cluster configuration. For example, you will see the storageClass applies succesfully to gp2 and PV has automatically been provisioned dynamically. Click here for more introduction upon [PV provision](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

```
kubectl get pv | grep tidbtest
```

![](img/get-pv.png)


## Step3. Scale on kubernete cluster

1. If you would like to change any configuration, for example, scaling up or down your replica number, simple revise the values.yml again and run the following command. For more guide upon scaling, refer to [this page](https://github.com/pingcap/docs-cn/blob/master/v3.0/tidb-in-kubernetes/scale-in-kubernetes.md)

```
helm upgrade tidb-cluster-test pingcap/tidb-cluster --namespace=tidbtest -f values.yml 
``` 

You will see this after the command.

![](img/helm-upgrade.png)


## Step 3: How to access TiDB 


###	Cluster Startup
1. Watch tidb-cluster up and running
```
	watch kubectl get pods --namespace tidbtest -l app.kubernetes.io/instance=tidb-cluster-test -o wide
```
1. List services in the tidb-cluster
```
	kubectl get services --namespace tidbtest -l app.kubernetes.io/instance=tidb-cluster-test
```

###	Cluster access

1. Access tidb-cluster using the MySQL client
```  	
  	kubectl port-forward -n tidbtest svc/tidb-cluster-test-tidb 4000:4000 &
    mysql -h 127.0.0.1 -P 4000 -u root -D test
```
1. Set a password for your user
```
    SET PASSWORD FOR 'root'@'%' = 'JEeRq8WbHu'; FLUSH PRIVILEGES;
```    
1. View monitor dashboard for TiDB cluster
```
   kubectl port-forward -n tidbtest svc/tidb-cluster-test-grafana 3000:3000
```   

Open browser at http://localhost:3000. The default username and password is admin/admin.
If you are running this from a remote machine, you must specify the server's external IP address.

## Delete resources

If your pv is retained, you may need to delete the released pv manually by running the following command.
```
kubectl delete pv pvc-2eb80d98-bd7XXXc1ae  -n tidbtest
```

## Limitation

According to [this page](https://pingcap.com/docs-cn/v3.0/tidb-in-kubernetes/faq/#tidb-%E7%9B%B8%E5%85%B3%E7%BB%84%E4%BB%B6%E5%8F%AF%E4%BB%A5%E9%85%8D%E7%BD%AE-hpa-%E6%88%96-vpa-%E4%B9%88), TiDB cluster hasn't supported HPA(Horizontal Pod Autoscaling) or VPA(Vertical Pod Autoscaling). The reason is because Tidb cluster uses statefulset instead of deployment as database is a stateful application and it's hard for a stateful application to scale simply based on CPU or memories.

![](img/stateful-set.png)

## Testing
Please refer to this page for [run sysbench on TiDB](https://github.com/pingcap/docs/blob/master/dev/benchmark/sysbench-v4.md)

## References

* [Pingcap](https://pingcap.com/docs-cn/dev/tidb-in-kubernetes/tidb-operator-overview/)
* [AWS EKS workshop](https://eksworkshop.com/scaling/)


