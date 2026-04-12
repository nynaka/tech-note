---
title: IP Masquerade (NAPT)
description: LinuxでのIP Masquerade (NAPT) の設定方法です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

IP Masquerade (NAPT)
===

## Kernel 設定

```bash title="一時的な設定"
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward > /dev/null
```

```bash title="永続設定"
echo "net.ipv4.ip_forward=1" \
    | sudo tee /etc/sysctl.d/ip_forward.conf > /dev/null
```


## IP Masquerade (NAPT)

### iptables のインストール

```bash title="Debian / Ubuntu 系"
sudo apt install -y iptables iptables-persistent
```

### 最小設定

wlp1s0 は無線 LAN インタフェースで、インターネット側のインタフェースです。  
有線 LAN 等の他のインタフェースから受信したトラフィックを wlp1s0 に中継する設定です。

```bash
# 既存設定の初期化
sudo iptables -F
sudo iptables -F -t nat
sudo iptables -X

# Deafult Rule
sudo iptables -P INPUT   ACCEPT
sudo iptables -P OUTPUT  ACCEPT
sudo iptables -P FORWARD ACCEPT

sudo iptables -t nat -A POSTROUTING -o wlp1s0 -j MASQUERADE
```

```bash title="iptables 設定の永続化"
sudo /etc/init.d/netfilter-persistent save
```

---

## LAN 側の設定

### NICの設定 (/etc/network/interface)

```diff
--- interfaces.origin   2025-11-29 16:05:19.639993475 +0900
+++ interfaces  2025-11-29 16:06:04.067991709 +0900
@@ -10,3 +10,7 @@
 # The primary network interface
 allow-hotplug enp2s0
 iface enp2s0 inet dhcp
+
+allow-hotplug enp3s0
+iface enp3s0 inet static
+    address 192.168.1.1/24
```

### DHCP サーバのインストール

```bash title="Debian / Ubuntu 系"
sudo apt install -y isc-dhcp-server
```

### DHCP サーバの設定

```diff title="/etc/dhcp/dhcpd.conf"
--- dhcpd.conf.origin   2025-11-29 16:12:55.603987434 +0900
+++ dhcpd.conf  2025-11-29 16:15:38.507980958 +0900
@@ -105,3 +105,15 @@
 #    range 10.0.29.10 10.0.29.230;
 #  }
 #}
+
+subnet 192.168.1.0 netmask 255.255.255.0 {
+    range 192.168.1.101 192.168.1.200;
+    option routers 192.168.1.1;
+    option domain-name-servers 8.8.8.8, 8.8.4.4;
+    #option rfc3442-classless-static-routes 24, 172,16,10, 192,168,1,10;
+
+    #host host1 {
+    #    hardware ethernet xx:xx:xx:xx:xx:xx;
+    #    fixed-address 192.168.1.xxx;
+    #}
+}
```

### DHCP サーバの自動起動設定

```bash
sudo systemctl start isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```
