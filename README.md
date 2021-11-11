# ROSA with AWS EFS CSI Driver

These instructions uses Helm to install the AWS EFS CSI Driver on ROSA using least privileged operation via integration with STS.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the AWS EFS Operator from Operator Hub as it does not support integration with STS as at the time of this writing.

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
	
You can obtain the OIDC endpoint using the rosa describe cluster command.

