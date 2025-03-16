# 

**_Fix permission issue I had with kubectl:_**

```
sudo chown -R $USER:$USER /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc
```

**_Fix for Cilium looking at 10. address for the API:_**
```
API_SERVER_IP=<IP>
API_SERVER_PORT=6443
cilium install \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set kubeProxyReplacement=true
```
**_BGP did not work from the gui, needed to ssh into the UDM SE:_**

```
sed -i 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
systemctl enable frr.service && service frr start
vi /etc/frr/frr.conf
```
```
#
log file stdout
hostname UDMSE
router bgp 64513
 bgp router-id 192.168.1.1
 bgp bestpath as-path multipath-relax
 no bgp ebgp-requires-policy
 neighbor 192.168.10.21 remote-as 64512
 neighbor 192.168.10.23 remote-as 64512
 neighbor 192.168.10.30 remote-as 64512
 neighbor 192.168.10.31 remote-as 64512
```
**_To apply the v2alpha CRD (this was all that worked for me):_**
```
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/main/pkg/k8s/apis/cilium.io/client/crds/v2alpha1/ciliumbgppeeringpolicies.yaml
```
**_To test from UDM:_**
```
vtysh -c "show ip bgp"
```
**_and_**
```
vtysh -c "show bgp summary"
```
**_The key to this was that I thought 'Active' meant it was talking but it does not, it needed to say Established before it was actually working. The issue was sort of bizzare, I originally set localASN: 64512 (K3s) and peerASN: 64513 (UDM SE) correctly in the file. Noticed in the log they both showed 64512, I reapplied the config and still it was wrong, except when I reapplied the config it was actually wrong in the file after. Once I changed it, they were no longer on the same ASN and that was when it ended up finally working._**

**_To check from Cilium side run:_**
```
cilium bgp peers
```
**EX:**__
```
josh@k3s-master1:~/plexyandyouknowit/k3s-manifests/storage$ cilium bgp peers
Node          Local AS   Peer AS   Peer Address   Session State   Uptime      Family         Received   Advertised
k3s-master1   64512      64513     192.168.1.1    established     10h18m43s   ipv4/unicast   0          0
                                                                              ipv6/unicast   0          0
k3s1          64512      64513     192.168.1.1    established     10h22m30s   ipv4/unicast   0          0
                                                                              ipv6/unicast   0          0
k3s2          64512      64513     192.168.1.1    established     10h22m45s   ipv4/unicast   0          0
                                                                              ipv6/unicast   0          0
k3s3          64512      64513     192.168.1.1    established     34m52s      ipv4/unicast   0          0
```
**_To apply storage configs:_**
```
kubectl apply -f persistent-volumes.yaml
kubectl apply -f persistent-volume-claims.yaml
```
**_To verify storage use:_**
```
kubectl get pv
kubectl get pvc
kubectl get storageclass
```
**_This is how it should look:_**
```
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
k3s-configs-pv   500Gi      RWX            Retain           Bound    default/k3s-configs-pvc                  <unset>                          3h47m
k3s-data-pv      30Ti       RWX            Retain           Bound    default/k3s-data-pvc                     <unset>                          3h47m
NAME              STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
k3s-configs-pvc   Bound    k3s-configs-pv   500Gi      RWX                           <unset>                 3h47m
k3s-data-pvc      Bound    k3s-data-pv      30Ti       RWX                           <unset>                 3h47m
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  18h

```
To apply the services:
```
kubectl apply -f arr.yaml
```


