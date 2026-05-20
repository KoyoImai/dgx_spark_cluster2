# ステップ11：QSFPスイッチの速度低下原因調査
QSFPスイッチ接続時に，通信速度（nccl-test）が低下する原因を調査する．

## 前準備1：一般用語まとめ
### Network Interface Card (NIC)
Network Interface Card (NIC)とは，パソコンやサーバーの中に物理的に存在する部品のことで，ネットワークと通信するための出入り口となる装置のことです．

DGX SparkにはConnectX-7 NICというものが用意されており，QSFPケーブルで接続可能です．
`ibdev2netdev`コマンドを使用すると以下のように4つのinterface名が表示されると思います．
```
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Down)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Up)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Down)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Up)
```

### Remote Direct Memory Access (RDMA)
Remote Direct Memory Access (RDMA)とは，ネットワーク経由で接続されたコンピュータ間で，CPUやOSを介さずに直接メモリにアクセスする技術です．
OSやCPUを介さないため，超低遅延と高いスループットを実現可能であり，PyTorchのNCCLでも使われています．

### Queue Pair (QP)
Queue Pair (QP)とは，RDMAにおいて，データの送受信を行うためにペアで構成されるキューの仕組みです．

### RDMA over Converged Ethernet (RoCE)
RDMA over Converged Ethernet (RoCE)は，RDMAをEthernet上で動かすための方式です．

### InfiniBand
InfiniBandは，HPC/AIクラスタ向けの高速ネットワーク企画です．
Ethernetとは別系統のネットワークで，低遅延・高帯域・RDMAを前提にした通信に使われています．

DGX SparkのConnectX-7ポートはInfiniBand modeではなく，Ethernet configurationのみに対応しています．

