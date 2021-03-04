# Using Packer to automate ESXi deployment

Follow this procedure to automate the deployment of VM's on ESXi.

This guide asumes a standalone ESXi server, ie. not managed by vCenter.

## Prerequisites
1. Install [Packer](https://learn.hashicorp.com/tutorials/packer/getting-started-install). This example assumes Debian or Ubuntu.

```
$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

$ sudo apt-get update && sudo apt-get install packer
```

2. Enable the VMware ESXi guest IP hack. This is needed because `packer` connects to the VM over SSH. It needs the VMs IP address for this to succeed. On the ESXi host, perform the following from the CLI. You need SSH access to your ESXi server.

```
# esxcli system settings advanced set -o /Net/GuestIPHack -i 1
```

3. Now, there are two options. Either you use ESXi <= 6.5 or you're running ESXi >= 6.7. This is relevant because at versions ESXi >= 6.7 (I think) VMware decided to offer VNC over a websocket connection so to integrate better with their new HTML5 GUI.

---

If using 6.5 and below:

Modify the ESXi firewall config to allow inbound VNC sessions to the VMs. See also these usefull resources:

* [Creating custom firewall rules](https://kb.vmware.com/s/article/2008226)

* [`vmware-iso` Packer docs](https://www.packer.io/docs/builders/vmware/iso)

The following section needs to be appended to the services defined in `/etc/vmware/firewall/service.xml`

```
<service id="1000">
  <id>packer-vnc</id>
  <rule id="0000">
    <direction>inbound</direction>
    <protocol>tcp</protocol>
    <porttype>dst</porttype>
    <port>
      <begin>5900</begin>
      <end>6000</end>
    </port>
  </rule>
  <enabled>true</enabled>
  <required>true</required>
</service>
```

Refresh the services with the command `esxcli network firewall refresh`. Check if the new rules have been applied:

```
esxcli network firewall ruleset list | grep vnc
```

Now, in your Packer template, be sure to add the correct values for `vnc_port_min` and `vnc_port_max`. Possibly, if using ESXi 6.5, also add the `vnc_disable_password: true` option to the template.

---

If using ESXi >= 6.7 then the following applies:

* [`vnc_over_websocket` documentation](https://www.packer.io/docs/builders/vmware-iso#vnc_over_websocket)
* [VNC to ESXi 6.x and 7.0](https://www.virtuallyghetto.com/2020/10/quick-tip-vmware-iso-builder-for-packer-now-supported-with-esxi-7-0.html)

There is no need to open special ports. VNC is multiplexed over the regular tcp/443 HTTPS connection to your ESXi host. However, in your Packer template, be sure to add the correct values for `vnc_over_websocket` and possibly `insecure_connection`.

---

4. Install OVFtool from the VMware website on your local workstation. Download from the [following URL](https://code.vmware.com/web/tool/4.4.0/ovf)

```
sudo sh ./VMware-ovftool-4.4.1-16812187-lin.x86_64.bundle --eulas-agreed --required --console
```

## Build a packer config

Set the credentials used to login to the ESXi server:

```
export ESXI_HOST="your.esxi.host"
export ESXI_DS="datastore1"
export HV_USER="your_esxi_admin"
export HV_PASS="SuperMegaStrongPassword"
```

Check the config:

```
packer validate -var-file=packer-esxi/esxi-variables.json packer-esxi/ubuntu-2004/ubuntu-2004.json
```

Build the VM:
```
packer build -var-file=packer-esxi/esxi-variables.json packer-esxi/ubuntu-2004/ubuntu-2004.json
```

Or when building k8s nodes:

```
VMNAME="k8s-master1" packer validate -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004-master.json
VMNAME="k8s-node1" packer validate -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004-node.json
```

Build the VM:
```
VMNAME="k8s-master1" packer build -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004-master.json
VMNAME="k8s-node1" packer build -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004-node.json
```

This concludes the Packer phase. Move on to Ansible.

# Make sure all prerequisites are set

On the host from which you will deploy Kubespray, install prerequisites:

```
sudo apt install python3 python3-pip sshpass -y
pip3 install -r requirements.txt
```

And deploy the Ansible playbooks to the freshly created hosts.

```
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i playbooks/inventory/k8s.yaml -u ubuntu -b -k -K --become-method sudo playbooks/k8s-prereq.yaml
```


# Deploy Kubernetes with kubespray

Our vm's are now preped enough to be turned into the nodes of a Kubernetes cluster. This guide
assumes the usage of [Kubespray](https://github.com/kubernetes-sigs/kubespray). 

## Set up the inventory
Note: in this case we don't execute the `pip3 install -r requirements.txt` because this command has already been executed and this sets up the Ansible
environment.

```
git clone git@github.com:kubernetes-sigs/kubespray.git
cd kubespray
cp -rfp inventory/sample inventory/k8s-cluster
```

## Tweak some stuff for useability and manageability

We need some tweaks to the default playbook:

* enable the `kube_read_only_port` so the `metrics-server` will work
* enable the Kubernetes dashboard (`dashboard_enabled: true`)
* enable Helm deployments (`helm_enabled: true`)
* enable the `metrics-server` (`metrics_server_enabled: true`)
* enable NGINX ingress controller (`ingress_nginx_enabled: true`)
* enable the Certificate Manager (`cert_manager_enabled: true`)

```
sed -i 's/# kube_read_only_port/kube_read_only_port/' \
    kubespray/inventory/k8s-cluster/group_vars/all/all.yml

sed -i 's/# dashboard_enabled: false/dashboard_enabled: true/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

sed -i 's/helm_enabled: false/helm_enabled: true/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

sed -i 's/metrics_server_enabled: false/metrics_server_enabled: true/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

sed -i 's/ingress_nginx_enabled: false/ingress_nginx_enabled: true/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

sed -i 's/cert_manager_enabled: true/cert_manager_enabled: false/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

sed -i 's/cert_manager_namespace:/# cert_manager_namespace:/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/addons.yml

```

Doublecheck the IP ranges used for services and pods. This should be unused IPv4 or IPv6 space:

```
grep kube_service_addresses -r
```

Check all other settings are OK:

```
vim kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/k8s-cluster.yml
```

## Change the default container runtime to CRI-O

See also the `docs/cri-o.md` in the Kubespray repo.

```
sed -i 's/# download_container: true/download_container: false/' \
    kubespray/inventory/k8s-cluster/group_vars/all/all.yml

echo "skip_downloads: false" >> \
    kubespray/inventory/k8s-cluster/group_vars/all/all.yml

sed -i 's/container_manager: docker/container_manager: crio/' \
    kubespray/inventory/k8s-cluster/group_vars/k8s-cluster/k8s-cluster.yml

sed -i 's/etcd_deployment_type: docker/etcd_deployment_type: host/' \
    kubespray/inventory/k8s-cluster/group_vars/etcd.yml

cat << EOF > kubespray/inventory/k8s-cluster/group_vars/all/crio.yaml
crio_registries_mirrors:
  - prefix: docker.io
    insecure: false
    blocked: false
    location: registry-1.docker.io
    mirrors:
      - location: mirror.gcr.io
        insecure: false
EOF
```

## Build the inventory

```
cd kubespray
CONFIG_FILE=inventory/k8s-cluster/hosts.yml python3 contrib/inventory_builder/inventory.py \
     k8s-master1,10.100.1.211 k8s-node1,10.100.1.213 k8s-node2,10.100.1.214
```

## Deploy the playbook

Deploy the Kubespray playbooks to create the K8S cluster:

```
ansible-playbook -i inventory/k8s-cluster/hosts.yml -u ubuntu -b -k -K -v --become-method sudo cluster.yml
```

## Using secure registries

To use secure registries, make sure the `global_auth_file` setting in `/etc/crio/crio.conf` is pointing to the
right `config.json`. The latter can be generated using a `docker login` command. For example:

```
global_auth_file = "/etc/crio/config.json"
```

TODO: find out how to automate using Kubespray.

## Install the API client

```
mkdir -p ~/.kube/

sudo curl -L "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl && sudo chmod +x /usr/local/bin/kubectl

scp root@k8s-master1:/etc/kubernetes/admin.conf ~/.kube/config

```

Give the playbook a few moments to finish. When finished (this could take up to 25 minutes). When finished, execute the following:

```
$ kubectl get nodes
NAME          STATUS   ROLES                  AGE   VERSION
k8s-master1   Ready    control-plane,master   48m   v1.20.2
k8s-master2   Ready    control-plane,master   47m   v1.20.2
k8s-node1     Ready    <none>                 46m   v1.20.2
k8s-node2     Ready    <none>                 46m   v1.20.2

```

## Useful tips

* https://k8slens.dev/
* https://www.redhat.com/sysadmin/kubespray-deploy-kubernetes
* https://medium.com/better-programming/kubernetes-tips-ha-cluster-with-kubespray-69e5bb2fa444
* https://levelup.gitconnected.com/installing-kubernetes-with-kubespray-on-nipa-cloud-a4fbeefb47ff

# Deploying ECK

To deploy the Elasticsearch, first install ECK (Elastic Cloud for Kubernetes, you should first 
apply the controller. When this is finished, you can deploy the instances.

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.4.0/all-in-one.yaml
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

Create the PV. The local filesystem must exist on the nodes (/data/es)

```
kubectl apply -f elastic-pv.yaml
```

Create the cluster:

```
kubectl apply -f elastic-711.yaml
kubectl get elasticsearch
kubectl get pods
```

ES Cluster state should be green. Now get the services:

```
$ kubectl get service
NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
elasticsearch-es-default     ClusterIP      None            <none>        9200/TCP         23h
elasticsearch-es-http        LoadBalancer   10.233.43.167   <pending>     9200:32487/TCP   23h
elasticsearch-es-transport   ClusterIP      None            <none>        9300/TCP         23h
kubernetes                   ClusterIP      10.233.0.1      <none>        443/TCP          8d
```

The automatically created superuser (`elastic`) secret can be pulled using:

```
$ PASSWORD=$(kubectl get secret elasticsearch-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
$ curl -k -u "elastic:$PASSWORD" https://k8s-node1:32487
{
  "name" : "elasticsearch-es-default-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "l4RZjpdvThSHGjjHVh5RzQ",
  "version" : {
    "number" : "7.11.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ff17057114c2199c9c1bbecc727003a907c0db7a",
    "build_date" : "2021-02-15T13:44:09.394032Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

Or, use the `kubectl port-forward` command for port-forwarding:

```
$ kubectl port-forward service/elasticsearch-es-http 9200
$ curl -k -u "elastic:$PASSWORD" https://localhost:9200
{
  "name" : "elasticsearch-es-default-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "l4RZjpdvThSHGjjHVh5RzQ",
  "version" : {
    "number" : "7.11.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ff17057114c2199c9c1bbecc727003a907c0db7a",
    "build_date" : "2021-02-15T13:44:09.394032Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

# Deploy Kibana

Now, to deploy Kibana:

```
kubectl apply -f elastic-kibana.yaml
kubectl get kibana
kubectl get pod
kubectl get service
```

To access the Kibana web interface, you can use the mapped `LoadBalancer` service port or you can use
a port-forwarder.

```
kubectl port-forward service/elasticsearch-kb-http 5601
```

The fire up a browser to https://localhost:5601


# Deploy a loadbalancer

## Installation

This example assumes the usage of the Metal-LB loadbalancer. You can also use keepalived. To install 
Metal-LB, make sure `strictARP` is set to `true` in the `kube-proxy` config.

Execute `kubectl edit configmap -n kube-system kube-proxy` and make sure the config is set accordingly:

```
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

To install MetalLB, install the manifest:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

## Configuration

Define the IP pool used on the respective network:

```
kubectl apply -f metallb-vip.yaml
```

The specified address should now be pingable:

```
$ kubectl describe configmap config -n metallb-system
Name:         config
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>

Data
====
config:
----
address-pools:
- name: vlan100
  protocol: layer2
  addresses:
  - 10.100.1.210-10.100.1.210

Events:  <none>
```

```
$ ping 10.100.1.210
PING 10.100.1.210 (10.100.1.210) 56(84) bytes of data.
64 bytes from 10.100.1.210: icmp_seq=1 ttl=64 time=0.717 ms
64 bytes from 10.100.1.210: icmp_seq=2 ttl=64 time=0.408 ms
^C
--- 10.100.1.210 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.408/0.562/0.717/0.156 ms
vincent@dev: eck$
```


