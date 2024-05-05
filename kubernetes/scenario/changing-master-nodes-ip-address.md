# Procedure for Changing Master Nodes IP in Kubernetes Cluster
This procedure ensures a safe and systematic approach to changing the master node's IP address in a Kubernetes cluster. Repeat these steps for each master node while ensuring only one node undergoes the process at a time for safety.
> [!Note]
>  Ensure to backup the /etc/kubernetes/pki directory on the intended master node to safeguard critical files.
## Draining Node
Execute `kubectl drain master-3` to gracefully evict all pods from the intended master node.

##  Changing IP and DNS
Update the IP address and DNS configuration of the intended master node.

## Identifying and Removing Certificates with Old IP
Navigate to **/etc/kubernetes/pki** and identify certificates with the old IP address as an alt name and remove these certificates.
- identify certificates
```bash
for f in $(find -name "*.crt"); do openssl x509 -in $f -text -noout > $f.txt; done
grep -Rl 192\.168\.181\.19 .
```
Result
```bash
# for me it was just apiserver
./apiserver.crt.txt
```
- Delete both the certificate and key for each identified certificate and extra files.
```bash
for f in $(find -name "*.crt"); do rm $f.txt apiserver.crt apiserver.key; done
```

## Modifying Configuration
- Pull the kubeadm configuration from the cluster into an external file using:
```bash
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm-config.yaml
```
- Edit the kubeadm-config.yaml file to include the new IP address, DNS, and hostnames under the **certSANs** section.
  <details><summary>Click to Expand: Detailed Guide for adding certSANs.</summary>
  <p>
  
  ```yaml
  apiServer:
  certSANs:
  - "192.168.171.19"
  - "kubernetes"
  - "kubernetes.default"
  - "kubernetes.default.svc"
  - "kubernetes.default.svc.cluster.local"
  - "master-3"
  - "master-3.k8s.local"
  extraArgs:  
  ```

</p>
</details>

- Edit the `kube-apiserver manifest` to updating the master node's IP address.
## Regenerating and check Certificates
```bash
# Regenerate
kubeadm init phase certs apiserver --config kubeadm-config.yaml
```
```bash
# Check the generated certificates
x509 -in apiserver.crt -text
```
## Identifying and Editing ConfigMaps
Identify ConfigMaps in the kube-system namespace that reference the old IP address and manually edit them to reflect the new IP.
```bash
kubectl -n kube-system get cm -o yaml | less
kubectl -n kube-system edit cm ....
```
## Restarting Services
Restart kubelet and containerd services to apply the changes effectively.

## Configuring Backend (HAProxy)
Ensure backend configurations like HAProxy are updated with the new master node IP.
```
backend k8s
    server  master-1 192.168.171.19:6443 check
```
## Uncordoning Node
Execute `kubectl uncordon master-3` to allow scheduling new pods on the node.
