# ステップ006：QSFPの4台接続（QSFPスイッチ）

QSFPケーブルでDGX Sparkを4台接続します．
接続にはQSFPスイッチを使用します．
構成は以下の通りです．
```
node15(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル1）
node16(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル2）
node17(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル3）
node18(enp1s0f0np0) ↔ QSFPスイッチ（ケーブル4）
```


## ステップ006.1：node15の設定
node15の設定を行います．
以下のコマンドを実行してください．
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.1/24
        - 10.0.10.15/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```

## ステップ006.2：node16の設定
node16の設定を行います．
以下のコマンドを実行してください．
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.2/24
        - 10.0.10.16/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```

## ステップ006.3：node17の設定
node17の設定を行います．
以下のコマンドを実行してください．
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.1/24
        - 10.0.10.17/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```

## ステップ006.4：node18の設定
node18の設定を行います．
以下のコマンドを実行してください．
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.2/24
        - 10.0.10.18/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```



## ステップ006.5：接続テスト
QSFPスイッチ経由で接続が可能かをテストします．
以下のコマンドをnode15で実行してpingが通るかを確認してください．
```
ping -c 3 10.0.10.16
ping -c 3 10.0.10.17
ping -c 3 10.0.10.18
```

## ステップ006.6：`/etc/hosts`への追記
全てのnodeで以下の内容を追記します．
```
sudo bash -c 'cat >> /etc/hosts << EOF

# QSFP switch network
10.0.10.15   node15-qsfp-sw
10.0.10.16   node16-qsfp-sw
10.0.10.17   node17-qsfp-sw
10.0.10.18   node18-qsfp-sw
EOF'
```


## ステップ006.7：リンク速度の設定と確認
node15で以下のコマンドを実行して，リンク速度を確認してください．
```
mprg@spark-fb97:~$ sudo ethtool enp1s0f0np0 | grep Speed
[sudo] mprg のパスワード: 
	Speed: 100000Mb/s
mprg@spark-fb97:~$ 
```
速度を確認すると`100Gbps=100000Mb/s`になっています．
スイッチ側でポートの速度を200Gbpsに手動設定し直す必要があります．

スイッチに接続して速度を設定し直します。
ここからは、DGX Spark8（Node8）とスイッチをLANケーブルで直接つないで、速度の設定をします。
まず、LANケーブルで接続したあとに、以下のコマンドを実行して一時的に接続したインターフェースに192.168.88.xのipアドレスを追加します。
```
sudo ip addr add 192.168.88.2/24 dev enx6c6e0705ec11
```
ipアドレスを追加したら、pingが通るか確認してみてください。
pingがとおれば問題ありません。
ブラウザで`http://192.168.88.1`を開いてください。
ログイン画面が出現するので、説明書どおりにユーザー名とパスワードを入力してログインしてください。

ログイン後、左上にある`≡`、`Advanced`、`Interface`の順にクリックしてください。
Interface画面が開くと、QSFPポートなどのリスト（qsfp56-1-1、qsfp56-1-2など）が表示されると思います。

`qsfp56-1-1`をクリックし、その後、`Ethernet`セクションを展開、`Auto Negotiation`をオフにして速度を`200G`に設定して`Ok`を押してください。
同様の手順で接続しているQSFPポート全てで設定してください。
（疲れたから後日詳細な設定を再度確認して残します。）

node15で以下のコマンドを実行してください。
```
ssh-copy-id mprg@10.0.10.16
ssh-copy-id mprg@10.0.10.17
ssh-copy-id mprg@10.0.10.18
```
