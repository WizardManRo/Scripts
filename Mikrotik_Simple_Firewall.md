# MikroTik Firewall Configuration

## Description

Firewall rules including BOGON address lists, DDoS detection, input/forward filtering, NAT, and service port hardening.

## Configuration

```routeros
/ip firewall address-list
add address=192.168.0.0/16 list=BOGON
add address=10.0.0.0/8 list=BOGON
add address=172.16.0.0/12 list=BOGON
add address=127.0.0.0/8 list=BOGON
add address=0.0.0.0/8 list=BOGON
add address=169.254.0.0/16 list=BOGON
add list=ddos-attackers
add list=ddos-targets

/ip firewall connection tracking
set udp-timeout=10s

/ip firewall filter
add action=return chain=detect-ddos dst-limit=32,32,src-and-dst-addresses/10s
add action=add-dst-to-address-list address-list=ddos-targets address-list-timeout=10m chain=detect-ddos
add action=add-src-to-address-list address-list=ddos-attackers address-list-timeout=10m chain=detect-ddos
add action=return chain=detect-ddos dst-limit=32,32,src-and-dst-addresses/10s protocol=tcp tcp-flags=syn,ack
add action=accept chain=input comment="Accept ICMP" protocol=icmp
add action=accept chain=input comment="Accept SSH" dst-port=5225 protocol=tcp
add action=accept chain=input comment="Accept WinBox" dst-port=5291 protocol=tcp
add action=accept chain=input comment="Accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="Drop public DNS requests (tcp)" dst-port=53 in-interface-list=WAN protocol=tcp
add action=drop chain=input comment="Drop public DNS requests (udp)" dst-port=53 in-interface-list=WAN protocol=udp
add action=drop chain=input comment="Drop all INPUT" in-interface-list=WAN
add action=drop chain=input comment="Drop invalid" connection-state=invalid
add action=accept chain=forward comment="Accept established,related, untracked" connection-state=established,related,untracked
add action=fasttrack-connection chain=forward comment=fasttrack connection-state=established,related
add action=drop chain=forward comment="Drop all from WAN not DSTNATed" connection-nat-state=!dstnat connection-state=new in-interface-list=WAN
add action=drop chain=forward comment="Drop invalid" connection-state=invalid

/ip firewall nat
add action=dst-nat chain=dstnat comment="PortFw Antena MikroTik" dst-port=5392 protocol=tcp to-addresses=192.168.111.2 to-ports=5291
add action=masquerade chain=srcnat comment=Masquerade ipsec-policy=out,none out-interface-list=WAN src-address=192.168.111.0/25

/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set h323 disabled=yes
set sip disabled=yes
set pptp disabled=yes
```
