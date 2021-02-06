---
title: "libvirt, dnsmasq, NetworkManager, and systemd-resolved"
summary: ""
authors: ["thomas"]
tags: ["linux", "network", "systemd"]
categories: []
date: 2021-01-31
---
Modern Linux can have complicated DNS setup. However this is now necessary, gone are the days when you could just set one static name server in /etc/resolv.conf and walk away. This little piece just documents how I've got my systems setup.

I've got systemd-resolved enabled and running, and I have also got /etc/resolv.conf linked to /run/systemd/resolve/stub-resolv.conf, this puts systemd-resolved into it's "normal" mode. It then runs a local stub DNS server and listens on localhost. It also takes care to create a working resolv.conf file. So in essence local DNS queries go to systemd-resolved. Now meanwhile NetworkManager is also running, it detects that systemd-resolved is calling the shots, so any DNS information it gets it pushed to systemd-resolved. For example a WiFi connection with a name server of 192.168.1.1 and DNS search of lan get pushed to systemd-resolved and things generally work, eg a search of lan is added and resolved sends queries to this given name server. Whats more a VPN connection that also uses some other custom domain and DNS server can also be used transparently without polluting DNS names to the wrong servers, eg a VPN that is given a DNS server of 10.0.0.1 and a search of work will mean that foo.lan and foo.work can both be resolved on the system using two different name severs.

What I wanted is more DNS integration with libvirt. The issue is that libvirt also does some network related activates, it's got it's own view of the world and can go a create bridges, various iptables to allow NAT to work and it also spawns a dnsmasq process as a helper for the VM's and this acts as a DHCP and DNS server.  By default on Debian you get an external network with 192.168.122.0/24 address that NAT's to the Internet. While this appears in NetworkManager it's created and managed via libvirt, eg it does not have an entry in /etc/NetworkManager/system-connections/, and even if it had libvirt would ignore it.

So to allow the system to talk to libvirts dnsmaq process there has to be some sort of linkage. The way this is configured usually is for NetworkManager to use a ~ syntax for the dns search, thus is ~virt is set in the search then foo.virt is directed to the dns servers set in that connection. Now because libvirt takes care to create and destroy it's networks there needs to be some sort of hook to modify the network.

Fortunately there are such hooks: /etc/NetworkManager/dispatcher.d/ which includes /etc/NetworkManager/dispatcher.d/01-ifupdown which in turn hooks: /etc/network/if-{up,down}.d. The idea is for this hook to add the necessary DNS server and search path. However I have found that when I update the DNS settings after libvirt has created the bridge that they don't take effect until the connection is downed and up again, eg when running "sudo nmcli connection modify virbr0 ipv4.dns 192.168.122.1" and "sudo nmcli connection modify virbr0 ipv4.dns-search '~virt;virt'" those settings don't appear in nmcli output. I have also see that if the underlying connection with the ~ search syntax is down, they the name servers don't apply, which means that unless there is at least one VM running the per connection DNS server is not used.

One hack round this is to create a dummy interface with the right information on, which has the dns servres set when the connection is created which takes effect immidiatly. I have found that in order for a NetworkManager connection to have ipv4 dns settings it requires either auto or manual ipv4.method, to prevent the interface sitting in an eternal connecting state as there is no DHCP server just giving it the last address of the external subnet works. Thus after libvirt creates virbr0 when you run: "sudo nmcli connection add type dummy ifname virbr0-dns con-name virbr0-dns ipv4.method manual ipv4.addresses 192.168.122.254/32 ipv4.dns 192.168.122.1 ipv4.dns-search '~virt virt' ipv6.method disabled" the queries for foo.virt are directed to 192.168.122.1 and whats more virt is added to the serch list so foo should also resolve.

While I was configuring this to test I found the following usefull to restart resolved, turn up it's debug level, flush it's cache, follow it's log and start a shark session to see what queries are going where:
```sudo systemctl restart systemd-resolved; sudo resolvectl log-level debug; sudo resolvectl flush-caches; sudo journalctl -fu systemd-resolved; sudo tshark -n -i any -f "port 53"```

After I used resolvectl to test lookups with: "resolvectl query debian.virt"

So to begin with I created a quick dispatched script "/etc/NetworkManager/dispatcher.d/libvirt":
```
#!/bin/sh -e
if [ -z "$1" ]; then
    echo "$0: called with no interface" 1>&2
    exit 1;
fi

export IFACE="$1"
case "$2" in
up)
    virsh --connect qemu:///system net-list \
        | sed "1,2d" | head -n -1 | awk '{print $1}' \
        | while read VIRTNET; do
            NETBR=$(virsh --connect qemu:///system net-info $VIRTNET | awk '/Bridge:/{print $2}')
            if [ $IFACE = $NETBR ]; then
                IP=$(nmcli connection show $IFACE | awk '/ipv4.addresses/{print $2}' | sed 's^.1/24^.254/32^')
                DNS=$(nmcli connection show $IFACE | awk '/ipv4.addresses/{print $2}' | sed 's^/24^^') 

                nmcli connection add type dummy \
                    con-name ${IFACE}-dns ifname ${IFACE}-dns \
                    ipv4.method manual ipv4.addresses $IP \
                    ipv4.dns $DNS \
                    ipv4.dns-search "~${VIRTNET}.virt ${VIRTNET}.virt" \
                    ipv6.method disabled
            fi
    done
    ;;

down)
    virsh --connect qemu:///system net-list \
        | sed "1,2d" | head -n -1 | awk '{print $1}' \
        | while read VIRTNET; do
            NETBR=$(virsh --connect qemu:///system net-info $VIRTNET | awk '/Bridge:/{print $2}')
            if [ $IFACE = $NETBR ]; then
                nmcli con del ${IFACE}-dns
            fi
    done
    ;;

vpn-up|vpn-down|hostname|dhcp4-change|dhcp6-change)
    ;;

*)
    echo "$0: called with unknown action \`$2'" 1>&2
    exit 1
    ;;

esac
```

Which then dnsmasq needs to be told about these new domains, so edit the libvirt network with: "virsh --connect qemu:///system net-edit default" then under the <network> create a domain tag like: "<domain name='default.libvirt' localOnly='yes'/>"

Then hosts can be created with:
$ virsh --connect qemu:///system net-update default add ip-dhcp-host "<host mac='52:54:00:de:97:a8' name='debian' ip='192.168.122.120' />" --live --config
$ virsh --connect qemu:///system net-update default add dns-host "<host ip='192.168.122.120'><hostname>debian</hostname></host>" --live --config

Perhaps, the above could be populated with a combination of:
virsh --connect qemu:///system net-dhcp-leases default
virsh --connect qemu:///system list --all

Perhaps a systemd service that monitors: /var/lib/libvirt/dnsmasq/default.* and updates DNS acordingly.

Perhaps add host entry for dummy interface