### GPUDirect RDMA
[DGX SparkはGPUDirect RDMAに未対応です．](https://docs.nvidia.com/dgx/dgx-spark-porting-guide/porting/cuda.html)

### Blackwell GPU
NVIDIAが開発した次世代高性能GPUおよびそのアーキテクチャです．

### NVIDIA Collective Communication Library (NCCL)


## 前準備2：DGX Spark関連の用語まとめ
### NVLink-C2C
NVLink-C2Cとは，DGX Sparkにも使用されるプロセッサ間を接続するための超高速・広帯域幅のチップ間インターコネクト技術です．
DGX Sparkは，Grace CPUとBlackwell GPUを統合した，CPU-GPU統合メモリという方式をとっています．
このCPU-GPU統合メモリの内部で，Grace CPUとBlackwell GPUを繋いでいるのがNVLink-C2Cです．

### NVIDIA GB10 Grace Blackwell
NVIDIA GB10 Grace Blackwellとは，ArmベースのGrace CPUとBlackwell GPUを1つのモジュールに統合したプロセッサです．
DGX Sparkは，NVIDIA GB10 Grace Blackwellを採用しており，128GBの共有メモリを持っています．
共有メモリの内部では，NVLink-C2CによってGrace CPUとBlackwell GPUの接続が行われています．
System of Chip(SoC)と表記する場合もある．

### LPDDR5x coherent unified system memory
LPDDR5x coherent unified system memoryとは，CPUとGPUがデータコピーのオーバーヘッドなしに単一の広大なメモリ空間を共有できる技術です．






## 起きている現象
QSFPスイッチ，もしくはQSFPケーブルで直接接続したDGX Spark複数台でnccl-testを行う際，通信速度が低下する現象が発生する．
現状判明していることは以下の通りです．

- DGX Sparkを再起動した直後は，QSFPスイッチやQSFPケーブル直接接続のどちらでも妥当な通信速度を達成する（20GBps〜22GBps程度）．
- QSFP接続の方式を変更すると，nccl-testの通信速度が低下する．約2.8Gbps〜3.2GBps程度で，RJ45の1.25GBpsよりは速いが，QSFPの理論値25GBpsよりも限りなく低速になる．
- QSFP接続方式を再起動時に戻しても，通信速度は改善しない．




## メモ
### 1.1：スイッチ接続 & 再起動時
```
# FEC の確認
mprg@spark-fb97:~$ sudo ethtool --show-fec enp1s0f1np1
[sudo] mprg のパスワード: 
FEC parameters for enp1s0f1np1:
Supported/Configured FEC encodings: Auto
Active FEC encoding: RS
mprg@spark-fb97:~$ 
```
```
# nccl-test
mprg@spark-fb97:~/nccl-tests$ cd ~/nccl-tests
mpirun -np 2 \
  -H 192.168.100.15:1,192.168.100.16:1 \
  --mca oob_tcp_if_include enp1s0f1np1 \
  --mca btl_tcp_if_include enp1s0f1np1 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f1np1 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 4294967296 step: 2(factor) warmup iters: 10 iters: 50 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid 548419 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid 546336 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
  1073741824     134217728     float    none      -1  27211.0   39.46   19.73       0  25806.7   41.61   20.80       0
  2147483648     268435456     float    none      -1  50573.4   42.46   21.23       0  48690.4   44.10   22.05       0
  4294967296     536870912     float    none      -1  96986.7   44.28   22.14       0  95610.9   44.92   22.46       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 21.4033 
#
# Collective test concluded: all_gather_perf
#

mprg@spark-fb97:~/nccl-tests$ 
```

### 1.2：QSFPケーブルを直結に差し替え
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --show-fec enp1s0f0np0
FEC parameters for enp1s0f0np0:
Supported/Configured FEC encodings: Auto
Active FEC encoding: None
mprg@spark-fb97:~/nccl-tests$ 
```
```
mprg@spark-fb97:~/nccl-tests$ cd ~/nccl-tests
mpirun -np 2 \
  -H 10.0.1.1:1,10.0.1.2:1 \
  --mca oob_tcp_if_include enp1s0f0np0 \
  --mca btl_tcp_if_include enp1s0f0np0 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 4294967296 step: 2(factor) warmup iters: 10 iters: 50 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid 548805 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid 547459 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
  1073741824     134217728     float    none      -1   191408    5.61    2.80       0   190310    5.64    2.82       0
  2147483648     268435456     float    none      -1   381038    5.64    2.82       0   379627    5.66    2.83       0
  4294967296     536870912     float    none      -1   760506    5.65    2.82       0   758014    5.67    2.83       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 2.82151 
#
# Collective test concluded: all_gather_perf
#

mprg@spark-fb97:~/nccl-tests$ 
```

### 1.3：結果の確認
1.2のQSFPケーブル直結への切り替えで，nccl-testの通信速度が低下していることがわかります．
具体的には，`Avg bus bandwidth : 21.4033`が`Avg bus bandwidth : 2.82151`まで低下します．
またこの時，Active FEC encodingが`RS`から`None`へと変化していることも確認できます．


### 1.4：Active FEC encodingが原因なのかを調査
Active FEC encodingを`RS`へと変更する．
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --set-fec enp1s0f0np0 encoding rs
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --show-fec enp1s0f0np0
FEC parameters for enp1s0f0np0:
Supported/Configured FEC encodings: RS
Active FEC encoding: RS
```
Active FEC encodingを`RS`へと変更し，再度nccl-testを実行する．
```
mprg@spark-fb97:~/nccl-tests$ cd ~/nccl-tests
mpirun -np 2 \
  -H 10.0.1.1:1,10.0.1.2:1 \
  --mca oob_tcp_if_include enp1s0f0np0 \
  --mca btl_tcp_if_include enp1s0f0np0 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 4294967296 step: 2(factor) warmup iters: 10 iters: 50 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid 568339 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid 567181 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
  1073741824     134217728     float    none      -1   191211    5.62    2.81       0   190000    5.65    2.83       0
  2147483648     268435456     float    none      -1   381061    5.64    2.82       0   379159    5.66    2.83       0
  4294967296     536870912     float    none      -1   760771    5.65    2.82       0   757765    5.67    2.83       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 2.8233 
#
# Collective test concluded: all_gather_perf
#

mprg@spark-fb97:~/nccl-tests$ 
```

### 1.5：結果の確認
FECをRS-FECに変更しても速度が変わらないため，FECは根本原因ではない可能性が高いです．
次の可能性として，RDMAを考える．
RDMAは，CPUやOSを介さずにメモリ間で直接データをやりとりする技術です．
可能性として，QSFPケーブルを差し替えた際に，なんらかの理由でRDMAが使用されず，代わりにSocket通信でCPUやOSを介した通信となってしまっているため，速度が低下している？
← そもそも，DGX Sparkはcpu-gpu統合メモリを使用しているのだがら，RDMAのようにCPUを介さずにGPUメモリで通信というのの意味がわからない．
実際，DGX SparkはRDMA非対応．

### 1.6：スイッチ接続に再度戻してnccl-test
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --show-fec enp1s0f0np0
[sudo] mprg のパスワード: 
FEC parameters for enp1s0f0np0:
Supported/Configured FEC encodings: Auto
Active FEC encoding: None
mprg@spark-fb97:~/nccl-tests$ 
```
```
mprg@spark-fb97:~/nccl-tests$ cd ~/nccl-tests
mpirun -np 2 \
  -H 192.168.100.15:1,192.168.100.16:1 \
  --mca oob_tcp_if_include enp1s0f1np1 \
  --mca btl_tcp_if_include enp1s0f1np1 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f1np1 \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 50 -w 10 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 4294967296 step: 2(factor) warmup iters: 10 iters: 50 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid 568914 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid 568096 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
  1073741824     134217728     float    none      -1   190841    5.63    2.81       0   189868    5.66    2.83       0
  2147483648     268435456     float    none      -1   380304    5.65    2.82       0   378653    5.67    2.84       0
  4294967296     536870912     float    none      -1   759256    5.66    2.83       0   756605    5.68    2.84       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 2.82776 
#
# Collective test concluded: all_gather_perf
#

mprg@spark-fb97:~/nccl-tests$
```


### 1.7：スイッチ接続に戻した & Active FEC encoding を RS に変更
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --set-fec enp1s0f0np0 encoding rs
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --show-fec enp1s0f1np1
FEC parameters for enp1s0f1np1:
Supported/Configured FEC encodings: Auto
Active FEC encoding: RS
```

### 1.8：現状の確認と仮説
現状確認している事実は以下の通りです．
- DGX Sparkを再起動した直後は，QSFPスイッチによる接続，もしくはQSFPケーブルの直接接続のどちらであっても理論値に近い通信速度を達成する．
- QSFPケーブルでの接続方法を変更すると，接続方法によらず通信速度が低下する．
- QSFpケーブルの接続方法を再起動時と同じ状態にしても通信速度は戻らず，再起動するまで低速な状態が続く．





