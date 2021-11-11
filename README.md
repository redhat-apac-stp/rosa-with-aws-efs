# ROSA with AWS EFS CSI Driver

These instructions uses Helm to install the AWS EFS CSI Driver on ROSA using least privileged operation via integration with STS.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the AWS EFS Operator from Operator Hub as it does not support integration with STS as at the time of this writing. Also do not use Velero/Restic to backup a dynamically provisioned EFS volume as there is a known issue (https://github.com/vmware-tanzu/velero/issues/2958) that prevents restoration. Use AWS Backup which is already integrated with EFS instead.

Create the following policy (e.g., aws-efs-csi-driver-policy) in IAM to allow dynamic provisioning of EFS access points by the EFS CSI driver.

	{
	    "Version": "2012-10-17",
	    "Statement": [
	        {
	            "Effect": "Allow",
	            "Action": [
	                "elasticfilesystem:DescribeAccessPoints",
	                "elasticfilesystem:DescribeFileSystems"
	            ],
	            "Resource": "*"
	        },
	        {
	            "Effect": "Allow",
	            "Action": [
	                "elasticfilesystem:CreateAccessPoint"
	            ],
	            "Resource": "*",
	            "Condition": {
	                "StringLike": {
	                    "aws:RequestTag/efs.csi.aws.com/cluster": "true"
	                }
	            }
	        },
	        {
	            "Effect": "Allow",
	            "Action": "elasticfilesystem:DeleteAccessPoint",
	            "Resource": "*",
	            "Condition": {
	                "StringEquals": {
	                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
	                }
	            }
	        }
	    ]
	}

Attach this policy to a role (e.g., aws-efs-csi-driver-irsa) in IAM and setup a trust relationship with the OIDC provider so that the service account can assume the role and obtain short-term credentials.

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
	        "Federated": "arn:aws:iam::<AWS account>:oidc-provider/<OIDC endpoint>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
	        "StringEquals": {
	          "<OIDC endpoint>:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
	        }
	      }
	    }
	  ]
	}
	
You can obtain the OIDC endpoint using the rosa describe cluster -c <cluster name> command.

Create the following service account in the kube-system namespace.
	
	apiVersion: v1
	kind: ServiceAccount
	metadata:
  	  name: efs-csi-controller-sa
  	  namespace: kube-system
  	  labels:
    	    app.kubernetes.io/name: aws-efs-csi-driver
  	  annotations:
    	    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS account>:role/aws-efs-csi-driver-irsa

The following instructions assume that Helm has been installed.
	
	helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
	helm repo update
	helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
	--namespace kube-system \
	--set controller.serviceAccount.create=false \
	--set controller.serviceAccount.name=efs-csi-controller-sa

Verify the controller and driver is present on each cluster node.
	
	oc get pod -n kube-system -o wide
	
	NAME                                  READY   STATUS    RESTARTS   AGE   IP             NODE                                              NOMINATED NODE   READINESS GATES
	efs-csi-controller-6fcd876856-68fbd   3/3     Running   0          19h   10.0.192.73    ip-10-0-192-73.ap-southeast-1.compute.internal    <none>           <none>
	efs-csi-controller-6fcd876856-lpmhv   3/3     Running   0          19h   10.0.140.2     ip-10-0-140-2.ap-southeast-1.compute.internal     <none>           <none>
	efs-csi-node-82tb6                    3/3     Running   0          19h   10.0.190.108   ip-10-0-190-108.ap-southeast-1.compute.internal   <none>           <none>
	efs-csi-node-8vzwz                    3/3     Running   0          19h   10.0.192.73    ip-10-0-192-73.ap-southeast-1.compute.internal    <none>           <none>
	efs-csi-node-9c2sq                    3/3     Running   0          19h   10.0.166.112   ip-10-0-166-112.ap-southeast-1.compute.internal   <none>           <none>
	efs-csi-node-c4mb9                    3/3     Running   0          19h   10.0.140.2     ip-10-0-140-2.ap-southeast-1.compute.internal     <none>           <none>
	efs-csi-node-c89tp                    3/3     Running   0          19h   10.0.224.152   ip-10-0-224-152.ap-southeast-1.compute.internal   <none>           <none>
	efs-csi-node-mhqrp                    3/3     Running   0          19h   10.0.159.82    ip-10-0-159-82.ap-southeast-1.compute.internal    <none>           <none>
	efs-csi-node-xw8rx                    3/3     Running   0          19h   10.0.175.60    ip-10-0-175-60.ap-southeast-1.compute.internal    <none>           <none>
	

From the AWS console select the EFS service and select create a new file system and associate it with the ROSA VPC. Choose either multi-AZ or single-AZ redundancy in alignment with your deployment model for ROSA.
	
Create a security group for the file system which allows all outbound traffic and inbound NFS traffic on TPC port 2049 from within the VPC only (or any other external sources that may require access depending on your needs).

Under the file system network options select manage and bind the file system to the security group. Ensure that there is one mount target per AZ and that it maps to the subnet in which the ROSA nodes are deployed.
	
Under the file system general section configure backups and lifecycle management in line with your corporate standards and select a preferred performance mode.

Create a storage class for EFS to support dynamic provisioning operations.
	
	apiVersion: storage.k8s.io/v1
	metadata:
	  name: efs-sc
	provisioner: efs.csi.aws.com
	mountOptions:
	  - tls
	parameters:
	  provisioningMode: efs-ap
	  fileSystemId: <file system ID>
	  directoryPerms: "700"

Validate dynamic creation of a persistent volume and corresponding access point in EFS by submitting a persistent volume claim request.
	
	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
	  name: efs-claim
	spec:
	  accessModes:
	    - ReadWriteMany
	  storageClassName: efs-sc
	  resources:
	    requests:
	      storage: 5Gi

It should look something like this inside AWS console. Note that the path for the access point should match the name of the persistent volume that was created by the claim.

https://github.com/redhat-apac-stp/rosa-with-aws-efs/blob/main/AWS-EFS-AP.png
	
Note that the operating system identity associated with the path is auto-generated when the access point is provisioned. When mounting this access point inside a pod the user:group of the mountpoint will match this numberic value irrespective of any pod-level securitycontext settings as per the listing below. This is by design and ensures the identity of the access path is consistent across different NFS clients. This should not impact the ability of the container to read/write from the mountpoint even if it runs as a non-root user that has a different numeric value.

	$ oc exec efs-app -- bash -c "id; mount -t nfs4; touch data/test; ls -l data"
	uid=1234(1234) gid=1234 groups=1234
	127.0.0.1:/ on /data type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,noresvport,proto=tcp,port=20155,timeo=600,retrans=2,sec=sys,clientaddr=127.0.0.1,local_lock=none,addr=127.0.0.1)
	total 28
	-rw-r--r--. 1 50000 50000     0 Nov 11 01:56 test

Finally note that the operating system identity is incremented by one (1) for each access point created. e.g., if another PVC is created the auto-generated path will by owned by 50001:50001.
