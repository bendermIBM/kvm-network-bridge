# Network Bridge with OSA

> A network bridge is a link-layer device which forwards traffic between networks based on a table of MAC addresses. The bridge builds the MAC addresses table by listening to network traffic and thereby learning what hosts are connected to each network. For example, you can use a software bridge on a Red Hat Enterprise Linux 8 host to emulate a hardware bridge or in virtualization environments, to integrate virtual machines (VM) to the same network as the host. 
>
> Sourced from [RHEL Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-a-network-bridge_configuring-and-managing-networking)

This is a guide for setting up a network bridge on a RHEL 8.3 KVM host and it will largely follow the process described in [RedHat's documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-a-network-bridge_configuring-and-managing-networking) in the "Configuring and Managing Networking" chapter. I will be referencing the documentation heavily in this guide to provide a consistent approach to setting up the bridge. 

## References

These are a collection of references that I used to build this guide. They are all very well written and are great sources of information, this guide just serving as an amalgamation of them.

[1] - [RHEL Chapter 11 : Configuring a network bridge](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-a-network-bridge_configuring-and-managing-networking)

[2] - [Libvirt Wiki - Networking](https://wiki.libvirt.org/page/Networking)

[3] - [Linux Config.org - How to use bridged networking with libvirt and KVM ](https://linuxconfig.org/how-to-use-bridged-networking-with-libvirt-and-kvm)

[4] - [JamieLinux.com - Libvirt Networking Handbook (bridged networking)](https://jamielinux.com/docs/libvirt-networking-handbook/bridged-network.html)

[5] - [RHEL Chapter 13 : Configuring Network Bonding](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-network-bonding_configuring-and-managing-networking)

## Configuration

- RHEL 8.3
- 4x OSA Cards
- 2x Bonded links, each assigned two OSA cards
- 1x Bridge, using one of the Bonded links

## Contents

1. [Setup the Linux Bridge](#setup-the-linux-bridge)
2. [Configure KVM with Linux Bridge](#setup-kvm-with-linux-bridge)
3. [Modify/Install KVM Guest to utilize Bridge Network](#set-up-kvm-guest-with-kvm-bridge-network)

##### Guide Note

In this guide we will heavily be using the `nmcli` interface. Another option would be to modify the files located in `/etc/sysconfig/network-scripts` directly, and is a matter of personal preference. For consistency, `nmcli` provides us the ability to copy & paste commands which template configuration files, rather than modifying them directly

## Creating "bond" interfaces

You can skip this section if you already have a bond interface created, otherwise follow these steps to get it configured. We will be setting up a bond with an "active-backup" configuration, however you may want to choose a different option. Extended information about the available configurations can be found in [Chapter 13, Section 4 of RedHat's documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-network-bonding_configuring-and-managing-networking)

1. Create a new bond interface

    ```
    nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup"
    ```

    To additionally set a Media Independent Interface (MII) monitoring interval, add the miimon=interval option to the bond.options property. ~ [Chapter 13](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-network-bonding_configuring-and-managing-networking)

    ```
    nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=1000"
    ```

2. Create OSA interfaces to the bond connection

    ```
    nmcli connection add type ethernet slave-type bond con-name bond0-port1 ifname enp7s0 master bond0
    ```

    ... and the secondary (backup) OSA

    ```
    nmcli connection add type ethernet slave-type bond con-name bond0-port2 ifname enp8s0 master bond0
    ```

    Alternatively if you already have linux interfaces defined for the OSA devices you will just have to modify those connections to assign their "master"

    ```
    nmcli connection modify enp7s0 master bond0
    nmcli connection modify enp8s0 master bond0
    ```

3. **Optional** - Assign IP to bond interface
    
    If this is the bond interface that you are going to use to back the KVM guest's network, you **do not** need to do this.

    ```
    nmcli connection modify bond0 ipv4.addresses '192.0.2.1/24'
    nmcli connection modify bond0 ipv4.gateway '192.0.2.254'
    nmcli connection modify bond0 ipv4.dns '192.0.2.253'
    nmcli connection modify bond0 ipv4.dns-search 'example.com'
    nmcli connection modify bond0 ipv4.method manual
    ```

    ```
    nmcli connection modify bond0 ipv6.addresses '2001:db8:1::1/64'
    nmcli connection modify bond0 ipv6.gateway '2001:db8:1::fffe'
    nmcli connection modify bond0 ipv6.dns '2001:db8:1::fffd'
    nmcli connection modify bond0 ipv6.dns-search 'example.com'
    nmcli connection modify bond0 ipv6.method manual
    ```

    If you **do not** want an IP configured to the interface, set `ipv4.method == disabled` and `ipv6.method == ignore`

    ```
    nmcli connection modify bond0 ipv4.method disabled
    nmcli connection modify bond0 ivp6.method ignore
    ```

4. Modify a few configuration settings to bring up the "slave" devices automatically with the bond

    ```
    nmcli connection modify bond0 connection.autoconnect-slaves 1
    ```

5. Activate Bond Connection

    ```
    nmcli connection up bond0
    ```

6. Verify bond is up by running `nmcli connection show`, additionally you can check the bond status by running...

    ```
    # cat /proc/net/bonding/bond0
    Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

    Bonding Mode: fault-tolerance (active-backup)
    Primary Slave: None
    Currently Active Slave: enp7s0
    MII Status: up
    MII Polling Interval (ms): 100
    Up Delay (ms): 0
    Down Delay (ms): 0

    Slave Interface: enp7s0
    MII Status: up
    Speed: Unknown
    Duplex: Unknown
    Link Failure Count: 0
    Permanent HW addr: 52:54:00:d5:e0:fb
    Slave queue ID: 0

    Slave Interface: enp8s0
    MII Status: up
    Speed: Unknown
    Duplex: Unknown
    Link Failure Count: 0
    Permanent HW addr: 52:54:00:b2:e2:63
    Slave queue ID: 0
    ```

## Setup the Linux Bridge

1. Remove IP Configuration from the guests' bond interface if present

2. Set up a Linux bridge interface on one of the bonded OSA pairs

    ```
    nmcli connection add type bridge con-name bridge0 ifname bridge0
    ```

3. Assign one of the bonds as a "slave" to the bridge interface

    ```
    nmcli connection modify <bond interface> master bridge0
    ```

4. **Optional** - Assign IP to bridge interface
    
    Given that this bonded OSA pair will be used for the guest network, it's not necessary to put an IP on the interface, but is an available option.

    ```
    # nmcli connection modify bridge0 ipv4.addresses '192.0.2.1/24'
    # nmcli connection modify bridge0 ipv4.gateway '192.0.2.254'
    # nmcli connection modify bridge0 ipv4.dns '192.0.2.253'
    # nmcli connection modify bridge0 ipv4.dns-search 'example.com'
    # nmcli connection modify bridge0 ipv4.method manual
    ```

    ```
    # nmcli connection modify bridge0 ipv6.addresses '2001:db8:1::1/64'
    # nmcli connection modify bridge0 ipv6.gateway '2001:db8:1::fffe'
    # nmcli connection modify bridge0 ipv6.dns '2001:db8:1::fffd'
    # nmcli connection modify bridge0 ipv6.dns-search 'example.com'
    # nmcli connection modify bridge0 ipv6.method manual
    ```

5. **Optional** - Modify/Disable Spanning Tree Protocol settings


    Spanning Tree Protocol alerts the rest of your network of a new bridge being initiated on the network, and can potentially cause issues with automated switch protection protocols protecting against L2 network loops. **Please have a discussion** with your network team in regards to this so that they can either modify your switch port, or recommend that you disable STP for this bridge. 


    ```
    nmcli connection modify bridge0 bridge.stp no
    ```

    or possibly.. 


    ```
    nmcli connection modify bridge0 bridge.priority '16384'
    ```

6. Modify a few configuration settings to bring up the "slave" devices automatically with the bridge

    ```
    nmcli connection modify bridge0 connection.autoconnect-slaves 1
    ```

7. Bring up the bridge connection

    ```
    nmcli connection up bridge0
    ```

8. Verify Setup

    There are a few things that we should now be able to do from the bridge interface to verify the functionality before moving onto the next steps.

    First we should see the physical OSA devices present in bridge0's `ip link` definition

    ```
    [root@zt93kd ~]# ip link show master bridge0
    4: enc180: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master bridge0 state UP mode DEFAULT group default qlen 1000
        link/ether aa:95:0b:1c:b8:94 brd ff:ff:ff:ff:ff:ff
    ```

    Additionally we can execute `bridge link show` to see some similar information

    ```
    [root@zt93kd ~]# bridge link show
    4: enc180: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master bridge0 state forwarding priority 32 cost 100
    ```

    Finally, to verify that everything is plugged in correctly we can execute a tcpdump on the bridge interface, ensuring that we are seeing ARP requests coming from the rest of the infrastructure plugged into the switch. 

    ```
    [root@zt93kd ~]# tcpdump -i bridge0 -e -nnn
    dropped privs to tcpdump
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on bridge0, link-type EN10MB (Ethernet), capture size 262144 bytes
    11:37:10.577304 52:54:00:97:41:2b > 33:33:00:00:00:02, ethertype 802.1Q (0x8100), length 74: vlan 1298, p 0, ethertype IPv6, fe80::5054:ff:fe97:412b > ff02::2: ICMP6, router solicitation, length 16
    11:37:10.626569 02:00:00:02:01:84 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 46: vlan 1308, p 0, ethertype ARP, Request who-has 10.20.XXX.XXX tell 10.20.XXX.XXX, length 28
    ```

## Setup KVM with Linux Bridge

Next we need to notify KVM of the new bridge interface that we have setup. The easiest way to do this is to template out a configuration file, and run `virsh net-define <filepath>` to get us set up.

#### Steps

1. Template out an XML file somewhere to the filesystem that a KVM user (or root) has access to. 
   
    ```
    $ cat /root/hostbridge.xml
    <network>
      <name>host-bridge</name>
      <forward mode="bridge"/>
      <bridge name="bridge0"/>
    </network>
    ```

2. Now, let's define that KVM bridge network & verify it gets defined.

    ```
    virsh net-define /root/hostbridge.xml
    ```

    ```
    [root@zt93kd ~]# virsh net-list
    Name          State    Autostart   Persistent
    ------------------------------------------------
    host-bridge   inactive no         yes
    ```

3. Next we need to start up that network, and persist it's definition.

    ```
    virsh net-start host-bridge && virsh net-autostart host-bridge
    ```

4. Verify that bridge is started and available

    ```
    [root@zt93kd ~]# virsh net-list
    Name          State    Autostart   Persistent
    ------------------------------------------------
    host-bridge   active   yes         yes
    ```

## Set up KVM Guest with KVM Bridge Network

We are now at a point where we can begin to re-configure Linux guests to utilize the new bridge network just created. If you are creating a new KVM guest it's as simple as passing in the correct parameter to `virt-install`. 

#### New Guest

The parameter that we want to ensure is set is `--network`. It would look something like

```
--network network=host-bridge
```

###### Full Example
```
virt-install --connect qemu:///system --name bender0 --vcpus 1 --memory 1024 --disk /var/lib/libvirt/images/bender0.qcow2  --network network=host-bridge --boot hd
```

#### Existing Guest

1. Shutdown the KVM Guest
2. Run `virsh edit <guestname>`
3. Remove existing `interface` definitions (if you are replacing the existing interface, otherwise skip)
4. Add in a new `interface` definition

    **NOTE** - you may want to modify `devno` to force an specific internal interface to be created on the KVM guest (instead of eth0, eth1)

    ```
    <interface type='network'>
      <source network='host-bridge'/>
      <model type='virtio'/>
      <address type='ccw' cssid='0xfe' ssid='0x0' devno='0x0001'/>
    </interface>
    ```

5. Save Definition & boot up KVM guest

## Post-Setup

You should now be able to interact with the KVM guests' internal ethernet interfaces and assign normal network configuration. After configuring the network on the guest's internal interfaces, network connection should be established from guest to guest, host to guest and guest to host at this point. 