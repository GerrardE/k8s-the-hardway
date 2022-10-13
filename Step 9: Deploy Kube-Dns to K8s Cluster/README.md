## Deploy Kube-Dns To K8s Cluster

- First, ssh into the controller node `ssh user@<PUBLIC_IP_ADDRESS>`

- Deploy kube-dns

`kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml`

You can try to exec into anypod and run an nslookup to a service or pod and see the result
