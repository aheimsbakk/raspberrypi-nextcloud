# Configure dnsmasq for OTG autoconfigure

This will configure the Raspberry Zero to give your PC an automatic IP. It will also set the PCs default gateway to the Zeros IP.

With this configuration all traffic will be routed to the Zero. 

Not useful now, but maybe later.


## Set static IP on USB interface 

```
sudo vi /etc/network/interfaces.d/usb0
```

```
auto usb0
iface usb0 inet static
    address 172.31.7.1
    netmask 255.255.255.0
```

### Set static MAC address

```
sudo vi /etc/modprobe.d/g_ether.config
```

```
options g_ether host_addr=26:34:d5:8e:55:5a
```


## Configure DNSMasq

```
sudo vi /etc/dnsmasq.d/usb0.config
```

```
domain=bob.org
dhcp-range=172.31.7.100,172.31.7.200,255.255.255.0,1h
interface=usb0
dhcp-host=26:34:d5:8e:55:5a,172.31.7.1

# Forward all traffic to PI Zero
dhcp-option=121,0.0.0.0/0,172.31.7.1
dhcp-option=249,0.0.0.0/0,172.31.7.1
dhcp-option=vendor:MSFT,2,1i
```

### Start DNSMasq at boot

```
sudo systemctl enable dnsmasq
```

## Forward traffic and secure 

```bash
# Forward IP traffic
cat <<EOF | sudo tee /etc/sysctl.d/50-forward.conf
net.ipv4.ip_forward = 1
net.ipv6.ip_forward = 1
EOF

# Reload changes
sudo sysctl --system

# Enable IPv4 firewall
sudo iptables -A POSTROUTING -t nat -o wlan0 -j MASQUERADE
sudo iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT ! -i wlan0 -m state --state NEW  -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 22 -m limit --limit 6/min -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -A FORWARD -i wlan0 -o wlan0 -j REJECT

# Enable IPv6 firewall
sudo ip6tables -A POSTROUTING -t nat -o wlan0 -j MASQUERADE
sudo ip6tables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo ip6tables -A INPUT ! -i wlan0 -m state --state NEW  -j ACCEPT
sudo ip6tables -A INPUT -p tcp -m tcp --dport 22 -m limit --limit 6/min -j ACCEPT
sudo ip6tables -P INPUT DROP
sudo ip6tables -A FORWARD -i wlan0 -o wlan0 -j REJECT

# Save current netfilter rules, and reload on boot
sudo DEBIAN_FRONTEND=noninteractive apt-get -yq install iptables-persistent
sudo netfilter-persistent save
```
