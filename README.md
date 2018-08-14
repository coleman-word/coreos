# coreos

# Step 1 - Etcd
Etcd is a "Distributed reliable key-value store for the most critical data of a distributed system". Kubernetes uses Etcd to store state about the cluster and service discovery between nodes. This state includes what nodes exist in the cluster, which nodes they are running on and what containers should be running.
CoreOS includes an Etcd2 deployment. You can check it's running using:
```
 etcdctl cluster-health
```
To deploy you need to create a systemd configuration file, similar to the one used by cat _usr_lib64/systemd_system_etcd2.service
The binary can be downloaded from  [https://github.com/coreos/etcd/releases](https://github.com/coreos/etcd/releases) 
In production you would want to run etcd on three separate machines to ensure maximum availability.

# Step 2 - Create SSL Certificate
To ensure that all communication is secured, it's important to generate SSL certificates used for communication and security.
# Task
Create the required SSL certificates for a Kubernetes deployment using the command below.
The command will generate### ca.pem### ,### ca-key.pem### ,### apiserver.pem### and### apiserver-key### . It allows traffic for the### kubernetes.default.svc.cluster.local### domain and the IP addresses 10.3.0.1 and the Host IP of the server. 10.3.0.1 is an internal address assigned to the DNS server.
```
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"
cat <<EOF > openssl.cnf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.3.0.1
IP.2 = 172.17.0.38
EOF
openssl genrsa -out apiserver-key.pem 2048
openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf
sudo mkdir -p /etc/kubernetes/ssl
sudo mv apiserver-key.pem apiserver.pem ca.pem ca-key.pem /etc/kubernetes/ssl
```

# Step 3 - Start Kubernetes
# Task
Launch a Kubernetes using the _start-cluster_ helper script.
```
sudo ADVERTISE_IP=172.17.0.38 ./start-cluster
```
This can take a couple of moments.
The helper script is from  [https://github.com/coreos/coreos-kubernetes/tree/master/single-node](https://github.com/coreos/coreos-kubernetes/tree/master/single-node) .
For multi-node clusters, refer to  [https://github.com/coreos/coreos-kubernetes/tree/master/multi-node](https://github.com/coreos/coreos-kubernetes/tree/master/multi-node) .
The following steps will explain the individual components being deployed.

# Step 4 - Kubelet and API Server
Kubernetes a combination of components, each run on the Master node. The Master is a single node which manages the cluster and the containers running it in. The master will launch new containers, configure networking and provide health information.
On each node a Kubelet is required to run the Kubernetes components. This is managed via Systemd.
```
cat /etc/systemd/system/kubelet.service
```
Once started, it's used to bootstrap other components of Kubernetes. For example, the API runs as a Kubernetes pod.
```
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```
## API Options
_insecure-bind-address_ binds the API to all IP addresses and makes it available via HTTP. This is useful for development purposes as it removes the need for certifications but should not be used in production.
_service-cluster-ip-range_ provides an IP range all containers will use.
_etcd_servers_ indicates where the API can find the etcd servers. This is a comma separated list of HTTP endpoints.

# Step 5 - Controller and Scheduler
The controller manager watches the state of the cluster via the API to ensure all requested services are running. When new services are deployed, the controller will communicate with the API and nodes to complete the required tasks.
```
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```
The Scheduler Server handles tracking resource use. It ensures containers can run on it's assigned node without overloading capacity.
```
cat /etc/kubernetes/manifests/kube-scheduler.yaml
```
hostname-override and address are used to bind the master to all IP addresses.
cluster_dns and cluster_domain define the DNS server (started in a future step) that allows containers to communicate via well-known names.
api-servers defines the URL the Master should use to contact the API.
config sets the manifest to use. manifests-multi is used with DNS. It represents the configuration for the cluster.

# Step 6 - Proxy
Each node in the cluster requires a running proxy server. The proxy is responsibility for managing communications by modifying the IPTables of the host machine. It also handles load balancing of traffic between containers on a host.
```
cat /etc/kubernetes/manifests/kube-proxy.yaml
```
The _master_ option is the HTTP of where the Master is running.

# Step 7 - KubeDNS / SkyDNS
When we started the Controller and Scheduler we defined a DNS IP which we'll now launch. Because Kubernetes uses etcd, it uses the related DNS service called SkyDNS.
The DNS allows containers to communicate based on well-known names instead of IP addresses.
To start a container you define a replication controller and a service in yaml files. The "Launching Guestbook" scenario covers the format and differences between rc and services (svc).
```
cat /srv/kubernetes/manifests/kube-dns-rc.json
```
```
cat /srv/kubernetes/manifests/kube-dns-svc.json
```

# Step 8 - Kube UI
Kubernetes has a UI that can be used to visualize the state of the cluster. As with the DNS service, the UI also runs inside the cluster.
```
cat /srv/kubernetes/manifests/kube-dashboard-rc.json
```
```
cat /srv/kubernetes/manifests/kube-dashboard-svc.json
```
After a few moments you will be able to visit the UI on port 8080 with the URL  [https://2886795302-8080-frugo04.environments.katacoda.com/ui/](https://2886795302-8080-frugo04.environments.katacoda.com/ui/) 

# Step 9 - Kubectl
Alongside the UI there is also a command line interface (CLI) called Kubectl. The kubectl is the command line client used to communicate with the API and the cluster.
```
curl -o ~/kubectl http://storage.googleapis.com/kubernetes-release/release/v1.3.4/bin/linux/amd64/kubectl
chmod u+x ~/kubectl
```


# Step 10 - Health Checks
With these steps completed the cluster is now running along with DNS and a UI. You can check the health running the following commands.
```
curl http://localhost:4001/version
curl http://localhost:8080/version
./kubectl cluster-info
./kubectl get nodes
```
The results from port 4001 are etcd. The results from 8080 are Kubernetes. The _export KUBERNETES_MASTER_ command sets the default server for kubectl.
