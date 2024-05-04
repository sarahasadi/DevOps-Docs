# Procedure for Changing External etcd Nodes IP
This document outlines the steps required to change the IP addresses of etcd nodes in an external Kubernetes cluster.
> [!Note]
>  Ensuring to perform each step on one node at a time to maintain system stability and safety.
## Table of Machines
| Name |  Purpose  |	Old IP	| New IP|
| :-------------: | :-------------: | :-------------: | :-------------: | 
| Bastion Host | Central point for managing | 192.168.181.10 | 192.168.181.10 |
| etcd-1 | Kubernetes ectd node | 192.168.181.14 | 192.168.171.14 | 
| etcd-2 | Kubernetes ectd node | 192.168.181.15 | 192.168.171.15 | 
| etcd-3 | Kubernetes ectd node | 192.168.181.16 | 192.168.171.16 |
| Master-1 | Kubernetes master node	 | 192.168.181.17 | 192.168.171.17 |
| Master-2 | Kubernetes master node	 | 192.168.181.18 | 192.168.171.18 |
| Master-3 | Kubernetes worker node |  192.168.181.19| 192.168.171.19 |

## Generate Certificates
Generate a certificate with old and new etcd nodes IP addresses and also wildcard etcd domain (e.g., *.etcd.local).
<details><summary>Click to Expand: Detailed Guide for Creating a Certificate Authority (CA)</summary>
<p>
  
```ruby
# On the Bastion Host

{
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes-etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IR",
      "O": "Kubernetes",
      "OU": "ETCD-CA",
      "ST": "Teh"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
}
```

</p>
</details>
<details><summary>Click to Expand: Generating Certificates for etcd Nodes</summary>
<p>
  
```ruby
# On the Bastion Host

ETCD_HOSTNAMES=*.etcd.local
ETCD_IP=127.0.0.1,192.168.171.14,192.168.171.15,192.168.171.16,192.168.181.14,192.168.181.15,192.168.181.16

cat > etcd-csr.json <<EOF
{
  "CN": "*.etcd.local",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IR",
      "O": "Kubernetes",
      "OU": "ETCD",
      "ST": "Teh"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${ETCD_IP},${ETCD_HOSTNAMES} \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd-cluster
```

</p>
</details>

## Distribute Certificates
Copy the necessary new certificate files to all etcd nodes and master nodes in the configured path location:
```bash
# On the Bastion Host

# For etcd nodes
for i in `seq 14 16`; do scp etcd-cluster-key.pem etcd-cluster.pem root@192.168.181.$i:/etc/etcd/pki; done
# For master nodes
for i in `seq 17 19`; do scp etcd-cluster-key.pem etcd-cluster.pem root@192.168.181.$i:/etc/kubernetes/pki/etcd/; done
```
## Edit Kubernetes Master Nodes and etcd Nodes Configuration
> [!Important]
> - Ensure proper resolution of etcd Host and DNS names.
> - Before proceeding with these changes, make sure you have backed up the directory /etc/kubernetes/.

**Step 1:** Update the etcd configuration **on all etcd** nodes with the new etcd certificates and restart the etcd service to apply changes after **step 3.**
<details><summary>Expand to see the part of etcd TLS configuration</summary>
  
```bash
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name etcd-1\
  --cert-file=/etc/etcd/pki/etcd-cluster.pem \
  --key-file=/etc/etcd/pki/etcd-cluster-key.pem \
  --peer-cert-file=/etc/etcd/pki/etcd-cluster.pem \
  --peer-key-file=/etc/etcd/pki/etcd-cluster-key.pem \
  --trusted-ca-file=/etc/etcd/pki/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/pki/ca.pem \
```
</details>

**Step 2:** On **one master node**, edit the kubeadm configmap with the new etcd certificate and change the etcd IP to etcd dns name.
```bash
kubectl -n kube-system edit cm kubeadm-config
```
<details><summary>Expand to see the part of external etcd configuration in the manifest </summary>
  
```bash
    etcd:
      external:
        caFile: /etc/kubernetes/pki/etcd/ca.pem
        certFile: /etc/kubernetes/pki/etcd/etcd-cluster.pem
        endpoints:
        - https://etcd-1.etcd.local:2379
        - https://etcd-2.etcd.local:2379
        - https://etcd-3.etcd.local:2379
        keyFile: /etc/kubernetes/pki/etcd/etcd-cluster-key.pem
```
</details>

