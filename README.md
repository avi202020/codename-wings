# codename-wings

## golang-server-webapp

Dockerized Golang based server web application. Starts HTTP server on port 8080 with web app
that has built-in Prometheus exporter and basic routing functionality, i.e:

* Displays which links are available: http://wings.hodzic.org
* Displays picture of Homer Simpson: http://wings.hodzic.org/homersimpson
* Displays current time in Amsterdam and in Covilha, Portugal: http://wings.hodzic.org/covilha

### Golang run example:

```
cd gofiles/; go run main.go
```

### Docker image build & publish using Makefile example:

```
make v=0.1
```

### Docker run example:

```
docker run -d -p 80:8080 docker.io/adnanhodzic/golang-server-webapp:0.1
```

## terraform-k8s-aws-eks-deploy

Collection of Terraform configuration files which allow you to seamlessly and automatically deploy Kubernetes cluster on Amazon EKS. Allowing you to easily deploy [Go server WebApp](https://github.com/AdnanHodzic/codename-wings#golang-server-webapp).

### Setup necessary tools

#### AWS credentials config

Pre-requistie is to have AWS key credentials with admin permissions.

To keep sync of any AWS key credential changes made to `~/.aws/config/` file:
```
ln -s ~/.aws/config ~/.aws/credentials
```

or to take a snapshot of current AWS key credentials in `~/.aws/config/` file:
```
cp ~/.aws/config ~/.aws/credentials
```

#### Install Kubectl

Command line interface for running commands against Kubernetes clusters. 

MacOs:
```
brew install aws-iam-authenticator`
```

Ubuntu:
```
sudo snap install kubectl --classic
```

#### Install aws-iam-authenticator

A tool to use AWS IAM credentials to authenticate to a Kubernetes cluster on AWS EKS.

MacOS/Ubuntu:

```
wget https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/tag/0.4.0-alpha.1
chmod +x aws-iam-authenticator_0.4.0-alpha.1_darwin_amd64
sudo mv aws-iam-authenticator_0.4.0-alpha.1_darwin_amd64 /usr/local/bin/heptio-authenticator-aws
heptio-authenticator-aws help
```

### Deploy Kubernetes cluster to AWS EKS

To initialize a working directory (no need to run every time) containing Terraform configuration files, run:

```
terraform init
```

To see execution plan or refresh of stack if changes are made:

```
terraform plan
```

Run to deploy Kubernetes cluster on Amazon EKS.

```
terraform apply
```

Once everything is successfuly deployed you need to configure kubectl (will overwrite existing config):

```
terraform output kubeconfig > ~/.kube/config
```

Configure config-map-aws-auth:

```
terraform output config-map-aws-auth > config-map-aws-auth.yml
kubectl apply -f config-map-aws-auth.yml
```

Verify that everything is working as expected and that you nodes that can communicate to cluster:

```
kubectl get nodes --watch
```

Once all nodes are in `Ready` state, proceed to next step.

### Deploy Go server WebApp to Kubernetes cluster on Amazon EKS

Deploy [golang-server-webapp](https://github.com/AdnanHodzic/codename-wings#golang-server-webapp) Docker image to be running on internal port: 8080

```
kubectl run go-server-webapp --image=docker.io/adnanhodzic/golang-server-webapp:0.1 --port=8080
```

Check the deployment state:

```
kubectl get deployments
kubectl get pods
```

Deploy loadbalancer to which we can connect publically on port 80, after which we're redirected to internal port 8080:

```
kubectl expose deployment go-server-webapp --type=LoadBalancer --port 80 --target-port 8080
```

Check state of our loadbalancer:

```
kubectl get services
```

Get full informaton about loadbalancer (and its A record):

```
kubectl describe services go-server-webapp
```

Update DNS records of your domain with values of `LoadBalancer Ingress` line which has our loadbalancer A record, i.e:
```
LoadBalancer Ingress:     a467d5230fb2311e8bfef0adf1c7b2a4-385229931.eu-west-1.elb.amazonaws.com
```

After this, in my case [golang-server-webapp](https://github.com/AdnanHodzic/codename-wings#golang-server-webapp) will now be running on http://wings.hodzic.org

![alt text](https://foolcontrol.org/wp-content/uploads/2018/12/wings-index.png)

### Decomission Kubernetes cluster from AWS EKS

Kuberenetes cluster and all its resources will be removed by running:

```
terraform destroy
```

## terraform-ansible-prometheus-deploy

Collection of Terraform configuration files which deploy AWS EC2 instance and performing necessary tasks for it to be SSH-able out of box. `hosts` file will also be automatically pouplated with its public IP, after which this hosts file can be use as an inventory file to run Ansible. Once Ansible is ran it will install Prometheus and configure it to scrap metrics from [golang-server-webapp](https://github.com/AdnanHodzic/codename-wings#golang-server-webapp).

### Deploy Prometheus server

#### Server deployment with Terraform

Before anything, run: 

```
terraform init
```

To deploy AWS EC2 instance which will be used as Prometheus server, and make all necessary infrastructal changes run:

```
terraform apply
```

After server is deployed, up and running you'll get following message:

```
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

ip = 54.171.81.213
```

This same IP has been automatically added to `hosts` file which makes it ready to run Anisble to install and configure Prometheus.

#### Install and configure Prometheus with Ansible

Run Ansible with following command:

```
ansible-playbook prometheus.yml -i hosts -b -u ubuntu
```

After Ansible play has finished running, Prometheus will be up and running.

#### Decomission Prometheus server

Prometheus server and all its resources will be removed by running:

```
terraform destroy
```
