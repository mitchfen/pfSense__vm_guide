# Virtualizing pfSense under QEMU/KVM

## 1. Starting out - Dependencies and Hardware

### Choosing a networking card

You are going to need a networking card with at least two interfaces for pfSense.
In my experience Intel cards work much better than realtek.

Here are a few options:

| Name         | # Ports |
| ------------ | ------- |
| Intel D33682 | 2       |
| Intel 0H092P | 4       |
| Intel I350 2 | 2       |

### Ensure your host has enough disk space and memory

Whichever computer you choose as your host is going to need at least 1GB of RAM and 16GB of disk space to allocate to the VM. The host should also have 2-4 cores you can give the VM. 1GB of RAM really is the bare minimum. You'll be better off if you can spare 2-4GB of RAM for the VM.

### Choose a host OS

This guide is geared towards Linux hosts. While it is possible to run pfSense under vSphere or Virtualbox, I will only be going over the setup instructions using virt-manager. I'm using Manjaro, an Arch based distro, but the steps should be similar for Ubuntu or Debian.

### Install virt-manager and QEMU

With the host OS installed, we can now install the necessary software.
`sudo pacman -S qemu virt-manager libvirt edk2-ovmf`

Next you will want to enable and start the libvirtd service (daemon) and the logging component that goes with it. You can enter this command:  
`sudo systemctl enable --now libvirtd && sudo systemctl enable --now virtlogd`

---

## 2. Installing pfSense

### Download the pfSense install media

