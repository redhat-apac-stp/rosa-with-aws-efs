# ROSA with AWS EFS CSI Driver

This installation method uses Helm to install the AWS EF CSI Driver on ROSA using a least privileged operational via integration with AWS STS.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the AWS EFS Operator from Operator Hub as it does not support integration with STS as of this writing.

