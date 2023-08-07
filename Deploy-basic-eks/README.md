# EKS INSTALACION
Vamos a deployar un EKS y probar con un pod

0) Crear LABORATORIO
Necesitamos las credenciales de aws.

1) Instalar terraforms:
https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli

2) Instalar y configurar awscli y terraforms

``` bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

4)  Siguiendo esta web instalar el EKS:
https://developer.hashicorp.com/terraform/tutorials/kubernetes/eks

``` bash
git clone https://github.com/onai254/terraform-eks-provision.git
cd terraform-eks-provision/
terraform init
terraform plan
terraform apply
.
.

Plan: 58 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + cluster_endpoint          = (known after apply)
  + cluster_name              = "Giovanni"
  + cluster_security_group_id = (known after apply)
  + region                    = "us-east-2"

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

random_string.suffix: Creating...
random_string.suffix: Creation complete after 0s [id=jyin8Haw]
module.eks.aws_iam_role.this[0]: Creating...
module.eks.module.eks_managed_node_group["one"].aws_iam_role.this[0]: Creating...
module.eks.aws_cloudwatch_log_group.this[0]: Creating...
module.vpc.aws_eip.nat[0]: Creating...
module.eks.module.eks_managed_node_group["two"].aws_iam_role.this[0]: Creating...
module.vpc.aws_vpc.this[0]: Creating...
module.eks.aws_cloudwatch_log_group.this[0]: Creation complete after 1s [id=/aws/eks/Giovanni/cluster]
.
.
.
.
.


aws_eks_addon.ebs-csi: Creation complete after 14m47s [id=Giovanni:aws-ebs-csi-driver]

Apply complete! Resources: 58 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "https://CF3BF71C559F999BABDC09C7DFA8687D.gr7.us-east-2.eks.amazonaws.com"
cluster_name = "Giovanni"
cluster_security_group_id = "sg-0d1741c59b1aaa2ad"
region = "us-east-2"
mascolo99@mascolo99-VirtualBox:~/terraform/eks2/terraform-eks-provision$

mascolo99@mascolo99-VirtualBox:~/terraform/eks2/terraform-eks-provision$ aws eks --region $(terraform output -raw region) update-kubeconfig \
>     --name $(terraform output -raw cluster_name)
Added new context arn:aws:eks:us-east-2:545629305864:cluster/Giovanni to /home/mascolo99/.kube/config

mascolo99@mascolo99-VirtualBox:~/terraform/eks2/terraform-eks-provision$ kubectl get nodes
NAME                                       STATUS   ROLES    AGE   VERSION
ip-10-0-1-33.us-east-2.compute.internal    Ready    <none>   81m   v1.24.15-eks-a5565ad
ip-10-0-1-98.us-east-2.compute.internal    Ready    <none>   81m   v1.24.15-eks-a5565ad
ip-10-0-2-115.us-east-2.compute.internal   Ready    <none>   81m   v1.24.15-eks-a5565ad
ip-10-0-2-83.us-east-2.compute.internal    Ready    <none>   81m   v1.24.15-eks-a5565ad
ip-10-0-3-101.us-east-2.compute.internal   Ready    <none>   81m   v1.24.15-eks-a5565ad
ip-10-0-3-97.us-east-2.compute.internal    Ready    <none>   81m   v1.24.15-eks-a5565ad
mascolo99@mascolo99-VirtualBox:~/terraform/eks2/terraform-eks-provision$

```

5) Creo este manifiesto para poder darle los roles a mi user y que pueda ver los recursos desde aws eks.

``` yaml
kubectl edit configmap aws-auth -n kube-system;

apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::XXXXXXXXXXX:role/node-group-1-eks-node-group-20230727102249306500000001
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::XXXXXXXXXXX:role/node-group-2-eks-node-group-20230727102249310600000003
      username: system:node:{{EC2PrivateDNSName}}
------> Agregar lo siguiente userarn: arn:aws:iam::ACCOUNT-ID:user/USER
  mapUsers: |
    - userarn: arn:aws:iam::XXXXXXXXXXX:user/mosco
      groups:
      - system:masters
```

6) Creo un pod de pruebas
``` yaml
mascolo99@mascolo99-VirtualBox:~/kubernetes/manifests$ cat <<EOF >static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```

7) Chequeo que responda la web

``` yaml
mascolo99@mascolo99-VirtualBox:~/terraform/terraform-eks-provision$ kubectl exec -ti static-web -- sh

# curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
#
```