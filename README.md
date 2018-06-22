# kubernetes-setup
Ansible Playbook scripts for quickly setting up a Kubernetes cluster w/ Docker from a fresh set of bare-metal servers

# Prerequisites
- Have at least 3 bare-metal servers (1 master, 2 workers)
- Have Ansible installed on local machine
- Have Python 3 installed on all target nodes
- Have SSH access to all target nodes (remember to SSH in at least once to verify host key)

# Steps
1. Edit `hosts` file and enter the IPs of the master and worker nodes 
2. Run `ansible-playbook -i hosts initial.yml`. This creates an `ubuntu` passwordless sudo user and authorizes local machine to SSH into `ubuntu`
3. Run `ansible-playbook -i hosts kube-dependencies.yml`. This installs packages necessary for the functioning of the cluser, such as Kubernetes, Docker, Kubelet, Kubeadm, etc.
4. Run `ansible-playbook -i hosts master.yml`. This sets up the master node.
5. SSH into the master node with the **UBUNTU** user: `ssh ubuntu@master_ip`. Run `kubectl get nodes`. You should see:

```
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.10.1
```
6. Run `ansible-playbook -i hosts workers.yml`. This sets up the worker nodes.
7. SSH into the master node with the **UBUNTU** user: `ssh ubuntu@master_ip`. Run `kubectl get nodes`. You should see:

```
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.10.1
worker1   Ready     <none>    1d        v1.10.1 
worker2   Ready     <none>    1d        v1.10.1
```

# Optional steps for extended setup
8. Create a secret in the cluster that holds Docker registry authentication token: `kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>`
9. Create a deployment. In the master node, create a directory `deployments` if it doesn't exist already, and create a YAML file `some-deployment.yml`. Here's an example:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <NAME OF DEPLOYMENT>
  labels:
    app: <DEPLOYMENT LABEL>
spec:
  replicas: <NUMBER OF REPLICA PODS>
  selector:
    matchLabels:
      app: <DEPLOYMENT LABEL>
  template:
    metadata:
      labels:
        app: <DEPLOYMENT LABEL>
    spec:
      containers:
      - name: <POD NAME>
        image: <IMAGE TAG>
        ports:
        - containerPort: <PORT TO EXPOSE e.g. 3000>
      imagePullSecrets:
      - name: regcred
```
10. Create a NodePort service to expose your app to the Internet and load balance between pods. In the master node, create a directory `services` if it doesn't exist already, and create a YAML file `some-service.yml`. Here's an example:

```
apiVersion: v1
kind: Service
metadata:
  name: <SERVICE NAME>
  labels:
    name: <SERVICE NAME>
spec:
  type: NodePort
  ports:
    - port: <PORT OF CONTAINER TO EXPOSE e.g. 3000>
      nodePort: <PORT TO EXPOSE TO THE INTERNET e.g. 30000>
      name: http
  selector:
    app: <DEPLOYMENT LABEL FROM ABOVE>
```
11. To start the deployment, on the master node, run `kubectl create -f ~/deployments/some-deployment.yml`
12. To start the service, on the master node, run `kubectl create -f ~/services/some-service.yml`
13. To connect the cluster to a CI pipeline for rolling updates, create a bash file in home directory in user `ubuntu` named `init-rolling-update.sh`. Here is an example:

```
if [ "$1" = "production" ]; then
        kubectl set image deployment/<DEPLOYMENT NAME> <DEPLOYMENT LABEL>="<PROD IMAGE TAG>:$2"
else
        kubectl set image deployment/<DEPLOYMENT NAME> <DEPLOYMENT LABEL>="<DEV IMAGE TAG>:$2"
fi
```

The CI build server would SSH into this master node and run this script, e.g. `./init-rolling-update.sh development v0.1.3`  