Go ahead and grab the [pfSense ISO](https://www.pfsense.org/download/#). Be sure to select AMD64 and CD Image (ISO). We do not want the memstick installer in this case.

### Creating the VM

Launch virt-manager and click "Create new virtual machine".
Select "Local install media", then choose the downloaded ISO.
Select FreeBSD 11.3 (Or newer depending on your version of pfSense).

Next you can select the amount of memory and disk space you'd like to allocate to the VM. By default the virtual disk is placed under `/var/lib/libvirt/images`.  
**IMPORTANT:** Before the last step, be sure to click "Customize configuration before install". If it prompts you to start the virtual network, choose no. We will not be using the virtual network.

### Configuring the VM before the first boot

You can remove the tablet input, as well as the USB redirectors. By default the VM should have one NIC (Network Interface Controller). Since we want to pass our NIC through to the VM, we need to add a second NIC using the "add hardware button", then configure both of them by modifying the XML.

Start by getting the names and MAC addressed of the network interfaces from your Linux host. The bash command `ip a` will list out the interfaces on your machine, but you can also check the gnome settings or network manager. In my case, I have one network adapter on my motherboard that I will be using for the host, and two network adapters to give over to pfSense: enp9s0f0 and enp9s0f1 (I am using the Intel D33682 from earlier).  
Once you know what interfaces you will be passing, and have added a second NIC to your VM, you can modify the XML of the virtual machine. It will look similar to this. Fill in the MAC address and change the slot numbers as needed.

```XML
<interface type="direct">
  <mac address="00:00:00:00:00:00"/>
  <source dev="enp9s0f0" mode="passthrough"/>
  <target dev="macvtap0"/>
  <model type="e1000"/>
  <link state="up"/>
  <alias name="net0"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x08" function="0x0"/>
</interface>
```

Modify the second NIC in the same fashion, being sure to use the second mac address and interface name. The slot number should also be different. In my case I used 0x08 and 0x09.
At this stage you can also take the opportunity to make sure that the host does not attempt to use these interfaces when it boots up. There is a setting in gnome-settings and network manager titled "Connect automatically" - make sure this is not checked for either interface being passed to pfSense.

### Boot into the install medium

At this stage you can click the play button to boot the VM. When you click into the VM it will capture your mouse and keyboard - you can press Ctrl+Alt to escape the VM and give your cursor back to the host.
Proceed through the install steps - they are self explanatory. The only option I will draw attention to is the partitioning scheme. You may want to choose the ZFS option. ZFS is a futuristic file system you can learn more about [here](https://en.wikipedia.org/wiki/ZFS).

---

## 3. Configuring pfSense

### Assign the WAN and LAN

Once your pfSense VM successfully boots, it will ask if you'd like to setup VLANS. You can choose no since we won't be using them for this guide. Next it will ask you to assign your WAN port. The WAN port corresponds to the port you will plug your cable modem into. The LAN port corresponds to the port on your networking card that will go out to all the other devices on your home network. In my case I selected em0 as the WAN and em1 as the LAN port.

### Log into the Web Interface

Next you can open a browser in your host OS and navigate to `192.168.1.1`. The pfSense web interface should appear and prompt you for a password.
The default username is admin, and the default password is pfsense.
Proceed through the Wizard to setup your DNS servers. Be sure to check the box to "Override DNS" since you don't want to use the slow DNS providers from your ISP. Learn more about DNS providers here.
The rest of the options, aside from setting your timezone, can be left at their default values. At the end of the wizard you can set your own password for the admin account.

### Celebrate! ✨🎉

Woohoo!! If you've made it this far then you have successfully setup pfSense in a virtual machine! Be sure to plug your LAN port into a switch and test the connectivity of your network. At this point you have an enterprise level router and firewall for your home network.

### Disable hardware offloading

While pfSense should be up and running at this point, you may notice that the connection is a bit slow. This is because of the way we are passing the NICs through to the VM. By default pfSense will try to offload some of the computation to the hardware of the networking card. This will not work in our case and so we need to disable this functionality. This will only increase the CPU overhead slightly but will significantly speed up your internet.

Click the System tab, then Advanced, then the Networking tab. Scroll to the bottom and disable all the hardware offloading options.

## 4. Installing Telegraf and Grafana

If you've reached this point you have a functional router running inside a virtual machine. These steps are optional - only if you want the Grafana dashboard.

### Install telegraf package on pfSense

Inside the pfSense web interface: select the System tab -> Package Manager -> Available packages. Install telegraf.

### Install InfluxDB and Grafana on the host

`sudo pacman -S influxdb grafana`

### Enable and start services on host

On your host OS, enter these commands into a terminal:  
`sudo systemctl enable --now influxdb && sudo systemctl enable --now grafana`

### Configure InfluxDB

Drop to a terminal and create an influx database on your host machine: see the influx documentation

### Configure Telegraf

Inside the pfSense web console at `192.168.1.1` click the services tab, then telegraf. Type in the name of the database you created as well as the IP address of your host machine.

If you plan to use pfBlocker-NG, place this in the additional configuration for telegraf:

```
[[inputs.logparser]]
files = ["/var/log/pfblockerng/dnsbl.log"]
from_beginning=true
[inputs.logparser.grok]
   measurement = "dnsbl_log"
   patterns = ["^%{WORD:BlockType}-%{WORD:BlockSubType},%{SYSLOGTIMESTAMP:timestamp:ts-syslog},%{IPORHOST:destination:tag},%{IPORHOST:source:tag},%{GREEDYDATA:call},%{WORD:BlockMethod},%{WORD:BlockList},%{IPORHOST:tld:tag},%{WORD:DefinedList:tag},%{GREEDYDATA:hitormiss}"]
   timezone = "Local"
   [inputs.logparser.tags]
      value = "1"

[[inputs.logparser]]
   files = ["/var/log/pfblockerng/ip_block.log"]
   from_beginning=true
   [inputs.logparser.grok]
      measurement = "ip_block_log"
      patterns = ["^%{SYSLOGTIMESTAMP:timestamp:ts-syslog},%{NUMBER:TrackerID},%{GREEDYDATA:Interface},%{WORD:InterfaceName},%{WORD:action},%{NUMBER:IPVersion},%{NUMBER:ProtocolID},%{GREEDYDATA:Protocol},%{IPORHOST:SrcIP:tag},%{IPORHOST:DstIP:tag},%{NUMBER:SrcPort},%{NUMBER:DstPort},%{WORD:Dir},%{WORD:GeoIP:tag},%{GREEDYDATA:AliasName},%{GREEDYDATA:IPEvaluated},%{GREEDYDATA:FeedName:tag},%{HOSTNAME:ResolvedHostname},%{HOSTNAME:ClientHostname},%{GREEDYDATA:ASN},%{GREEDYDATA:DuplicateEventStatus}"]
      timezone = "Local"
```

### Configure Grafana

Navigate to `http://localhost:3000` and use the default credentials to login (admin, admin). Set your new password, then click the option to add a data source. Plug in the influxDB information from earlier.

### Create or Import a dashboard

At this point you can either import a dashboard made by someone else, or create your own. You can view and download my dashboard from my GitHub repository if you'd like. Many people have created dashboards in JSON format if you'd like to try some.

## 5. Finishing up

### Ensure the VM runs on host startup

You will want your router to startup when you boot up your host OS. To set this, open virt-manager and check "Start virtual machine on host boot up" under the Boot options.

Congratulations!
You now have a highly overengineered router and are well on your way to having a homelab.
