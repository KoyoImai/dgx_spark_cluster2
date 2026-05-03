## ステップ３：NAT（インターネット接続の共有）の設定
まずspark08（管理者node）でIPフォワーディングを有効化します。
IPフォワーディングとは、あるネットワークインターフェースから受け取ったパケットを別のインターフェースに転送する機能です。
以下のような構成です。
```
計算node（10.0.0.x）
    ↓ RJ45スイッチ
管理者node（10.0.0.8） ← クラスタ内部NIC
管理者node（192.168.111.x） ← 研究室LAN NIC
    ↓
研究室インターネット
```
計算nodeは研究室インターネットに直接つながっていないため、そのままではインターネットに出られません。
IPフォワーディングを有効にすることで、管理者nodeがルーターの役割を果たし、計算nodeのインターネット通信を中継できるようになります。

以下のコマンドを実行してください。
```
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
`etc/sysctl.conf`を確認します。
```
mprg@spark-3894:~/Desktop$ cat /etc/sysctl.conf 
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

###################################################################
# Functions previously found in netbase
#

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
#net.ipv4.conf.default.rp_filter=1
#net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
# See http://lwn.net/Articles/277146/
# Note: This may impact IPv6 TCP sessions too
#net.ipv4.tcp_syncookies=1

# Uncomment the next line to enable packet forwarding for IPv4
#net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
#net.ipv6.conf.all.forwarding=1


###################################################################
# Additional settings - these settings can improve the network
# security of the host and prevent against some network attacks
# including spoofing attacks and man in the middle attacks through
# redirection. Some network environments, however, require that these
# settings are disabled so review and enable them as needed.
#
# Do not accept ICMP redirects (prevent MITM attacks)
#net.ipv4.conf.all.accept_redirects = 0
#net.ipv4.conf.default.accept_redirects = 0
# _or_
# Accept ICMP redirects only for gateways listed in our default
# gateway list (enabled by default)
# net.ipv4.conf.all.secure_redirects = 1
#
# Do not send ICMP redirects (we are not a router)
#net.ipv4.conf.all.send_redirects = 0
#
# Log Martian Packets
#net.ipv4.conf.all.log_martians = 1
#

###################################################################
# Magic system request Key
# 0=disable, 1=enable all, >1 bitmask of sysrq functions
# See https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html
# for what other values do
#kernel.sysrq=438

net.ipv4.ip_forward=1
mprg@spark-3894:~/Desktop$ 
```
次にNATの設定を行います。
管理者nodeの研究室LAN側のインターフェース名を確認します。
以下のコマンドを実行してください。
```
mprg@spark-3894:~/Desktop$ ip a | grep "192.168.111"
    inet 192.168.111.42/24 brd 192.168.111.255 scope global dynamic noprefixroute enP7s7
    inet 192.168.111.133/24 brd 192.168.111.255 scope global dynamic noprefixroute wlP9s9
mprg@spark-3894:~/Desktop$ 
```
`enP7s7（有線）`と`wlP9s9（WiFi）`の両方が研究室LANに接続されています。
有線の`enP7s7`をインターネット側インターフェースとしてNATを設定します。
以下のコマンドを実行します。
```
sudo iptables -t nat -A POSTROUTING -o enP7s7 -j MASQUERADE
sudo iptables -A FORWARD -i enx6c6e0705ec11 -o enP7s7 -j ACCEPT
sudo iptables -A FORWARD -i enP7s7 -o enx6c6e0705ec11 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
この設定を再起動後も維持できるよう永続化します。
```
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```
途中でipv4とipv6の設定について聞かれます。ipv4は「はい」、ipv6は「いいえ」で勧めてください。
次にiptables-persistentの保存を実行します。
```
sudo netfilter-persistent save
```
結果は以下のようになります。
```
mprg@spark-3894:~/Desktop$ sudo netfilter-persistent save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables save
mprg@spark-3894:~/Desktop$ 
```
正常に保存されました。
次に計算node側でデフォルトゲートウェイを管理者node（10.0.0.8）に設定します。
これにより計算nodeのインターネット通信が管理者node経由で行われるようになります。
以下のコマンドを全ての計算用nodeで実行してください。
```
sudo nmcli con mod "有線接続 3" \
  ipv4.gateway "10.0.0.8" \
  ipv4.dns "8.8.8.8"

sudo nmcli con up "有線接続 3"
```
