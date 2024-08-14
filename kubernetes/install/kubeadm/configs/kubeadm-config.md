Create a file called kubeadm-config.yaml with the following contents
``` yaml
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" # change this (see below)
controllerManager: {}
dns: {}
etcd:
  external:
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd.pem
    endpoints:
      - https://ETCD_0_IP:2379 # change ETCD_0_IP appropriately
      - https://ETCD_1_IP:2379 # change ETCD_1_IP appropriately
      - https://ETCD_2_IP:2379 # change ETCD_2_IP appropriately
    keyFile: /etc/kubernetes/pki/etcd/etcd-key.pem
imageRepository: REGISTERY/kubernetes
kind: ClusterConfiguration
kubernetesVersion: v1.28.11
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.16.0/20
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```
