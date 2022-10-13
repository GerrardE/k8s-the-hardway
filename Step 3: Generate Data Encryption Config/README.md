## Generating a Data Encryption Config for Kubernetes

- First, ssh into the workspace `ssh user@<PUBLIC_IP_ADDRESS>`

##### Generating a Data Encryption Key

`export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)`

```
cat > encryption-config.yaml << EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

##### Copy the File to the Kubernetes Controller Servers

- `scp encryption-config.yaml user@<PUBLIC_IP>:~/`

`scp encryption-config.yaml cloud_user@44.200.14.124:~/`

`scp encryption-config.yaml cloud_user@3.239.242.52:~/`
