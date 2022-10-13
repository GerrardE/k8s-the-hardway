## Bootstrap An etcd Cluster

- First, ssh into the controller nodes `ssh user@<PUBLIC_IP_ADDRESS>`

##### Install the etcd Binary on Each Control Node:

- Download the etcd binaries with the following command on each node:

`wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.5/etcd-v3.3.5-linux-amd64.tar.gz"`

- Extract the etcd archive on the *nodes*:

`tar -xvf etcd-v3.3.5-linux-amd64.tar.gz`

- Move the extracted files to the `/usr/local/bin` directory in *nodes*:

`sudo mv etcd-v3.3.5-linux-amd64/etcd* /usr/local/bin/`

##### Configure and Start the etcd Service on Each Control Node:

- Create the ffg directories on each node:

`sudo mkdir -p /etc/etcd /var/lib/etcd`

- Copy the files to the `/etc/etcd/` directory of each node:

`sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/`

- Set the ffg env variables on each node:

```
PRIVATE_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
ETCD_NAME=controller-0
INTERNAL_IP=$(PRIVATE_IP)
CONTROLLER_0_INTERNAL_IP=$(PRIVATE_IP_0)
CONTROLLER_1_INTERNAL_IP=$(PRIVATE_IP_1)
```

e.g for controller 0

```
ETCD_NAME=controller-0
INTERNAL_IP=10.0.1.147
CONTROLLER_0_INTERNAL_IP=10.0.1.147
CONTROLLER_1_INTERNAL_IP=10.0.1.54
```

controller 1

```
ETCD_NAME=controller-1
INTERNAL_IP=10.0.1.54
CONTROLLER_0_INTERNAL_IP=10.0.1.147
CONTROLLER_1_INTERNAL_IP=10.0.1.54
```

- Create the etcd systemd unit file:

```
cat << EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://${CONTROLLER_0_INTERNAL_IP}:2380,controller-1=https://${CONTROLLER_1_INTERNAL_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- Enable and start etcd on each node:

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

- Verify that the etcd cluster works well:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```
