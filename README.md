# ocp4-upi-install

Created by: Stephen Nimmo

#### #######Changelog########## 
Andrew Seifert:
- updated general assembly of the infrastructure.
- added rhel8.2 installation notes
- added ansible on mac installation notes
- added coreOS install information
- added notation around pathing in ansible scripts
- added htpassword notation updates
- updated quick installation instructions to include earlier steps
- switched around a few sections for better flow
- updated cluster.yml information to clarify needs
- added more regarding what services the node1 carries on it

# About

These playbooks are designed to perform a bare metal install of OCP 4.5 on Intel NUCs. Instructions are tailored for a bare metal install using 4 machines, where one machine hosts the services needed for the cluster and the other three are control plane nodes. Network is required to download parts of the install.

The ansible playbooks are designed to:
- Generate the necessary configs and install all services on the service node. 
- Download all the necessary RHCOS and OCP install packages.
- Generate the ignition files and set them up to be served from the service node's httpd service, along with the rhcos kernel.

The networking assumes a standard class C networking subnet ( can be adjusted to whatever you want. Must be big enough to include all of the servers and the services vm required. (5 total) )

- network: {networkIP}, mask: 255.255.255.0, gateway: {gatewayIP}, dns: {dnsIP}, 8.8.8.8 (optional)
- domain: {yourdomain}.com
- {VMNodeName}.{yourdomain}.com - {subnet}.10
- {node1Name}.{yourdomain}.com - {subnet}.11
- {node2Name}.{yourdomain}.com - {subnet}.12
- {node3Name}.{yourdomain}.com - {subnet}.13
- {node4Name}.{yourdomain}.com - {subnet}.14

#### Note: Ips are not specific and can be whatever one wants.  A good idea would be to whitelist these IPs on your home network to make sure no IPs are taken during install, reboots, etc.  Check with the router's information pages to understand network whitelisting if needed.

The services node {node1} will provide the cluster dns, nfs, http, tftp, and haproxy.  For the purpose of this install it is best to allow these to be the default and any changes can be either done separately after the install or feel free to fork the project and make a new version of it.

# Quick Instructions for Use

- Install Red Hat Enterprise Linux 8.2 or higher onto node 1 ( services node ) from a USB
- Copy a pull secret down from Red Hat cloud.redhat.com (select openshift > cloud options > download pull secret)
- Copy over the coreOS install onto a USB for later use
- Install Ansible onto a local workstation and clone this git repo to that workstation
- Copy cluster_example.yml from /templates to the root of the project.
- Rename cluster_example.yml to cluster.yml
- Update the cluster.yml with the cluster information for your install.
- Execute `ansible-playbook -e @cluster.yml main.yml --ask-become-pass` from the root of the project.
- Create network bridge for {serviceNode}
- Create service node( {serviceNode})
- Execute `/ocp/openshift-install wait-for bootstrap-complete --dir=/ocp/install --log-level=debug`
- Install CoreOS on {node2name}, {node3name}, and {node4name}
- Execute `/ocp/openshift-install wait-for install-complete --dir=/ocp/install --log-level=debug` on {node1}
- Create a user after cluster is complete.

# Prerequisites

- Install Ansible on the node you will be using to configure your scripts.  This generally will be your local machine and not part of the the 4 nucs. You can however choose to run these installs from {node1} with no issues.  General rule of thumb is to install Ansible on a linux box but instructions for MacOS are below as this can get weird fast with MAC.

  - Install on MacOS:
    - Homebrew: ```homebrew install ansible```
    - Python: If python is already installed run ```pip(or pip3 depending on version) install -g ansible```
      - If Python is not installed yet install the latest version of Python and then follow the pip installs above.

- Create(if needed) and copy an ssh key for Ansible to connect to the RHEL server with
  - `ssh-keygen -f /pathTokey/keyname`
  - For this install it would be recommended to leave the password blank for the key.  Ansible will get tripped up in its defuault configuration.

#### NOTE: Ansible supports Python 2.7 and 3.5 or higher.

- Install RHEL 8.2 on services machine. ({node1})
    - Partitioning
        - / - 100 GiB
        - /boot 1024 MiB
        - /swap 15 GiB
        - /home - 100 GiB
        - /nfs - the rest! This will be used as an NFS server for OCP storage.  You can just hit enter when creating the partition to use the rest of the storage.
    - Server Install
      - When installing your RHEL server select the default 'Server with GUI' option (top of list).  
      - Remove all options for extra installation on the right panel of the screen.
- Create or copy an ssh pub key of the server you will be running ansible with to use for login.  To validate go to the node1 and check /root/.ssh/authorized keys for the key applied in this file.
    - `ssh-copy-id -i ~/pathToSSHKey/{yourPublicKey} root@{node1}`

- Setup a network bridge for libvirt to use for the bootstrap node.  This must be done first before you create your VM.
```
nmcli connection add type bridge con-name {bridgeName} ifname {bridgeName}
nmcli connection add type ethernet slave-type bridge con-name {bridgeName}-{adapter-name} ifname {adapter-name} master {bridgename}
nmcli connection modify br0 ipv4.addresses '{node1IP}/24'
nmcli connection modify br0 ipv4.gateway '{gatewayIP}'
nmcli connection modify br0 ipv4.dns '{dnsIP} 8.8.8.8'
nmcli connection modify br0 ipv4.method manual
nmcli connection modify br0 bridge.priority '16384'
nmcli connection up br0 && nmcli connection down {adapter-name}
```
## Generate your scripts via Ansible from {serviceNode} or local host

