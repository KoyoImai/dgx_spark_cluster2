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
~実際，DGX SparkはRDMA非対応．~
DGX Sparkが非対応なのは，RDMAではなくDirectRDMAです．
← DGX Sparkの統合メモリは，あくまでCPUとGPUがメモリ空間を直接

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

### 1.8：現状の整理と低速化の仮説
現状確認している事実は以下の通りです．
- DGX Sparkを再起動した直後は，QSFPスイッチによる接続，もしくはQSFPケーブルの直接接続のどちらであっても理論値に近い通信速度を達成する．
- QSFPケーブルでの接続方法を変更すると，接続方法によらず通信速度が低下する．
- QSFpケーブルの接続方法を再起動時と同じ状態にしても通信速度は戻らず，再起動するまで低速な状態が続く．

上記の事実から，考えられる仮説
- 通信速度は低下しても，2.8GBps〜3.2GBps程度は出ていることから，RJ45（理論値1.25GBps）で通信してしまっている可能性は低い
- 通信速度の低下の直接的なトリガーは，QSFPケーブル差し替えによる物理リンクの切断




### 1.9：TCPソケット通信になっている可能性の調査
再起動直後のQSFPスイッチ経由通信でのnccl-test
```
mprg@spark-fb97:~/nccl-tests$ DISPLAY= \
mpirun -np 2 \
  -H 192.168.100.15:1,192.168.100.16:1 \
  --mca oob_tcp_if_include enp1s0f1np1 \
  --mca btl_tcp_if_include enp1s0f1np1 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f1np1 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_DEBUG_SUBSYS=INIT,NET \
  ./build/all_gather_perf -b 1G -e 1G -f 2 -n 1 -w 1 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 1073741824 step: 2(factor) warmup iters: 1 iters: 1 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid  23491 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid  24302 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
spark-fb97:23491:23491 [0] NCCL INFO ENV/Plugin: Could not find: libnccl-env.so
spark-fb97:23491:23491 [0] NCCL INFO NCCL_SOCKET_IFNAME set to enp1s0f1np1
spark-fb97:23491:23491 [0] NCCL INFO cudaDriverVersion 13000
spark-fb97:23491:23491 [0] NCCL INFO NCCL version 2.28.9+cuda13.0
spark-4440:24302:24302 [0] NCCL INFO ENV/Plugin: Could not find: libnccl-env.so
spark-4440:24302:24302 [0] NCCL INFO cudaDriverVersion 13000
spark-4440:24302:24302 [0] NCCL INFO NCCL_SOCKET_IFNAME set to enp1s0f1np1
spark-4440:24302:24302 [0] NCCL INFO NCCL version 2.28.9+cuda13.0
spark-4440:24302:24302 [0] NCCL INFO NET/Plugin: Could not find: libnccl-net.so
spark-4440:24302:24302 [0] NCCL INFO dlvsym failed on mlx5dv_get_data_direct_sysfs_path - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_get_data_direct_sysfs_path, version MLX5_1.25 version MLX5_1.25
spark-4440:24302:24302 [0] NCCL INFO dlvsym failed on mlx5dv_reg_dmabuf_mr - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_reg_dmabuf_mr, version MLX5_1.25 version MLX5_1.25
spark-fb97:23491:23491 [0] NCCL INFO NET/Plugin: Could not find: libnccl-net.so
spark-fb97:23491:23491 [0] NCCL INFO dlvsym failed on mlx5dv_get_data_direct_sysfs_path - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_get_data_direct_sysfs_path, version MLX5_1.25 version MLX5_1.25
spark-fb97:23491:23491 [0] NCCL INFO dlvsym failed on mlx5dv_reg_dmabuf_mr - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_reg_dmabuf_mr, version MLX5_1.25 version MLX5_1.25
spark-4440:24302:24302 [0] NCCL INFO NET/IB: [1] rocep1s0f1:uverbs1:1/RoCE provider=Mlx5 speed=200000 context=0xc53b90db27f0 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-4440:24302:24302 [0] NCCL INFO NET/IB : Made virtual device [0] name=rocep1s0f1 speed=200000 ndevs=1
spark-fb97:23491:23491 [0] NCCL INFO NET/IB: [1] rocep1s0f1:uverbs1:1/RoCE provider=Mlx5 speed=200000 context=0xbca554388f70 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-fb97:23491:23491 [0] NCCL INFO NET/IB : Made virtual device [0] name=rocep1s0f1 speed=200000 ndevs=1
spark-4440:24302:24302 [0] NCCL INFO NET/IB: [3] roceP2p1s0f1:uverbs3:1/RoCE provider=Mlx5 speed=200000 context=0xc53b90df3ef0 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
spark-4440:24302:24302 [0] NCCL INFO NET/IB : Made virtual device [1] name=roceP2p1s0f1 speed=200000 ndevs=1
spark-4440:24302:24302 [0] NCCL INFO NET/IB : Using [0]rocep1s0f1:1/RoCE [1]roceP2p1s0f1:1/RoCE [RO]; OOB enp1s0f1np1:192.168.100.16<0>
spark-4440:24302:24302 [0] NCCL INFO Initialized NET plugin IB
spark-4440:24302:24302 [0] NCCL INFO Assigned NET plugin IB to comm
spark-4440:24302:24302 [0] NCCL INFO Assigned GIN plugin GIN_IB_GDAKI to comm
spark-4440:24302:24302 [0] NCCL INFO Using network IB
spark-fb97:23491:23491 [0] NCCL INFO NET/IB: [3] roceP2p1s0f1:uverbs3:1/RoCE provider=Mlx5 speed=200000 context=0xbca5543ca670 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
spark-fb97:23491:23491 [0] NCCL INFO NET/IB : Made virtual device [1] name=roceP2p1s0f1 speed=200000 ndevs=1
spark-fb97:23491:23491 [0] NCCL INFO NET/IB : Using [0]rocep1s0f1:1/RoCE [1]roceP2p1s0f1:1/RoCE [RO]; OOB enp1s0f1np1:192.168.100.15<0>
spark-fb97:23491:23491 [0] NCCL INFO Initialized NET plugin IB
spark-fb97:23491:23491 [0] NCCL INFO Assigned NET plugin IB to comm
spark-fb97:23491:23491 [0] NCCL INFO Assigned GIN plugin GIN_IB_GDAKI to comm
spark-fb97:23491:23491 [0] NCCL INFO Using network IB
spark-4440:24302:24302 [0] NCCL INFO ncclCommInitRankConfig comm 0xc53b8f7f6ab0 rank 1 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0x3470ca318760efc8 - Init START
spark-fb97:23491:23491 [0] NCCL INFO ncclCommInitRankConfig comm 0xbca552dd2650 rank 0 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0x3470ca318760efc8 - Init START
spark-fb97:23491:23491 [0] NCCL INFO RAS client listening socket at 127.0.0.1<28028>
spark-4440:24302:24302 [0] NCCL INFO RAS client listening socket at 127.0.0.1<28028>
spark-fb97:23491:23491 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 0 'rocep1s0f1'
spark-4440:24302:24302 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 0 'rocep1s0f1'
spark-fb97:23491:23491 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 1 'roceP2p1s0f1'
spark-4440:24302:24302 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 1 'roceP2p1s0f1'
spark-fb97:23491:23491 [0] NCCL INFO ncclTopoGetCpuAffinity: Affinity for GPU 0 is empty, ignoring. (GPU affinity =  ; CPU affinity = 0).
spark-4440:24302:24302 [0] NCCL INFO ncclTopoGetCpuAffinity: Affinity for GPU 0 is empty, ignoring. (GPU affinity =  ; CPU affinity = 0).
spark-fb97:23491:23491 [0] NCCL INFO comm 0xbca552dd2650 rank 0 nRanks 2 nNodes 2 localRanks 1 localRank 0 MNNVL 0
spark-fb97:23491:23491 [0] NCCL INFO Channel 00/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 01/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 02/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 03/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 04/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 05/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 06/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 07/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 08/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 09/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 10/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 11/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 12/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 13/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 14/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Channel 15/16 : 0 1
spark-fb97:23491:23491 [0] NCCL INFO Trees [0] 1/-1/-1->0->-1 [1] 1/-1/-1->0->-1 [2] 1/-1/-1->0->-1 [3] 1/-1/-1->0->-1 [4] 1/-1/-1->0->-1 [5] 1/-1/-1->0->-1 [6] 1/-1/-1->0->-1 [7] 1/-1/-1->0->-1 [8] -1/-1/-1->0->1 [9] -1/-1/-1->0->1 [10] -1/-1/-1->0->1 [11] -1/-1/-1->0->1 [12] -1/-1/-1->0->1 [13] -1/-1/-1->0->1 [14] -1/-1/-1->0->1 [15] -1/-1/-1->0->1
spark-fb97:23491:23491 [0] NCCL INFO P2P Chunksize set to 131072
spark-4440:24302:24302 [0] NCCL INFO comm 0xc53b8f7f6ab0 rank 1 nRanks 2 nNodes 2 localRanks 1 localRank 0 MNNVL 0
spark-4440:24302:24302 [0] NCCL INFO Trees [0] -1/-1/-1->1->0 [1] -1/-1/-1->1->0 [2] -1/-1/-1->1->0 [3] -1/-1/-1->1->0 [4] -1/-1/-1->1->0 [5] -1/-1/-1->1->0 [6] -1/-1/-1->1->0 [7] -1/-1/-1->1->0 [8] 0/-1/-1->1->-1 [9] 0/-1/-1->1->-1 [10] 0/-1/-1->1->-1 [11] 0/-1/-1->1->-1 [12] 0/-1/-1->1->-1 [13] 0/-1/-1->1->-1 [14] 0/-1/-1->1->-1 [15] 0/-1/-1->1->-1
spark-4440:24302:24302 [0] NCCL INFO P2P Chunksize set to 131072
spark-fb97:23491:23491 [0] NCCL INFO PROFILER/Plugin: Could not find: libnccl-profiler.so
spark-fb97:23491:23491 [0] NCCL INFO Check P2P Type isAllDirectP2p 1 directMode 0 isAllCudaP2p 1
spark-4440:24302:24302 [0] NCCL INFO PROFILER/Plugin: Could not find: libnccl-profiler.so
spark-4440:24302:24302 [0] NCCL INFO Check P2P Type isAllDirectP2p 1 directMode 0 isAllCudaP2p 1
spark-4440:24302:24310 [0] NCCL INFO [Proxy Service] Device 0 CPU core 0
spark-fb97:23491:23501 [0] NCCL INFO [Proxy Service] Device 0 CPU core 0
spark-fb97:23491:23502 [0] NCCL INFO [Proxy Service UDS] Device 0 CPU core 0
spark-4440:24302:24311 [0] NCCL INFO [Proxy Service UDS] Device 0 CPU core 0
spark-fb97:23491:23491 [0] NCCL INFO TUNER/Plugin: Could not find: libnccl-tuner.so
spark-fb97:23491:23491 [0] NCCL INFO threadThresholds 8/8/64 | 16/8/64 | 512 | 512
spark-fb97:23491:23491 [0] NCCL INFO 16 coll channels, 16 collnet channels, 0 nvls channels, 16 p2p channels, 2 p2p channels per peer
spark-fb97:23491:23491 [0] NCCL INFO CC Off, workFifoBytes 1048576
spark-fb97:23491:23491 [0] NCCL INFO ncclCommInitRankConfig comm 0xbca552dd2650 rank 0 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0x3470ca318760efc8 - Init COMPLETE
spark-fb97:23491:23491 [0] NCCL INFO Init timings - ncclCommInitRankConfig: rank 0 nranks 2 total 0.25 (kernels 0.18, alloc 0.06, bootstrap 0.01, allgathers 0.00, topo 0.00, graphs 0.00, connections 0.01, rest 0.00)
spark-4440:24302:24302 [0] NCCL INFO TUNER/Plugin: Could not find: libnccl-tuner.so
spark-4440:24302:24302 [0] NCCL INFO threadThresholds 8/8/64 | 16/8/64 | 512 | 512
spark-4440:24302:24302 [0] NCCL INFO 16 coll channels, 16 collnet channels, 0 nvls channels, 16 p2p channels, 2 p2p channels per peer
spark-4440:24302:24302 [0] NCCL INFO ncclCommInitRankConfig comm 0xc53b8f7f6ab0 rank 1 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0x3470ca318760efc8 - Init COMPLETE
spark-4440:24302:24302 [0] NCCL INFO Init timings - ncclCommInitRankConfig: rank 1 nranks 2 total 0.25 (kernels 0.17, alloc 0.05, bootstrap 0.02, allgathers 0.00, topo 0.00, graphs 0.00, connections 0.01, rest 0.00)
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
spark-4440:24302:24312 [0] NCCL INFO [Proxy Progress] Device 0 CPU core 0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 0 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000d20
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 0 from local rank 0, transport 2
spark-fb97:23491:23503 [0] NCCL INFO [Proxy Progress] Device 0 CPU core 0
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000d20
spark-4440:24302:24302 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 00/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 1 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000d98
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 1 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000d98
spark-4440:24302:24302 [0] NCCL INFO Channel 01/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 01/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 2 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000e10
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 2 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000e10
spark-4440:24302:24302 [0] NCCL INFO Channel 02/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 02/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 3 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000e88
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 3 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000e88
spark-4440:24302:24302 [0] NCCL INFO Channel 03/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 03/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 4 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000f00
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 4 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000f00
spark-4440:24302:24302 [0] NCCL INFO Channel 04/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 04/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 5 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000f78
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 5 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000f78
spark-fb97:23491:23491 [0] NCCL INFO Channel 05/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 05/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 6 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4000ff0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 6 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68000ff0
spark-fb97:23491:23491 [0] NCCL INFO Channel 06/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 06/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 7 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001068
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 7 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001068
spark-fb97:23491:23491 [0] NCCL INFO Channel 07/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 07/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 8 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40010e0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 8 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680010e0
spark-4440:24302:24302 [0] NCCL INFO Channel 08/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 08/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 9 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001158
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 9 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001158
spark-4440:24302:24302 [0] NCCL INFO Channel 09/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 09/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 10 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40011d0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 10 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680011d0
spark-4440:24302:24302 [0] NCCL INFO Channel 10/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 10/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 11 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001248
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 11 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001248
spark-fb97:23491:23491 [0] NCCL INFO Channel 11/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 11/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 12 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40012c0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 12 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680012c0
spark-fb97:23491:23491 [0] NCCL INFO Channel 12/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 12/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 13 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001338
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 13 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001338
spark-4440:24302:24302 [0] NCCL INFO Channel 13/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 13/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 14 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40013b0
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 14 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680013b0
spark-fb97:23491:23491 [0] NCCL INFO Channel 14/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 14/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy recv connection 15 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001428
spark-4440:24302:24310 [0] NCCL INFO New proxy recv connection 15 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001428
spark-fb97:23491:23491 [0] NCCL INFO Channel 15/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 15/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 16 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40014a0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 16 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680014a0
spark-fb97:23491:23491 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 00/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 17 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001518
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 17 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001518
spark-fb97:23491:23491 [0] NCCL INFO Channel 01/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 01/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 18 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001590
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 18 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001590
spark-fb97:23491:23491 [0] NCCL INFO Channel 02/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 02/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 19 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001608
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 19 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001608
spark-fb97:23491:23491 [0] NCCL INFO Channel 03/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 03/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 20 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001680
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 20 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001680
spark-fb97:23491:23491 [0] NCCL INFO Channel 04/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 04/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 21 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680016f8
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 21 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40016f8
spark-4440:24302:24302 [0] NCCL INFO Channel 05/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 05/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 22 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001770
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 22 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001770
spark-4440:24302:24302 [0] NCCL INFO Channel 06/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 06/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 23 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680017e8
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 23 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40017e8
spark-4440:24302:24302 [0] NCCL INFO Channel 07/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 07/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 24 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001860
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 24 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001860
spark-4440:24302:24302 [0] NCCL INFO Channel 08/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 08/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 25 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680018d8
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 25 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40018d8
spark-4440:24302:24302 [0] NCCL INFO Channel 09/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 09/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 26 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001950
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 26 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001950
spark-4440:24302:24302 [0] NCCL INFO Channel 10/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 10/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 27 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a680019c8
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 27 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f40019c8
spark-4440:24302:24302 [0] NCCL INFO Channel 11/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 11/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 28 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001a40
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 28 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001a40
spark-4440:24302:24302 [0] NCCL INFO Channel 12/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23491 [0] NCCL INFO Channel 12/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 29 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001ab8
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 29 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001ab8
spark-4440:24302:24302 [0] NCCL INFO Channel 13/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23491:23491 [0] NCCL INFO Channel 13/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 30 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001b30
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 30 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001b30
spark-fb97:23491:23491 [0] NCCL INFO Channel 14/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:24302:24302 [0] NCCL INFO Channel 14/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23491:23501 [0] NCCL INFO New proxy send connection 31 from local rank 0, transport 2
spark-fb97:23491:23491 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff89f4001ba8
spark-4440:24302:24310 [0] NCCL INFO New proxy send connection 31 from local rank 0, transport 2
spark-4440:24302:24302 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xfd9a68001ba8
spark-fb97:23491:23491 [0] NCCL INFO Channel 15/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:24302:24302 [0] NCCL INFO Channel 15/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 576 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b9e9c fifoLkey=0x1b9e9c
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 576 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 704 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1ddcd6 fifoLkey=0x1ddcd6
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 704 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 577 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b9591 fifoLkey=0x1b9591
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 577 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 705 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1dd0cc fifoLkey=0x1dd0cc
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 705 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 576 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 704 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1deeeb fifoLkey=0x1deeeb
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 704 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 578 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1bbbb8 fifoLkey=0x1bbbb8
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 578 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 704 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 707 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1dece9 fifoLkey=0x1dece9
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 707 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 581 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1bafac fifoLkey=0x1bafac
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 581 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 577 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 704 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 705 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 578 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 707 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 581 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 582 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1beeeb fifoLkey=0x1beeeb
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 582 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 582 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1ba7a5 fifoLkey=0x1ba7a5
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 582 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 582 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 582 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 710 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1deae7 fifoLkey=0x1deae7
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 710 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 585 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1b9f9d fifoLkey=0x1b9f9d
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 585 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 710 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1db6b2 fifoLkey=0x1db6b2
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 710 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 711 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1de7e6 fifoLkey=0x1de7e6
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 711 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 586 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1b9795 fifoLkey=0x1b9795
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 586 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 712 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1de5e5 fifoLkey=0x1de5e5
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 712 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 585 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b7374 fifoLkey=0x1b7374
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 585 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 587 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1b8d8c fifoLkey=0x1b8d8c
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 587 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 710 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 713 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1d9b9d fifoLkey=0x1d9b9d
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 713 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 588 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b5e5c fifoLkey=0x1b5e5c
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 588 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 585 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 716 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1d8c8c fifoLkey=0x1d8c8c
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 716 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 591 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b4442 fifoLkey=0x1b4442
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 591 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 711 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 710 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 586 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 585 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 712 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 587 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 713 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 588 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 716 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 591 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 719 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1d7577 fifoLkey=0x1d7577
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 719 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 594 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1b2d2a fifoLkey=0x1b2d2a
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 594 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 719 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1de4dd fifoLkey=0x1de4dd
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 719 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 720 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1d6c6c fifoLkey=0x1d6c6c
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 720 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 594 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1b6865 fifoLkey=0x1b6865
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 594 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 722 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1dd9d7 fifoLkey=0x1dd9d7
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 722 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 719 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 597 mtu 3 GID 3 (0/F64A8C0FFFF0000) fifoRkey=0x1b524f fifoLkey=0x1b524f
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 597 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 719 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 594 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 594 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 720 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 722 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 597 mtu 3 GID 3 (0/1064A8C0FFFF0000) fifoRkey=0x1bece9 fifoLkey=0x1bece9
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 597 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 597 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 725 mtu 3 GID 0 (80FE/9DFB2FFEFF47BB4E) fifoRkey=0x1ddbd1 fifoLkey=0x1ddbd1
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 725 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 597 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 725 mtu 3 GID 0 (80FE/46442FFEFF47BB4E) fifoRkey=0x1d4a49 fifoLkey=0x1d4a49
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 725 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23501 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 725 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:24302:24310 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 725 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23491:23491 [0] NCCL INFO Connected all rings, use ring PXN 0 GDR 0
spark-4440:24302:24302 [0] NCCL INFO Connected all rings, use ring PXN 0 GDR 0
  1073741824     134217728     float    none      -1  24414.9   43.98   21.99       0  24472.5   43.88   21.94       0
spark-fb97:23491:23491 [0] NCCL INFO comm 0xbca552dd2650 rank 0 nranks 2 cudaDev 0 busId f01000 - Destroy COMPLETE
spark-4440:24302:24302 [0] NCCL INFO comm 0xc53b8f7f6ab0 rank 1 nranks 2 cudaDev 0 busId f01000 - Destroy COMPLETE
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 21.9636 
#
# Collective test concluded: all_gather_perf
#

spark-4440:24302:24302 [0] NCCL INFO ENV/Plugin: Closing env plugin ncclEnvDefault
spark-fb97:23491:23491 [0] NCCL INFO ENV/Plugin: Closing env plugin ncclEnvDefault
mprg@spark-fb97:~/nccl-tests$ 
```

