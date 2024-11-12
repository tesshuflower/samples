# ACM RH OV ( RedHat OpenShift Virtualization ) backup using OADP and ACM Policies

This policy can be used to backup RHOV resources running on managed clusters or hub. 
The policy is installed on the hub and is placed on managed clusters ( or hub ) using a label annotation `acm-virt-config`:`value`, where is the name of a ConfigMap, available on the hub in the same namespace with this policy. This ConfigMap defines the backup configuration for the cluster, such as : OADP version, backup schedule cron job, backup storage location, backup storage credentials.
An example of such configuration is available [here](./acm-virt-config.yaml) and [here](./acm-virt-config-13.yaml).

The Policy installs OADP on the cluster tagged with the `acm-virt-config`:`value` label, creats the DPA and creates a backup schedule to backup all VM resources.