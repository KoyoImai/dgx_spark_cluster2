# ステップ10
NCCL testによる通信速度のテストを実行する．

## 前置き
ステップ6〜ステップ9までで，QSFPスイッチによる4台接続のテストを行ってきたが，NCCL testによる通信速度が著しく遅いという現象が発生していた．
現在でも原因は不明だが，QSFPケーブルの差し替え（スイッチ経由からQSFPケーブル直接接続の切り替え）によって速度が低下するということが判明した．
DGX Spark自体を再起動すれば速度は正常に戻るが，その後QSFPケーブルの差し替えを行うことで速度が低下する．
根本的な原因解明と対策はまだできていないが，一旦この問題は傍に置いておいて，NCCL testによる速度調査を行う．

## ステップ10.1：IPアドレスの永続化
ステップ9では各nodeのIPアドレスを手動で固定していましたが，再起動のたびにリセットされるので永続化をします．
各計算nodeで以下のコマンドを実行してください．
```
# node15
sudo tee /etc/netplan/41-cx7-sw.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.15/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/41-cx7-sw.yaml
sudo netplan apply

# 確認
ip a show enp1s0f1np1 | grep inet
```
```
# node16
sudo tee /etc/netplan/41-cx7-sw.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.16/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/41-cx7-sw.yaml
sudo netplan apply
ip a show enp1s0f1np1 | grep inet
```
```
# node17
sudo tee /etc/netplan/41-cx7-sw.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.17/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/41-cx7-sw.yaml
sudo netplan apply
ip a show enp1s0f1np1 | grep inet
```
```
# node18
sudo tee /etc/netplan/41-cx7-sw.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.18/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/41-cx7-sw.yaml
sudo netplan apply
ip a show enp1s0f1np1 | grep inet
```

## ステップ10.2：FECモードの固定
QSFPケーブルの差し替えで速度が低下する原因として，FEC（Forward Error Correction）モードの自動ネゴシエーションを考えていますが，本当にこれが原因かはわからないので，一旦後回しにします．


## ステップ10.3：`/etc/hosts`とSSH鍵の整備
全ての計算nodeで以下のコマンドを実行して，`/etc/hosts`にアドレス登録をしてください．
```
sudo bash -c 'cat >> /etc/hosts << EOF

# QSFP switch network (192.168.100.x)
192.168.100.15   node15-sw
192.168.100.16   node16-sw
192.168.100.17   node17-sw
192.168.100.18   node18-sw
EOF'
```
続いて，node15から他のnodeへとssh接続がで切るように鍵を配布します．
以下のコマンドをnode15で実行してください．
```
ssh-keyscan 192.168.100.16 >> ~/.ssh/known_hosts
ssh-keyscan 192.168.100.17 >> ~/.ssh/known_hosts
ssh-keyscan 192.168.100.18 >> ~/.ssh/known_hosts

ssh-copy-id mprg@192.168.100.16
ssh-copy-id mprg@192.168.100.17
ssh-copy-id mprg@192.168.100.18

# 疎通の確認
for ip in 192.168.100.16 192.168.100.17 192.168.100.18; do
  echo "=== $ip ===" && ssh mprg@$ip hostname
done
```


## ステップ10.4：NCCL Testの実行


