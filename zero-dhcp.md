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
dhcp-option=3,172.31.7.1
```

### Start DNSMasq at boot

```
sudo systemctl enable dnsmasq
```
