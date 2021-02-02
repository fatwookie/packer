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
VMNAME="k8s-vm1" packer validate -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004.json
```

Build the VM:
```
VMNAME="k8s-vm1" packer build -var-file=packer-esxi/esxi-variables.json packer-esxi/k8s-2004/k8s-2004.json
```

This concludes the Packer phase. Move on to Ansible.

# Make sure all prerequisites are set

On the host from which you will deploy Kubespray, install prerequisites:

```
sudo apt install ansible sshpass -y
```

# Deploy Kubernetes with kubespray

Our vm's are now preped enough to be turned into the nodes of a Kubernetes cluster. This guide
assumes the usage of [Kubespray|https://github.com/kubernetes-sigs/kubespray]
