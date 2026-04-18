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

### ufw のインストール

```bash title="Debian / Ubuntu 系"
sudo apt install -y ufw
```

### フォワーディングポリシーの変更

ufw のデフォルトではパケット転送が拒否されているため、`ACCEPT` に変更します。

```diff title="/etc/default/ufw"
-DEFAULT_FORWARD_POLICY="DROP"
+DEFAULT_FORWARD_POLICY="ACCEPT"
```

### NAT ルールの追加

`/etc/ufw/before.rules` の先頭（`*filter` セクションより前）に NAT テーブルのルールを追加します。  
`wlp1s0` はインターネット側（WAN 側）のインタフェース名です。環境に合わせて変更してください。

```diff title="/etc/ufw/before.rules"
+# NAT table rules
+*nat
+:POSTROUTING ACCEPT [0:0]
+
+# Masquerade traffic going out via wlp1s0 (WAN interface)
+-A POSTROUTING -o wlp1s0 -j MASQUERADE
+
+COMMIT
+
 # Don't delete these required lines, otherwise there will be errors
 *filter
```

### LAN 側からのトラフィックを許可

```bash
# LAN 側インタフェース (enp3s0) からのトラフィックを許可
sudo ufw allow in on enp3s0 from 192.168.1.0/24
```

### SSH の通信許可

```bash
# SSH
sudo ufw allow in on enp3s0 from 192.168.1.0/24 to any port 22 proto tcp
```

### mDNS (Avahi) (オプション)

mDNS はマルチキャスト UDP パケット（宛先 `224.0.0.251:5353`）を使用します。  
ufw の `allow` ルールはユニキャストにのみ適用されるため、`/etc/ufw/before.rules` に直接ルールを追記します。

```diff title="/etc/ufw/before.rules"
 # allow all on loopback
 -A ufw-before-input -i lo -j ACCEPT
 -A ufw-before-output -o lo -j ACCEPT
+
+# allow mDNS (multicast DNS) on LAN interface
+-A ufw-before-input -i enp3s0 -p udp -d 224.0.0.251 --dport 5353 -j ACCEPT
```

### Samba (オプション)

```bash
# Samba (NetBIOS + SMB)
sudo ufw allow in on enp3s0 from 192.168.1.0/24 to any app Samba
```

### ufw の有効化

```bash
sudo ufw enable
sudo ufw status verbose
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
