# Wireguard vps setup

This guide sets up a WireGuard tunnel between a VPS (with public IPv4) and a homelab (IPv6-only) in order to:

- Route outbound traffic from the homelab through the VPS's IPv4:
    - Forward game server ports (e.g. Minecraft, Valheim) via static ipv4 to services in your homelab
    - Enable LXC containers in your homelab to use a static IP and reach the internet via the VPN tunnel

---

## Setup vps host
- allow Linux kernel port forwarding sudo vim /etc/sysctl.conf:
- Uncomment the next line to enable packet forwarding for IPv4
- #net.ipv4.ip_forward=1
- install and configure:

```
apt update && apt upgrade -y
sudo apt-get install wireguard
sudo -i
cd /etc/wireguard
umask 077
wg genkey | tee server.key | wg pubkey > server.pub
```
- sudo vim /etc/wireguard/wg0.conf add:
```
[Interface]
Address = 10.100.0.1/24
ListenPort = {port}
```
- `echo "PrivateKey = $(cat server.key)" >> /etc/wireguard/wg0.conf`
```
PostUp = iptables -A FORWARD -i eth0 -o %i -j ACCEPT
PostUp = iptables -A FORWARD -i %i -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# valheim port-forward
PostUp = iptables -t nat -A PREROUTING -i eth0 -p udp --dport 2456:2457 -j DNAT --to-destination 10.100.0.2:2456-2457
PostUp = iptables -A FORWARD -p udp -d 10.100.0.2 --dport 2456:2457 -j ACCEPT

PostDown = iptables -D FORWARD -i eth0 -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
# valheim port-forward
PostDown = iptables -t nat -D PREROUTING -i eth0 -p udp --dport 2456:2457 -j DNAT --to-destination 10.100.0.2:2456-2457
PostDown = iptables -D FORWARD -p udp -d 10.100.0.2 --dport 2456:2457 -j ACCEPT

[Peer]
```
- start wireguard on boot: `sudo systemctl start wg-quick@wg0 && sudo systemctl enable wg-quick@wg0`
- add firewall rule: inbound udp {port}
- configure clients: 

## setup homeserver client
- create deb wireguard client lxc

```
apt update && apt upgrade -y
sudo apt-get install wireguard
sudo -i
cd /etc/wireguard
umask 077
wg genkey | tee homelabe-client.key | wg pubkey > homelab-client.pub
```

- allow Linux kernel port forwarding sudo vim /etc/sysctl.conf:
    - Uncomment the next line to enable packet forwarding for IPv4
    - #net.ipv4.ip_forward=1
    - sysctl -p

- on wireguard server add peer vim /etc/wireguard/wg0.conf:
```
[Peer]
PublicKey = client-pub-key
AllowedIPs = 10.100.0.2/32
```
- reload wireguard:
```
systemctl reload wg-quick@wg0
wg show
```

### setup client config in deb wireguard lxc: 
- `apt install iptables`
- `vim client.conf`
```
[Interface]
Address = 10.100.0.2/32
ListenPort = 47112
DNS = 8.8.8.8
PrivateKey = {private-client-key} 

PostUp = iptables -t nat -A POSTROUTING -o %i -j MASQUERADE
PostUp = iptables -t nat -A PREROUTING -i %i -p udp --dport 2456:2458 -j DNAT --to-destination 192.168.0.223:2456-2458
PostUp = iptables -A FORWARD -i %i -o ens18 -p udp --dport 2456:2458 -d 192.168.0.223 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o ens18 -p udp --dport 2456:2458 -d 192.168.0.223 -j MASQUERADE

PostDown = iptables -t nat -D POSTROUTING -o %i -j MASQUERADE
PostDown = iptables -t nat -D PREROUTING -i %i -p udp --dport 2456:2458 -j DNAT --to-destination 192.168.0.223:2456-2458
PostDown = iptables -D FORWARD -i %i -o ens18 -p udp --dport 2456:2458 -d 192.168.0.223 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ens18 -p udp --dport 2456:2458 -d 192.168.0.223 -j MASQUERADE


[Peer]
PublicKey = {servers-public-key} 
Endpoint = {serers-ip}:47112
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
~                        
```

- start wireguard on boot: `systemctl start wg-quick@client && systemctl enable wg-quick@client`
- reboot `wg-quick down client && wg-quick up client`