QSFP直結に切り替えてnccl-testを実行．
```
DISPLAY= \
mpirun -np 2 \
  -H 10.0.1.1:1,10.0.1.2:1 \
  --mca oob_tcp_if_include enp1s0f0np0 \
  --mca btl_tcp_if_include enp1s0f0np0 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_DEBUG_SUBSYS=INIT,NET \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 5 -w 2 -g 1
```
```
mprg@spark-fb97:~/nccl-tests$ DISPLAY= \
mpirun -np 2 \
  -H 10.0.1.1:1,10.0.1.2:1 \
  --mca oob_tcp_if_include enp1s0f0np0 \
  --mca btl_tcp_if_include enp1s0f0np0 \
  -x LD_LIBRARY_PATH \
  -x NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_DEBUG_SUBSYS=INIT,NET \
  ./build/all_gather_perf -b 1G -e 4G -f 2 -n 5 -w 2 -g 1
Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

Authorization required, but no authorization protocol specified

# nccl-tests version 2.18.3 nccl-headers=22809 nccl-library=22809
# Collective test starting: all_gather_perf
# nThread 1 nGpus 1 minBytes 1073741824 maxBytes 4294967296 step: 2(factor) warmup iters: 2 iters: 5 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid  23940 on spark-fb97 device  0 [000f:01:00] NVIDIA GB10
#  Rank  1 Group  0 Pid  25264 on spark-4440 device  0 [000f:01:00] NVIDIA GB10
spark-fb97:23940:23940 [0] NCCL INFO ENV/Plugin: Could not find: libnccl-env.so
spark-fb97:23940:23940 [0] NCCL INFO NCCL_SOCKET_IFNAME set to enp1s0f0np0
spark-fb97:23940:23940 [0] NCCL INFO cudaDriverVersion 13000
spark-fb97:23940:23940 [0] NCCL INFO NCCL version 2.28.9+cuda13.0
spark-4440:25264:25264 [0] NCCL INFO ENV/Plugin: Could not find: libnccl-env.so
spark-4440:25264:25264 [0] NCCL INFO cudaDriverVersion 13000
spark-4440:25264:25264 [0] NCCL INFO NCCL_SOCKET_IFNAME set to enp1s0f0np0
spark-4440:25264:25264 [0] NCCL INFO NCCL version 2.28.9+cuda13.0
spark-fb97:23940:23940 [0] NCCL INFO NET/Plugin: Could not find: libnccl-net.so
spark-fb97:23940:23940 [0] NCCL INFO dlvsym failed on mlx5dv_get_data_direct_sysfs_path - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_get_data_direct_sysfs_path, version MLX5_1.25 version MLX5_1.25
spark-fb97:23940:23940 [0] NCCL INFO dlvsym failed on mlx5dv_reg_dmabuf_mr - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_reg_dmabuf_mr, version MLX5_1.25 version MLX5_1.25
spark-fb97:23940:23940 [0] NCCL INFO NET/IB: [0] rocep1s0f0:uverbs0:1/RoCE provider=Mlx5 speed=200000 context=0xc1abd84b7eb0 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : Made virtual device [0] name=rocep1s0f0 speed=200000 ndevs=1
spark-4440:25264:25264 [0] NCCL INFO NET/Plugin: Could not find: libnccl-net.so
spark-4440:25264:25264 [0] NCCL INFO dlvsym failed on mlx5dv_get_data_direct_sysfs_path - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_get_data_direct_sysfs_path, version MLX5_1.25 version MLX5_1.25
spark-4440:25264:25264 [0] NCCL INFO dlvsym failed on mlx5dv_reg_dmabuf_mr - /lib/aarch64-linux-gnu/libmlx5.so: undefined symbol: mlx5dv_reg_dmabuf_mr, version MLX5_1.25 version MLX5_1.25
spark-4440:25264:25264 [0] NCCL INFO NET/IB: [0] rocep1s0f0:uverbs0:1/RoCE provider=Mlx5 speed=200000 context=0xad6cd9442240 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-4440:25264:25264 [0] NCCL INFO NET/IB : Made virtual device [0] name=rocep1s0f0 speed=200000 ndevs=1
spark-fb97:23940:23940 [0] NCCL INFO NET/IB: [2] roceP2p1s0f0:uverbs2:1/RoCE provider=Mlx5 speed=200000 context=0xc1abd84f95b0 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : Made virtual device [1] name=roceP2p1s0f0 speed=200000 ndevs=1
spark-4440:25264:25264 [0] NCCL INFO NET/IB: [2] roceP2p1s0f0:uverbs2:1/RoCE provider=Mlx5 speed=200000 context=0xad6cd9483940 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
spark-4440:25264:25264 [0] NCCL INFO NET/IB : Made virtual device [1] name=roceP2p1s0f0 speed=200000 ndevs=1
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : Using [0]rocep1s0f0:1/RoCE [1]roceP2p1s0f0:1/RoCE [RO]; OOB enp1s0f0np0:10.0.1.1<0>
spark-fb97:23940:23940 [0] NCCL INFO Initialized NET plugin IB
spark-fb97:23940:23940 [0] NCCL INFO Assigned NET plugin IB to comm
spark-fb97:23940:23940 [0] NCCL INFO Assigned GIN plugin GIN_IB_GDAKI to comm
spark-fb97:23940:23940 [0] NCCL INFO Using network IB
spark-4440:25264:25264 [0] NCCL INFO NET/IB : Using [0]rocep1s0f0:1/RoCE [1]roceP2p1s0f0:1/RoCE [RO]; OOB enp1s0f0np0:10.0.1.2<0>
spark-4440:25264:25264 [0] NCCL INFO Initialized NET plugin IB
spark-4440:25264:25264 [0] NCCL INFO Assigned NET plugin IB to comm
spark-4440:25264:25264 [0] NCCL INFO Assigned GIN plugin GIN_IB_GDAKI to comm
spark-4440:25264:25264 [0] NCCL INFO Using network IB
spark-fb97:23940:23940 [0] NCCL INFO ncclCommInitRankConfig comm 0xc1abd6f01570 rank 0 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0xe6b90f987a9283f2 - Init START
spark-4440:25264:25264 [0] NCCL INFO ncclCommInitRankConfig comm 0xad6cd7e86360 rank 1 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0xe6b90f987a9283f2 - Init START
spark-fb97:23940:23940 [0] NCCL INFO RAS client listening socket at 127.0.0.1<28028>
spark-4440:25264:25264 [0] NCCL INFO RAS client listening socket at 127.0.0.1<28028>
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 0 'rocep1s0f0'
spark-4440:25264:25264 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 0 'rocep1s0f0'
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 1 'roceP2p1s0f0'
spark-4440:25264:25264 [0] NCCL INFO NET/IB : GPU Direct RDMA Disabled for HCA 1 'roceP2p1s0f0'
spark-fb97:23940:23940 [0] NCCL INFO ncclTopoGetCpuAffinity: Affinity for GPU 0 is empty, ignoring. (GPU affinity =  ; CPU affinity = 0).
spark-4440:25264:25264 [0] NCCL INFO ncclTopoGetCpuAffinity: Affinity for GPU 0 is empty, ignoring. (GPU affinity =  ; CPU affinity = 0).
spark-fb97:23940:23940 [0] NCCL INFO comm 0xc1abd6f01570 rank 0 nRanks 2 nNodes 2 localRanks 1 localRank 0 MNNVL 0
spark-fb97:23940:23940 [0] NCCL INFO Channel 00/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 01/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 02/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 03/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 04/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 05/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 06/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 07/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 08/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 09/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 10/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 11/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 12/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 13/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 14/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Channel 15/16 : 0 1
spark-fb97:23940:23940 [0] NCCL INFO Trees [0] 1/-1/-1->0->-1 [1] 1/-1/-1->0->-1 [2] 1/-1/-1->0->-1 [3] 1/-1/-1->0->-1 [4] 1/-1/-1->0->-1 [5] 1/-1/-1->0->-1 [6] 1/-1/-1->0->-1 [7] 1/-1/-1->0->-1 [8] -1/-1/-1->0->1 [9] -1/-1/-1->0->1 [10] -1/-1/-1->0->1 [11] -1/-1/-1->0->1 [12] -1/-1/-1->0->1 [13] -1/-1/-1->0->1 [14] -1/-1/-1->0->1 [15] -1/-1/-1->0->1
spark-fb97:23940:23940 [0] NCCL INFO P2P Chunksize set to 131072
spark-fb97:23940:23940 [0] NCCL INFO PROFILER/Plugin: Could not find: libnccl-profiler.so
spark-fb97:23940:23940 [0] NCCL INFO Check P2P Type isAllDirectP2p 1 directMode 0 isAllCudaP2p 1
spark-4440:25264:25264 [0] NCCL INFO comm 0xad6cd7e86360 rank 1 nRanks 2 nNodes 2 localRanks 1 localRank 0 MNNVL 0
spark-4440:25264:25264 [0] NCCL INFO Trees [0] -1/-1/-1->1->0 [1] -1/-1/-1->1->0 [2] -1/-1/-1->1->0 [3] -1/-1/-1->1->0 [4] -1/-1/-1->1->0 [5] -1/-1/-1->1->0 [6] -1/-1/-1->1->0 [7] -1/-1/-1->1->0 [8] 0/-1/-1->1->-1 [9] 0/-1/-1->1->-1 [10] 0/-1/-1->1->-1 [11] 0/-1/-1->1->-1 [12] 0/-1/-1->1->-1 [13] 0/-1/-1->1->-1 [14] 0/-1/-1->1->-1 [15] 0/-1/-1->1->-1
spark-4440:25264:25264 [0] NCCL INFO P2P Chunksize set to 131072
spark-4440:25264:25264 [0] NCCL INFO PROFILER/Plugin: Could not find: libnccl-profiler.so
spark-4440:25264:25264 [0] NCCL INFO Check P2P Type isAllDirectP2p 1 directMode 0 isAllCudaP2p 1
spark-fb97:23940:23949 [0] NCCL INFO [Proxy Service] Device 0 CPU core 0
spark-fb97:23940:23950 [0] NCCL INFO [Proxy Service UDS] Device 0 CPU core 0
spark-4440:25264:25272 [0] NCCL INFO [Proxy Service] Device 0 CPU core 0
spark-4440:25264:25273 [0] NCCL INFO [Proxy Service UDS] Device 0 CPU core 0
spark-fb97:23940:23940 [0] NCCL INFO TUNER/Plugin: Could not find: libnccl-tuner.so
spark-fb97:23940:23940 [0] NCCL INFO threadThresholds 8/8/64 | 16/8/64 | 512 | 512
spark-fb97:23940:23940 [0] NCCL INFO 16 coll channels, 16 collnet channels, 0 nvls channels, 16 p2p channels, 2 p2p channels per peer
spark-fb97:23940:23940 [0] NCCL INFO CC Off, workFifoBytes 1048576
spark-fb97:23940:23940 [0] NCCL INFO ncclCommInitRankConfig comm 0xc1abd6f01570 rank 0 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0xe6b90f987a9283f2 - Init COMPLETE
spark-fb97:23940:23940 [0] NCCL INFO Init timings - ncclCommInitRankConfig: rank 0 nranks 2 total 0.25 (kernels 0.17, alloc 0.06, bootstrap 0.01, allgathers 0.00, topo 0.00, graphs 0.00, connections 0.01, rest 0.00)
spark-4440:25264:25264 [0] NCCL INFO TUNER/Plugin: Could not find: libnccl-tuner.so
spark-4440:25264:25264 [0] NCCL INFO threadThresholds 8/8/64 | 16/8/64 | 512 | 512
spark-4440:25264:25264 [0] NCCL INFO 16 coll channels, 16 collnet channels, 0 nvls channels, 16 p2p channels, 2 p2p channels per peer
spark-4440:25264:25264 [0] NCCL INFO ncclCommInitRankConfig comm 0xad6cd7e86360 rank 1 nranks 2 cudaDev 0 nvmlDev 0 busId f01000 commId 0xe6b90f987a9283f2 - Init COMPLETE
spark-4440:25264:25264 [0] NCCL INFO Init timings - ncclCommInitRankConfig: rank 1 nranks 2 total 0.25 (kernels 0.18, alloc 0.05, bootstrap 0.01, allgathers 0.00, topo 0.00, graphs 0.00, connections 0.01, rest 0.00)
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 0 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000d20
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 0 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000d20
spark-4440:25264:25274 [0] NCCL INFO [Proxy Progress] Device 0 CPU core 0
spark-4440:25264:25264 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23951 [0] NCCL INFO [Proxy Progress] Device 0 CPU core 0
spark-fb97:23940:23940 [0] NCCL INFO Channel 00/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 1 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000d98
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 1 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000d98
spark-4440:25264:25264 [0] NCCL INFO Channel 01/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 01/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 2 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000e10
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 2 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000e10
spark-4440:25264:25264 [0] NCCL INFO Channel 02/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 02/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 3 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000e88
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 3 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000e88
spark-4440:25264:25264 [0] NCCL INFO Channel 03/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 03/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 4 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000f00
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 4 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000f00
spark-4440:25264:25264 [0] NCCL INFO Channel 04/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 04/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 5 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000f78
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 5 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000f78
spark-4440:25264:25264 [0] NCCL INFO Channel 05/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 05/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 6 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260000ff0
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 6 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4000ff0
spark-4440:25264:25264 [0] NCCL INFO Channel 06/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 06/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 7 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001068
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 7 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001068
spark-4440:25264:25264 [0] NCCL INFO Channel 07/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 07/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 8 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600010e0
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 8 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40010e0
spark-4440:25264:25264 [0] NCCL INFO Channel 08/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 08/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 9 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001158
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 9 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001158
spark-4440:25264:25264 [0] NCCL INFO Channel 09/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 09/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 10 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600011d0
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 10 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40011d0
spark-4440:25264:25264 [0] NCCL INFO Channel 10/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 10/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 11 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001248
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 11 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001248
spark-4440:25264:25264 [0] NCCL INFO Channel 11/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 11/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 12 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600012c0
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 12 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40012c0
spark-4440:25264:25264 [0] NCCL INFO Channel 12/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 12/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 13 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001338
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 13 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001338
spark-4440:25264:25264 [0] NCCL INFO Channel 13/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 13/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 14 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600013b0
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 14 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40013b0
spark-4440:25264:25264 [0] NCCL INFO Channel 14/0 : 0[0] -> 1[0] [receive] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 14/0 : 1[0] -> 0[0] [receive] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy recv connection 15 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001428
spark-fb97:23940:23949 [0] NCCL INFO New proxy recv connection 15 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001428
spark-4440:25264:25264 [0] NCCL INFO Channel 15/0 : 0[0] -> 1[0] [receive] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 15/0 : 1[0] -> 0[0] [receive] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 16 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600014a0
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 16 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40014a0
spark-4440:25264:25264 [0] NCCL INFO Channel 00/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 17 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001518
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 17 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001518
spark-4440:25264:25264 [0] NCCL INFO Channel 01/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 01/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 18 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001590
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 18 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001590
spark-4440:25264:25264 [0] NCCL INFO Channel 02/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 02/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 19 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001608
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 19 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001608
spark-4440:25264:25264 [0] NCCL INFO Channel 03/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 03/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 20 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001680
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 20 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001680
spark-4440:25264:25264 [0] NCCL INFO Channel 04/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 04/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 21 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600016f8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 21 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40016f8
spark-4440:25264:25264 [0] NCCL INFO Channel 05/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 05/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 22 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001770
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 22 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001770
spark-4440:25264:25264 [0] NCCL INFO Channel 06/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 06/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 23 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600017e8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 23 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40017e8
spark-4440:25264:25264 [0] NCCL INFO Channel 07/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 07/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 24 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001860
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 24 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001860
spark-4440:25264:25264 [0] NCCL INFO Channel 08/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 08/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 25 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600018d8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 25 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40018d8
spark-4440:25264:25264 [0] NCCL INFO Channel 09/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 09/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 26 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001950
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 26 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001950
spark-4440:25264:25264 [0] NCCL INFO Channel 10/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 10/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 27 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff82600019c8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 27 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f40019c8
spark-4440:25264:25264 [0] NCCL INFO Channel 11/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 11/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 28 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001a40
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 28 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001a40
spark-4440:25264:25264 [0] NCCL INFO Channel 12/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 12/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 29 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001ab8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 29 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001ab8
spark-4440:25264:25264 [0] NCCL INFO Channel 13/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 13/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 30 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001b30
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 30 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001b30
spark-4440:25264:25264 [0] NCCL INFO Channel 14/0 : 1[0] -> 0[0] [send] via NET/IB/0
spark-fb97:23940:23940 [0] NCCL INFO Channel 14/0 : 0[0] -> 1[0] [send] via NET/IB/0
spark-4440:25264:25272 [0] NCCL INFO New proxy send connection 31 from local rank 0, transport 2
spark-4440:25264:25264 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xff8260001ba8
spark-fb97:23940:23949 [0] NCCL INFO New proxy send connection 31 from local rank 0, transport 2
spark-fb97:23940:23940 [0] NCCL INFO Connected to proxy localRank 0 -> connection 0xe683f4001ba8
spark-4440:25264:25264 [0] NCCL INFO Channel 15/0 : 1[0] -> 0[0] [send] via NET/IB/1
spark-fb97:23940:23940 [0] NCCL INFO Channel 15/0 : 0[0] -> 1[0] [send] via NET/IB/1
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 345 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17f3a3 fifoLkey=0x17f3a3
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 345 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 345 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x180dc1 fifoLkey=0x180dc1
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 345 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 345 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 476 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x199076 fifoLkey=0x199076
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 476 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 345 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 484 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19b596 fifoLkey=0x19b596
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 484 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 476 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 484 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 349 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x181ecd fifoLkey=0x181ecd
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 349 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 503 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19c8b4 fifoLkey=0x19c8b4
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 503 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 349 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 350 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x1828d5 fifoLkey=0x1828d5
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 350 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 504 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19d5c2 fifoLkey=0x19d5c2
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 504 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 351 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x182bdc fifoLkey=0x182bdc
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 351 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 505 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19e1cf fifoLkey=0x19e1cf
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 505 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 494 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x19b19b fifoLkey=0x19b19b
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 494 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 353 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x1806ae fifoLkey=0x1806ae
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 353 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 506 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19e9d9 fifoLkey=0x19e9d9
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 506 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 354 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x1810b9 fifoLkey=0x1810b9
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 354 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 507 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19c1aa fifoLkey=0x19c1aa
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 507 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 355 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x180bb3 fifoLkey=0x180bb3
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 355 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 351 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17faa2 fifoLkey=0x17faa2
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 351 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 357 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x17b660 fifoLkey=0x17b660
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 357 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 497 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x19a190 fifoLkey=0x19a190
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 497 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 503 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 354 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17eb9a fifoLkey=0x17eb9a
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 354 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 350 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 500 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x199786 fifoLkey=0x199786
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 500 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 358 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17e192 fifoLkey=0x17e192
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 358 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 504 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 503 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x198d7d fifoLkey=0x198d7d
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 503 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 363 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17d986 fifoLkey=0x17d986
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 363 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 351 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 507 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x198571 fifoLkey=0x198571
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 507 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 505 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 366 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17cd7d fifoLkey=0x17cd7d
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 366 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 494 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 353 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 506 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 351 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 354 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 497 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 507 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 355 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 354 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 520 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x18f9e8 fifoLkey=0x18f9e8
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 520 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 357 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 500 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 358 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 503 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 363 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 507 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 366 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 511 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x197769 fifoLkey=0x197769
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 511 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 520 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 371 mtu 3 GID 3 (0/101000AFFFF0000) fifoRkey=0x17c574 fifoLkey=0x17c574
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 371 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 515 mtu 3 GID 0 (80FE/9CFB2FFEFF47BB4E) fifoRkey=0x19785d fifoLkey=0x19785d
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 515 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 0 IbDev 0 Port 1 qpn 371 mtu 3 GID 3 (0/201000AFFFF0000) fifoRkey=0x1832e1 fifoLkey=0x1832e1
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 371 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 511 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: NCCL Dev 1 IbDev 1 Port 1 qpn 523 mtu 3 GID 0 (80FE/45442FFEFF47BB4E) fifoRkey=0x19ead0 fifoLkey=0x19ead0
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 523 query_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 371 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23949 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 515 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 0 Port 1 qpn 371 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-4440:25264:25272 [0] NCCL INFO NET/IB: IbDev 1 Port 1 qpn 523 set_ece={supported=1, vendor_id=0x15b3, options=0x30000002, comp_mask=0x0}
spark-fb97:23940:23940 [0] NCCL INFO Connected all rings, use ring PXN 0 GDR 0
spark-4440:25264:25264 [0] NCCL INFO Connected all rings, use ring PXN 0 GDR 0
  1073741824     134217728     float    none      -1   190308    5.64    2.82       0   189469    5.67    2.83       0
  2147483648     268435456     float    none      -1   379952    5.65    2.83       0   378491    5.67    2.84       0
ssh: connect to host 192.168.100.16 port 22: Connection timed out
ssh: connect to host 192.168.100.15 port 22: Connection timed out
  4294967296     536870912     float    none      -1   758966    5.66    2.83       0   755246    5.69    2.84       0
spark-fb97:23940:23940 [0] NCCL INFO comm 0xc1abd6f01570 rank 0 nranks 2 cudaDev 0 busId f01000 - Destroy COMPLETE
spark-4440:25264:25264 [0] NCCL INFO comm 0xad6cd7e86360 rank 1 nranks 2 cudaDev 0 busId f01000 - Destroy COMPLETE
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 2.83174 
#
# Collective test concluded: all_gather_perf
#

spark-4440:25264:25264 [0] NCCL INFO ENV/Plugin: Closing env plugin ncclEnvDefault
spark-fb97:23940:23940 [0] NCCL INFO ENV/Plugin: Closing env plugin ncclEnvDefault
mprg@spark-fb97:~/nccl-tests$ 
```

