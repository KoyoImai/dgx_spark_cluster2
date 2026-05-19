# ステップ10
NCCL testによる通信速度のテストを実行する．

## 前置き
ステップ6〜ステップ9までで，QSFPスイッチによる4台接続のテストを行ってきたが，NCCL testによる通信速度が著しく遅いという現象が発生していた．
現在でも原因は不明だが，QSFPケーブルの差し替え（スイッチ経由からQSFPケーブル直接接続の切り替え）によって速度が低下するということが判明した．
DGX Spark自体を再起動すれば速度は正常に戻るが，その後QSFPケーブルの差し替えを行うことで速度が低下する．
根本的な原因解明と対策はまだできていないが，一旦この問題は傍に置いておいて，NCCL testによる速度調査を行う．

## ステップ10.1：IPアドレスの永続化
ステップ9では各nodeのIPアドレスを手動で固定していましたが，再起動のたびにリセットされるので永続化をします．
ここでは，QSFPスイッチを用いた接続でのIPアドレス固定を行います．
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
NCCL Testによる通信速度のテストを行います．
まず，以下のコマンドを全てのnodeで実行して環境変数を設定してください．
```
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64:$MPI_HOME/lib:$LD_LIBRARY_PATH"
```

### テスト1：2台（QSFPスイッチ）
node15で以下のコマンドを実行してください．
```
cd ~/nccl-tests

mpirun -np 2 \
  -H 192.168.100.15:1,192.168.100.16:1 \
  --mca oob_tcp_if_include enp1s0f1np1 \
  --mca btl_tcp_if_include enp1s0f1np1 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f1np1 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
```


### テスト2：4台（QSFPスイッチ）
node15で以下のコマンドを実行してください．
```
mpirun -np 4 \
  -H 192.168.100.15:1,192.168.100.16:1,192.168.100.17:1,192.168.100.18:1 \
  --mca oob_tcp_if_include enp1s0f1np1 \
  --mca btl_tcp_if_include enp1s0f1np1 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f1np1 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
```


### テスト3：4台（RJ45）
node15で以下のコマンドを実行してください．
```
export NCCL_IB_DISABLE=1

mpirun -np 4 \
  -H 10.0.0.15:1,10.0.0.16:1,10.0.0.17:1,10.0.0.18:1 \
  --mca oob_tcp_if_include enP7s7 \
  --mca btl_tcp_if_include enP7s7 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enP7s7 \
  -x NCCL_IB_DISABLE \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1

unset NCCL_IB_DISABLE
```



markdown## ステップ10.4 NCCL Test 結果

### 実験設定

| 項目 | 値 |
|------|-----|
| テスト種別 | all_gather_perf |
| メッセージサイズ | 1GB / 2GB / 4GB |
| 反復回数 | 50回 |
| ウォームアップ | 10回 |
| GPU数/node | 1 |

### 測定結果

| 構成 | 1GB | 2GB | 4GB | Avg busbw |
|------|-----|-----|-----|-----------|
| 2台 QSFPスイッチ | 21.55 GB/s | 22.27 GB/s | 22.61 GB/s | 22.14 GB/s |
| 4台 QSFPスイッチ | 21.71 GB/s | 22.08 GB/s | 22.42 GB/s | 22.07 GB/s |
| 4台 RJ45 (10GbE) |  1.18 GB/s |  1.18 GB/s |  1.18 GB/s |  1.18 GB/s |

※ out-of-place と in-place の平均値

### 理論値との比較

| 構成 | Avg busbw | 理論値 | 理論比 |
|------|-----------|--------|--------|
| 2台 QSFPスイッチ | 22.14 GB/s | 25.00 GB/s | 88.6% |
| 4台 QSFPスイッチ | 22.07 GB/s | 25.00 GB/s | 88.3% |
| 4台 RJ45 (10GbE) |  1.18 GB/s |  1.25 GB/s | 94.4% |
| **QSFP / RJ45 比** | **約18.7倍** | **20倍** | — |

### 考察

- QSFPスイッチ接続では理論値（200Gbps = 25 GB/s）の約88%を達成しており良好な結果
- 2台と4台でほぼ同じ busbw となっており，スイッチがボトルネックになっていないことを示している
- RJ45（10GbE）は理論値（10Gbps = 1.25 GB/s）の94.4%で、こちらも上限付近で動作している
- QSFPはRJ45と比べて約18.7倍の帯域幅を実現しており，大規模モデルの分散学習においてQSFP接続が有効である

