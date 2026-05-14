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

