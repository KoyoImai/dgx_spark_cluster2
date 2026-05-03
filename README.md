# DGX Spark Cluster

## 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QSFPケーブル2本 \
・LANケーブル6本 \
・[USB-Cハブ](https://www.ankerjapan.com/products/a8352?srsltid=AfmBOoq95ZKB998T5GecohoCODQpk4HWPhwSNI8mhbB-wakpWkvt89U1)（管理者nodeのRJ45増設用）

## 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \
・管理者nodeのみ研究室インターネットに接続しユーザーがログイン可能

## ステップ1：ipアドレスの固定
### 管理者nodeでipアドレスを固定
まず、DGX Sparkは内蔵RJ45が1つしかないため、USB-Cハブを使って増設します。
現在の接続名を確認します。確認には`nmcli con show`を実行してください。
```
mprg@spark-3894:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
有線接続 3  79e325db-0583-39c2-8b2f-dec17007e14b  ethernet  enP7s7  
MPRG        185c28a2-7880-4798-afcc-552a08e6f618  wifi      wlP9s9  
lo          013843c6-cbba-4cd0-ba26-76441a926dab  loopback  lo      
docker0     22db8f7b-81c5-42c1-b726-5979db9461e3  bridge    docker0 
有線接続 1  a8df8c28-1e9f-3804-8483-d04d679c138a  ethernet  --      
mprg@spark-3894:~/Desktop$
```
本来ならここで`有線接続1`~`有線接続6`が出てくるはずですが、今回は出てきませんでした。
なので、`ip a`を実行し、USB-Cハブが刺さっているかを確認します。
```
mprg@spark-3894:~/Desktop$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:38:94 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 192.168.111.42/24 brd 192.168.111.255 scope global dynamic noprefixroute enP7s7
       valid_lft 257674sec preferred_lft 257674sec
    inet6 fe80::bcd6:beed:5578:cd9a/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
7: wlP9s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 58:02:05:f5:cc:fe brd ff:ff:ff:ff:ff:ff
    altname wlP9p1s0
    inet 192.168.111.133/24 brd 192.168.111.255 scope global dynamic noprefixroute wlP9s9
       valid_lft 84912sec preferred_lft 84912sec
    inet6 fe80::5e3d:9b51:c2dd:c0f/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether a6:f2:10:1f:b3:24 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
11: enx6c6e0705ec11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 6c:6e:07:05:ec:11 brd ff:ff:ff:ff:ff:ff
mprg@spark-3894:~/Desktop$
```
`enx6c6e0705ec11`がUSB-Cハブ経由のRJ45インターフェースです。
`nmcli con show`で対応する接続名が出てないので、ネットワークを新規に接続します。
以下を実行してください。
```
sudo nmcli con add \
  type ethernet \
  con-name "cluster-internal" \
  ifname enx6c6e0705ec11 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.8/24 \
  ipv4.gateway "" \
  ipv4.dns ""
```
上記のコマンドを実行する次のような出力が出ると思います。
```
接続 'cluster-internal' (2257ccd1-3a1c-42cb-92c7-21190bd84ef0) が正常に追加されました。
```
接続の作成が完了したので、この接続を有効にします。
以下のようにコマンドを実行してください。
```
mprg@spark-3894:~/Desktop$ sudo nmcli con up "cluster-internal"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/7793)
mprg@spark-3894:~/Desktop$
```
