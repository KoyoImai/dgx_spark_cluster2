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
これで管理者nodeのip固定が完了したと思います。
以下のようなコマンドを実行してipアドレスが固定されていることを確認してください。
```
mprg@spark-3894:~/Desktop$ ip a show enx6c6e0705ec11
11: enx6c6e0705ec11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 6c:6e:07:05:ec:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.8/24 brd 10.0.0.255 scope global noprefixroute enx6c6e0705ec11
       valid_lft forever preferred_lft forever
    inet6 fe80::545d:10c:f7f9:dc01/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-3894:~/Desktop$ 
```


### 計算用node15でipアドレスを固定
`nmcli con show`を実行し、現在のインターフェース名を確認します。
```
mprg@spark-fb97:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
MPRG        48bc8a9d-daf2-492b-b762-9a0c35693f52  wifi      wlP9s9  
有線接続 3  14f7abde-ac36-3ab9-b8a0-da0f36968bfa  ethernet  enP7s7  
lo          16095b3e-5b89-4962-8966-38e9fbf8963c  loopback  lo      
docker0     42bb5e29-3ae7-46c8-bb13-ddca370ded0e  bridge    docker0 
有線接続 1  492530d1-b50c-3801-b94b-d05a90b15cc5  ethernet  --      
有線接続 2  2f48d38e-8faf-322a-aa19-dc1c270dcf6c  ethernet  --      
有線接続 4  3f0a3f8f-2d8d-301b-8a02-6943394b0ffc  ethernet  --      
有線接続 5  c9b59ca5-6e71-32ac-a86e-d2955c621042  ethernet  --      
mprg@spark-fb97:~/Desktop$ 
```
`有線接続 3`が`enP7s7（内蔵RJ45）`に対応しています。
もし有線接続1~5のどれがenP7s7（RJ45）に対応しているかわからない場合、`nmcli con show "有線接続 x" | grep interface`を実行して確認してください。
これにIPを固定します。
以下を実行してください。
```
mprg@spark-fb97:~/Desktop$ sudo nmcli con mod "有線接続 3" \
  connection.interface-name enP7s7 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.15/24 \
  ipv4.gateway "" \
  ipv4.dns ""
[sudo] mprg のパスワード: 
mprg@spark-fb97:~/Desktop$ sudo nmcli con up "有線接続 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/15844)
mprg@spark-fb97:~/Desktop$ 
```
最後にipアドレスが固定されているかを確認します。
```
mprg@spark-fb97:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:97 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.15/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
    inet6 fe80::b487:3457:e80:a948/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-fb97:~/Desktop$ 
```


### 計算用node16でipアドレスを固定
`nmcli con show`を実行し、現在のインターフェース名を確認します。
```
mprg@spark-4440:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
MPRG        61537807-a36a-4fe4-a2e8-88dd4f0839f4  wifi      wlP9s9  
有線接続 3  c7c294e2-f5f4-366f-ab99-52cea54e6676  ethernet  enP7s7  
lo          d36ce37d-ea96-4433-8306-5ec39052c4b0  loopback  lo      
docker0     8548a3df-3aef-42cc-aef3-c2a00d3c05d7  bridge    docker0 
有線接続 1  af0d74cf-7256-32b2-a0ba-8f0741ebfc7a  ethernet  --      
有線接続 2  693c8ec1-f0a3-3b2c-95ce-d1efe41e8aac  ethernet  --      
有線接続 4  b5d089c9-24a0-3cd7-af20-3f601681c8c4  ethernet  --      
有線接続 5  c5342c81-18eb-3105-bdfc-ec53b8ae96f4  ethernet  --      
mprg@spark-4440:~/Desktop$
```
`有線接続 3`が`enP7s7（内蔵RJ45）`に対応しています。
もし有線接続1~5のどれがenP7s7（RJ45）に対応しているかわからない場合、`nmcli con show "有線接続 x" | grep interface`を実行して確認してください。
これにIPを固定します。
以下を実行してください。
```
mprg@spark-4440:~/Desktop$ sudo nmcli con mod "有線接続 3" \
  connection.interface-name enP7s7 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.16/24 \
  ipv4.gateway "" \
  ipv4.dns ""
[sudo] mprg のパスワード: 
mprg@spark-4440:~/Desktop$ sudo nmcli con up "有線接続 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/12799)
mprg@spark-4440:~/Desktop$ 
```
最後にipアドレスが固定されているかを確認します。
```
mprg@spark-4440:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:44:40 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.16/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
    inet6 fe80::d81e:3a8c:dd94:6885/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-4440:~/Desktop$ 
```

