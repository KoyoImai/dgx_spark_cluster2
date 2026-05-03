## ステップ５：QSFPの4台接続（2ペア＋クロス接続）
QSFPケーブルでDGX Sparkを4台接続します。
接続構成は「2ペア＋クロス接続」で、以下の構成にします。
```
【現在】
node15(enp1s0f0np0) ↔ node16(enp1s0f0np0)（ケーブル1・既存）
node17(enp1s0f0np0) ↔ node18(enp1s0f0np0)（ケーブル2・既存）

【追加】
node15(enp1s0f1np1) ↔ node17(enp1s0f1np1)（ケーブル3・新規）
node16(enp1s0f1np1) ↔ node18(enp1s0f1np1)（ケーブル4・新規）
```
上記の通りにDGX SparkをQSFPケーブルで接続します。

### node15でip固定
QSFPインターフェースの状態を確認します。
以下のコマンドを実行してください。
```
mprg@spark-fb97:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Up)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Up)
mprg@spark-fb97:~$ 
```
`enP2p1s0f1np1 (Up)`となっているため、このインターフェースを使用します。
以下のコマンドをnode15で実行してください。
```
sudo tee /etc/netplan/41-cx7-p2.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 10.0.3.1/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/41-cx7-p2.yaml
sudo netplan apply
```
ipアドレスが正しく設定されているか確認します。
`ip a show enp1s0f1np1`を実行してください。
```
mprg@spark-fb97:~$ ip a show enp1s0f1np1
4: enp1s0f1np1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:99 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.1/24 brd 10.0.3.255 scope global noprefixroute enp1s0f1np1
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2f:fb99/64 scope link 
       valid_lft forever preferred_lft forever
mprg@spark-fb97:~$
```
上記の結果から`enp1s0f1np1`インターフェースのipが`10.0.3.1`に設定されていることが確認できます。



