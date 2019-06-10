# Coloclue-proxmox-pfsense

Fork of pekare/hetznet-proxmox-pfsense

I did not really like the NAT solutions recommended for Proxmox/SmartOS on Hetzner.
The perfectionist in me wanted to have the hypervisor behind the same firewall as the VM's.
This is how I managed to implement pfSense with 1 NIC (1 IP) in Proxmox using PCI passthrough.

P.S. This was written with pfSense 2.3 in mind.
Version 2.4 do not use the same installer and do not offer the option of enabling serial-console.
I suggest you consult https://doc.pfsense.org/index.php/Installing_pfSense for an alternative solution.

## step 1: install proxmox
Request a LARA (their nickname for KVM/IPMI) session.
I had a flash drive installed since exploring SmartOS earlier, so I just dd'ed the .iso from the linux rescue system.
Either that or request them to mount the iso in the LARA (KVM/IPMI) session.

## step 2: install openvswitch
edit /etc/apt/sources.list.d/pve-enterprise.list
```
#deb https://enterprise.proxmox.com/debian/pve stretch pve-enterprise
deb http://download.proxmox.com/debian/pve stretch pve-no-subscription
```
```
apt update
apt upgrade
apt install openvswitch-switch
```

## step 3: enable pci passthrough
edit /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
edit /etc/modules
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
```
update-grub
reboot
```
## step 4: create pfsense vm
- Create a VM with 1 virtio interface.
- Boot vm and install as EMBEDDED.
- Shutdown VM after first reboot.
- Enable autostart in options of VM.
- Locate your ethernet card using "lspci". The address should be in the form of: 04:00.0.

edit /etc/pve/qemu-server/VMID.conf
```
serial0: socket
hostpci0: 04:00.0
```

## step 5: change hypervisor network
edit /etc/network/interfaces
```
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet manual
        ovs_type OVSBridge
        ovs_ports int0

allow-vmbr0 int0
iface int0 inet static
        address  10.0.11.2
        netmask  255.255.255.0
        gateway  10.0.11.1
        ovs_type OVSIntPort
        ovs_bridge vmbr0
        ovs_options tag=11
```
edit /etc/hosts
```
127.0.0.1 localhost.localdomain localhost
10.0.11.2 xxx xxx pvelocalhost
```
```
reboot
```
## step 6: configure pfsense
- Back to IPMI, use 'qm terminal VMID' to connect to pfsense VM.
- Finish initial setup.
  - WAN -> em0 -> v4/DHCP4: ${external_ip}/${cidr}
  - LAN -> vtnet0.11 -> v4: 10.0.11.1/24 (vlan 11 in this case)
- Enter shell (option 8), disable pf 'pfctl -d'
- Go to https://${external_ip} and System -> Advanced -> Networking
check 'Disable hardware checksum offload'. (Not sure why, but that hindered me from connecting to webui/ssh on hypervisor.)
- Setup port forwardring or vpn to your liking..

## step 7: profit