### 計算用node17でipアドレスを固定
計算用node17のipアドレス固定はnode15-16と同じ手順で実行します。
`nmcli con show`を実行し、現在のインターフェース名を確認します。
```
mprg@spark-755c:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
MPRG        e1a14021-f068-4ce8-b97b-7b85de4f9788  wifi      wlP9s9  
有線接続 3  c5a2e270-52e4-337e-9b61-97e50407c434  ethernet  enP7s7  
lo          bad9e95d-51f0-4271-a1d9-f54a0b57b330  loopback  lo      
docker0     f34cdb96-d3b6-4ecb-8d9a-55222494db48  bridge    docker0 
有線接続 1  9d19d00d-7fc7-3080-9f46-f31adc2abd1c  ethernet  --      
有線接続 2  43e7ad4b-eea4-3f55-b78a-e8b6a3ac124f  ethernet  --      
有線接続 4  e7a74228-4663-3d9a-8ef8-7534a8eccc75  ethernet  --      
有線接続 5  9f4db453-56ef-3162-b64d-45c51c870146  ethernet  --      
mprg@spark-755c:~/Desktop$ nmcli con show "有線接続 3" | grep interface
connection.interface-name:              enP7s7
mprg@spark-755c:~/Desktop$ 
```
`有線接続 3`が`enP7s7（内蔵RJ45）`に対応しています。
これにIPを固定します。
以下を実行してください。
```
mprg@spark-755c:~/Desktop$ sudo nmcli con mod "有線接続 3" \
  connection.interface-name enP7s7 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.17/24 \
  ipv4.gateway "" \
  ipv4.dns ""
[sudo] mprg のパスワード: 
mprg@spark-755c:~/Desktop$ sudo nmcli con up "有線接続 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/8466)
mprg@spark-755c:~/Desktop$ 
```
最後にipアドレスが固定されているかを確認します。
```
mprg@spark-755c:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2a:75:5c brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.17/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
    inet6 fe80::1f25:6f31:a658:22ba/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-755c:~/Desktop$ 
```


### 計算用node18でipアドレスを固定
`nmcli con show`を実行し、現在のインターフェース名を確認します。
```
mprg@spark-07a2:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
MPRG        8129ee4e-3f2d-4223-9f6d-a8489628fda4  wifi      wlP9s9  
lo          49178b70-9df5-4fca-aa8c-085234785b63  loopback  lo      
docker0     a93efcbe-fb68-4b41-b0f1-149e4c3dec5a  bridge    docker0 
有線接続 1  f0afafdc-8c7c-375b-a621-4a14dfad45d0  ethernet  --      
有線接続 2  d3fe4537-bd81-3054-a641-0802b9400eed  ethernet  --      
有線接続 3  26900b63-e53d-3ba7-8021-13fd44d88f66  ethernet  --      
有線接続 4  9f4d1b05-84ad-3b68-9ae7-787cc390d1da  ethernet  --      
有線接続 5  261629bd-f8e9-3cf9-a3cd-816290c3228a  ethernet  --      
mprg@spark-07a2:~/Desktop$ nmcli con show "有線接続 3" | grep interface
connection.interface-name:              enP7s7
mprg@spark-07a2:~/Desktop$ 
```
`有線接続 3`が`enP7s7（内蔵RJ45）`に対応しています。
これにIPを固定します。
以下を実行してください。
```
mprg@spark-07a2:~/Desktop$ sudo nmcli con mod "有線接続 3" \
  connection.interface-name enP7s7 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.18/24 \
  ipv4.gateway "" \
  ipv4.dns ""
[sudo] mprg のパスワード: 
mprg@spark-07a2:~/Desktop$ sudo nmcli con up "有線接続 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/8309)
mprg@spark-07a2:~/Desktop$ 
```
最後にipアドレスが固定されているかを確認します。
```
mprg@spark-07a2:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2e:07:a2 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.18/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
    inet6 fe80::962f:c37c:5c40:b4fc/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-07a2:~/Desktop$
```


### `/etc/hosts`の設定
まず管理者nodeで`/etc/hosts`を確認します。
```
mprg@spark-3894:~/Desktop$ cat /etc/hosts
127.0.0.1       localhost
127.0.0.1       spark-3894  

mprg@spark-3894:~/Desktop$ 
```
node間でnode名が解決できるように`/etc/hosts`を設定します。
以下のコマンドを実行してください。
```
sudo bash -c 'cat >> /etc/hosts << EOF

# DGX Spark cluster
10.0.0.8    node8
10.0.0.15   node15
10.0.0.16   node16
10.0.0.17   node17
10.0.0.18   node18
EOF'
```
一応`ssh mprg@nodexx`でssh接続できるかを確かめてください。

### ssh鍵の生成と共有
ssh鍵の生成と共有を行います。
これによって、node間でパスワードなしでssh接続ができるようになります。
管理者nodeで以下を実行してください。
```
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```
その後以下を実行してください。
```
ssh-copy-id -i ~/.ssh/id_ed25519.pub mprg@node15
ssh-copy-id -i ~/.ssh/id_ed25519.pub mprg@node16
ssh-copy-id -i ~/.ssh/id_ed25519.pub mprg@node17
ssh-copy-id -i ~/.ssh/id_ed25519.pub mprg@node18
```
これで管理者nodeから計算用nodeへのssh接続はパスワードなしで可能になります。
