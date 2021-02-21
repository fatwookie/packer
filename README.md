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