## Configure Kubectl To Access A Remote Cluster

- First, ssh into the bastion/workspace `ssh user@<PUBLIC_IP_ADDRESS>`

- Set the cluster public IP:
`KUBERNETES_PUBLIC_ADDRESS=<Controller public IP>`

- Setup Cluster data:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
```

- Set kubectl credential

```
kubectl config set-credentials admin \
 --client-certificate=admin.pem \
 --client-key=admin-key.pem
```

- Set the context for kubectl

```
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

- Configure kubectl to use the new context:

`kubectl config use-context kubernetes-the-hard-way`
