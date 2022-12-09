# HowTo: Deploy RHEL image with govc (VMware)

Red Hats online Image Builder for Red Hat Enterprise Linux can build and generate images RHEL for AWS, Azure, GCP, VMware, .qcow2 and .iso.

If your are a Red Hat customer, logon to https://console.redhat.com - Select Red Hat Enterprise Server and Insights, and Image Builder will be available in the menu. Really easy to use, so will not cover that part here.

So, after we have downloaded our freshly created .vmdk file, we can of course create a machine and add the disk, but there's an issue; We do not have any known user we can logon with to the machine. So, we need to inject cloud-init information into the machine on first boot. To do this, we need to have a look at a tool named govc. Documentation for this tool and downloads is available here: https://github.com/vmware/govmomi/blob/master/govc/README.md

Red Hat created documentation for this process here:

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/creating_customized_rhel_images_using_the_image_builder_service/assembly_creating-and-uploading-a-customized-rhel-vmdk-system-image-to-vsphere

But if you are not familiar with the govc tool, this HowTo will complement this documentation with more specific examples.

This are the files (with correct naming) that need to be created:

## metadata.yaml
This file only defines name for the virtual machine, as well as hostname.

```
instance-id: t_rhel9-gm
local-hostname: rhel9-gm
```

## userdata.yaml
This file containes the userdata including ssh-keys that you want to inject into the machine. My file is really basic, so for other needs, please refer to general cloud-init documentation as well as docs for govc.

```
#cloud-config
users:
  - default
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1y............yMweeyTc7wkdAYhn demo@hal9000.io
```

After both metadata.yaml and userdata.yaml has been created, run the following to export that information to the environment using gzip compression and base64 encoding. Will be used in the last step of this guide.

```
export METADATA=$(gzip -c9 <metadata.yaml | { base64 -w0 2>/dev/null || base64; }) \
USERDATA=$(gzip -c9 <userdata.yaml | { base64 -w0 2>/dev/null || base64; })
```

## Configure environment for govc

After you have copied the commandline tool to your preferred binary path, we need to declare some strings.

First of all, we need to be able to login to vCenter og ESXi with govc. And for that, we need these:

```
export GOVC_URL=vc.ekeberg.io
export GOVC_USERNAME=demo@ekeberg.io
export GOVC_PASSWORD=SuperSecret2!
export GOVC_INSECURE=true
```
GOVC_INSECURE disables SSL verification, and the rest is pretty obvious.

Next step is to configure some more strings based on data from vCenter. As a minimum, we need to specify datastore and network port group. I've added what folder in vCenter I want the machines created in too.

To figure out paths, use the 'govc ls' command as show in this example:

```
❯ govc ls
/SkyNet/vm
/SkyNet/network
/SkyNet/host
/SkyNet/datastore

❯ govc ls /SkyNet/network
/SkyNet/network/VLAN1
/SkyNet/network/mgmtnetwork
/SkyNet/network/DSwitch 1-DVUplinks-1021
/SkyNet/network/DSwitch 1-VLAN1
/SkyNet/network/DSwitch 1-mgmtnetwork
/SkyNet/network/DSwitch 1-VLAN1-ephemeral
/SkyNet/network/DSwitch
```

That gives me the paths I need to declare this:

```
export GOVC_DATASTORE='/SkyNet/datastore/Local01_2'
export GOVC_NETWORK='/SkyNet/network/DSwitch 1-VLAN1'
export GOVC_FOLDER=/SkyNet/vm/templates
```
In the folder I have downloaded my vmdk file, I run the following to copy the vmdk to the datastore I defined in GOVC_DATASTORE. Filename first, then foldername that the vm will reside in.

```
govc import.vmdk t_rhel9-gm.vmdk t_rhel9-gm
```
Now we are ready to create the virtual machine. Change RAM, vCPU, OS as desired, just be aware that the image only supports IDE controller for the boot disk we have just created. Machine will be name t_rhel9-gm.

```
govc vm.create \
-net.adapter=vmxnet3 \
-m=4096 -c=2 -g=rhel9_64Guest \
-firmware=bios -disk='t_rhel9-gm/t_rhel9-gm.vmdk' \
-disk.controller=ide -on=false \
 t_rhel9-gm
```
The last thing we need to do before we boot the machine, is to inject our cloud-init information into the vm, as defined in our yaml files and exported to the environment earlier.

```
govc vm.change -vm t_rhel8-gm \
-e guestinfo.metadata="${METADATA}" \
-e guestinfo.metadata.encoding="gzip+base64" \
-e guestinfo.userdata="${USERDATA}" \
-e guestinfo.userdata.encoding="gzip+base64"
```

Now, power on, find the IP and try to logon with SSH.

```
govc vm.power -on t_rhel9-gm
HOST=$(govc vm.ip t_rhel9-gm)
ssh demo@$HOST
```