**Step 3:** Additionally, update the same as Step 2 **one by one** in the kube-apiserver manifest on each master node. It is strongly recommended to keep them both in sync.

```bash
# Restart etcd service

systemctl daemon-reload
systemctl start etcd
```

## Check Member ID and Cluster Health
```bash
# Check Cluster Health

ETCDCTL_API=3 etcdctl endpoint health \
--write-out=table \
--endpoints=https://192.168.181.14:2379,https://192.168.181.15:2379,https://192.168.181.16:2379 \
--cacert=/etc/etcd/pki/ca.pem \
--cert=/etc/etcd/pki/etcd.pem \
--key=/etc/etcd/pki/etcd-key.pem
```
```bash
# Check Member IDs

ETCDCTL_API=3 etcdctl endpoint status \
--write-out=table \
--endpoints=https://192.168.181.14:2379,https://192.168.181.15:2379,https://192.168.181.16:2379 \
--cacert=/etc/etcd/pki/ca.pem \
--cert=/etc/etcd/pki/etcd.pem \
--key=/etc/etcd/pki/etcd-key.pem
```
## Remove etcd Node
- Stop etcd service on the node to be removed:
```bash
systemctl stop etcd
```
- Execute the following command on existing nodes to remove the specified etcd member:
```bash
ETCDCTL_API=3 etcdctl member remove f9184ef2d52a2163 \
--endpoints=https://192.168.181.15:2379,https://192.168.181.16:2379 \
--cacert=/etc/etcd/pki/ca.pem \
--cert=/etc/etcd/pki/etcd-cluster.pem \
--key=/etc/etcd/pki/etcd-cluster-key.pem
```
- Delete the removed etcd data directory:
```bash
rm -rf /var/lib/etcd/*
```
## Change IP and DNS Records
Modify the IP address and DNS record for the removed etcd node on all etcd and master nodes in the cluster.
## Update etcd configuration on the previously removed etcd node as a new Member
```bash
# Example
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name etcd-1\
  --cert-file=/etc/etcd/pki/etcd-cluster.pem \
  --key-file=/etc/etcd/pki/etcd-cluster-key.pem \
  --peer-cert-file=/etc/etcd/pki/etcd-cluster.pem \
  --peer-key-file=/etc/etcd/pki/etcd-cluster-key.pem \
  --trusted-ca-file=/etc/etcd/pki/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/pki/ca.pem \
  --peer-client-cert-auth=true\
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.171.14:2380 \
  --listen-peer-urls https://192.168.171.14:2380 \
  --listen-client-urls https://192.168.171.14:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://etcd-1.etcd.local:2379,https://192.168.171.14:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd-1=https://192.168.171.14:2380,etcd-2=https://192.168.181.15:2380,etcd-3=https://192.168.181.16:2380 \
  --initial-cluster-state existing \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
## Add New Member
- On one of the existing nodes:
```bash
ETCDCTL_API=3 etcdctl member add etcd-1 \
--peer-urls=https://etcd-1.etcd.local:2380 \
--endpoints=https://192.168.181.15:2379,https://192.168.181.16:2379 \
--cacert=/etc/etcd/pki/ca.pem \
--cert=/etc/etcd/pki/etcd-cluster.pem \
--key=/etc/etcd/pki/etcd-cluster-key.pem
```
- Start the new member:
```bash
systemctl daemon-reload
systemctl start etcd
```
## Restart Existing Members
- Change the etcd configuration for existing members and restart the etcd service:
```bash
systemctl daemon-reload
systemctl restart etcd
```
## Edit Kubernetes Configuration
- On one master node, edit the kubeadm configmap with the new etcd certificate and change the etcd IP to etcd name.
- Additionally, update this step in the kube-apiserver manifest.

> Repeat these steps for each etcd node in the external etcd cluster for Kubernetes.
### Result
```bash
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
--endpoints=https://etcd-1.etcd.local:2379,https://etcd-2.etcd.local:2379,https://etcd-3.etcd.local:2379 \
--cacert=/etc/etcd/pki/ca.pem \
--cert=/etc/etcd/pki/etcd-cluster.pem \
--key=/etc/etcd/pki/etcd-cluster-key.pem
```
![pic](https://github.com/sarahasadi/kubernetes/assets/157595779/767875d3-b5bb-4a56-999e-d1cf10745b3c)
