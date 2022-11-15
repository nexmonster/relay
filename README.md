# Relay :satellite:

If you've ever wanted to receive CSI packets on a device other than the nexmon_csi device, this is how you do it:

```bash
sudo nft add table ip nexmoncsi
sudo nft 'add chain ip nexmoncsi INPUT { type filter hook input priority -150; policy accept; }'

sudo nft add rule ip nexmoncsi INPUT iifname "wlan0" ip protocol udp ip saddr 10.10.10.10 ip daddr 255.255.255.255 udp sport 5500 udp dport 5500 counter dup to IP-WHERE-YOU-RECEIVE
```

Remember to replace `wlan0` with whichever interface your device gets csi packets from.

Run this on your nexmon_csi device (rpi, asus rt-ac86, nexus devices), and replace the `IP-WHERE-YOU-RECEIVE` with the ip of the device where you'd like to receive CSI packets. Packets are duplicated instead of being forwarded, so you can capture them on both the devices.

It's also possible to send the packets to multiple devices by running the last line multiple times. You can also send them to the broadcast ip of your network and receive them on all the devices on your network.

#### Persistence

Also, these rules disappear after reboot, so if you'd like to make them permanent, add the rules to your `/etc/rc.local` at the end, before the `exit 0` line.

#### Recieving

To listen to the packets on your devices, use the usual tcpdump command.

```
sudo tcpdump -i eth0 dst port 5500 -vv
```

Replace `eth0` with whichever interface you can reach the csi device. And that's it!


#### Legacy
If nftables aren't supported on your devices, you can also use `iptables`. Iptables are deprecated and are no longer included in some distros by default, so prefer nftables where you can.

```bash
iptables -t mangle -A INPUT -i wlan0 -p udp -s 10.10.10.10 -d 255.255.255.255 --dport 5500 --sport 5500 -j TEE --gateway 10.20.30.207 IP-desktop
```

#### Advanced usage

The default destination address of packets is `255.255.255.255` (broadcast), which makes some routers not forward packets. You can mangle the IP and UDP headers to change these:

```bash

nft add table ip nexmoncsi

nft 'add chain ip nexmoncsi input { type filter hook input priority -150; policy accept; }'

nft add rule ip nexmoncsi input iifname "wlan0" ip protocol udp ip saddr 10.10.10.10 ip daddr 255.255.255.255 udp sport 5500 udp dport 5500 counter mark set 900 dup to IP-WHERE-YOU-RECEIVE device "eth0"

nft 'add chain ip nexmoncsi output { type filter hook output priority 150; policy accept; }'

nft add rule ip nexmoncsi output oifname "eth0" meta mark 900 counter ip saddr set 10.10.10.1 udp sport set 5500 # many mangling opportunities here

# ip saddr set 10.10.10.1 ; sets source address to 10.10.10.1
# ip daddr set 10.20.30.1 ; sets destination address to 10.20.30.1
# udp sport set 5400      ; sets UDP source port to 5400
# udp dport set 5600      ; sets UDP destination port to 5600

# Setting daddr here won't send it to a different device. You have to set the ip where you'd like to receive in the 3rd line.

```

#### More Resources

- https://serverfault.com/questions/1115546/forward-udp-broadcasts-to-another-ip/1115550#1115550
- https://github.com/nomeata/udp-broadcast-relay
- https://github.com/udp-redux/udp-broadcast-relay-redux
- https://github.com/udp-redux/udp-broadcast-relay-redux/issues/12
- https://github.com/udp-redux/udp-broadcast-relay-redux/issues/13


