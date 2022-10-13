## Smoke Testing A K8s Cluster

- First, ssh into the controller node `ssh user@<PUBLIC_IP_ADDRESS>`

##### Verify Cluster's Ability TO Perform Data Encryption

- Create test secret:

`kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"`

- Review raw data from etcd:

```
sudo ETCDCTL_API=3 etcdctl get --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/kubernetes.pem --key=/etc/etcd/kubernetes-key.pem/registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

Output should look somewhat like so:

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a fc 21 ee dc e5 84 8a  |:v1:key1:.!.....|
00000050  53 8e fd a9 72 a8 75 25  65 30 55 0e 72 43 1f 20  |S...r.u%e0U.rC. |
00000060  9f 07 15 4f 69 8a 79 a4  70 62 e9 ab f9 14 93 2e  |...Oi.y.pb......|
00000070  e5 59 3f ab a7 b2 d8 d6  05 84 84 aa c3 6f 8d 5c  |.Y?..........o.\|
00000080  09 7a 2f 82 81 b5 d5 ec  ba c7 23 34 46 d9 43 02  |.z/.......#4F.C.|
00000090  88 93 57 26 66 da 4e 8e  5c 24 44 6e 3e ec 9c 8e  |..W&f.N.\$Dn>...|
000000a0  83 ff 40 9a fb 94 07 3c  08 52 0e 77 50 81 c9 d0  |..@....<.R.wP...|
000000b0  b7 30 68 ba b1 b3 26 eb  b1 9f 3f f1 d7 76 86 09  |.0h...&...?..v..|
000000c0  d8 14 02 12 09 30 b0 60  b2 ad dc bb cf f5 77 e0  |.....0.`......w.|
000000d0  4f 0b 1f 74 79 c1 e7 20  1d 32 b2 68 01 19 93 fc  |O..ty.. .2.h....|
000000e0  f5 c8 8b 0b 16 7b 4f c2  6a 0a                    |.....{O.j.|
000000ea
```

##### Verify That Deployments Work

- Create a new deployment:

`kubectl run nginx --image=nginx`

- Verify that pod was created and its running:

`kubectl get pods -l run=nginx`

##### Verify That Portforwarding works

- Get name of nginx pod:

`POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")`

- Forward the port 80 to 8081 on the localsystem

`kubectl port-forward $POD_NAME 8081:80`

- Verify the portfoward works by hitting the port

`curl --head http://127.0.0.1:8081`

##### Verify That Container Logs Can Be Accessed

- Get name of pod:

`POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")`

- `kubectl logs $POD_NAME`

##### Verify That Commands Can Be Executed In The Pod

- `kubectl exec -ti $POD_NAME -- nginx -v`

##### Verify That Services Work

- Create a service

`kubectl expose deployment nginx --port 80 --type NodePort`

- Get the nodeport newly assigned to the service and assign it to a local variable:

`NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')`

- Access it from one of the worker node:

`curl -I 10.0.1.102:$NODE_PORT`
