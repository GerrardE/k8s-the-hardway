## Bootstrap The K8s Control Plane

- First, ssh into the controller nodes `ssh user@<PUBLIC_IP_ADDRESS>`

##### Download And Install Binaries

Run the following commands on all the control planes:

- Create a directory like so: `sudo mkdir -p /etc/kubernetes/config`

- Download the service binaries:

```
wget -q --show-progress --https-only --timestamping "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-apiserver" "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-controller-manager" "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-scheduler" "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl"
```

- Make the binaries executable:

`chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl`

- Move the files:

`sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/`

##### Configure the kube-apiserver Service

- Make a directory:

`sudo mkdir -p /var/lib/kubernetes/`

- Copy these files into it:

`sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem encryption-config.yaml /var/lib/kubernetes/`

- Set the internal Ip variable:

`INTERNAL_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)`

- Set the following variables:

```
ETCD_SERVER_0=<CONTROLLER_0_PRIVATE_IP>
ETCD_SERVER_1=<CONTROLLER_1_PRIVATE_IP>
```

- Create systemd unit for both servers:

```
cat << EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://${ETCD_SERVER_0}:2379,https://${ETCD_SERVER_1}:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2 \\
  --kubelet-preferred-address-types=InternalIP,InternalDNS,Hostname,ExternalIP,ExternalDNS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

##### Configure the kube-controller-manager Service

Run the following commands

- Move the config file:

`sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/`

- Create systemd unit control file:

```
cat << EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

##### Configure the kube-scheduler Service

- Move the config file to the k8s dir:

`sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/`

- Create config file for kube-scheduler:

```
cat << EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

- Create the systemd unit file:

```
cat << EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

##### Start All of the Services

- Enable and start the control plane services

`sudo systemctl daemon-reload`

`sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler`

`sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler`

- Verify their status:

`kubectl get componentstatuses --kubeconfig admin.kubeconfig`


##### Enable HTTP Health Checks

- Install nginx 

`sudo apt-get install -y nginx`

- Create nginx config file:

```
cat > kubernetes.default.svc.cluster.local << EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

- Move the file

`sudo mv kubernetes.default.svc.cluster.local /etc/nginx/sites-available/kubernetes.default.svc.cluster.local`

- Create a symlink to the `sites-enabled` dir:

`sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/`

- Start and enable nginx:

`sudo systemctl restart nginx`

`sudo systemctl enable nginx`

- Verify that http health checks are working: `curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz`


##### Set Up RBAC for Kubelet Authorization

Run the following command in one control plane node:

- Create a clusterrole for the k8s api server to access the kubelet

```
cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

- Assign the user to kube-apiserver user:

```
cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### Setup the Kube Api Frontend Loadbalancer

##### Install Nginx on the Load Balancer Server

ssh into the load balancer:

- Run the following commands to install, enable and start nginx on the load balancer:

```
sudo apt-get install -y nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

- Create a folder for nginx config:

`sudo mkdir -p /etc/nginx/tcpconf.d`

- Edit the main nginx file to include configs from the folder above:

`include /etc/nginx/tcpconf.d/*;`

- Create an nginx config file for loadbalancing across the 2 controllers:

```
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {
    upstream kubernetes {
        server <CONTROLLER 0 PRIVATE IP>:6443;
        server <CONTROLLER 1 PRIVATE IP>:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF
```

(Don't forget to replace the controller IPs)

- Reload the nginx config

`sudo nginx -s reload`

- Make a call to the loadbalancer to check that everything works fine

`curl -k https://localhost:6443/version`
