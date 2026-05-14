## ステップ004：QSFPの2台接続

**[参考1:DGX Sparkの2台接続](https://build.nvidia.com/spark/connect-two-sparks/stacked-sparks)** \
**[参考２:DGX Sparkの2台接続](https://dev.classmethod.jp/articles/dgx-spark-two-node-clustering/)**

QSFPケーブルで2台のDGX Sparkを接続してください。
ここでは、node15とnode16を接続して作業を勧めていきます。

### node15でip固定
QSFPケーブルで2台のDGX Sparkを接続したら、ネットワークインターフェースの設定を行います。
NVIDIA公式に従って、自動IP割り当てで設定します。
まず、2台が接続できているかを、以下のコマンドで確認してください。
```
mprg@spark-fb97:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
mprg@spark-fb97:~$ 
```
enp1s0f0np0 (Up)がUpとなっているため、このインターフェースを使用します。
以下のコマンドを、node15で実行してください。
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.1/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```
ipアドレスが正しく設定されているかを確認します。
`ip a show enp1s0f0np0`を実行して設定を確認します。
```
mprg@spark-fb97:~$ ip a show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:98 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.1/24 brd 10.0.1.255 scope global noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2f:fb98/64 scope link 
       valid_lft forever preferred_lft forever
mprg@spark-fb97:~$ 
```
上記の結果から`enp1s0f0np0`インターフェースのipが`10.0.1.1`に設定されていることが確認できます。

### node16でip固定
続いてnode16でもip固定を行います。
まず、node15と同様にQSFPインターフェースの状態を確認します
```
mprg@spark-4440:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
mprg@spark-4440:~$ 
```
enp1s0f0np0 (Up)がUpとなっているため、このインターフェースを使用します。
以下のコマンドをnode16で実行してください。
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.2/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```
ipアドレスが正しく設定されているかを確認します。
`ip a show enp1s0f0np0`を実行して設定を確認します。
```
mprg@spark-4440:~$ ip a show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:44:41 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.2/24 brd 10.0.1.255 scope global noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2f:4441/64 scope link 
       valid_lft forever preferred_lft forever
mprg@spark-4440:~$ 
```
上記の結果から`enp1s0f0np0`インターフェースのipが`10.0.1.2`に設定されていることが確認できます。

### node17でip固定
続いてnode17でもip固定を行います。
まず、node15と同様にQSFPインターフェースの状態を確認します
```
mprg@spark-755c:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
mprg@spark-755c:~$ 
```
enp1s0f0np0 (Up)がUpとなっているため、このインターフェースを使用します。
以下のコマンドをnode17で実行してください。
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.1/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```
ipアドレスが正しく設定されているかを確認します。
`ip a show enp1s0f0np0`を実行して設定を確認します。
```
mprg@spark-755c:~$ ip a show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2a:75:5d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.1/24 brd 10.0.2.255 scope global noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2a:755d/64 scope link 
       valid_lft forever preferred_lft forever
mprg@spark-755c:~$ 
```

### node18でip固定
続いてnode18でもip固定を行います。
まず、node15と同様にQSFPインターフェースの状態を確認します
```
mprg@spark-07a2:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
mprg@spark-07a2:~$ 
```
enp1s0f0np0 (Up)がUpとなっているため、このインターフェースを使用します。
以下のコマンドをnode18で実行してください。
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.2/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```
ipアドレスが正しく設定されているかを確認します。
`ip a show enp1s0f0np0`を実行して設定を確認します。
```
mprg@spark-07a2:~$ ip a show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2e:07:a3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.2/24 brd 10.0.2.255 scope global noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2e:7a3/64 scope link 
       valid_lft forever preferred_lft forever
mprg@spark-07a2:~$ 
```





