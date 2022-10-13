## Setting Up K8s Networking

- First, ssh into the worker nodes `ssh user@<PUBLIC_IP_ADDRESS>`

- Enable IP Forwarding On All Worker Nodes

`sudo sysctl net.ipv4.conf.all.forwarding=1`

`echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf`

- Install weavenet on the cluster via the controller node

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.200.0.0/16"` OR

`kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml`
