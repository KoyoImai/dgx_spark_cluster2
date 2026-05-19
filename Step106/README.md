# ステップ106：LLM学習速度調査（QSFPも含めて再計測）
QSFPスイッチも含めて再計測を行います．
ステップ6〜8でQSFP直結の設定を削除してしまっているので，それらの復元などもここで行います．

## 前準備
### `40-cx7.yaml`の作成
QSFPケーブルでDGX Sparkを直接接続するために，`40-cx7.yaml`を再作成します．
各計算nodeで以下のコマンドを実行してください．
```
# node15
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
ip a show enp1s0f0np0 | grep inet
```

```
# node16
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
ip a show enp1s0f0np0 | grep inet
```

```
# node17
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
ip a show enp1s0f0np0 | grep inet
```

```
# node18
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
ip a show enp1s0f0np0 | grep inet
```

### `/etc/hosts`への追記
全てのnodeで以下のコマンドを実行してください．
```
sudo bash -c 'cat >> /etc/hosts << EOF

# QSFP direct connection (10.0.1.x / 10.0.2.x)
10.0.1.1   node15-qsfp
10.0.1.2   node16-qsfp
10.0.2.1   node17-qsfp
10.0.2.2   node18-qsfp
EOF'
```

### SSH鍵配布（node15で実行）
現在QSFPスイッチへと接続しているので，できていません．
後でやります．


## ステップ106.1：環境確認
全ての計算nodeで以下を実行して環境を確認します．
```
# NCCLの確認
ls -l ~/nccl/build/lib/libnccl.so*

# 3系統のネットワーク確認
ip a show enP7s7       | grep inet  # RJ45
ip a show enp1s0f0np0  | grep inet  # QSFP直結
ip a show enp1s0f1np1  | grep inet  # QSFPスイッチ
```


## ステップ106.2：学習スクリプトの用意
学習プログラムは，すでに作成済みの`/home4cluster/lora_train/train_alpaca.py`を使用します．
起動スクリプトは全て`/home4cluster/lora_train/step106/`ディレクトリに配置します．
以下のコマンドを管理者nodeで実行して起動スクリプトを作成してください．








