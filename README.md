# Provision an EKS Cluster for the Splunk K8s Operator
  
Terraform configuration files to provision an EKS cluster on AWS.  
  
### Error
If you get the error:  
```bash
Error: Get "http://localhost/api/v1/namespaces/kube-system/configmaps/aws-auth": dial tcp 127.0.0.1:80: connect: connection refused
```
then run the command  
```bash
terraform state rm module.eks.kubernetes_config_map.aws_auth
```
