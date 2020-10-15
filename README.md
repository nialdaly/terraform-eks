# Terraform EKS
The following project will provision a VPC, security groups and an Amazon EKS cluster using Terraform.

## Prerequisites
* AWS account
* [Terraform])(https://learn.hashicorp.com/tutorials/terraform/install-cli)
* [wget](https://stackoverflow.com/questions/33886917/how-to-install-wget-in-macos)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/docs/intro/install/)

## Amazon EKS Cluster provisioning
To provision an Amazon EKS cluster the following Terraform files are needed.

`vpc.tf` provisions a new VPC, subnets and availability zones using the AWS VPC Module.

`security-groups.tf` provisions the security groups used by the EKS cluster.

`eks-cluster.tf` provisions all the resources needed to set up an EKS cluster in the private subnets and bastion servers to access the cluster using the AWS EKS Module. The AutoScaling group configuration will contain three nodes.

The Terraform workspace can be initialized by using the `terraform init` command. `terraform validate` can be used to check 

The `terraform apply` command can be used to provision the resources and build the EKS cluster.

## kubectl Configuration
The access credentials of the EKS cluster can be retrieved and used to configure kubectl using the following command.
`aws eks --region $(terraform output region) update-kubeconfig --name $(terraform output cluster_name)`

## Deploying the Kubernetes Metrics Server & Dashboard
The metrics server can be downloaded and unzipped 
```
wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
```

It can then be deployed to the cluster using the following command.
`kubectl apply -f metrics-server-0.3.6/deploy/1.8+/`

The folliowing command be used to verify that the metrics server has been deployed.
```
kubectl get deployment metrics-server -n kube-system
```

The Kubernetes Dashboard resources can be scheduled using the following command.
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml`

A proxy server can be created using the following command.
```
kubectl proxy
```

The proxy server allows you to navigate to the Kubernetes dashboard from the browser on your own machine. This proxy can be stopped by CTRL + C. Now the Kubernetes dashboard should be accessible via the following URL.
```
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## Authenticating the Kubernetes Dashboard
To use the Kubernetes dashboard, you need to create a ClusterRoleBinding and provide an authorization token. This gives the cluster-admin permission to access the kubernetes-dashboard.

Open another terminal window while the kubectl proxy process is open. The ClusterRoleBinding resource can be created using the following command.

```
kubectl apply -f https://raw.githubusercontent.com/hashicorp/learn-terraform-provision-eks-cluster/master/kubernetes-dashboard-admin.rbac.yaml
```

The authorization token can be generated using the following command.

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```

The generated token can then be copied and pasted in the Kubernetes dashboard authentiction screen and used to sign in.

## Tiller (Helm Server) Setup
The Helm ServiceAccount can be applied using the following command.
```
kubectl apply -f tiller-user.yaml
```

The Helm Jenkins stable repo can be downloaded using the following command.
```
helm repo add jenkins https://charts.jenkins.io
```

## Jenkins Installation using Helm
Jenkins can be installed on the cluster using the following command.
```
helm install jenkins-cicd jenkins/jenkins -f jenkins-values.yaml
```

After Jenkins has been installed, we can access the load balancer address via the following command.
```
kubectl get svc
```

Alongside the `jenkins-cicd` service the external ip address should be available. To access the password for the Jenkins server, run the following command.
```
printf $(kubectl get secret --namespace default jenkins-cicd -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

## Deleting the Jenkins Service
The Jenkins service can be deleted using the following commands.
```
kubectl delete services jenkins-cicd
kubectl delete services jenkins-cicd-agent
helm uninstall jenkins-cicd
```

The EKS cluster and the associated resources can be deleted using the `terraform destroy` command.
