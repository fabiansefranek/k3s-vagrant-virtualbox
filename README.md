# K3S Setup 

## Vagrant file
The vagrant file firstly specifies the IP addresses of the server node and the agent nodes.

After that, the provision scripts are defined:
### Server script
1. Installs curl
2. Fetches k3s download script and installs it with the following parameters:
    * server: indicates that the node will be in the control plane
    * cluster-init: tells k3s to initialize k3s cluster with etcd
    * node-external-ip: specifies the external ip address to advertise for this node
    * flannel-iface: overrides the default networking interface used by flannel (used due to issues with flannel not taking correct interface)
3. Waits for k3s service to be up
4. Copy token to shared vagrant folder, so agents are able to join with it

### Agent script
1. Installs curl
2. Fetches k3s download script and installs it with the following environment variables:
    * K3S_TOKEN_FILE: path to the token, required to join the cluster
    * K3S_URL: url of the server node

Next, the general VM config is provided:
1. Use alpine 3.18 box
2. Create shared folder if not exists, else use it
3. Define system resources (2gb ram, 2 cpus)

Then the server VM is initialized with its IP as a private network, it's hostname and the serverprovision script.

Finally, each agent gets created with its IP, name and also the agent provision script.

After executing `vagrant up`, we can test if the setup worked by getting the nodes of the cluster with `kubectl get nodes` on the server agent.

Output:
```
NAME     STATUS   ROLES                       AGE   VERSION
agent1   Ready    <none>                      13m   v1.29.5+k3s1
agent2   Ready    <none>                      12m   v1.29.5+k3s1
server   Ready    control-plane,etcd,master   14m   v1.29.5+k3s1
```

## Deployment

To deploy a simple NGINX image to the cluster, we can utilize the deployment file located at `./deployments/nginx.yaml`.

This yaml file specifies that it is a `Deployment`, with the name `nginx-deployment`.

The `template > metadata > labels` field defines the label which is assigned to every pod created by this deployment 

The `spec > selector > matchLabels` field specifies how to identify the pods that belong to this deployment. In this case, the selector matches pods with the label `app: nginx`

The `spec > replicas` tells Kubernetes how many pods of this type we want. In our case, two nginx pods are created.

The `spec > containers` field lists the containers that will run in a pod. In this case, we have one container with the name `nginx` and it uses the docker image `nginx:1.27-alpine`. Furthermore, it exposes the port 80 of the container

To create this deployment, execute the command `kubectl apply -f /vagrant_shared/nginx-deployment.yaml` (the file needs to be put into the shared folder beforehand)

Now we can verify if this deployment works by getting all pods with `kubectl get pods -o wide`. Additionally, to check if Nginx works, we send a GET request to one of the pod's internal IP address: `curl http://{ip}`

```<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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
```

## Service

Although there is a deployment, until now we cannot access the Nginx server yet.
To make it accessible, we create a kubernetes service called `nginx-service.yaml`.

This configuration file is of kind `Service`, with the name `nginx-service`, it applies to all pods with the label we specified in the deployment (i.e `app: nginx`). It is of type `NodePort`, which provides a way to expose pods to the outside world. In our case, we expose port 80 of the TCP stack and map it to the outside port 30001

This configuration file must also be applied with `kubectl apply -f /vagrant_shared/nginx-service.yaml`

Now, if we go to the server ip (with port 30001) in the host browser, we can see the Nginx default page

```
Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```