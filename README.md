# Relay :satellite:

It's possible to forward the CSI packets from your Nexmon device to another device (eg: your laptop) and collect / analyse them there. Example below shows how you would do it on a RaspberryPi:

```bash
# Run as root. nft is a built-in command.

nft add table ip nexmon

nft 'add chain ip nexmon input  { type filter hook input  priority -150; policy accept; }'
nft 'add chain ip nexmon output { type filter hook output priority  150; policy accept; }'

nft add rule ip nexmon input  iifname "wlan0" ip protocol udp ip saddr 10.10.10.10 ip daddr 255.255.255.255 udp sport 5500 udp dport 5500 counter mark set 900 dup to IP-OF-LAPTOP device "eth0"
nft add rule ip nexmon output oifname "eth0"  meta mark 900 counter ip saddr set IP-OF-NEXMON-DEVICE ip daddr set IP-OF-LAPTOP
```
Note that the above example is for a Raspberry Pi. On a Raspberry Pi running Nexmon, CSI packets arrive from `wlan0`, and the laptop is reachable via `eth0` to the Pi. If you're running this on a different device, you will need to replace `wlan0` and `eth0` with the names of your interfaces.


To record them on your laptop, you can use tcpdump:

```bash
# eno1 is the Ethernet interface on my laptop.
# Replace it with the name of the interface you can reach the Pi from. (eth0, wlan0, etc)

sudo tcpdump -i eno1 dst port 5500

# If you have multiple devices sending you CSI packets,
# you can filter them by IP like this

sudo tcpdump -i eno1 dst port 5500 and src IP-OF-DEVICE-YOU-WANT-CSI-FROM
```

## Advanced Usage

### Persistence

The forwarding rules we set will be cleared on reboot. To make them survive reboots, you can use the `/etc/rc.local` file.

First, create a file called `setup.sh`, and put the following code in it:

```bash
#!/bin/sh

[[ $UID != 0 ]] && echo "Please run as root. Exiting." && exit -1

# Start Nexmon_CSI
ifconfig wlan0 up

nexutil -Iwlan0 -s500 -b -l34 -vKuABEQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==

iw dev wlan0 interface add mon0 type monitor
ip link set mon0 up

# Start CSI Forwarding with NF Tables
nft add table ip nexmon

nft 'add chain ip nexmon input { type filter hook input priority -150; policy accept; }'
nft 'add chain ip nexmon output { type filter hook output priority 150; policy accept; }'

nft add rule ip nexmon input iifname "wlan0" ip protocol udp ip saddr 10.10.10.10 ip daddr 255.255.255.255 udp sport 5500 udp dport 5500 counter mark set 900 dup to IP-OF-LAPTOP device "eth0"

nft add rule ip nexmon output oifname "eth0" meta mark 900 counter ip saddr set IP-OF-NEXMON-DEVICE ip daddr set IP-OF-LAPTOP

echo "Finished setting up Nexmon and CSI forwarding"
```

Make sure you change the interface names, IPs, and the makecsiparams string.

Now if you run `setup.sh` (with `sudo bash setup.sh`), your device will start collecting CSI and will forward it to your laptop.

Test whether the script works by running it once, and add it to your `/etc/rc.local` file, before the `exit 0`.
```bash
# /etc/rc.local

......
......
......

# Add at the end, before the exit 0
bash /home/pi/setup.sh

exit 0
```

Your device will start collecting CSI as soon as it boots and will forward it to your laptop now.

### Legacy - IP Tables

If nftables aren't supported on your devices, you can also use `iptables`. Iptables are deprecated and are no longer included in some distros by default, so prefer nftables where you can.

```bash
iptables -t mangle -A INPUT -i wlan0 -p udp -s 10.10.10.10 -d 255.255.255.255 --dport 5500 --sport 5500 -j TEE --gateway IP-OF-LAPTOP
```

### Mangling

By default, CSI packets have a source IP of `10.10.10.10`, source port 5500, destination IP `255.255.255.255`, and destination port 5500.

IP `255.255.255.255` is used for broadcasts, and some routers don't forward them.
Which is why we changed the source and destination ips of the packets in the final nft command 
(`nft add rule ip nexmon output oifname "eth0"  meta mark 900 counter ip saddr set IP-OF-NEXMON-DEVICE ip daddr set IP-OF-LAPTOP`).

It's possible to change the other properties of the packets as well.
Add any of the following to the final line to change the corresponding property:

```bash
ip saddr set 10.10.10.1 # sets source address to 10.10.10.1
ip daddr set 10.20.30.1 # sets destination address to 10.20.30.1
udp sport set 5400      # sets UDP source port to 5400
udp dport set 5600      # sets UDP destination port to 5600
```

**Example**:

To make the packets have source IP `10.20.30.40`, and destination port `5600`,
```bash
nft add rule ip nexmon output oifname "eth0"  meta mark 900 counter ip saddr set 10.20.30.40 udp dport set 5600
```

Now to receive the packets on your laptop, you have to use `dst port 5600` with tcpdump.

Note that even though you change the destination ip with `ip dadder set ...`, the packet will still arrive
at the IP you set with `dup to IP-OF-LAPTOP` in the third command. This is probably because routing happens by Mac ID
in the local network.

To see your current ruleset, run `nft list ruleset`. To delete all rules and start over, run `nft flush ruleset`.
Chech the netfilter documentation for more details and usage.

## Issues?

Create a new Issue or Discussion in this repo and tag me ([@zeroby0](https://github.com/zeroby0/)).

Please include as much detail you can include in your issue, and how you've tried to solve / debug the issue. Please
include steps to reproduce your problem as well. Most bugs solve themselves or become very trivial to solve when you figure out how to reproduce them.    

## More Resources

- https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
- https://netfilter.org/projects/nftables/
- https://serverfault.com/questions/1115546/forward-udp-broadcasts-to-another-ip/1115550#1115550
- https://github.com/nomeata/udp-broadcast-relay
- https://github.com/udp-redux/udp-broadcast-relay-redux
- https://github.com/udp-redux/udp-broadcast-relay-redux/issues/12
- https://github.com/udp-redux/udp-broadcast-relay-redux/issues/13


