# Install PMN Gateway

## Pre-requisites

 - [ ]  `jq` will be installed by default on the Platform9 machine.
> [!NOTE]
> *To install it, follow the steps below:*
>
> [centos@pf9-mgmt ~]$ yum install -y epel-release
>  
> [centos@pf9-mgmt ~]$ yum install jq -y
### Check the status of the node
```
[centos@p9-mgmt ~]$ kubectl get nodes
NAME             STATUS   ROLES    AGE   VERSION
192.100.100.14   Ready    master   15d   v1.17.9
```
> [!IMPORTANT]
> In the event that you encounter a `cluster unreachable` or `unauthorized access` message, please follow the steps below:
>
> [centos@pf9-mgmt ~]$ bash clustercli.sh -a config -i /home/centos/inventory/intel-cluster1/ -t intel-cluster1 -s airgap

###  Copy rootCA.pem file of PMN orchestrator
```
[centos@pf9-mgmt ~]$ mkdir -p /home/centos/pmn-gw-files/
[centos@pf9-mgmt ~]$ mv /home/centos/rootCA.pem /home/centos/pmn-gw-files/
```
#### Create a Kubernetes secret using the file "rootCA.pem"
```
[centos@pf9-mgmt ~]$ cd /home/centos/pmn-gw/configs/
[centos@pf9-mgmt configs]$ kubectl create secret generic agwc-secret-certs --from-file=rootCA.pem=rootCA.pem --namespace pmn
```
### Pull the docker images for openebs
The following script can be used to pull helm chart,Docker images, tag them, and push them to a bevolake registry and installs the helm chart:
```
[centos@pf9-mgmt ~]$ cd /home/centos/23.03_packages/pmn-gw/scripts/
[centos@pf9-mgmt scripts]$ ./pull_openrepo_imgs.sh
```
### Mark the `openebs-hostpath` as default storage class
Check the status of openebs pods:
```
[centos@pf9-mgmt scripts]$ kubectl get pods -n openebs
```
The output is similar to this:
```
NAME                                           READY   STATUS    RESTARTS   AGE
openebs-localpv-provisioner-587cc4cd7d-9966s   1/1     Running   10         12d
openebs-ndm-operator-5c5784458b-87gxm          1/1     Running   8          12d
openebs-ndm-w97h7                              1/1     Running   6          12d
```
Path the storage class:
```
[centos@pf9-mgmt scripts]$ kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
### Verify that your chosen StorageClass is default:
```
[centos@pf9-mgmt scripts]$ kubectl get storageclass
```
The output is similar to this:
```
NAME                         PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device               openebs.io/local               Delete          WaitForFirstConsumer   false                  12d
openebs-hostpath (default)   openebs.io/local               Delete          WaitForFirstConsumer   false                  12d
sc-local-cil                 kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  44d
sc-local-crdlcluster         kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  44d
sc-local-elasticsearch       kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  44d
sc-local-oms                 kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  43d
sc-local-prometheus          kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  44d
```
### Generate the snowflake file on 'on-prem' node
```
[centos@on-prem ~]$ sudo su -c 'uuidgen > /etc/snowflake'
```
### Copy the required config files
```
[centos@pf9-mgmt ~]$ cd /home/centos/pmn-gw/
[centos@pf9-mgmt ~]$ scp -r configs/ on-prem:~/
[centos@on-prem ~]$ sudo mkdir -p /etc/magma
[centos@on-prem ~]$ sudo cp -r configs/* /etc/magma/
```
### List of scripts
```
[centos@pf9-mgmt ~]$ cd /home/centos/23.03_packages/pmn-gw/scripts/
[centos@pf9-mgmt ~]$ ls
total 20
-rwxr-xr-x 1 centos centos 1344 Aug 30 13:27 configure_aws.sh
-rwxr-xr-x 1 centos centos 3747 Aug 30 17:16 data_relay.sh
-rwxr-xr-x 1 centos centos  424 Aug 30 13:39 install_pmn_gw.sh
-rwxrwxr-x 1 centos centos 2124 Aug 30 17:15 pull_openrepo_imgs.sh
-rwxr-xr-x 1 centos centos 1118 Aug 30 13:27 pull_pkgs.sh
```
### Managing AWS Credentials with Cross-Account Access Support

The script `data_relay.sh` provides the flexibility to deploy AWS resources either using cross-account access or by directly utilizing a designated IAM user. To configure the AWS credentials appropriately, follow the steps below:
##### 1.  **Storing AWS Credentials**
By using the `.credentials.json` file, you eliminate the need to hardcode your credentials directly into shell scripts, enhancing security and maintainability.
```
[centos@pf9-mgmt ~]$ cat << EOF > .credentials.json
{
	"ACCESS_KEY": "<YOUR_AWS_ACCESS_KEY>",
	"SECRET_KEY": "<YOUR_AWS_SECRET_ACCESS_KEY>",
	"REGION": "YOUR_AWS_REGION",
	"CROSS_REGION_ACCESS": "True",
	"IAM_ROLE_ARN": "<IAM_ROLE_ARN>",
	"PMN_GW_VERSION": "97f6db401", # Override the latest commit ID.
	"ECR_REGISTRY": "<ECR_REGISTRY_NAME>",
	"ECR_REPO": "<ECR_REPO>",
	"AWS_PROFILE": "<PROFILE_NAME>",
    "CHART_VERSION": "1.0.0"
}
EOF
```


> [!NOTE]
> Replace the above values with yours.
>
> Set `CROSS_ACCOUNT_ACCESS` to "True" to enable cross-account access support.
 
> [!IMPORTANT]
> If the `CROSS_ACCOUNT_ACCESS` is set to True:
>
> [ Mandatory ]: Set `"IAM_ROLE_ARN"` to the ARN of the IAM role in the target account.
>
> [ Mandatory ]: Set `"SOURCE_PROFILE"` to the AWS CLI named profile containing the source account's credentials.
### Install PMN gw
**Note**: Download the scripts [here](https://github.com/wavelabsai/pmn-systems/tree/development/Devops/pmn-gw/scripts)
```
[centos@pf9-mgmt pmn-gw-scripts]$ ./data_relay.sh
Enter the path where the aws creds are stored: /home/centos/.credentials.json

-----------Output ommitted-----------
Release "pmn-gw" has been upgraded. Happy Helming!
NAME: pmn-gw
LAST DEPLOYED: Mon Jul 31 04:39:43 2023
NAMESPACE: pmn
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
[ Successfully installed PMN GW ... ]

- Check the status by running: 

[ kubectl get pods -n pmn ]

- To fetch the Challenge key and HDW ID:

[ kubectl exec -n pmn -it magmad-<pod-hash> -- show_gateway_info.py ]

- To check in the gateway in the PMN orchestrator:

[ kubectl exec -n pmn -it magmad-<pod-hash> -- checkin_cli.py ]

- To list the subscribers:

[ kubectl exec -n pmn -it magmad-<pod-hash> subscriber_cli.py list ]
```
### Verify the pods status

```
[centos@pf9-mgmt ~]$ kubectl get pods -n pmn
NAME                             READY   STATUS    RESTARTS   AGE
control-proxy-6bcf44cbcf-x4dvx   1/1     Running   1          2d3h
eventd-76898675ff-brqwh          1/1     Running   1          2d3h
health-bcd66bf49-b4sc7           1/1     Running   1          2d3h
magmad-7b7b9f87d4-t654c          1/1     Running   1          2d3h
policydb-6f556576bc-ckkl8        1/1     Running   1          2d3h
redis-7d549c744d-4k5nz           1/1     Running   1          2d3h
state-5dd596c85b-kknz8           1/1     Running   1          2d3h
subscriberdb-9d4c7f4c-4vfzd      1/1     Running   1          2d3h
td-agent-bit-7f9c8dc6b-lqphn     1/1     Running   1          2d3h
```
