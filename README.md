# Provision an EKS Cluster for the Splunk K8s Operator
  
Terraform configuration files to provision an EKS cluster on AWS.  
  
### 1. Deploy EKS Cluster
Clone the repo
```bash
git clone 
cd 
```
  
Initiate the Terraform
```bash
terraform init -update
```
  
Apply
```bash
terraform apply
```
  
This process should take approximately 10 minutes. Upon successful application, your terminal prints the outputs  
```bash
Apply complete! Resources: 51 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "https://80CD543ECDB40DCF1AC9xxxxxxx9710F.sk1.eu-north-1.eks.amazonaws.com"
cluster_id = "rees-eks-kXpc1xxxx"
cluster_name = "rees-eks-kXpcxxxx"
cluster_security_group_id = "sg-03614b30a2xxxxxd7"
config_map_aws_auth = [
  {
    "binary_data" = tomap(null) /* of string */
....
....
```
  
### 2. Configure kubectl
Now that you've provisioned your EKS cluster, you need to configure kubectl.
  
Run the following command to retrieve the access credentials for your cluster and automatically configure kubectl.  
  
```bash
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```
  

### 3. Deploy Kubernetes Metrics Server
The Kubernetes Metrics Server, used to gather metrics such as cluster CPU and memory usage over time, is not deployed by default in EKS clusters.  
  
Download and unzip the metrics server by running the following command.  
```bash  
wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
```
  
Deploy the metrics server to the cluster by running the following command.
```bash
kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
```
  
Verify that the metrics server has been deployed. If successful, you should see something like this.
```bash
kubectl get deployment metrics-server -n kube-system
```
  
### 4. Deploy Kubernetes Dashboard and Proxy
The following command will schedule the resources necessary for the dashboard.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```
  
Your output will look like this  
```bash
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```
  
Now, create a proxy server that will allow you to navigate to the dashboard from the browser on your local machine. This will continue running until you stop the process by pressing `CTRL + C`.
  
```bash
kubectl proxy
```
  
Your output will be:  
```bash
Starting to serve on 127.0.0.1:8001
```
  
You can reach the Kubernetes dashboard here - http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
  

### 5. Authenticate the dashboard  (New Terminal)
To use the Kubernetes dashboard, you need to create a ClusterRoleBinding and provide an authorization token.
  
In another terminal (do not close the kubectl proxy process), create the ClusterRoleBinding resource.
```bash
kubectl apply -f https://raw.githubusercontent.com/hashicorp/learn-terraform-provision-eks-cluster/master/kubernetes-dashboard-admin.rbac.yaml
```
  
Then, generate the authorization token.
```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```
  
Select "Token" on the Dashboard UI then copy and paste the entire token you receive into the dashboard authentication screen to sign in. You are now signed in to the dashboard for your Kubernetes cluster.
  
Navigate to the "Cluster" page by clicking on "Cluster" in the left navigation bar. You should see a list of nodes in your cluster.
  
![dashboard](/images/dashboard.png)

### 6. Installing the Splunk Operator
A Kubernetes cluster administrator can install and start the Splunk Operator by running:
```bash
kubectl apply -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-install.yaml
```
  
After the Splunk Operator starts, you'll see a single pod running within your current namespace:
```bash
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
splunk-operator-75f5d4d85b-8pshn   1/1     Running   0          5s
```
  
![operator](/images/operator.png)

### 7. Deploy a Standalone deployment of Splunk Enterprise
  
Let’s ask our operator pod to build us a standalone demo instance to play with!  
  
```bash
kubectl -n default apply -f s1.yaml 
```
  
You will now see the sevices running:
```bash
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
kubernetes                      ClusterIP   172.20.0.1      <none>        443/TCP                               115m
splunk-operator-metrics         ClusterIP   172.20.51.109   <none>        8383/TCP,8686/TCP                     85m
splunk-s1-standalone-headless   ClusterIP   None            <none>        8000/TCP,8088/TCP,8089/TCP,9997/TCP   67s
splunk-s1-standalone-service    ClusterIP   172.20.91.91    <none>        8000/TCP,8088/TCP,8089/TCP,9997/TCP   67s
```
  
![services](/images/services.png)
  

### 8. Splunk Web Access
Get your Pod details  
```bash
kubectl get pods                             
NAME                                  READY   STATUS    RESTARTS   AGE
splunk-default-monitoring-console-0   1/1     Running   0          46m
splunk-operator-5845f6d45c-m4stq      1/1     Running   0          133m
splunk-s1-standalone-0                1/1     Running   0          48m
```
  
You can use a simple network port forward to open port 8000 for Splunk Web access:  
```bash
kubectl port-forward splunk-s1-standalone-0 8000
```
  
To access our Spunk Operator for Kubernetes built instance, we will need to grab the secret which contains the HEC token and password, among other secrets the Operator syncs to the Splunk instance. 
```bash
kubectl -n default get secret splunk-default-secret -o yaml
```
  
Your output will be:
```yaml
apiVersion: v1
data:
  hec_token: QUI4NUFBREMtRDkyNy0yNzJBLUFDNDAtNjRCQ0M2QzQ4RjI2
  idxc_secret: Mm51S3BTWDBveE43WklHQm9pZnF5VE5s
  pass4SymmKey: eWlvRUc5Nzk0cGJEWkF2UFJ5VEtVV1VY
  password: UXJiRG5IUVJud1hDYTdlR0pTc2x6YWt3
  shc_secret: ZXRYQzNZV3VYOHRrbHBuWUFOeVpMZDBn
kind: Secret
```
  
To `Decode` your passwords use the following:
```bash
kubectl get secret splunk-default-secret -o go-template=' {{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```
  
Your Output will be decoded like this:
```bash
kubectl get secret splunk-default-secret -o go-template=' {{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
 hec_token: AB85AADC-D927-272A-AC40-64BCC6C48F26
idxc_secret: 2nuKpSX0oxN7ZIGBoifqyTNl
pass4SymmKey: yioEG9794pbDZAvPRyTKUWUX
password: QrbDnHQRnwXCa7eGJSslzakw
shc_secret: etXC3YWuX8tklpnYANyZLd0g
```


Log into Splunk Enterprise at http://localhost:8000 using the `admin` account with the password.
  
  
### 9. Clean Up
To delete your standalone deployment, run:
```bash
kubectl delete standalone s1
```
  
Watch as the deployment is terminated:
```bash
kubectl -n default get pods -w
```
  
Destroy the EKS Cluster  
```bash
terraform destroy
```

### Error
If you get the error:  
```bash
Error: Get "http://localhost/api/v1/namespaces/kube-system/configmaps/aws-auth": dial tcp 127.0.0.1:80: connect: connection refused
```
then run the command  
```bash
terraform state rm module.eks.kubernetes_config_map.aws_auth
```
