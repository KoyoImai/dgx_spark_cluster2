## ステップ４：QSFPの2台接続
まず、QSFPケーブルで2台のDGX Sparkを接続してください。
ここでは、node15とnode16を接続して作業を勧めていきます。

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
