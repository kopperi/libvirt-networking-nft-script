# libvirt-networking-nft-script
This is a new version of now archived scripts to manage virtual interfaces for virtual machines. Key difference compared to older version is, that this script utilizes nftables instead of nftables. 

# Prerequisities
Created for bash 5.0.3
- nftables 
- DNSmasq for each virtual interface (see 2.1)
- sudo / root 
- Router admin access for creation of static routes (for set_bridge, see 2.3.2)

# 1 What are these for?
These scripts have been created to manually control active network interfaces in system relying on systemd. 
* set_nat creates a NAT'ed virtual network. Traffic from VMs  seems to originate from host's IP address.
* set_iso creates isolated network, which has no connections to other networks. VMs in same isolated network can see each other, but they have no access to other networks.
* set_bridge creates a routed subnet, where VMs are visible to other networks (given, that routing is set up properly). This may require changes in routing, since e.g. router or default gateway might not be aware of creation of new subnet. See details below.

## 1.1 What is this based on?
Scripts are mostly based on libvirt networking manual Jamie Nguyen's tutorial which can be found here: [Libvirt Networking Handbook](https://jamielinux.com/docs/libvirt-networking-handbook/index.html)  


## 1.2 What's the difference between set_iso, set_nat and set_bridge?
All scripts have same workflow, and only differences coming from the firewall rules and interface configuration. All scripts rely on DNSMasq to provide DHCP for VMs. 

## 1.3 What do these scripts do exactly?
All scripts take argument up or down, and nothing else. All have same steps. 

### 1.3.1 Argument UP
1) Create network interface (bridge interface with user defined parameters
2) Enable and start pre-configured DNSmasq service for the network (Systemd)
3) Add nftables rules from file rules/\*.nft
4) Bring the network interface UP after all previous steps have completed

### 1.3.2 Argument DOWN
1) Stop networks DNSmasq service and disable it (Systemd)  
2) Bring network interface DOWN, and remove the interface
3) Flush the rules previously added by UP argument and set rules according to baseline.nft

# 2. Installation steps
## 2.1 DNSmasq service configuration
DNSmasq configuration can be summed up with following. 
```
#install dnsmasq
apt install dnsmasq

#copy dnsmasq config files to correct folder 
cp -R $HOME/git/libvirtd-networking-script/var/lib/dnsmasq/* /var/lib/dnsmasq/
```
You may want to adjust dnsmasq.conf in each folder (e.g. ../virbr-bridge/dnsmasq.conf). You may also want to create systemd unit files for different dnsmasq instances to manage them with systemctl. These scripts do it for you, but if you prefer to use systemd, [Jamie's guide](https://jamielinux.com/docs/libvirt-networking-handbook/appendix/run-dnsmasq-with-systemd.html) has good instructions on it. 

For reference, original guides for running DNSmasq can be found here:

[NAT network](https://jamielinux.com/docs/libvirt-networking-handbook/custom-nat-based-network.html#run-dnsmasq)  
[Routed network](https://jamielinux.com/docs/libvirt-networking-handbook/custom-routed-network.html#run-dnsmasq)


## 2.2 Define baseline rules to contain your normal firewall setup
Before you use this *create baseline.nft under rules directory* by issuing the following (as root)
```
echo "#!/usr/bin/nft -f" > rules/baseline.nft
nft list ruleset >> rules/baseline.nft
```

After that, adjust nftables rules to fit your environment. *At least* check that IP ranges in script match your liking. Rules can be found in rules/\*.nft files
 
## 2.3 Create symlinks
To easily use the scripts, you can create symlinks to e.g. /usr/local/bin for each script
```
#E.g.
ln -s $HOME/git/libvirt-networking-nft-script/set_bridge_nft set-bridge
```
After this, you can use the scripts as you would any other command, e.g. set-bridge up.

## 2.4 Configuring your VMs to use right kind of network (Mandatory) 
Last step is to configure your virtualmachine to use the network. Instructions below concern only workin on QEMU/KVM. These should work on other virtualized environments as well (e.g. VMWare and Virtualbox), though I haven't tested them personally.

### 2.4.1 Configuration of libvirt VM (Qemu/KVM)
Virtualmachines in Qemu/KVM won't start if your configured network device doesn't exist. You can use virt-manager GUI to define used network interfaces or you can use virsh edit <Virtualmachine ID> to perform changes to VM's XML.

If you want to use virsh edit. following should be added (or replaced) in your libvirt VM's configuration file. You can find the missing information (INTERFACE_MAC_ADDRESS and INTERFACE_NAME) in function file set_(nat/iso/bridge) at same place where you configure the parameters.
```
interface type='bridge'    
mac address='<add_random_mac_address'   #Usually set automatically by libvirt     
source bridge'='<interface_name>'       #E.g. virbr-bridge     
model type='<Virtual_NIC_device>'       #E.g. virtio
```
### 2.3.2 Additional steps for configuration of custom routed network (set_bridge)
In order for a router to understand where to send packets belonging to VM in subnet created by set_bridge, it may be necessary to create a static route. In router settings, set static route with $ip_cidr via <host_interface>. For example, if $ip_cidr=192.168.3.1/24 and host's NIC IP is 192.168.1.5, then set static route as 192.168.3.0/24 via 192.168.1.5/24.
With some routers functioning as gateway between internal and external networks, it might be necessary to also create a firewall rule to NAT connections coming from virtual subnet towards external network.

# 3 Using the scripts
Add them to folder included in $PATH, e.g. /usr/local/bin. You can use soft links as well.  Once the scripts are found in $PATH, you may use them like other commands. Scripts do need sudo or root to make changes to nftables, interfaces, and managing services.

 You can use following commands to verify that scripts are working as intended.
 ```
 #check that interfaces have been added / removed properly
 ip a 
 #check that DNSmasq service for network is running/killed
 systemctl status dnsmasq@virbr_XXXX.service 
 #check that firewall rules are in place / removed
 nftables list ruleset 
```
