# EKS Getting Started Guide Configuration

This is the full configuration from https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html

See that guide for additional information.

NOTE: This full configuration utilizes the [Terraform http provider](https://www.terraform.io/docs/providers/http/index.html) to call out to icanhazip.com to determine your local workstation external IP for easily configuring EC2 Security Group access to the Kubernetes master servers. Feel free to replace this as necessary.

# Create the EKS cluster via Terraform

1. `terraform apply`
2. Run `terraform output config_map_aws_auth > config_map_aws_auth.yaml` and save the configuration into a file, e.g. config_map_aws_auth.yaml

```➜  aws-eks git:(master) ✗ terraform output config_map_aws_auth


apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::131778002569:role/terraform-eks-demo-node
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

3. Than apply it to the cluster which allows worker nodes to be able to join the cluster

```
➜  aws-eks git:(master) ✗ kubectl apply -f config_map_aws_auth.yaml
configmap "aws-auth" created
```

4. Verify nodes are joining to the cluster.

```
kubectl get nodes --watch
```

# Kubectl setup on osx

1. Check the version of your kubectl<br />
   `kubectl version --short --client`
2. Get the recommended version of kubectl if you have not already. [docs](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)<br />
   `curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/kubectl`
3. Get the iam authenticator<br />
   `curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator`
4. Update eks kubeconfig
   `aws eks update-kubeconfig --name terraform-eks-demo --region=us-west-2`

```
➜  .kube git:(master) aws eks update-kubeconfig --name terraform-eks-demo --region=us-west-2
Added new context arn:aws:eks:us-west-2:131778002569:cluster/terraform-eks-demo to /Users/shaytac/.kube/config

module 'distutils' has no attribute 'spawn'
```

5. Check to see if you can issue commands against cluster. If you receive username/password prompt that is most likely due to [outdated kubectl version](https://stackoverflow.com/questions/51334054/setting-up-aws-eks-dont-know-username-and-password-for-config)

```
➜ kubectl get svc                                                                                                                          │
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE                                                                                              │
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   52m

```

6. [Managing Users or IAM Roles for your Cluster](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)

7. Deploy the Kubernetes dashboard to your cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

8. Deploy Hipster

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
```

9. Deploy the influxdb backend for heapster to your cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
```

10. Create the heapster cluster role binding for the dashboard:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml

```

11. Create an eks admin account [file](./eks-admin-account.yml]

```
kubectl apply -f eks-admin-service-account.yml
```

12. Bind the admin role to eks admin account

```
kubectl apply -f eks-admin-cluster-role-binding.yml
```

13. Now using the `eks-admin` account connect to dashbard.

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

14. Start the kubectl proxy

```
➜  aws-eks git:(master) ✗ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

15. Connect to dashboard.

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

Output

```
Name:         eks-admin-token-b5zv4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=eks-admin
              kubernetes.io/service-account.uid=bcfe66ac-39be-11e8-97e8-026dce96b6e8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      ------> <authentication_token> <------
```

```
➜  ~ kubectl --namespace kube-system get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   172.20.0.10      <none>        53/UDP,53/TCP   1h
kubernetes-dashboard   ClusterIP   172.20.175.241   <none>        443/TCP         22m
```

# Installing Helm on Kubernetes cluster

1. `brew install helm`
2. Make sure to have current context set to desired cluster

```
➜  ~ kubectl config current-context
arn:aws:eks:us-west-2:131778002569:cluster/terraform-eks-demo
```

3. Execute `helm init`

In case of following error

```
➜  ~ helm install --name my-release stable/nginx-ingress
Error: release my-release failed: namespaces "default" is forbidden: User "system:serviceaccount:kube-system:default" cannot get namespaces in the namespace "default"
```

# Installing Nginx ingress

Based on [this guide](https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md)

1.

```
➜  nginx-ingress git:(master) ✗ cat ns-and-sa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-ingress
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress
  namespace: nginx-ingress%

➜  aws-eks git:(master) ✗ kubectl apply -f nginx-ingress/ns-and-sa.yaml
namespace "nginx-ingress" created
serviceaccount "nginx-ingress" created
```

2.

```
➜  aws-eks git:(master) ✗ kubectl apply -f ./nginx-ingress/default-server-secret.yaml
secret "default-server-secret" created
```

3.

```
kubectl apply -f common/nginx-config.yaml
```

4. Create an nginx ingress deployment

```
➜  nginx-ingress git:(master) ✗ kubectl apply -f ./deployments/nginx-ingress.yaml
deployment.extensions "nginx-ingress" created
```

5. Create a DaemonSet

```
➜  nginx-ingress git:(master) ✗ kubectl apply -f daemon-set/nginx-ingress.yaml
daemonset.extensions "nginx-ingress" created

```

6. Apply Rbac

```
➜  nginx-ingress git:(master) ✗ kubectl apply -f rbac.yaml
clusterrole.rbac.authorization.k8s.io "nginx-ingress" created
clusterrolebinding.rbac.authorization.k8s.io "nginx-ingress" created

```

If this is not setup correctly pods will be crashing with the following error:

```
➜  nginx-ingress git:(master) ✗ kubectl --namespace=nginx-ingress logs -p nginx-ingress-6gmkp
I0210 03:25:38.390831       1 main.go:118] Starting NGINX Ingress controller Version=edge GitCommit=9a21a40b
F0210 03:25:38.399951       1 main.go:177] Error when getting nginx-ingress/default-server-secret: secrets "default-server-secret" is forbidden: User "system:serviceaccount:nginx-ingress:nginx-ingress" cannot get secrets in the namespace "nginx-ingress"
```

7. Create a service with the Type LoadBalancer

```
➜  nginx-ingress git:(master) ✗ kubectl apply -f service/loadbalancer-aws-elb.yaml
service "nginx-ingress" created

```

**Optional demo app deployment with ingress based on [nginx docs](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/complete-example)**

8. Create `coffee` and `tea` services that we will be using to test the ingress controller deployment

```
kubectl create -f cafe.yaml
```

9. Create ingress resource for demo-app

```
➜  aws-eks git:(master) ✗ kubectl create -f cafe-ingress.yaml
ingress.extensions "cafe-ingress" created
```

10. To test set `IC_HTTPS_PORT` and `IC_IP`

```
➜  demo-app git:(master) ✗ nslookup your-elb-1934633290.us-west-2.elb.amazonaws.com
Server:         1.1.1.1
Address:        1.1.1.1#53

Non-authoritative answer:
Name:   your-elb-1934633290.us-west-2.elb.amazonaws.com
Address: 34.218.166.164
Name:   your-elb-1934633290.us-west-2.elb.amazonaws.com
Address: 34.211.105.237

➜  demo-app git:(master) ✗ export IC_HTTPS_PORT=443

➜  demo-app git:(master) ✗ export IC_IP=34.218.166.164
➜  demo-app git:(master) curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecure

Server address: 10.0.1.204:80
Server name: coffee-6c47b9cb9c-z67wc
Date: 10/Feb/2019:18:16:48 +0000
URI: /coffee
Request ID: f11da1407f0567199da637a0db3c9ca8

➜  demo-app git:(master) curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/tea --insecure

Server address: 10.0.0.186:80
Server name: tea-58d4697745-l5xmt
Date: 10/Feb/2019:18:17:16 +0000
URI: /tea
Request ID: f4805c41008b3300c3c78fbcd35bf1e5

```