Copy cluster_example.yml from /templates to the root of the project.
- Rename cluster_example.yml to cluster.yml
- Update the cluster.yml with the cluster information for your install.
  - Replace the ssh-key variable with the full public ssh key (cat the file) - leave the tick marks.
  - Replace the pull secret variable with the pull secret (cat the file) - leave the tick marks.
  - Enter your cidr key where it says 'slash notation'
  - Enter the node names and ips where its asking for them
  - Enter the ip for the bootstrap node - dont change the name
- Execute `ansible-playbook -e @cluster.yml main.yml --ask-become-pass` from the root of the project.
  - This will generate all the configuration files needed as well as set up the services on your {node1}

#### NOTE:  When running ansible from MacOS there might be an issue where Ansible is not looking in the right location for ssh keys.  If an error caused by 'permission denied' stops the scripts - run the ansible code with '-vvv' at the end to troubleshoot the issue.

## Checking the Configuration

- Create Configuration Files for {node1} - `ansible-playbook -e @cluster.yml generate_inventory.yml generate_config.yml`
    - This creates a '.config' directory in the root of the project with the generated configs in the folders relative to where they should be put on {node1}.

# Start the Bootstrap VM

From {node1}, start the virtual machine. *You will need to be logged in to the UI of the server*. Run the command below, then open the Virtual Machine Manager, select the VM to open. 

`virt-install --name=bootstrap --vcpus=4 --ram=8192 --disk path=/nfs/libvirt/images/bootstrap.qcow2,size=120,bus=virtio --os-variant rhel8.0 --network bridge=br0,model=virtio --boot menu=on --cdrom /ocp/downloads/installer.iso`

Once booted, press tab on the boot menu and append the following to the end of the boot options. Copy the below out, edit the required fields before pasting or writing into the VM boot config.

```
ip={BootstrapNodeIP}::{gatewayIP}:255.255.255.0:bootstrap.{yourDomain}.com::none nameserver={node1IP} coreos.inst.install_dev=vda coreos.inst.image_url=http://{node1IP}:8008/rhcos/metal.raw.gz coreos.inst.ignition_url=http://{node1IP}:8008/ignition/bootstrap.ign
```

# Bootstrap the OCP cluster on {node1}

`/ocp/openshift-install wait-for bootstrap-complete --dir=/ocp/install --log-level=debug`

Once you see this.

`INFO Waiting up to 40m0s for bootstrapping to complete...`

Proceed to install the control plane machines. 

# Install RHCOS on Control Plane Machines

Download the RHCOS installer and flash a USB drive. The correct version to download can be found at https://mirror.openshift.com/pub/openshift-v4//dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer.x86_64.iso

Boot into your instance using the RHCOS ISO Installer

Once booted, press 'e' on the boot menu and append the following to the end of the boot options. The text below should be added after the text 'coreos.inst.install=true' (or something like that)

} The example below is for the first control plane machine.

```
ip={node2IP}::{gatewayIP}:255.255.255.0:{node2Name}.{yourDomain}.com::none nameserver={node1IP} coreos.inst.install_dev=sda coreos.inst.image_url=http://{node1IP}:8008/rhcos/metal.raw.gz coreos.inst.ignition_url=http://{node1IP}:8008/ignition/master.ign
```

Do that for the other two masters({node3},{node4}), changing the ip and hostname respectively. 

Then wait for the currently executing bootstrap-complete command to say it's successful. If it fails just run the bootstrap command again.

`INFO It is now safe to remove the bootstrap resources`

# Wait for the Install

From the {serviceNode}:

`/ocp/openshift-install wait-for install-complete --dir=/ocp/install --log-level=debug`

While you are waiting...

`vim /etc/haproxy/haproxy.cfg`

Remove the bootstrap entries. 

```
frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap {serviceNodeIP}:22623 check
    server node2 {node2IP}:22623 check
    server node3 {node3IP}:22623 check
    server node4 {node4IP}:22623 check
```

Then go delete your bootstrap VM and release the disk space. 

# Day 2

## Add htpasswd for admin user

From local or {node1}:

```
htpasswd -c -B -b ./{openshift.htpasswd.file} {user} {passwd}
```

#### Note: openshift.htpasswd.file can be any name you want

```
oc create secret generic {htpass-secret} --from-file=htpasswd=./openshift.htpasswd -n openshift-config
```

#### Note: htpass-secret can be anything you want to name it.  


Create your oauth.yml file:

```
vim htpasswd-oauth.yaml
```

Copy the below information into the oauth.yml file. Take note tthat the fileData: name: {htpass-secret} must be the name given during the 'oc create secret' script above.
```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider 
    mappingMethod: claim 
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

Apply the authentication to your new cluster:

```
oc apply -f htpasswd-oauth.yaml
```

Add newly added users to be cluster admins...

```
oc adm groups new admins
oc adm groups add-users admins USER
oc adm policy add-cluster-role-to-group cluster-admin admins
```

https://docs.openshift.com/container-platform/4.5/authentication/identity_providers/configuring-htpasswd-identity-provider.html

# Adding Container Storage

https://docs.openshift.com/container-platform/4.5/registry/configuring_registry_storage/configuring-registry-storage-baremetal.html#installation-registry-storage-non-production_configuring-registry-storage-baremetal


## Dynamic NFS Client Provisioner

```
oc new-project nfs-client-provisioner
```

https://medium.com/faun/openshift-dynamic-nfs-persistent-volume-using-nfs-client-provisioner-fcbb8c9344e

Don't use the helm chart. It doesn't work. Do the step by step instructions. 

# Other Day 2

`kamel install --cluster-setup`



