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
現状判明していることは以下の通り．

- DGX Sparkを再起動した直後は，QSFPスイッチやQSFPケーブル直接接続のどちらでも妥当な通信速度を達成する（20GBps〜22GBps程度）．
- QSFP接続の方式を変更すると，nccl-testの通信速度が低下する．約2.8Gbps〜3.2GBps程度で，RJ45の1.25GBpsよりは速いが，QSFPの理論値25GBpsよりも限りなく低速になる．
- QSFP接続方式を再起動時に戻しても，通信速度は改善しない．
- DGX Spark自体を再起動すると通信速度が高速に戻る




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
1.2のQSFPケーブル直結への切り替えで，nccl-testの通信速度が低下していることがわかる．
具体的には，`Avg bus bandwidth : 21.4033`が`Avg bus bandwidth : 2.82151`まで低下する．
またこの時，Active FEC encodingが`RS`から`None`へと変化していることも確認できる．


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
FECをRS-FECに変更しても速度が変わらないため，FECは根本原因ではない可能性が高い．
次の可能性として，RDMAを考える．
RDMAは，CPUやOSを介さずにメモリ間で直接データをやりとりする技術のことである．
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

上記の事実から考えられる仮説
- 通信速度は低下しても，2.8GBps〜3.2GBps程度は出ていることから，RJ45（理論値1.25GBps）で通信してしまっている可能性は低い
- 通信速度の低下の直接的なトリガーは，QSFPケーブル差し替えによる何かしらが原因
- RJ45での通信ではなく，それでいて22GBpsの速度が出ない原因は，通信以外の余計な処理が挟まっている可能性
- 例えば，RoCEではなくTCPソケット通信になってCPU周りで通信データのコピーが発生して通信速度のボトルネックとなっている



### 1.9：TCPソケット通信になっている可能性の調査
<details>
<summary>再起動直後のQSFPスイッチ経由通信でのnccl-test</summary>

<pre><code>
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
</code></pre>
</details>











<details>
<summary>QSFP直結に切り替えてnccl-testを実行</summary>

<pre><code>
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
</code></pre>
</details>



### 1.10：結果の解釈
1.9の結果を見る限り，TCPソケット通信をしているわけではなく，RDMA通信を行なっている．

低速化後のログでも，NCCLはRoCEデバイスを認識している．
```
spark-fb97:23940:23940 [0] NCCL INFO NET/IB: [0] rocep1s0f0:uverbs0:1/RoCE provider=Mlx5 speed=200000 context=0xc1abd84b7eb0 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-4440:25264:25264 [0] NCCL INFO NET/IB: [0] rocep1s0f0:uverbs0:1/RoCE provider=Mlx5 speed=200000 context=0xad6cd9442240 pciPath=/sys/devices/pci0000:00/0000:00:00.0/0000:01:00.0 ar=0
spark-fb97:23940:23940 [0] NCCL INFO NET/IB: [2] roceP2p1s0f0:uverbs2:1/RoCE provider=Mlx5 speed=200000 context=0xc1abd84f95b0 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
spark-4440:25264:25264 [0] NCCL INFO NET/IB: [2] roceP2p1s0f0:uverbs2:1/RoCE provider=Mlx5 speed=200000 context=0xad6cd9483940 pciPath=/sys/devices/pci0002:00/0002:00:00.0/0002:01:00.0 ar=0
```
また，低速化後でも`NET/IB`トランスポートを使用している．
```
spark-fb97:23940:23940 [0] NCCL INFO NET/IB : Using [0]rocep1s0f0:1/RoCE [1]roceP2p1s0f0:1/RoCE [RO]; OOB enp1s0f0np0:10.0.1.1<0>
spark-fb97:23940:23940 [0] NCCL INFO Using network IB
spark-4440:25264:25264 [0] NCCL INFO NET/IB : Using [0]rocep1s0f0:1/RoCE [1]roceP2p1s0f0:1/RoCE [RO]; OOB enp1s0f0np0:10.0.1.2<0>
spark-4440:25264:25264 [0] NCCL INFO Using network IB
```
上記の結果から，RoCE / RDMA系transportを使っているにもかかわらず，速度が 20〜22 GB/s ではなく 2.8 GB/s 程度に落ちていることがわかる．


### 1.11：現状確認と通信速度低下の原因に対する仮説
QSFPケーブルの接続変更前後で，RDMA通信を行なっているのは1.9と1.10で確認した．
このことから，TCPソケット通信になってCPU周りでの処理がボトルネックになっているわけではないことは証明できている．
RoCE/RDMA系transportを使用しているのに，通信速度が低下するということは，「RoCEが何らかの理由で本来の性能を発揮できなくなった」状態へと，QSFPケーブルの抜き差しで固定されてしまったと考えられる．
それでは，どのような状態ならば，RoCEで速度が低下するのか．
どういった現象が発生すれば，RoCEで速度が低下するのか．

考えるべきは，QSFPケーブルの抜き差し前後でどのような挙動がDGX Spark内部で発生しているのか．

### 1.12：QSFPケーブル抜き差し前後でのDGX Sparkの内部挙動
```
mprg@spark-fb97:~/nccl-tests$ sudo dmesg -w
[sudo] mprg のパスワード: 
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd871]
[    0.000000] Linux version 6.17.0-1018-nvidia (buildd@bos03-arm64-016) (aarch64-linux-gnu-gcc-13 (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #18-Ubuntu SMP PREEMPT_DYNAMIC Tue May  5 21:28:33 UTC 2026 (Ubuntu 6.17.0-1018.18-nvidia 6.17.13)
[    0.000000] KASLR enabled
[    0.000000] earlycon: uart0 at MMIO32 0x0000000016a00000 (options '')
[    0.000000] printk: legacy bootconsole [uart0] enabled
[    0.000000] efi: EFI v2.9 by American Megatrends
[    0.000000] efi: RTPROP=0x86f4fd18 ACPI 2.0=0x8707e018 SMBIOS=0x86dd0000 SMBIOS 3.0=0x86dc0000 TPMFinalLog=0x86fe0000 ESRT=0x855a4198 MOKvar=0x86d10000 INITRD=0x853a8a18 RNG=0x87065d18 MEMRESERVE=0x853a8598 
[    0.000000] random: crng init done
[    0.000000] secureboot: Secure boot disabled
[    0.000000] esrt: Reserving ESRT space from 0x00000000855a4198 to 0x00000000855a4220.
[    0.000000] ACPI: Early table checksum verification disabled
[    0.000000] ACPI: RSDP 0x000000008707E018 000024 (v02 ALASKA)
[    0.000000] ACPI: XSDT 0x000000008707E098 0000AC (v01 ALASKA A M I    01072009 AMI  01000013)
[    0.000000] ACPI: FACP 0x000000008707EB18 000114 (v06 ALASKA A M I    01072009 AMI  00010013)
[    0.000000] ACPI: DSDT 0x000000008706C018 011717 (v02 ALASKA A M I    01072009 INTL 20230628)
[    0.000000] ACPI: FIDT 0x000000008707EF18 00009C (v01 ALASKA A M I    01072009 AMI  00010013)
[    0.000000] ACPI: SSDT 0x000000008706A018 001A08 (v02 MTKINC ANGUS    00000001 INTL 20230628)
[    0.000000] ACPI: SSDT 0x0000000087069018 000305 (v02 MTKINC ANGUSCPU 000000FF INTL 20230628)
[    0.000000] ACPI: CSRT 0x0000000087069E98 000068 (v00 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: DBG2 0x000000008707EC98 000175 (v00 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: GTDT 0x0000000087069418 0001D8 (v03 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: APIC 0x0000000087067018 0006A8 (v05 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: MCFG 0x0000000087067E98 00012C (v01 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: PPTT 0x0000000087066018 0008B8 (v02 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: SPCR 0x0000000087069F98 000050 (v02 OVMF   OVMFEDK2 20130221 OVMF 00000099)
[    0.000000] ACPI: SDEV 0x0000000087066E98 000024 (v01 NVIDIA NVSDEV   00000000 NVDA 00000001)
[    0.000000] ACPI: SSDT 0x0000000087065018 000793 (v02 HPQOEM 89EA     00000001 INTL 20230628)
[    0.000000] ACPI: SSDT 0x0000000087065A98 000171 (v01 NVIDIA NBCI     00000001 INTL 20230628)
[    0.000000] ACPI: IORT 0x0000000087068018 000F40 (v06 MEDTEK MTKSGI   00000000 MTK  00000001)
[    0.000000] ACPI: FPDT 0x0000000087066F18 000044 (v01 ALASKA A M I    01072009 AMI  01000013)
[    0.000000] ACPI: WSMT 0x0000000087066F98 000028 (v01 ALASKA A M I    01072009 AMI  00010013)
[    0.000000] ACPI: SPCR: console: uart,mmio32,0x16a00000,115200
[    0.000000] ACPI: Use ACPI SPCR as default console: Yes
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x000000207fffffff]
[    0.000000] NODE_DATA(0) allocated [mem 0x2069677440-0x206967cfff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000080000000-0x00000000ffffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   [mem 0x0000000100000000-0x000000207fffffff]
[    0.000000]   Device   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x000000008007ffff]
[    0.000000]   node   0: [mem 0x0000000080080000-0x0000000083d5ffff]
[    0.000000]   node   0: [mem 0x0000000083d60000-0x0000000083d6ffff]
[    0.000000]   node   0: [mem 0x0000000083d70000-0x000000008655ffff]
[    0.000000]   node   0: [mem 0x0000000086560000-0x000000008701ffff]
[    0.000000]   node   0: [mem 0x0000000087020000-0x000000008707ffff]
[    0.000000]   node   0: [mem 0x0000000087080000-0x0000000088100fff]
[    0.000000]   node   0: [mem 0x0000000088200000-0x000000008a0fffff]
[    0.000000]   node   0: [mem 0x000000008b080000-0x000000008e07ffff]
[    0.000000]   node   0: [mem 0x0000000090000000-0x00000000933dffff]
[    0.000000]   node   0: [mem 0x0000000093400000-0x00000000afffffff]
[    0.000000]   node   0: [mem 0x00000000b9600000-0x00000000b97fffff]
[    0.000000]   node   0: [mem 0x00000000c2e00000-0x00000000d59fffff]
[    0.000000]   node   0: [mem 0x00000000d5a00000-0x00000000ddffffff]
[    0.000000]   node   0: [mem 0x00000000e0000000-0x00000000e001ffff]
[    0.000000]   node   0: [mem 0x00000000e0020000-0x00000000f9ffffff]
[    0.000000]   node   0: [mem 0x00000000fa000000-0x00000000fa001fff]
[    0.000000]   node   0: [mem 0x00000000fa002000-0x00000000ffcfffff]
[    0.000000]   node   0: [mem 0x00000000ffd00000-0x00000000ffd1ffff]
[    0.000000]   node   0: [mem 0x00000000ffd20000-0x000000027fffffff]
[    0.000000]   node   0: [mem 0x0000000280000000-0x00000003237fffff]
[    0.000000]   node   0: [mem 0x0000000323800000-0x000000207df6ffff]
[    0.000000]   node   0: [mem 0x000000207df70000-0x000000207fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000207fffffff]
[    0.000000] On node 0, zone DMA: 255 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 3968 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 8064 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 32 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 5632 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 38400 pages in unavailable ranges
[    0.000000] On node 0, zone DMA: 8192 pages in unavailable ranges
[    0.000000] cma: Reserved 128 MiB at 0x00000000f2000000
[    0.000000] crashkernel size resulted in zero bytes
[    0.000000] psci: probing for conduit method from ACPI.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.5
[    0.000000] percpu: Embedded 57 pages/cpu s108056 r8192 d117224 u233472
[    0.000000] pcpu-alloc: s108056 r8192 d117224 u233472 alloc=57*4096
[    0.000000] pcpu-alloc: [0] 00 [0] 01 [0] 02 [0] 03 [0] 04 [0] 05 [0] 06 [0] 07 
[    0.000000] pcpu-alloc: [0] 08 [0] 09 [0] 10 [0] 11 [0] 12 [0] 13 [0] 14 [0] 15 
[    0.000000] pcpu-alloc: [0] 16 [0] 17 [0] 18 [0] 19 
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: Address authentication (architected QARMA3 algorithm)
[    0.000000] CPU features: detected: GICv3 CPU interface
[    0.000000] CPU features: detected: HCRX_EL2 register
[    0.000000] CPU features: detected: Virtualization Host Extensions
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] CPU features: detected: Spectre-BHB
[    0.000000] CPU features: detected: SSBS not fully self-synchronizing
[    0.000000] alternatives: applying boot alternatives
[    0.000000] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-6.17.0-1018-nvidia root=UUID=a151b0ce-dde9-400c-b84e-ae7b9859c406 ro init_on_alloc=0 iommu.passthrough=0 console=tty0 plymouth.ignore-serial-consoles plymouth.use-simpledrm earlycon=uart,mmio32,0x16A00000 console=tty0 console=ttyS0,921600 crashkernel=1G-:0M quiet splash initcall_blacklist=tegra234_cbb_init pci=pcie_bus_safe vt.handoff=7
[    0.000000] blacklisting initcall tegra234_cbb_init
[    0.000000] Unknown kernel command line parameters "splash", will be passed to user space.
[    0.000000] printk: log buffer data + meta data: 262144 + 917504 = 1179648 bytes
[    0.000000] Dentry cache hash table entries: 8388608 (order: 14, 67108864 bytes, linear)
[    0.000000] Inode-cache hash table entries: 4194304 (order: 13, 33554432 bytes, linear)
[    0.000000] software IO TLB: area num 32.
[    0.000000] software IO TLB: mapped [mem 0x00000000fbd00000-0x00000000ffd00000] (64MB)
[    0.000000] Fallback order for Node 0: 0 
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 33457121
[    0.000000] Policy zone: Normal
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=20, Nodes=1
[    0.000000] ftrace: allocating 69220 entries in 272 pages
[    0.000000] ftrace: allocated 272 pages with 2 groups
[    0.000000] Dynamic Preempt: none
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=512 to nr_cpu_ids=20.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] 	Rude variant of Tasks RCU enabled.
[    0.000000] 	Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 100 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=20
[    0.000000] RCU Tasks: Setting shift to 5 and lim to 1 rcu_task_cb_adjust=1 rcu_task_cpu_ids=20.
[    0.000000] RCU Tasks Rude: Setting shift to 5 and lim to 1 rcu_task_cb_adjust=1 rcu_task_cpu_ids=20.
[    0.000000] RCU Tasks Trace: Setting shift to 5 and lim to 1 rcu_task_cb_adjust=1 rcu_task_cpu_ids=20.
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: GIC: Using split EOI/Deactivate mode
[    0.000000] GICv3: 960 SPIs implemented
[    0.000000] GICv3: 672 Extended SPIs implemented
[    0.000000] Root IRQ handler: gic_handle_irq
[    0.000000] GICv3: GICv3 features: 16 PPIs, DirectLPI
[    0.000000] GICv3: GICv4 features: DirectLPI RVPEID Valid+Dirty 
[    0.000000] GICv3: GICD_CTLR.DS=0, SCR_EL3.FIQ=1
[    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x0000000006880000
[    0.000000] ITS [mem 0x06840000-0x0685ffff]
[    0.000000] ITS@0x0000000006840000: Single VMOVP capable
[    0.000000] ITS@0x0000000006840000: Using GICv4.1 mode 00000000 00000001
[    0.000000] ITS@0x0000000006840000: allocated 8192 Devices @100330000 (indirect, esz 8, psz 64K, shr 1)
[    0.000000] ITS@0x0000000006840000: allocated 32768 Interrupt Collections @100340000 (flat, esz 2, psz 64K, shr 1)
[    0.000000] ITS@0x0000000006840000: allocated 8192 Virtual CPUs @100350000 (flat, esz 8, psz 64K, shr 1)
[    0.000000] GICv3: using LPI property table @0x0000000100360000
[    0.000000] ITS: Using DirectLPI for VPE invalidation
[    0.000000] ITS: Enabling GICv4 support
[    0.000000] GICv3: CPU0: using allocated LPI pending table @0x0000000100380000
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] ACPI GTDT: found 1 memory-mapped timer block(s).
[    0.000000] arch_timer: cp15 and mmio timer(s) running at 1000.00MHz (phys/virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0x1fffffffffffffff max_cycles: 0x1cd42e4dffb, max_idle_ns: 881590591483 ns
[    0.000000] sched_clock: 61 bits at 1000MHz, resolution 1ns, wraps every 4398046511103ns
[    0.000208] Console: colour dummy device 80x25
[    0.000217] printk: legacy console [tty0] enabled
[    0.000339] ACPI: Core revision 20250404
[    0.000616] Calibrating delay loop (skipped), value calculated using timer frequency.. 2000.00 BogoMIPS (lpj=1000000)
[    0.000624] pid_max: default: 32768 minimum: 301
[    0.000723] LSM: initializing lsm=lockdown,capability,landlock,yama,apparmor,ima,evm
[    0.000766] landlock: Up and running.
[    0.000769] Yama: becoming mindful.
[    0.000831] AppArmor: AppArmor initialized
[    0.000986] Mount-cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.001029] Mountpoint-cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.002199] rcu: Hierarchical SRCU implementation.
[    0.002204] rcu: 	Max phase no-delay instances is 400.
[    0.002307] Timer migration: 2 hierarchy levels; 8 children per group; 2 crossnode level
[    0.002813] fsl-mc MSI: ITS@0x6840000 domain created
[    0.002852] Remapping and enabling EFI services.
[    0.003361] smp: Bringing up secondary CPUs ...
[    0.003709] Detected PIPT I-cache on CPU1
[    0.003737] GICv3: CPU1: found redistributor 100 region 0:0x00000000068c0000
[    0.003749] GICv3: CPU1: using allocated LPI pending table @0x0000000100390000
[    0.003773] CPU1: Booted secondary processor 0x0000000100 [0x410fd871]
[    0.004243] Detected PIPT I-cache on CPU2
[    0.004274] GICv3: CPU2: found redistributor 200 region 0:0x0000000006900000
[    0.004286] GICv3: CPU2: using allocated LPI pending table @0x00000001003a0000
[    0.004313] CPU2: Booted secondary processor 0x0000000200 [0x410fd871]
[    0.004782] Detected PIPT I-cache on CPU3
[    0.004815] GICv3: CPU3: found redistributor 300 region 0:0x0000000006940000
[    0.004827] GICv3: CPU3: using allocated LPI pending table @0x00000001003b0000
[    0.004854] CPU3: Booted secondary processor 0x0000000300 [0x410fd871]
[    0.005320] Detected PIPT I-cache on CPU4
[    0.005357] GICv3: CPU4: found redistributor 400 region 0:0x0000000006980000
[    0.005369] GICv3: CPU4: using allocated LPI pending table @0x00000001003c0000
[    0.005397] CPU4: Booted secondary processor 0x0000000400 [0x410fd871]
[    0.005939] Detected PIPT I-cache on CPU5
[    0.005967] GICv3: CPU5: found redistributor 500 region 0:0x00000000069c0000
[    0.005977] GICv3: CPU5: using allocated LPI pending table @0x00000001003d0000
[    0.005998] CPU5: Booted secondary processor 0x0000000500 [0x410fd851]
[    0.006467] Detected PIPT I-cache on CPU6
[    0.006497] GICv3: CPU6: found redistributor 600 region 0:0x0000000006a00000
[    0.006506] GICv3: CPU6: using allocated LPI pending table @0x00000001003e0000
[    0.006527] CPU6: Booted secondary processor 0x0000000600 [0x410fd851]
[    0.006997] Detected PIPT I-cache on CPU7
[    0.007028] GICv3: CPU7: found redistributor 700 region 0:0x0000000006a40000
[    0.007037] GICv3: CPU7: using allocated LPI pending table @0x00000001003f0000
[    0.007059] CPU7: Booted secondary processor 0x0000000700 [0x410fd851]
[    0.007548] Detected PIPT I-cache on CPU8
[    0.007582] GICv3: CPU8: found redistributor 800 region 0:0x0000000006a80000
[    0.007592] GICv3: CPU8: using allocated LPI pending table @0x0000000100400000
[    0.007614] CPU8: Booted secondary processor 0x0000000800 [0x410fd851]
[    0.008093] Detected PIPT I-cache on CPU9
[    0.008128] GICv3: CPU9: found redistributor 900 region 0:0x0000000006ac0000
[    0.008137] GICv3: CPU9: using allocated LPI pending table @0x0000000100410000
[    0.008159] CPU9: Booted secondary processor 0x0000000900 [0x410fd851]
[    0.008856] Detected PIPT I-cache on CPU10
[    0.008912] GICv3: CPU10: found redistributor 10000 region 0:0x0000000006b00000
[    0.008930] GICv3: CPU10: using allocated LPI pending table @0x0000000100420000
[    0.008968] CPU10: Booted secondary processor 0x0000010000 [0x410fd871]
[    0.009512] Detected PIPT I-cache on CPU11
[    0.009551] GICv3: CPU11: found redistributor 10100 region 0:0x0000000006b40000
[    0.009567] GICv3: CPU11: using allocated LPI pending table @0x0000000100430000
[    0.009595] CPU11: Booted secondary processor 0x0000010100 [0x410fd871]
[    0.010090] Detected PIPT I-cache on CPU12
[    0.010134] GICv3: CPU12: found redistributor 10200 region 0:0x0000000006b80000
[    0.010150] GICv3: CPU12: using allocated LPI pending table @0x0000000100440000
[    0.010180] CPU12: Booted secondary processor 0x0000010200 [0x410fd871]
[    0.010677] Detected PIPT I-cache on CPU13
[    0.010721] GICv3: CPU13: found redistributor 10300 region 0:0x0000000006bc0000
[    0.010737] GICv3: CPU13: using allocated LPI pending table @0x0000000100450000
[    0.010768] CPU13: Booted secondary processor 0x0000010300 [0x410fd871]
[    0.011279] Detected PIPT I-cache on CPU14
[    0.011327] GICv3: CPU14: found redistributor 10400 region 0:0x0000000006c00000
[    0.011344] GICv3: CPU14: using allocated LPI pending table @0x0000000100460000
[    0.011375] CPU14: Booted secondary processor 0x0000010400 [0x410fd871]
[    0.011899] Detected PIPT I-cache on CPU15
[    0.011938] GICv3: CPU15: found redistributor 10500 region 0:0x0000000006c40000
[    0.011952] GICv3: CPU15: using allocated LPI pending table @0x0000000100470000
[    0.011978] CPU15: Booted secondary processor 0x0000010500 [0x410fd851]
[    0.012473] Detected PIPT I-cache on CPU16
[    0.012515] GICv3: CPU16: found redistributor 10600 region 0:0x0000000006c80000
[    0.012528] GICv3: CPU16: using allocated LPI pending table @0x0000000100480000
[    0.012554] CPU16: Booted secondary processor 0x0000010600 [0x410fd851]
[    0.013130] Detected PIPT I-cache on CPU17
[    0.013172] GICv3: CPU17: found redistributor 10700 region 0:0x0000000006cc0000
[    0.013185] GICv3: CPU17: using allocated LPI pending table @0x0000000100490000
[    0.013212] CPU17: Booted secondary processor 0x0000010700 [0x410fd851]
[    0.013712] Detected PIPT I-cache on CPU18
[    0.013756] GICv3: CPU18: found redistributor 10800 region 0:0x0000000006d00000
[    0.013769] GICv3: CPU18: using allocated LPI pending table @0x00000001004a0000
[    0.013793] CPU18: Booted secondary processor 0x0000010800 [0x410fd851]
[    0.014312] Detected PIPT I-cache on CPU19
[    0.014357] GICv3: CPU19: found redistributor 10900 region 0:0x0000000006d40000
[    0.014371] GICv3: CPU19: using allocated LPI pending table @0x00000001004b0000
[    0.014395] CPU19: Booted secondary processor 0x0000010900 [0x410fd851]
[    0.014557] smp: Brought up 1 node, 20 CPUs
[    0.014605] SMP: Total of 20 processors activated.
[    0.014608] CPU: All CPU(s) started at EL2
[    0.014613] CPU features: detected: Branch Target Identification
[    0.014617] CPU features: detected: ARMv8.4 Translation Table Level
[    0.014620] CPU features: detected: Data cache clean to the PoU not required for I/D coherence
[    0.014623] CPU features: detected: Common not Private translations
[    0.014626] CPU features: detected: CRC32 instructions
[    0.014629] CPU features: detected: Data cache clean to Point of Deep Persistence
[    0.014631] CPU features: detected: Data cache clean to Point of Persistence
[    0.014634] CPU features: detected: Data independent timing control (DIT)
[    0.014637] CPU features: detected: E0PD
[    0.014639] CPU features: detected: Enhanced Counter Virtualization
[    0.014641] CPU features: detected: Enhanced Counter Virtualization (CNTPOFF)
[    0.014644] CPU features: detected: Enhanced Privileged Access Never
[    0.014646] CPU features: detected: Enhanced Virtualization Traps
[    0.014649] CPU features: detected: Fine Grained Traps
[    0.014652] CPU features: detected: Generic authentication (architected QARMA3 algorithm)
[    0.014656] CPU features: detected: RCpc load-acquire (LDAPR)
[    0.014659] CPU features: detected: LSE atomic instructions
[    0.014661] CPU features: detected: Privileged Access Never
[    0.014664] CPU features: detected: PMUv3
[    0.014666] CPU features: detected: RAS Extension Support
[    0.014669] CPU features: detected: RASv1p1 Extension Support
[    0.014671] CPU features: detected: Speculation barrier (SB)
[    0.014673] CPU features: detected: Stage-2 Force Write-Back
[    0.014676] CPU features: detected: TLB range maintenance instructions
[    0.014678] CPU features: detected: WFx with timeout
[    0.014681] CPU features: detected: Memory Partitioning And Monitoring
[    0.014683] CPU features: detected: Memory Partitioning And Monitoring Virtualisation
[    0.014687] CPU features: detected: Speculative Store Bypassing Safe (SSBS)
[    0.014689] CPU features: detected: Scalable Vector Extension
[    0.014840] alternatives: applying system-wide alternatives
[    0.016455] CPU features: detected: Activity Monitors Unit (AMU) on CPU0-19
[    0.016468] CPU features: detected: Hardware dirty bit management on CPU0-19
[    0.016473] SVE: maximum available vector length 16 bytes per vector
[    0.016477] SVE: default vector length 16 bytes per vector
[    0.017129] Memory: 127348184K/133828484K available (26048K kernel code, 6390K rwdata, 19336K rodata, 14784K init, 1192K bss, 6324684K reserved, 131072K cma-reserved)
[    0.018613] devtmpfs: initialized
[    0.037364] initcall tegra234_cbb_init blacklisted
[    0.037856] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
[    0.037871] posixtimers hash table entries: 16384 (order: 6, 262144 bytes, linear)
[    0.037960] futex hash table entries: 8192 (524288 bytes on 1 NUMA nodes, total 512 KiB, linear).
[    0.038239] 2G module region forced by RANDOMIZE_MODULE_REGION_FULL
[    0.038242] 0 pages in range for non-PLT usage
[    0.038243] 507280 pages in range for PLT usage
[    0.038366] pinctrl core: initialized pinctrl subsystem
[    0.038771] SMBIOS 3.3.0 present.
[    0.038776] DMI: NVIDIA NVIDIA_DGX_Spark/P4242, BIOS 5.36_0ACUM018 08/06/2025
[    0.038826] DMI: Memory slots populated: 1/1
[    0.040492] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.041434] DMA: preallocated 16384 KiB GFP_KERNEL pool for atomic allocations
[    0.041947] DMA: preallocated 16384 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.042433] DMA: preallocated 16384 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.042449] audit: initializing netlink subsys (disabled)
[    0.042521] audit: type=2000 audit(0.041:1): state=initialized audit_enabled=0 res=1
[    0.042808] thermal_sys: Registered thermal governor 'fair_share'
[    0.042812] thermal_sys: Registered thermal governor 'bang_bang'
[    0.042815] thermal_sys: Registered thermal governor 'step_wise'
[    0.042817] thermal_sys: Registered thermal governor 'user_space'
[    0.042819] thermal_sys: Registered thermal governor 'power_allocator'
[    0.042854] cpuidle: using governor ladder
[    0.042883] cpuidle: using governor menu
[    0.042989] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.043249] ASID allocator initialised with 65536 entries
[    0.043555] acpiphp: ACPI Hot Plug PCI Controller Driver version: 0.5
[    0.043695] Serial: AMBA PL011 UART driver
[    0.044455] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.044457] HugeTLB: 0 KiB vmemmap can be freed for a 1.00 GiB page
[    0.044460] HugeTLB: registered 32.0 MiB page size, pre-allocated 0 pages
[    0.044461] HugeTLB: 0 KiB vmemmap can be freed for a 32.0 MiB page
[    0.044463] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
[    0.044465] HugeTLB: 0 KiB vmemmap can be freed for a 2.00 MiB page
[    0.044467] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages
[    0.044469] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page
[    0.045525] ACPI: Added _OSI(Module Device)
[    0.045527] ACPI: Added _OSI(Processor Device)
[    0.045529] ACPI: Added _OSI(Processor Aggregator Device)
[    0.056050] ACPI: 5 ACPI AML tables successfully acquired and loaded
[    0.061244] ACPI: USB4 _OSC: OS supports USB3+ DisplayPort+ PCIe+ XDomain+
[    0.061247] ACPI: USB4 _OSC: OS controls USB3+ DisplayPort+ PCIe+ XDomain+
[    0.061770] ACPI: Interpreter enabled
[    0.061772] ACPI: Using GIC for interrupt routing
[    0.062715] ACPI: MCFG table detected, 16 entries
[    0.066437] ACPI: \_SB_.P0RR: New power resource
[    0.066477] ACPI: \_SB_.R0RR: New power resource
[    0.066517] ACPI: \_SB_.P2RR: New power resource
[    0.066561] ACPI: \_SB_.R2RR: New power resource
[    0.066599] ACPI: \_SB_.P4RR: New power resource
[    0.066638] ACPI: \_SB_.R4RR: New power resource
[    0.066678] ACPI: \_SB_.P6RR: New power resource
[    0.066715] ACPI: \_SB_.R6RR: New power resource
[    0.066754] ACPI: \_SB_.P7RR: New power resource
[    0.066791] ACPI: \_SB_.R7RR: New power resource
[    0.066830] ACPI: \_SB_.P8RR: New power resource
[    0.066869] ACPI: \_SB_.R8RR: New power resource
[    0.066907] ACPI: \_SB_.P9RR: New power resource
[    0.066945] ACPI: \_SB_.R9RR: New power resource
[    0.070267] ACPI: \_SB_.USB5.RHUB.PRT2.PWFR: New power resource
[    0.072447] ACPI: \_SB_.PBRR: New power resource
[    0.072486] ACPI: \_SB_.RBRR: New power resource
[    0.072526] ACPI: \_SB_.PCRR: New power resource
[    0.072566] ACPI: \_SB_.RCRR: New power resource
[    0.072605] ACPI: \_SB_.PDRR: New power resource
[    0.072644] ACPI: \_SB_.RDRR: New power resource
[    0.072684] ACPI: \_SB_.PERR: New power resource
[    0.072722] ACPI: \_SB_.RERR: New power resource
[    0.072761] ACPI: \_SB_.PFRR: New power resource
[    0.074864] ACPI: CPU0 has been hot-added
[    0.074909] ACPI: CPU1 has been hot-added
[    0.074951] ACPI: CPU2 has been hot-added
[    0.074991] ACPI: CPU3 has been hot-added
[    0.075035] ACPI: CPU4 has been hot-added
[    0.075076] ACPI: CPU5 has been hot-added
[    0.075117] ACPI: CPU6 has been hot-added
[    0.075156] ACPI: CPU7 has been hot-added
[    0.075198] ACPI: CPU8 has been hot-added
[    0.075238] ACPI: CPU9 has been hot-added
[    0.075286] ACPI: CPU10 has been hot-added
[    0.075327] ACPI: CPU11 has been hot-added
[    0.075369] ACPI: CPU12 has been hot-added
[    0.075410] ACPI: CPU13 has been hot-added
[    0.075450] ACPI: CPU14 has been hot-added
[    0.075490] ACPI: CPU15 has been hot-added
[    0.075531] ACPI: CPU16 has been hot-added
[    0.075579] ACPI: CPU17 has been hot-added
[    0.075619] ACPI: CPU18 has been hot-added
[    0.075659] ACPI: CPU19 has been hot-added
[    0.076658] ACPI: PCI Root Bridge [PCI0] (domain 0000 [bus 00-0f])
[    0.076666] acpi PNP0A08:00: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.076749] acpi PNP0A08:00: _OSC: platform does not support [SHPCHotplug DPC]
[    0.076885] acpi PNP0A08:00: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.077174] acpi PNP0A08:00: ECAM area [mem 0xf300000000-0xf300ffffff] reserved by PNP0C02:00
[    0.077184] acpi PNP0A08:00: ECAM at [mem 0xf300000000-0xf300ffffff] for [bus 00-0f]
[    0.077209] ACPI: Remapped I/O 0x0000000067000000 to [io  0x0000-0xffff window]
[    0.077322] PCI host bridge to bus 0000:00
[    0.077360] pci_bus 0000:00: root bus resource [io  0x0000-0xffff window] (bus address [0x67000000-0x6700ffff])
[    0.077363] pci_bus 0000:00: root bus resource [mem 0x67010000-0x697fffff window]
[    0.077365] pci_bus 0000:00: root bus resource [mem 0x7500000000-0x82ffffffff window]
[    0.077368] pci_bus 0000:00: root bus resource [bus 00-0f]
[    0.077370] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.077399] pci 0000:00:00.0: [10de:22ce] type 01 class 0x060400 PCIe Root Port
[    0.077413] pci 0000:00:00.0: PCI bridge to [bus 01-0f]
[    0.077473] pci 0000:00:00.0: PME# supported from D0 D3hot D3cold
[    0.078046] pci 0000:01:00.0: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[    0.078301] pci 0000:01:00.0: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[    0.078332] pci 0000:01:00.0: ROM [mem 0x00000000-0x000fffff pref]
[    0.079190] pci 0000:01:00.0: PME# supported from D3cold
[    0.079535] pci 0000:01:00.0: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[    0.079538] pci 0000:01:00.0: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[    0.081109] pci 0000:01:00.1: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[    0.081343] pci 0000:01:00.1: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[    0.081373] pci 0000:01:00.1: ROM [mem 0x00000000-0x000fffff pref]
[    0.081984] pci 0000:01:00.1: PME# supported from D3cold
[    0.082318] pci 0000:01:00.1: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[    0.082321] pci 0000:01:00.1: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[    0.083279] pci_bus 0000:00: max bus depth: 1 pci_try_num: 2
[    0.083288] pci 0000:00:00.0: bridge window [mem 0x7500000000-0x7505ffffff 64bit pref]: assigned
[    0.083291] pci 0000:00:00.0: bridge window [mem 0x67100000-0x672fffff]: assigned
[    0.083296] pci 0000:01:00.0: BAR 0 [mem 0x7500000000-0x7501ffffff 64bit pref]: assigned
[    0.083348] pci 0000:01:00.1: BAR 0 [mem 0x7502000000-0x7503ffffff 64bit pref]: assigned
[    0.083399] pci 0000:01:00.0: ROM [mem 0x67100000-0x671fffff pref]: assigned
[    0.083402] pci 0000:01:00.0: VF BAR 0 [mem 0x7504000000-0x75047fffff 64bit pref]: assigned
[    0.083432] pci 0000:01:00.1: ROM [mem 0x67200000-0x672fffff pref]: assigned
[    0.083435] pci 0000:01:00.1: VF BAR 0 [mem 0x7504800000-0x7504ffffff 64bit pref]: assigned
[    0.083456] pci 0000:00:00.0: PCI bridge to [bus 01-0f]
[    0.083468] pci 0000:00:00.0:   bridge window [mem 0x67100000-0x672fffff]
[    0.083472] pci 0000:00:00.0:   bridge window [mem 0x7500000000-0x7505ffffff 64bit pref]
[    0.083475] pci_bus 0000:00: resource 4 [io  0x0000-0xffff window]
[    0.083478] pci_bus 0000:00: resource 5 [mem 0x67010000-0x697fffff window]
[    0.083480] pci_bus 0000:00: resource 6 [mem 0x7500000000-0x82ffffffff window]
[    0.083483] pci_bus 0000:01: resource 1 [mem 0x67100000-0x672fffff]
[    0.083485] pci_bus 0000:01: resource 2 [mem 0x7500000000-0x7505ffffff 64bit pref]
[    0.083490] pci 0000:00:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.083542] pci 0000:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.083593] pci 0000:01:00.1: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.083652] ACPI: PCI Root Bridge [PCI2] (domain 0002 [bus 00-0f])
[    0.083657] acpi PNP0A08:01: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.083735] acpi PNP0A08:01: _OSC: platform does not support [SHPCHotplug DPC]
[    0.083869] acpi PNP0A08:01: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.084153] acpi PNP0A08:01: ECAM area [mem 0xf320000000-0xf320ffffff] reserved by PNP0C02:00
[    0.084160] acpi PNP0A08:01: ECAM at [mem 0xf320000000-0xf320ffffff] for [bus 00-0f]
[    0.084180] ACPI: Remapped I/O 0x000000005d000000 to [io  0x10000-0x1ffff window]
[    0.084275] PCI host bridge to bus 0002:00
[    0.084312] pci_bus 0002:00: root bus resource [io  0x10000-0x1ffff window] (bus address [0x5d000000-0x5d00ffff])
[    0.084315] pci_bus 0002:00: root bus resource [mem 0x5d010000-0x5f7fffff window]
[    0.084317] pci_bus 0002:00: root bus resource [mem 0x3d00000000-0x4affffffff window]
[    0.084320] pci_bus 0002:00: root bus resource [bus 00-0f]
[    0.084322] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.084344] pci 0002:00:00.0: [10de:22ce] type 01 class 0x060400 PCIe Root Port
[    0.084357] pci 0002:00:00.0: PCI bridge to [bus 01-0f]
[    0.084413] pci 0002:00:00.0: PME# supported from D0 D3hot D3cold
[    0.084935] pci 0002:01:00.0: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[    0.085190] pci 0002:01:00.0: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[    0.085221] pci 0002:01:00.0: ROM [mem 0x00000000-0x000fffff pref]
[    0.086099] pci 0002:01:00.0: PME# supported from D3cold
[    0.086453] pci 0002:01:00.0: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[    0.086455] pci 0002:01:00.0: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[    0.088017] pci 0002:01:00.1: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[    0.088260] pci 0002:01:00.1: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[    0.088292] pci 0002:01:00.1: ROM [mem 0x00000000-0x000fffff pref]
[    0.088909] pci 0002:01:00.1: PME# supported from D3cold
[    0.089254] pci 0002:01:00.1: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[    0.089256] pci 0002:01:00.1: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[    0.090230] pci_bus 0002:00: max bus depth: 1 pci_try_num: 2
[    0.090235] pci 0002:00:00.0: bridge window [mem 0x3d00000000-0x3d05ffffff 64bit pref]: assigned
[    0.090238] pci 0002:00:00.0: bridge window [mem 0x5d100000-0x5d2fffff]: assigned
[    0.090241] pci 0002:01:00.0: BAR 0 [mem 0x3d00000000-0x3d01ffffff 64bit pref]: assigned
[    0.090295] pci 0002:01:00.1: BAR 0 [mem 0x3d02000000-0x3d03ffffff 64bit pref]: assigned
[    0.090349] pci 0002:01:00.0: ROM [mem 0x5d100000-0x5d1fffff pref]: assigned
[    0.090351] pci 0002:01:00.0: VF BAR 0 [mem 0x3d04000000-0x3d047fffff 64bit pref]: assigned
[    0.090382] pci 0002:01:00.1: ROM [mem 0x5d200000-0x5d2fffff pref]: assigned
[    0.090385] pci 0002:01:00.1: VF BAR 0 [mem 0x3d04800000-0x3d04ffffff 64bit pref]: assigned
[    0.090406] pci 0002:00:00.0: PCI bridge to [bus 01-0f]
[    0.090419] pci 0002:00:00.0:   bridge window [mem 0x5d100000-0x5d2fffff]
[    0.090421] pci 0002:00:00.0:   bridge window [mem 0x3d00000000-0x3d05ffffff 64bit pref]
[    0.090425] pci_bus 0002:00: resource 4 [io  0x10000-0x1ffff window]
[    0.090427] pci_bus 0002:00: resource 5 [mem 0x5d010000-0x5f7fffff window]
[    0.090430] pci_bus 0002:00: resource 6 [mem 0x3d00000000-0x4affffffff window]
[    0.090432] pci_bus 0002:01: resource 1 [mem 0x5d100000-0x5d2fffff]
[    0.090434] pci_bus 0002:01: resource 2 [mem 0x3d00000000-0x3d05ffffff 64bit pref]
[    0.090438] pci 0002:00:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.090492] pci 0002:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.090544] pci 0002:01:00.1: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[    0.090596] ACPI: PCI Root Bridge [PCI4] (domain 0004 [bus 00-0f])
[    0.090600] acpi PNP0A08:02: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.090677] acpi PNP0A08:02: _OSC: platform does not support [SHPCHotplug DPC]
[    0.090811] acpi PNP0A08:02: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.091093] acpi PNP0A08:02: ECAM area [mem 0xf340000000-0xf340ffffff] reserved by PNP0C02:00
[    0.091099] acpi PNP0A08:02: ECAM at [mem 0xf340000000-0xf340ffffff] for [bus 00-0f]
[    0.091119] ACPI: Remapped I/O 0x0000000062000000 to [io  0x20000-0x2ffff window]
[    0.091212] PCI host bridge to bus 0004:00
[    0.091248] pci_bus 0004:00: root bus resource [io  0x20000-0x2ffff window] (bus address [0x62000000-0x6200ffff])
[    0.091251] pci_bus 0004:00: root bus resource [mem 0x62010000-0x647fffff window]
[    0.091253] pci_bus 0004:00: root bus resource [mem 0x5900000000-0x66ffffffff window]
[    0.091256] pci_bus 0004:00: root bus resource [bus 00-0f]
[    0.091257] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.091279] pci 0004:00:00.0: [10de:22ce] type 01 class 0x060400 PCIe Root Port
[    0.091291] pci 0004:00:00.0: PCI bridge to [bus 01-0f]
[    0.091296] pci 0004:00:00.0:   bridge window [mem 0x62100000-0x621fffff]
[    0.091353] pci 0004:00:00.0: PME# supported from D0 D3hot D3cold
[    0.091707] pci 0004:01:00.0: [144d:a810] type 00 class 0x010802 PCIe Endpoint
[    0.091737] pci 0004:01:00.0: BAR 0 [mem 0x62100000-0x62103fff 64bit]
[    0.094669] pci 0004:00:00.0: bridge window [io  0x1000-0x0fff] to [bus 01-0f] add_size 1000
[    0.094673] pci 0004:00:00.0: bridge window [mem 0x00100000-0x000fffff 64bit pref] to [bus 01-0f] add_size 200000 add_align 100000
[    0.094677] pci 0004:00:00.0: bridge window [mem 0x00100000-0x001fffff] to [bus 01-0f] add_size 100000 add_align 100000
[    0.094681] pci 0004:00:00.0: bridge window [mem 0x62100000-0x622fffff]: assigned
[    0.094684] pci 0004:00:00.0: bridge window [mem 0x5900000000-0x59001fffff 64bit pref]: assigned
[    0.094686] pci 0004:00:00.0: bridge window [io  0x20000-0x20fff]: assigned
[    0.094690] pci 0004:01:00.0: BAR 0 [mem 0x62100000-0x62103fff 64bit]: assigned
[    0.094697] pci 0004:00:00.0: PCI bridge to [bus 01-0f]
[    0.094699] pci 0004:00:00.0:   bridge window [io  0x20000-0x20fff]
[    0.094703] pci 0004:00:00.0:   bridge window [mem 0x62100000-0x622fffff]
[    0.094705] pci 0004:00:00.0:   bridge window [mem 0x5900000000-0x59001fffff 64bit pref]
[    0.094709] pci_bus 0004:00: resource 4 [io  0x20000-0x2ffff window]
[    0.094711] pci_bus 0004:00: resource 5 [mem 0x62010000-0x647fffff window]
[    0.094714] pci_bus 0004:00: resource 6 [mem 0x5900000000-0x66ffffffff window]
[    0.094716] pci_bus 0004:01: resource 0 [io  0x20000-0x20fff]
[    0.094718] pci_bus 0004:01: resource 1 [mem 0x62100000-0x622fffff]
[    0.094720] pci_bus 0004:01: resource 2 [mem 0x5900000000-0x59001fffff 64bit pref]
[    0.094725] pci 0004:00:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  256
[    0.094730] pci 0004:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  256
[    0.094781] ACPI: PCI Root Bridge [PCI6] (domain 0006 [bus 00-0f])
[    0.094784] acpi PNP0A08:03: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.094859] acpi PNP0A08:03: _OSC: platform does not support [SHPCHotplug DPC]
[    0.094992] acpi PNP0A08:03: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.095274] acpi PNP0A08:03: ECAM area [mem 0xf360000000-0xf360ffffff] reserved by PNP0C02:00
[    0.095280] acpi PNP0A08:03: ECAM at [mem 0xf360000000-0xf360ffffff] for [bus 00-0f]
[    0.095299] ACPI: Remapped I/O 0x000000006c000000 to [io  0x30000-0x3ffff window]
[    0.095392] PCI host bridge to bus 0006:00
[    0.095428] pci_bus 0006:00: root bus resource [io  0x30000-0x3ffff window] (bus address [0x6c000000-0x6c00ffff])
[    0.095431] pci_bus 0006:00: root bus resource [mem 0x6c010000-0x6e7fffff window]
[    0.095433] pci_bus 0006:00: root bus resource [mem 0x9100000000-0x9effffffff window]
[    0.095435] pci_bus 0006:00: root bus resource [bus 00-0f]
[    0.095437] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.095464] pci_bus 0006:00: resource 4 [io  0x30000-0x3ffff window]
[    0.095466] pci_bus 0006:00: resource 5 [mem 0x6c010000-0x6e7fffff window]
[    0.095468] pci_bus 0006:00: resource 6 [mem 0x9100000000-0x9effffffff window]
[    0.095507] ACPI: PCI Root Bridge [PCI7] (domain 0007 [bus 00-0f])
[    0.095511] acpi PNP0A08:04: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.095584] acpi PNP0A08:04: _OSC: platform does not support [SHPCHotplug DPC]
[    0.095719] acpi PNP0A08:04: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.096001] acpi PNP0A08:04: ECAM area [mem 0xf370000000-0xf370ffffff] reserved by PNP0C02:00
[    0.096007] acpi PNP0A08:04: ECAM at [mem 0xf370000000-0xf370ffffff] for [bus 00-0f]
[    0.096026] ACPI: Remapped I/O 0x000000006e800000 to [io  0x40000-0x4ffff window]
[    0.096118] PCI host bridge to bus 0007:00
[    0.096154] pci_bus 0007:00: root bus resource [io  0x40000-0x4ffff window] (bus address [0x6e800000-0x6e80ffff])
[    0.096157] pci_bus 0007:00: root bus resource [mem 0x6e810000-0x70ffffff window]
[    0.096159] pci_bus 0007:00: root bus resource [mem 0x9f00000000-0xacffffffff window]
[    0.096161] pci_bus 0007:00: root bus resource [bus 00-0f]
[    0.096163] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.096188] pci 0007:00:00.0: [10de:22d0] type 01 class 0x060400 PCIe Root Port
[    0.096202] pci 0007:00:00.0: PCI bridge to [bus 01-0f]
[    0.096207] pci 0007:00:00.0:   bridge window [io  0x40000-0x40fff]
[    0.096210] pci 0007:00:00.0:   bridge window [mem 0x6e900000-0x6e9fffff]
[    0.096275] pci 0007:00:00.0: PME# supported from D0 D3hot D3cold
[    0.096650] pci 0007:01:00.0: [10ec:8127] type 00 class 0x020000 PCIe Endpoint
[    0.096697] pci 0007:01:00.0: BAR 0 [io  0x40000-0x400ff]
[    0.096702] pci 0007:01:00.0: BAR 2 [mem 0x6e900000-0x6e93ffff 64bit]
[    0.096707] pci 0007:01:00.0: BAR 4 [mem 0x6e940000-0x6e943fff 64bit]
[    0.096822] pci 0007:01:00.0: supports D1 D2
[    0.096825] pci 0007:01:00.0: PME# supported from D0 D1 D2 D3hot D3cold
[    0.099678] pci 0007:00:00.0: bridge window [io  0x1000-0x1fff] to [bus 01-0f] add_size 1000
[    0.099681] pci 0007:00:00.0: bridge window [mem 0x00100000-0x000fffff 64bit pref] to [bus 01-0f] add_size 200000 add_align 100000
[    0.099684] pci 0007:00:00.0: bridge window [mem 0x00100000-0x001fffff] to [bus 01-0f] add_size 100000 add_align 100000
[    0.099688] pci 0007:00:00.0: bridge window [mem 0x6e900000-0x6eafffff]: assigned
[    0.099691] pci 0007:00:00.0: bridge window [mem 0x9f00000000-0x9f001fffff 64bit pref]: assigned
[    0.099693] pci 0007:00:00.0: bridge window [io  0x40000-0x41fff]: assigned
[    0.099697] pci 0007:01:00.0: BAR 2 [mem 0x6e900000-0x6e93ffff 64bit]: assigned
[    0.099707] pci 0007:01:00.0: BAR 4 [mem 0x6e940000-0x6e943fff 64bit]: assigned
[    0.099717] pci 0007:01:00.0: BAR 0 [io  0x40000-0x400ff]: assigned
[    0.099722] pci 0007:00:00.0: PCI bridge to [bus 01-0f]
[    0.099724] pci 0007:00:00.0:   bridge window [io  0x40000-0x41fff]
[    0.099727] pci 0007:00:00.0:   bridge window [mem 0x6e900000-0x6eafffff]
[    0.099730] pci 0007:00:00.0:   bridge window [mem 0x9f00000000-0x9f001fffff 64bit pref]
[    0.099734] pci_bus 0007:00: resource 4 [io  0x40000-0x4ffff window]
[    0.099737] pci_bus 0007:00: resource 5 [mem 0x6e810000-0x70ffffff window]
[    0.099739] pci_bus 0007:00: resource 6 [mem 0x9f00000000-0xacffffffff window]
[    0.099741] pci_bus 0007:01: resource 0 [io  0x40000-0x41fff]
[    0.099743] pci_bus 0007:01: resource 1 [mem 0x6e900000-0x6eafffff]
[    0.099745] pci_bus 0007:01: resource 2 [mem 0x9f00000000-0x9f001fffff 64bit pref]
[    0.099750] pci 0007:00:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  256
[    0.099758] pci 0007:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  256
[    0.099807] ACPI: PCI Root Bridge [PCI8] (domain 0008 [bus 00-0f])
[    0.099810] acpi PNP0A08:05: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.099885] acpi PNP0A08:05: _OSC: platform does not support [SHPCHotplug DPC]
[    0.100017] acpi PNP0A08:05: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.100298] acpi PNP0A08:05: ECAM area [mem 0xf380000000-0xf380ffffff] reserved by PNP0C02:00
[    0.100303] acpi PNP0A08:05: ECAM at [mem 0xf380000000-0xf380ffffff] for [bus 00-0f]
[    0.100322] ACPI: Remapped I/O 0x0000000071000000 to [io  0x50000-0x5ffff window]
[    0.100415] PCI host bridge to bus 0008:00
[    0.100451] pci_bus 0008:00: root bus resource [io  0x50000-0x5ffff window] (bus address [0x71000000-0x7100ffff])
[    0.100453] pci_bus 0008:00: root bus resource [mem 0x71010000-0x737fffff window]
[    0.100456] pci_bus 0008:00: root bus resource [mem 0xad00000000-0xbaffffffff window]
[    0.100458] pci_bus 0008:00: root bus resource [bus 00-0f]
[    0.100460] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.100486] pci_bus 0008:00: resource 4 [io  0x50000-0x5ffff window]
[    0.100489] pci_bus 0008:00: resource 5 [mem 0x71010000-0x737fffff window]
[    0.100491] pci_bus 0008:00: resource 6 [mem 0xad00000000-0xbaffffffff window]
[    0.100530] ACPI: PCI Root Bridge [PCI9] (domain 0009 [bus 00-0f])
[    0.100534] acpi PNP0A08:06: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.100607] acpi PNP0A08:06: _OSC: platform does not support [SHPCHotplug DPC]
[    0.100741] acpi PNP0A08:06: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.101020] acpi PNP0A08:06: ECAM area [mem 0xf390000000-0xf390ffffff] reserved by PNP0C02:00
[    0.101025] acpi PNP0A08:06: ECAM at [mem 0xf390000000-0xf390ffffff] for [bus 00-0f]
[    0.101044] ACPI: Remapped I/O 0x0000000073800000 to [io  0x60000-0x6ffff window]
[    0.101138] PCI host bridge to bus 0009:00
[    0.101174] pci_bus 0009:00: root bus resource [io  0x60000-0x6ffff window] (bus address [0x73800000-0x7380ffff])
[    0.101177] pci_bus 0009:00: root bus resource [mem 0x73810000-0x75ffffff window]
[    0.101179] pci_bus 0009:00: root bus resource [mem 0xbb00000000-0xc8ffffffff window]
[    0.101181] pci_bus 0009:00: root bus resource [bus 00-0f]
[    0.101183] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.101208] pci 0009:00:00.0: [10de:22d0] type 01 class 0x060400 PCIe Root Port
[    0.101224] pci 0009:00:00.0: PCI bridge to [bus 01-0f]
[    0.101229] pci 0009:00:00.0:   bridge window [mem 0x73a00000-0x73cfffff]
[    0.101300] pci 0009:00:00.0: PME# supported from D0 D3hot D3cold
[    0.101700] pci 0009:01:00.0: [14c3:7925] type 00 class 0x028000 PCIe Endpoint
[    0.101760] pci 0009:01:00.0: BAR 0 [mem 0x73a00000-0x73bfffff 64bit]
[    0.101765] pci 0009:01:00.0: BAR 2 [mem 0x73c00000-0x73c07fff 64bit]
[    0.101872] pci 0009:01:00.0: PME# supported from D0 D3hot D3cold
[    0.104688] pci 0009:00:00.0: bridge window [io  0x1000-0x0fff] to [bus 01-0f] add_size 1000
[    0.104691] pci 0009:00:00.0: bridge window [mem 0x00100000-0x000fffff 64bit pref] to [bus 01-0f] add_size 200000 add_align 100000
[    0.104695] pci 0009:00:00.0: bridge window [mem 0x73900000-0x73bfffff]: assigned
[    0.104697] pci 0009:00:00.0: bridge window [mem 0xbb00000000-0xbb001fffff 64bit pref]: assigned
[    0.104700] pci 0009:00:00.0: bridge window [io  0x60000-0x60fff]: assigned
[    0.104703] pci 0009:01:00.0: BAR 0 [mem 0x73a00000-0x73bfffff 64bit]: assigned
[    0.104716] pci 0009:01:00.0: BAR 2 [mem 0x73900000-0x73907fff 64bit]: assigned
[    0.104728] pci 0009:00:00.0: PCI bridge to [bus 01-0f]
[    0.104732] pci 0009:00:00.0:   bridge window [io  0x60000-0x60fff]
[    0.104735] pci 0009:00:00.0:   bridge window [mem 0x73900000-0x73bfffff]
[    0.104738] pci 0009:00:00.0:   bridge window [mem 0xbb00000000-0xbb001fffff 64bit pref]
[    0.104742] pci_bus 0009:00: resource 4 [io  0x60000-0x6ffff window]
[    0.104744] pci_bus 0009:00: resource 5 [mem 0x73810000-0x75ffffff window]
[    0.104746] pci_bus 0009:00: resource 6 [mem 0xbb00000000-0xc8ffffffff window]
[    0.104748] pci_bus 0009:01: resource 0 [io  0x60000-0x60fff]
[    0.104751] pci_bus 0009:01: resource 1 [mem 0x73900000-0x73bfffff]
[    0.104753] pci_bus 0009:01: resource 2 [mem 0xbb00000000-0xbb001fffff 64bit pref]
[    0.104758] pci 0009:00:00.0: Max Payload Size set to  256/ 512 (was  128), Max Read Rq  256
[    0.104767] pci 0009:01:00.0: Max Payload Size set to  256/ 256 (was  128), Max Read Rq  256
[    0.108745] platform NVDA8800:00: failed to claim resource 0: [mem 0x05170000-0x051cffff]
[    0.108750] acpi NVDA8800:00: platform device creation failed: -16
[    0.108821] platform NVDA8900:00: failed to claim resource 0: [mem 0xc8000000-0xd7ffffff]
[    0.108824] acpi NVDA8900:00: platform device creation failed: -16
[    0.109644] ACPI: PCI Root Bridge [PCIF] (domain 000f [bus 00-01])
[    0.109648] acpi PNP0A08:0b: _OSC: OS supports [ExtendedConfig ASPM ClockPM Segments MSI EDR HPX-Type3]
[    0.109720] acpi PNP0A08:0b: _OSC: platform does not support [SHPCHotplug DPC]
[    0.109835] acpi PNP0A08:0b: _OSC: OS now controls [PCIeHotplug PME AER PCIeCapability LTR]
[    0.110690] acpi PNP0A08:0b: ECAM area [mem 0x29000000-0x291fffff] reserved by PNP0C02:01
[    0.110697] acpi PNP0A08:0b: ECAM at [mem 0x29000000-0x291fffff] for [bus 00-01]
[    0.110807] PCI host bridge to bus 000f:00
[    0.110841] pci_bus 000f:00: root bus resource [mem 0x24000000-0x281fffff window]
[    0.110843] pci_bus 000f:00: root bus resource [bus 00-01]
[    0.110845] PCI: OF: of_root node is NULL, cannot create PCI host bridge node
[    0.110890] pci 000f:00:00.0: [10de:22d1] type 01 class 0x060400 PCIe Root Port
[    0.110925] pci 000f:00:00.0: PCI bridge to [bus 01]
[    0.110950] pci 000f:00:00.0:   bridge window [mem 0x24000000-0x27ffffff 64bit pref]
[    0.111107] pci 000f:00:00.0: PME# supported from D0 D3hot
[    0.111583] pci 000f:01:00.0: [10de:2e12] type 00 class 0x030000 PCIe Endpoint
[    0.111649] pci 000f:01:00.0: BAR 0 [mem 0x24000000-0x27ffffff 64bit pref]
[    0.111719] pci 000f:01:00.0: Enabling HDA controller
[    1.123520] pci 000f:01:00.0: DOE: [2c8] ABORT timed out
[    1.123523] pci 000f:01:00.0: DOE: [2c8] failed to reset mailbox with abort command : -5
[    1.123532] pci 000f:01:00.0: DOE: [2c8] failed to create mailbox: -5
[    1.123593] pci 000f:01:00.0: 0.000 Gb/s available PCIe bandwidth, limited by Unknown x0 link at 000f:00:00.0 (capable of 32.000 Gb/s with 2.5 GT/s PCIe x16 link)
[    1.123812] pci 000f:00:00.0: PCI bridge to [bus 01]
[    1.123832] pci 000f:00:00.0: PCI bridge to [bus 01]
[    1.123839] pci 000f:00:00.0:   bridge window [mem 0x24000000-0x27ffffff 64bit pref]
[    1.123846] pci_bus 000f:00: resource 4 [mem 0x24000000-0x281fffff window]
[    1.123849] pci_bus 000f:01: resource 2 [mem 0x24000000-0x27ffffff 64bit pref]
[    1.123858] pci 000f:00:00.0: Max Payload Size set to  256/ 512 (was  128), Max Read Rq  512
[    1.123869] pci 000f:01:00.0: Max Payload Size set to  256/ 256 (was  128), Max Read Rq  512
[    1.124584] iommu: Default domain type: Translated (set via kernel command line)
[    1.124587] iommu: DMA domain TLB invalidation policy: lazy mode
[    1.125831] SCSI subsystem initialized
[    1.125900] libata version 3.00 loaded.
[    1.125937] ACPI: bus type USB registered
[    1.125961] usbcore: registered new interface driver usbfs
[    1.125975] usbcore: registered new interface driver hub
[    1.125990] usbcore: registered new device driver usb
[    1.126064] pps_core: LinuxPPS API ver. 1 registered
[    1.126066] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    1.126072] PTP clock support registered
[    1.126194] EDAC MC: Ver: 3.0.0
[    1.126386] scmi_core: SCMI protocol bus registered
[    1.126559] efivars: Registered efivars operations
[    1.126880] mpam:mpam_msc_driver_init: No MSC devices found in firmware
[    1.127164] NetLabel: Initializing
[    1.127165] NetLabel:  domain hash size = 128
[    1.127167] NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
[    1.127189] NetLabel:  unlabeled traffic allowed by default
[    1.127252] mctp: management component transport protocol core
[    1.127254] NET: Registered PF_MCTP protocol family
[    1.127340] pci 000f:01:00.0: vgaarb: setting as boot VGA device
[    1.127342] pci 000f:01:00.0: vgaarb: bridge control possible
[    1.127344] pci 000f:01:00.0: vgaarb: VGA device added: decodes=io+mem,owns=none,locks=none
[    1.127347] vgaarb: loaded
[    1.127570] clocksource: Switched to clocksource arch_sys_counter
[    1.127836] VFS: Disk quotas dquot_6.6.0
[    1.127848] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    1.129127] AppArmor: AppArmor Filesystem Enabled
[    1.129181] pnp: PnP ACPI init
[    1.129465] system 00:00: [mem 0xf300000000-0xf3ffffffff window] could not be reserved
[    1.129470] system 00:00: [mem 0x1d790000-0x1d790fff] has been reserved
[    1.129473] system 00:00: [mem 0x1d690000-0x1d690fff] has been reserved
[    1.129476] system 00:00: [mem 0x1d600000-0x1d600fff] has been reserved
[    1.129479] system 00:00: [mem 0x1d640000-0x1d640fff] has been reserved
[    1.129481] system 00:00: [mem 0x16bd0000-0x16bd0fff] has been reserved
[    1.129817] system 00:01: [mem 0x29000000-0x291fffff window] could not be reserved
[    1.129838] pnp: PnP ACPI: found 2 devices
[    1.132713] NET: Registered PF_INET protocol family
[    1.132794] IP idents hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    1.157045] tcp_listen_portaddr_hash hash table entries: 65536 (order: 8, 1048576 bytes, linear)
[    1.157717] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    1.157733] TCP established hash table entries: 524288 (order: 10, 4194304 bytes, linear)
[    1.160289] TCP bind hash table entries: 65536 (order: 9, 2097152 bytes, linear)
[    1.161615] TCP: Hash tables configured (established 524288 bind 65536)
[    1.161800] MPTCP token hash table entries: 65536 (order: 9, 1572864 bytes, linear)
[    1.161908] UDP hash table entries: 65536 (order: 10, 4194304 bytes, linear)
[    1.165325] UDP-Lite hash table entries: 65536 (order: 10, 4194304 bytes, linear)
[    1.168821] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    1.168843] NET: Registered PF_XDP protocol family
[    1.168937] PCI: CLS 0 bytes, default 64
[    1.168969] ARM FF-A: Driver version 1.2
[    1.168971] ARM FF-A: Firmware version 1.2 found
[    1.169041] Trying to unpack rootfs image as initramfs...
[    1.173104] kvm [1]: nv: 567 coarse grained trap handlers
[    1.173330] kvm [1]: nv: 664 fine grained trap handlers
[    1.173491] kvm [1]: IPA Size Limit: 40 bits
[    1.173519] kvm [1]: GICv4 support disabled
[    1.173523] kvm [1]: GICv3: no GICV resource entry
[    1.173527] kvm [1]: disabling GICv2 emulation
[    1.173591] kvm [1]: GIC system register CPU interface enabled
[    1.173615] kvm [1]: vgic interrupt IRQ9
[    1.173665] kvm [1]: VHE mode initialized successfully
[    1.235947] Initialise system trusted keyrings
[    1.235982] Key type blacklist registered
[    1.236136] workingset: timestamp_bits=40 max_order=25 bucket_order=0
[    1.236811] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    1.237045] fuse: init (API version 7.44)
[    1.237346] integrity: Platform Keyring initialized
[    1.237361] integrity: Machine keyring initialized
[    1.261670] Key type asymmetric registered
[    1.261677] Asymmetric key parser 'x509' registered
[    1.261722] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 239)
[    1.261899] io scheduler mq-deadline registered
[    1.265161] ledtrig-cpu: registered to indicate activity on CPUs
[    1.266308] input: Power Button as /devices/LNXSYSTM:00/LNXSYBUS:00/PNP0C0C:00/input/input0
[    1.266371] ACPI: button: Power Button [PWRB]
[    1.266437] input: Lid Switch as /devices/LNXSYSTM:00/LNXSYBUS:00/PNP0C0D:00/input/input1
[    1.266474] ACPI: button: Lid Switch [LID0]
[    1.266530] input: Sleep Button as /devices/LNXSYSTM:00/LNXSYBUS:00/PNP0C0E:00/input/input2
[    1.266588] ACPI: button: Sleep Button [SLPB]
[    1.280649] thermal LNXTHERM:00: registered as thermal_zone0
[    1.280661] ACPI: thermal: Thermal Zone [TSOC] (51 C)
[    1.280822] thermal LNXTHERM:01: registered as thermal_zone1
[    1.280827] ACPI: thermal: Thermal Zone [TS0E] (49 C)
[    1.280971] thermal LNXTHERM:02: registered as thermal_zone2
[    1.280975] ACPI: thermal: Thermal Zone [TS0P] (48 C)
[    1.281109] thermal LNXTHERM:03: registered as thermal_zone3
[    1.281112] ACPI: thermal: Thermal Zone [TS1E] (49 C)
[    1.281248] thermal LNXTHERM:04: registered as thermal_zone4
[    1.281251] ACPI: thermal: Thermal Zone [TS1P] (48 C)
[    1.281379] thermal LNXTHERM:05: registered as thermal_zone5
[    1.281382] ACPI: thermal: Thermal Zone [TGPU] (51 C)
[    1.281515] thermal LNXTHERM:06: registered as thermal_zone6
[    1.281518] ACPI: thermal: Thermal Zone [TUNC] (50 C)
[    1.281802] ACPI GTDT: found 1 SBSA generic Watchdog(s).
[    1.286671] Serial: 8250/16550 driver, 32 ports, IRQ sharing enabled
[    1.291093] printk: legacy console [ttyS0] disabled
[    1.311479] MTKI0511:00: ttyS0 at MMIO 0x16a00000 (irq = 45, base_baud = 1625000) is a ST16650V2
[    1.311529] printk: legacy console [ttyS0] enabled
[    1.311533] printk: legacy bootconsole [uart0] disabled
[    1.312300] msm_serial: driver initialized
[    1.312381] SuperH (H)SCI(F) driver initialized
[    1.312737] arm-smmu-v3 arm-smmu-v3.0.auto: option mask 0x0
[    1.312760] arm-smmu-v3 arm-smmu-v3.0.auto: ias 40-bit, oas 40-bit (features 0x0396dfbf)
[    1.313262] arm-smmu-v3 arm-smmu-v3.0.auto: allocated 65536 entries for cmdq
[    1.313592] arm-smmu-v3 arm-smmu-v3.0.auto: allocated 32768 entries for evtq
[    1.314018] arm-smmu-v3 arm-smmu-v3.0.auto: allocated 65536 entries for priq
[    1.314020] arm-smmu-v3 arm-smmu-v3.0.auto: 2-level strtab only covers 25/32 bits of SID
[    1.314349] arm-smmu-v3 arm-smmu-v3.0.auto: msi_domain absent - falling back to wired irqs
[    1.315276] platform NVDA8000:00: Adding to iommu group 0
[    1.315293] platform NVDA8000:01: Adding to iommu group 1
[    1.315309] platform NVDA8000:02: Adding to iommu group 2
[    1.315324] platform NVDA8000:03: Adding to iommu group 3
[    1.315340] platform NVDA8001:00: Adding to iommu group 4
[    1.315356] platform NVDA8000:04: Adding to iommu group 5
[    1.316045] pci 0000:00:00.0: Adding to iommu group 6
[    1.316197] pci 0000:01:00.0: Adding to iommu group 7
[    1.316228] pci 0000:01:00.1: Adding to iommu group 8
[    1.316676] pci 0002:00:00.0: Adding to iommu group 9
[    1.317076] pci 0002:01:00.0: Adding to iommu group 10
[    1.317107] pci 0002:01:00.1: Adding to iommu group 11
[    1.317369] pci 0004:00:00.0: Adding to iommu group 12
[    1.317637] pci 0004:01:00.0: Adding to iommu group 13
[    1.317904] pci 0007:00:00.0: Adding to iommu group 14
[    1.318173] pci 0007:01:00.0: Adding to iommu group 15
[    1.318441] pci 0009:00:00.0: Adding to iommu group 16
[    1.318986] pci 0009:01:00.0: Adding to iommu group 17
[    1.319828] arm-smmu-v3 arm-smmu-v3.1.auto: option mask 0x0
[    1.319846] arm-smmu-v3 arm-smmu-v3.1.auto: ias 40-bit, oas 40-bit (features 0x0396dfbf)
[    1.320149] arm-smmu-v3 arm-smmu-v3.1.auto: allocated 65536 entries for cmdq
[    1.320541] arm-smmu-v3 arm-smmu-v3.1.auto: allocated 32768 entries for evtq
[    1.321033] arm-smmu-v3 arm-smmu-v3.1.auto: allocated 65536 entries for priq
[    1.321042] arm-smmu-v3 arm-smmu-v3.1.auto: 2-level strtab only covers 25/32 bits of SID
[    1.322481] arm-smmu-v3 arm-smmu-v3.1.auto: msi_domain absent - falling back to wired irqs
[    1.324373] platform NVDA2014:00: Adding to iommu group 18
[    1.324677] pci 000f:00:00.0: Adding to iommu group 19
[    1.325167] pci 000f:01:00.0: Adding to iommu group 20
[    1.394039] arm-smmu-v3 arm-smmu-v3.2.auto: option mask 0x0
[    1.394072] arm-smmu-v3 arm-smmu-v3.2.auto: ias 40-bit, oas 40-bit (features 0x0396dfbf)
[    1.394474] arm-smmu-v3 arm-smmu-v3.2.auto: allocated 65536 entries for cmdq
[    1.395107] arm-smmu-v3 arm-smmu-v3.2.auto: allocated 32768 entries for evtq
[    1.395732] arm-smmu-v3 arm-smmu-v3.2.auto: allocated 65536 entries for priq
[    1.395735] arm-smmu-v3 arm-smmu-v3.2.auto: 2-level strtab only covers 25/32 bits of SID
[    1.396153] arm-smmu-v3 arm-smmu-v3.2.auto: msi_domain absent - falling back to wired irqs
[    1.396536] arm-smmu-v3 arm-smmu-v3.2.auto: no priq irq - PRI will be broken
[    1.396929] platform NVDA2861:00: Adding to iommu group 21
[    1.397114] platform NVDA0210:00: Adding to iommu group 22
[    1.397143] platform NVDA0210:01: Adding to iommu group 23
[    1.397172] platform NVDA0210:02: Adding to iommu group 24
[    1.412515] Freeing initrd memory: 82164K
[    1.421647] loop: module loaded
[    1.422968] ACPI: bus type drm_connector registered
[    1.423271] tun: Universal TUN/TAP device driver, 1.6
[    1.423739] PPP generic driver version 2.4.2
[    1.424265] mousedev: PS/2 mouse device common for all mice
[    1.426119] rtc-efi rtc-efi.0: registered as rtc0
[    1.426501] rtc-efi rtc-efi.0: setting system clock to 2026-05-20T09:44:13 UTC (1779270253)
[    1.426732] i2c_dev: i2c /dev entries driver
[    1.427030] device-mapper: core: CONFIG_IMA_DISABLE_HTABLE is disabled. Duplicate IMA measurements will not be recorded in the IMA log.
[    1.427069] device-mapper: uevent: version 1.0.3
[    1.427169] device-mapper: ioctl: 4.50.0-ioctl (2025-04-28) initialised: dm-devel@lists.linux.dev
[    1.427683] SMCCC: SOC_ID: ID = jep106:0426:8901 Revision = 0x00000000
[    1.427687] Arm LFA: Live Firmware activation: no firmware agent found
[    1.431696] hw perfevents: enabled with armv8_pmuv3_0 PMU driver, 21 (0,800fffff) counters available
[    1.434891] hw perfevents: enabled with armv8_pmuv3_1 PMU driver, 32 (0,ffffffff) counters available
[    1.435091] drop_monitor: Initializing network drop monitor service
[    1.435217] NET: Registered PF_INET6 protocol family
[    1.435307] watchdog: NMI not fully supported
[    1.435309] watchdog: Hard watchdog permanently disabled
[    1.436860] Segment Routing with IPv6
[    1.436880] In-situ OAM (IOAM) with IPv6
[    1.436927] NET: Registered PF_PACKET protocol family
[    1.437552] Key type dns_resolver registered
[    1.441641] registered taskstats version 1
[    1.449276] Loading compiled-in X.509 certificates
[    1.451189] Loaded X.509 cert 'Build time autogenerated kernel key: 8c8cf838f840476e22f3b6565677dd5f2a28eb1c'
[    1.451797] Loaded X.509 cert 'Canonical Ltd. Live Patch Signing 2025 Kmod: d541cef61dc7e793b7eb7e899970a2eef0b5dc8c'
[    1.452388] Loaded X.509 cert 'Canonical Ltd. Live Patch Signing: 14df34d1a87cf37625abec039ef2bf521249b969'
[    1.452984] Loaded X.509 cert 'Canonical Ltd. Kernel Module Signing: 88f752e560a1e0737e31163a466ad7b70a850c19'
[    1.452997] blacklist: Loading compiled-in revocation X.509 certificates
[    1.453015] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing: 61482aa2830d0ab2ad5af10b7250da9033ddcef0'
[    1.453027] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (2017): 242ade75ac4a15e50d50c84b0d45ff3eae707a03'
[    1.453037] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (ESM 2018): 365188c1d374d6b07c3c8f240f8ef722433d6a8b'
[    1.453046] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (2019): c0746fd6c5da3ae827864651ad66ae47fe24b3e8'
[    1.453056] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (2021 v1): a8d54bbb3825cfb94fa13c9f8a594a195c107b8d'
[    1.453065] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (2021 v2): 4cf046892d6fd3c9a5b03f98d845f90851dc6a8c'
[    1.453075] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (2021 v3): 100437bb6de6e469b581e61cd66bce3ef4ed53af'
[    1.453084] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing (Ubuntu Core 2019): c1d57b8f6b743f23ee41f4f7ee292f06eecadfb9'
[    1.459622] Demotion targets for Node 0: null
[    1.460251] Key type .fscrypt registered
[    1.460253] Key type fscrypt-provisioning registered
[    1.460340] Key type big_key registered
[    1.486459] Key type encrypted registered
[    1.486478] AppArmor: AppArmor sha256 policy hashing enabled
[    1.488304] ima: secureboot mode disabled
[    1.488327] ima: No TPM chip found, activating TPM-bypass!
[    1.488336] Loading compiled-in module X.509 certificates
[    1.488993] Loaded X.509 cert 'Build time autogenerated kernel key: 8c8cf838f840476e22f3b6565677dd5f2a28eb1c'
[    1.488998] ima: Allocated hash algorithm: sha256
[    1.489028] ima: No architecture policies found
[    1.489063] evm: Initialising EVM extended attributes:
[    1.489065] evm: security.selinux
[    1.489067] evm: security.SMACK64
[    1.489069] evm: security.SMACK64EXEC
[    1.489071] evm: security.SMACK64TRANSMUTE
[    1.489073] evm: security.SMACK64MMAP
[    1.489075] evm: security.apparmor
[    1.489077] evm: security.ima
[    1.489078] evm: security.capability
[    1.489080] evm: HMAC attrs: 0x1
[    1.490912] pcieport 0000:00:00.0: PME: Signaling with IRQ 329
[    1.491228] pcieport 0000:00:00.0: AER: enabled with IRQ 330
[    1.492642] pcieport 0002:00:00.0: PME: Signaling with IRQ 332
[    1.492950] pcieport 0002:00:00.0: AER: enabled with IRQ 333
[    1.494325] pcieport 0004:00:00.0: PME: Signaling with IRQ 335
[    1.494636] pcieport 0004:00:00.0: AER: enabled with IRQ 336
[    1.494688] pcieport 0004:00:00.0: pciehp: Slot #4 AttnBtn- PwrCtrl- MRL- AttnInd- PwrInd- HotPlug+ Surprise+ Interlock- NoCompl+ IbPresDis- LLActRep+
[    1.496327] pcieport 0007:00:00.0: PME: Signaling with IRQ 338
[    1.496608] pcieport 0007:00:00.0: AER: enabled with IRQ 339
[    1.496645] pcieport 0007:00:00.0: pciehp: Slot #7 AttnBtn- PwrCtrl- MRL- AttnInd- PwrInd- HotPlug+ Surprise+ Interlock- NoCompl+ IbPresDis- LLActRep+
[    1.498270] pcieport 0009:00:00.0: PME: Signaling with IRQ 341
[    1.498564] pcieport 0009:00:00.0: AER: enabled with IRQ 342
[    1.498605] pcieport 0009:00:00.0: pciehp: Slot #9 AttnBtn- PwrCtrl- MRL- AttnInd- PwrInd- HotPlug+ Surprise+ Interlock- NoCompl+ IbPresDis- LLActRep+
[    1.499364] pcieport 000f:00:00.0: PME: Signaling with IRQ 343
[    1.499680] pcieport 000f:00:00.0: AER: enabled with IRQ 345
[    1.509257] clk: Disabling unused clocks
[    1.509265] PM: genpd: Disabling unused power domains
[    1.513155] Freeing unused kernel memory: 14784K
[    1.861122] Checked W+X mappings: passed, no W+X pages found
[    1.861140] Run /init as init process
[    1.861142]   with arguments:
[    1.861146]     /init
[    1.861148]     splash
[    1.861150]   with environment:
[    1.861152]     HOME=/
[    1.861154]     TERM=linux
[    2.195740] xhci-hcd NVDA8000:00: xHCI Host Controller
[    2.195758] xhci-hcd NVDA8000:00: new USB bus registered, assigned bus number 1
[    2.196041] xhci-hcd NVDA8000:00: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.196066] xhci-hcd NVDA8000:00: irq 96, io mem 0x1db60000
[    2.196156] xhci-hcd NVDA8000:00: xHCI Host Controller
[    2.196160] xhci-hcd NVDA8000:00: new USB bus registered, assigned bus number 2
[    2.196164] xhci-hcd NVDA8000:00: Host supports USB 3.2 Enhanced SuperSpeed
[    2.196599] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.196603] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.196606] usb usb1: Product: xHCI Host Controller
[    2.196608] usb usb1: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.196610] usb usb1: SerialNumber: NVDA8000:00
[    2.215534] hub 1-0:1.0: USB hub found
[    2.215594] hub 1-0:1.0: 1 port detected
[    2.216876] Key type psk registered
[    2.217111] r8127 Ethernet controller driver 11.014.00-NAPI loaded
[    2.217388] usb usb2: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.217573] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.217583] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.217587] usb usb2: Product: xHCI Host Controller
[    2.217591] usb usb2: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.217594] usb usb2: SerialNumber: NVDA8000:00
[    2.222846] hub 2-0:1.0: USB hub found
[    2.222884] hub 2-0:1.0: 1 port detected
[    2.226650] xhci-hcd NVDA8000:01: xHCI Host Controller
[    2.226682] xhci-hcd NVDA8000:01: new USB bus registered, assigned bus number 3
[    2.226982] xhci-hcd NVDA8000:01: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.227011] xhci-hcd NVDA8000:01: irq 97, io mem 0x1db90000
[    2.227123] xhci-hcd NVDA8000:01: xHCI Host Controller
[    2.227129] xhci-hcd NVDA8000:01: new USB bus registered, assigned bus number 4
[    2.227136] xhci-hcd NVDA8000:01: Host supports USB 3.2 Enhanced SuperSpeed
[    2.227435] usb usb3: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.227445] usb usb3: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.227449] usb usb3: Product: xHCI Host Controller
[    2.227453] usb usb3: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.227457] usb usb3: SerialNumber: NVDA8000:01
[    2.230110] hub 3-0:1.0: USB hub found
[    2.230149] hub 3-0:1.0: 1 port detected
[    2.231867] usb usb4: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.231983] usb usb4: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.231987] usb usb4: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.231990] usb usb4: Product: xHCI Host Controller
[    2.231993] usb usb4: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.231995] usb usb4: SerialNumber: NVDA8000:01
[    2.232796] hub 4-0:1.0: USB hub found
[    2.232833] hub 4-0:1.0: 1 port detected
[    2.233703] xhci-hcd NVDA8000:02: xHCI Host Controller
[    2.233723] xhci-hcd NVDA8000:02: new USB bus registered, assigned bus number 5
[    2.233977] xhci-hcd NVDA8000:02: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.233998] xhci-hcd NVDA8000:02: irq 98, io mem 0x1dde0000
[    2.234082] xhci-hcd NVDA8000:02: xHCI Host Controller
[    2.234085] xhci-hcd NVDA8000:02: new USB bus registered, assigned bus number 6
[    2.234089] xhci-hcd NVDA8000:02: Host supports USB 3.2 Enhanced SuperSpeed
[    2.234179] usb usb5: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.234183] usb usb5: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.234185] usb usb5: Product: xHCI Host Controller
[    2.234187] usb usb5: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.234189] usb usb5: SerialNumber: NVDA8000:02
[    2.234761] hub 5-0:1.0: USB hub found
[    2.234810] hub 5-0:1.0: 1 port detected
[    2.236020] usb usb6: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.236111] usb usb6: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.236114] usb usb6: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.236116] usb usb6: Product: xHCI Host Controller
[    2.236118] usb usb6: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.236120] usb usb6: SerialNumber: NVDA8000:02
[    2.236533] hub 6-0:1.0: USB hub found
[    2.236547] hub 6-0:1.0: 1 port detected
[    2.237212] xhci-hcd NVDA8000:03: xHCI Host Controller
[    2.237218] xhci-hcd NVDA8000:03: new USB bus registered, assigned bus number 7
[    2.237433] xhci-hcd NVDA8000:03: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.237444] xhci-hcd NVDA8000:03: irq 99, io mem 0x1de10000
[    2.237520] xhci-hcd NVDA8000:03: xHCI Host Controller
[    2.237523] xhci-hcd NVDA8000:03: new USB bus registered, assigned bus number 8
[    2.237525] xhci-hcd NVDA8000:03: Host supports USB 3.2 Enhanced SuperSpeed
[    2.237599] usb usb7: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.237603] usb usb7: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.237606] usb usb7: Product: xHCI Host Controller
[    2.237608] usb usb7: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.237610] usb usb7: SerialNumber: NVDA8000:03
[    2.238579] hub 7-0:1.0: USB hub found
[    2.238610] hub 7-0:1.0: 1 port detected
[    2.242097] r8127: This product is covered by one or more of the following patents: US6,570,884, US6,115,776, and US6,327,625.
[    2.242511] r8127  Copyright (C) 2025 Realtek NIC software team <nicfae@realtek.com> 
                This program comes with ABSOLUTELY NO WARRANTY; for details, please see <http://www.gnu.org/licenses/>. 
                This is free software, and you are welcome to redistribute it under certain conditions; see <http://www.gnu.org/licenses/>. 
[    2.242662] usb usb8: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.242722] usb usb8: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.242725] usb usb8: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.242727] usb usb8: Product: xHCI Host Controller
[    2.242729] usb usb8: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.242731] usb usb8: SerialNumber: NVDA8000:03
[    2.247472] hub 8-0:1.0: USB hub found
[    2.247530] hub 8-0:1.0: 1 port detected
[    2.256308] xhci-hcd NVDA8001:00: xHCI Host Controller
[    2.256327] xhci-hcd NVDA8001:00: new USB bus registered, assigned bus number 9
[    2.256687] xhci-hcd NVDA8001:00: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.256719] xhci-hcd NVDA8001:00: irq 100, io mem 0x1d860000
[    2.256833] xhci-hcd NVDA8001:00: xHCI Host Controller
[    2.256840] xhci-hcd NVDA8001:00: new USB bus registered, assigned bus number 10
[    2.256848] xhci-hcd NVDA8001:00: Host supports USB 3.2 Enhanced SuperSpeed
[    2.257187] usb usb9: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.257197] usb usb9: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.257202] usb usb9: Product: xHCI Host Controller
[    2.257206] usb usb9: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.257210] usb usb9: SerialNumber: NVDA8001:00
[    2.268818] nvme nvme0: pci function 0004:01:00.0
[    2.274067] nvme nvme0: D3 entry latency set to 10 seconds
[    2.274575] hub 9-0:1.0: USB hub found
[    2.274614] hub 9-0:1.0: 1 port detected
[    2.275241] r8127 0007:01:00.0 enP7s7: renamed from eth0
[    2.275289] usb usb10: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.275405] usb usb10: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.275419] usb usb10: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.275424] usb usb10: Product: xHCI Host Controller
[    2.275428] usb usb10: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.275432] usb usb10: SerialNumber: NVDA8001:00
[    2.275900] hub 10-0:1.0: USB hub found
[    2.275930] hub 10-0:1.0: 1 port detected
[    2.276419] xhci-hcd NVDA8000:04: xHCI Host Controller
[    2.276443] xhci-hcd NVDA8000:04: new USB bus registered, assigned bus number 11
[    2.276723] xhci-hcd NVDA8000:04: hcc params 0x01844f91 hci version 0x120 quirks 0x0008000000000010
[    2.276742] xhci-hcd NVDA8000:04: irq 101, io mem 0x1d870000
[    2.276829] xhci-hcd NVDA8000:04: xHCI Host Controller
[    2.276832] xhci-hcd NVDA8000:04: new USB bus registered, assigned bus number 12
[    2.276835] xhci-hcd NVDA8000:04: Host supports USB 3.2 Enhanced SuperSpeed
[    2.276947] usb usb11: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.17
[    2.276955] usb usb11: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.276958] usb usb11: Product: xHCI Host Controller
[    2.276960] usb usb11: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.276963] usb usb11: SerialNumber: NVDA8000:04
[    2.277310] hub 11-0:1.0: USB hub found
[    2.277322] hub 11-0:1.0: 2 ports detected
[    2.277729] usb usb12: We don't know the algorithms for LPM for this host, disabling LPM.
[    2.277954] usb usb12: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.17
[    2.277965] usb usb12: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    2.277971] usb usb12: Product: xHCI Host Controller
[    2.277975] usb usb12: Manufacturer: Linux 6.17.0-1018-nvidia xhci-hcd
[    2.277979] usb usb12: SerialNumber: NVDA8000:04
[    2.278723] hub 12-0:1.0: USB hub found
[    2.278764] hub 12-0:1.0: 1 port detected
[    2.279637] nvme nvme0: 15/0/0 default/read/poll queues
[    2.283134]  nvme0n1: p1 p2
[    2.297852] mlx5_core 0000:01:00.0: enabling device (0000 -> 0002)
[    2.297987] mlx5_core 0000:01:00.0: firmware version: 28.45.4028
[    2.298008] mlx5_core 0000:01:00.0: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[    2.541525] usb 2-1: new SuperSpeed USB device number 2 using xhci-hcd
[    2.564307] usb 2-1: New USB device found, idVendor=05e3, idProduct=0626, bcdDevice= 6.63
[    2.564320] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[    2.564325] usb 2-1: Product: USB3.1 Hub
[    2.564329] usb 2-1: Manufacturer: GenesysLogic
[    2.567925] hub 2-1:1.0: USB hub found
[    2.569099] hub 2-1:1.0: 4 ports detected
[    2.666169] mlx5_core 0000:01:00.0: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[    2.666723] mlx5_core 0000:01:00.0: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[    2.669180] usb 11-2: new high-speed USB device number 2 using xhci-hcd
[    2.669233] usb 1-1: new high-speed USB device number 2 using xhci-hcd
[    2.671973] mlx5_core 0000:01:00.0: Flow counters bulk query buffer size increased, bulk_query_len(8)
[    2.679106] mlx5_core 0000:01:00.0: Port module event: module 0, Cable unplugged
[    2.679865] mlx5_core 0000:01:00.0: mlx5_pcie_event:326:(pid 11): Detected insufficient power on the PCIe slot (27W).
[    2.697597] mlx5_core 0000:01:00.0: mlx5e: IPSec ESP acceleration enabled
[    2.796572] usb 11-2: New USB device found, idVendor=13d3, idProduct=3630, bcdDevice= 1.00
[    2.796588] usb 11-2: New USB device strings: Mfr=5, Product=6, SerialNumber=7
[    2.796594] usb 11-2: Product: Wireless_Device
[    2.796599] usb 11-2: Manufacturer: MediaTek Inc.
[    2.796603] usb 11-2: SerialNumber: 000000000
[    2.796814] usb 1-1: New USB device found, idVendor=05e3, idProduct=0610, bcdDevice= 6.63
[    2.796820] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[    2.796824] usb 1-1: Product: USB2.1 Hub
[    2.796827] usb 1-1: Manufacturer: GenesysLogic
[    2.797953] hub 1-1:1.0: USB hub found
[    2.798286] hub 1-1:1.0: 4 ports detected
[    2.829143] mlx5_core 0000:01:00.0: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[    2.837293] mlx5_core 0000:01:00.1: enabling device (0000 -> 0002)
[    2.837428] mlx5_core 0000:01:00.1: firmware version: 28.45.4028
[    2.837449] mlx5_core 0000:01:00.1: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[    3.095171] usb 1-1.1: new full-speed USB device number 3 using xhci-hcd
[    3.200019] usb 1-1.1: not running at top speed; connect to a high speed hub
[    3.205469] usb 1-1.1: New USB device found, idVendor=291a, idProduct=8355, bcdDevice= 1.12
[    3.205480] usb 1-1.1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    3.205484] usb 1-1.1: Product: USB BillBoard
[    3.205486] usb 1-1.1: Manufacturer: Anker Type-C Hub Device
[    3.205489] usb 1-1.1: SerialNumber: SN23456789
[    3.205607] mlx5_core 0000:01:00.1: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[    3.205895] mlx5_core 0000:01:00.1: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[    3.211205] mlx5_core 0000:01:00.1: Flow counters bulk query buffer size increased, bulk_query_len(8)
[    3.217526] mlx5_core 0000:01:00.1: Port module event: module 1, Cable plugged
[    3.218330] mlx5_core 0000:01:00.1: mlx5_pcie_event:326:(pid 165): Detected insufficient power on the PCIe slot (27W).
[    3.226405] mlx5_core 0000:01:00.1: mlx5e: IPSec ESP acceleration enabled
[    3.303206] usb 1-1.3: new high-speed USB device number 4 using xhci-hcd
[    3.386146] mlx5_core 0000:01:00.1: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[    3.393362] mlx5_core 0002:01:00.0: enabling device (0000 -> 0002)
[    3.393518] mlx5_core 0002:01:00.0: firmware version: 28.45.4028
[    3.393546] mlx5_core 0002:01:00.0: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[    3.419809] usb 1-1.3: New USB device found, idVendor=0b95, idProduct=7720, bcdDevice= 0.01
[    3.419820] usb 1-1.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    3.419823] usb 1-1.3: Product: AX88772A
[    3.419826] usb 1-1.3: Manufacturer: ASIX Elec. Corp.
[    3.419828] usb 1-1.3: SerialNumber: 000387
[    3.762193] mlx5_core 0002:01:00.0: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[    3.763309] mlx5_core 0002:01:00.0: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[    3.773710] mlx5_core 0002:01:00.0: Flow counters bulk query buffer size increased, bulk_query_len(8)
[    3.781193] mlx5_core 0002:01:00.0: Port module event: module 0, Cable unplugged
[    3.781508] mlx5_core 0002:01:00.0: mlx5_pcie_event:326:(pid 402): Detected insufficient power on the PCIe slot (27W).
[    3.793057] mlx5_core 0002:01:00.0: mlx5e: IPSec ESP acceleration enabled
[    3.936209] asix 1-1.3:1.0 (unnamed net_device) (uninitialized): PHY [usb-001:004:10] driver [Asix Electronics AX88772A] (irq=POLL)
[    3.945738] Asix Electronics AX88772A usb-001:004:10: attached PHY driver (mii_bus:phy_addr=usb-001:004:10, irq=POLL)
[    3.946089] asix 1-1.3:1.0 eth2: register 'asix' at usb-NVDA8000:00-1.3, ASIX AX88772 USB 2.0 Ethernet, 00:0e:c6:45:a2:b4
[    3.946155] usbcore: registered new interface driver asix
[    3.949939] mlx5_core 0002:01:00.0: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[    3.950200] asix 1-1.3:1.0 enx000ec645a2b4: renamed from eth2
[    3.955279] mlx5_core 0002:01:00.1: enabling device (0000 -> 0002)
[    3.955419] mlx5_core 0002:01:00.1: firmware version: 28.45.4028
[    3.955442] mlx5_core 0002:01:00.1: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[    4.318644] mlx5_core 0002:01:00.1: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[    4.319411] mlx5_core 0002:01:00.1: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[    4.324955] mlx5_core 0002:01:00.1: Flow counters bulk query buffer size increased, bulk_query_len(8)
[    4.332365] mlx5_core 0002:01:00.1: Port module event: module 1, Cable plugged
[    4.333262] mlx5_core 0002:01:00.1: mlx5_pcie_event:326:(pid 165): Detected insufficient power on the PCIe slot (27W).
[    4.345920] mlx5_core 0002:01:00.1: mlx5e: IPSec ESP acceleration enabled
[    4.515733] mlx5_core 0002:01:00.1: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[    4.521574] mlx5_core 0002:01:00.1 enP2p1s0f1np1: renamed from eth2
[    4.521868] mlx5_core 0000:01:00.1 enp1s0f1np1: renamed from eth1
[    4.522156] mlx5_core 0002:01:00.0 enP2p1s0f0np0: renamed from eth3
[    4.522456] mlx5_core 0000:01:00.0 enp1s0f0np0: renamed from eth0
[    4.534296] MACsec IEEE 802.1AE
[    5.955560] raid6: neonx8   gen()  5698 MB/s
[    5.972556] raid6: neonx4   gen()  5696 MB/s
[    5.989557] raid6: neonx2   gen()  5506 MB/s
[    6.006558] raid6: neonx1   gen()  4806 MB/s
[    6.023560] raid6: int64x8  gen()  3206 MB/s
[    6.040563] raid6: int64x4  gen()  3150 MB/s
[    6.057560] raid6: int64x2  gen()  2718 MB/s
[    6.074562] raid6: int64x1  gen()  2201 MB/s
[    6.074565] raid6: using algorithm neonx8 gen() 5698 MB/s
[    6.091557] raid6: .... xor() 4309 MB/s, rmw enabled
[    6.091560] raid6: using neon recovery algorithm
[    6.097272] xor: measuring software checksum speed
[    6.097452]    8regs           : 18790 MB/sec
[    6.097636]    32regs          : 18219 MB/sec
[    6.097768]    arm64_neon      : 25346 MB/sec
[    6.097770] xor: using function: arm64_neon (25346 MB/sec)
[    6.100662] async_tx: api initialized (async)
[    6.236327] Btrfs loaded, zoned=yes, fsverity=yes
[    6.332499] EXT4-fs (nvme0n1p2): mounted filesystem a151b0ce-dde9-400c-b84e-ae7b9859c406 ro with ordered data mode. Quota mode: none.
[    6.490031] systemd[1]: Inserted module 'autofs4'
[    6.523165] systemd[1]: systemd 255.4-1ubuntu8.15 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT -GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY +P11KIT +QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified)
[    6.523178] systemd[1]: Detected architecture arm64.
[    6.524071] systemd[1]: Hostname set to <spark-fb97>.
[    6.726969] systemd[1]: Configuration file /etc/systemd/system/dgx-dashboard-admin.service is marked world-inaccessible. This has no effect as configuration data is accessible via APIs without restrictions. Proceeding anyway.
[    6.765414] systemd[1]: Queued start job for default target graphical.target.
[    6.799290] systemd[1]: Created slice system-modprobe.slice - Slice /system/modprobe.
[    6.799869] systemd[1]: Created slice system-serial\x2dgetty.slice - Slice /system/serial-getty.
[    6.800352] systemd[1]: Created slice system-systemd\x2dfsck.slice - Slice /system/systemd-fsck.
[    6.800679] systemd[1]: Created slice user.slice - User and Session Slice.
[    6.800745] systemd[1]: Started systemd-ask-password-wall.path - Forward Password Requests to Wall Directory Watch.
[    6.800982] systemd[1]: Set up automount proc-sys-fs-binfmt_misc.automount - Arbitrary Executable File Formats File System Automount Point.
[    6.801003] systemd[1]: Expecting device dev-disk-by\x2duuid-924B\x2d73A3.device - /dev/disk/by-uuid/924B-73A3...
[    6.801011] systemd[1]: Expecting device dev-ttyS0.device - /dev/ttyS0...
[    6.801036] systemd[1]: Reached target integritysetup.target - Local Integrity Protected Volumes.
[    6.801077] systemd[1]: Reached target nss-user-lookup.target - User and Group Name Lookups.
[    6.801101] systemd[1]: Reached target slices.target - Slice Units.
[    6.801121] systemd[1]: Reached target snapd.mounts-pre.target - Mounting snaps.
[    6.801156] systemd[1]: Reached target veritysetup.target - Local Verity Protected Volumes.
[    6.801244] systemd[1]: Listening on dm-event.socket - Device-mapper event daemon FIFOs.
[    6.801372] systemd[1]: Listening on lvm2-lvmpolld.socket - LVM2 poll daemon socket.
[    6.801476] systemd[1]: Listening on multipathd.socket - multipathd control socket.
[    6.808602] systemd[1]: Listening on rpcbind.socket - RPCbind Server Activation Socket.
[    6.809186] systemd[1]: Listening on syslog.socket - Syslog Socket.
[    6.809315] systemd[1]: Listening on systemd-fsckd.socket - fsck to fsckd communication Socket.
[    6.809395] systemd[1]: Listening on systemd-initctl.socket - initctl Compatibility Named Pipe.
[    6.809510] systemd[1]: Listening on systemd-journald-dev-log.socket - Journal Socket (/dev/log).
[    6.809653] systemd[1]: Listening on systemd-journald.socket - Journal Socket.
[    6.809832] systemd[1]: Listening on systemd-networkd.socket - Network Service Netlink Socket.
[    6.809877] systemd[1]: systemd-pcrextend.socket - TPM2 PCR Extension (Varlink) was skipped because of an unmet condition check (ConditionSecurity=measured-uki).
[    6.810111] systemd[1]: Listening on systemd-udevd-control.socket - udev Control Socket.
[    6.810209] systemd[1]: Listening on systemd-udevd-kernel.socket - udev Kernel Socket.
[    6.811420] systemd[1]: Mounting dev-hugepages.mount - Huge Pages File System...
[    6.812344] systemd[1]: Mounting dev-mqueue.mount - POSIX Message Queue File System...
[    6.813100] systemd[1]: Mounting proc-fs-nfsd.mount - NFSD configuration filesystem...
[    6.814103] systemd[1]: Mounting sys-kernel-debug.mount - Kernel Debug File System...
[    6.814788] systemd[1]: Mounting sys-kernel-tracing.mount - Kernel Trace File System...
[    6.820429] systemd[1]: Starting systemd-journald.service - Journal Service...
[    6.820524] systemd[1]: auth-rpcgss-module.service - Kernel Module supporting RPCSEC_GSS was skipped because of an unmet condition check (ConditionPathExists=/etc/krb5.keytab).
[    6.821741] systemd[1]: Starting keyboard-setup.service - Set the console keyboard layout...
[    6.822655] systemd[1]: Starting kmod-static-nodes.service - Create List of Static Device Nodes...
[    6.823678] systemd[1]: Starting lvm2-monitor.service - Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling...
[    6.824724] systemd[1]: Starting modprobe@configfs.service - Load Kernel Module configfs...
[    6.825566] systemd[1]: Starting modprobe@dm_mod.service - Load Kernel Module dm_mod...
[    6.827019] systemd[1]: Starting modprobe@drm.service - Load Kernel Module drm...
[    6.828175] systemd[1]: Starting modprobe@efi_pstore.service - Load Kernel Module efi_pstore...
[    6.829083] systemd[1]: Starting modprobe@fuse.service - Load Kernel Module fuse...
[    6.829959] systemd[1]: Starting modprobe@loop.service - Load Kernel Module loop...
[    6.830801] systemd[1]: Starting modprobe@nvme_fabrics.service - Load Kernel Module nvme_fabrics...
[    6.830933] systemd[1]: netplan-ovs-cleanup.service - OpenVSwitch configuration for cleanup was skipped because of an unmet condition check (ConditionFileIsExecutable=/usr/bin/ovs-vsctl).
[    6.831390] systemd[1]: systemd-fsck-root.service - File System Check on Root Device was skipped because of an unmet condition check (ConditionPathExists=!/run/initramfs/fsck-root).
[    6.832619] systemd[1]: Starting systemd-modules-load.service - Load Kernel Modules...
[    6.832637] systemd[1]: systemd-pcrmachine.service - TPM2 PCR Machine ID Measurement was skipped because of an unmet condition check (ConditionSecurity=measured-uki).
[    6.833812] systemd[1]: Starting systemd-remount-fs.service - Remount Root and Kernel File Systems...
[    6.833873] systemd[1]: systemd-tpm2-setup-early.service - TPM2 SRK Setup (Early) was skipped because of an unmet condition check (ConditionSecurity=measured-uki).
[    6.835058] systemd[1]: Starting systemd-udev-trigger.service - Coldplug All udev Devices...
[    6.836673] systemd[1]: Mounted dev-hugepages.mount - Huge Pages File System.
[    6.836788] systemd[1]: Mounted dev-mqueue.mount - POSIX Message Queue File System.
[    6.836885] systemd[1]: Mounted sys-kernel-debug.mount - Kernel Debug File System.
[    6.836967] systemd[1]: Mounted sys-kernel-tracing.mount - Kernel Trace File System.
[    6.837244] systemd[1]: Finished kmod-static-nodes.service - Create List of Static Device Nodes.
[    6.837584] systemd[1]: modprobe@configfs.service: Deactivated successfully.
[    6.837737] systemd[1]: Finished modprobe@configfs.service - Load Kernel Module configfs.
[    6.837988] systemd[1]: modprobe@dm_mod.service: Deactivated successfully.
[    6.838130] systemd[1]: Finished modprobe@dm_mod.service - Load Kernel Module dm_mod.
[    6.838344] systemd[1]: modprobe@drm.service: Deactivated successfully.
[    6.838473] systemd[1]: Finished modprobe@drm.service - Load Kernel Module drm.
[    6.838683] systemd[1]: modprobe@fuse.service: Deactivated successfully.
[    6.838810] systemd[1]: Finished modprobe@fuse.service - Load Kernel Module fuse.
[    6.839020] systemd[1]: modprobe@loop.service: Deactivated successfully.
[    6.839147] systemd[1]: Finished modprobe@loop.service - Load Kernel Module loop.
[    6.840250] systemd[1]: Mounting sys-fs-fuse-connections.mount - FUSE Control File System...
[    6.841110] systemd[1]: Mounting sys-kernel-config.mount - Kernel Configuration File System...
[    6.841163] systemd[1]: systemd-repart.service - Repartition Root Disk was skipped because no trigger condition checks were met.
[    6.842326] systemd[1]: Starting systemd-tmpfiles-setup-dev-early.service - Create Static Device Nodes in /dev gracefully...
[    6.842900] systemd-journald[622]: Collecting audit messages is disabled.
[    6.845889] systemd[1]: Mounted sys-fs-fuse-connections.mount - FUSE Control File System.
[    6.846915] pstore: Using crash dump compression: deflate
[    6.846978] IPMI message handler: version 39.2
[    6.847738] systemd[1]: modprobe@nvme_fabrics.service: Deactivated successfully.
[    6.847929] systemd[1]: Finished modprobe@nvme_fabrics.service - Load Kernel Module nvme_fabrics.
[    6.848646] ipmi device interface
[    6.850052] systemd[1]: Mounted sys-kernel-config.mount - Kernel Configuration File System.
[    6.851359] pstore: Registered efi_pstore as persistent store backend
[    6.852201] systemd[1]: modprobe@efi_pstore.service: Deactivated successfully.
[    6.852367] systemd[1]: Finished modprobe@efi_pstore.service - Load Kernel Module efi_pstore.
[    6.853469] mstflint_access: loading out-of-tree module taints kernel.
[    6.854614]   MST::  : mst_init 1715: Mellanox Technologies Software Tools Driver - version 2.0.0
[    6.854624]   MST::  : mst_init 1726: found device - domain=0x0, bus=0x1, slot=0x0, func=0x0, vendor=0x15b3, device=0x1021
[    6.854840]   MST::  : mst_init 1726: found device - domain=0x0, bus=0x1, slot=0x0, func=0x1, vendor=0x15b3, device=0x1021
[    6.854983]   MST::  : mst_init 1726: found device - domain=0x2, bus=0x1, slot=0x0, func=0x0, vendor=0x15b3, device=0x1021
[    6.855139]   MST::  : mst_init 1726: found device - domain=0x2, bus=0x1, slot=0x0, func=0x1, vendor=0x15b3, device=0x1021
[    6.856009] RPC: Registered named UNIX socket transport module.
[    6.856018] RPC: Registered udp transport module.
[    6.856020] RPC: Registered tcp transport module.
[    6.856022] RPC: Registered tcp-with-tls transport module.
[    6.856024] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    6.856276] systemd[1]: Finished systemd-modules-load.service - Load Kernel Modules.
[    6.857586] systemd[1]: Starting systemd-sysctl.service - Apply Kernel Variables...
[    6.861016] EXT4-fs (nvme0n1p2): re-mounted a151b0ce-dde9-400c-b84e-ae7b9859c406 r/w.
[    6.862249] systemd[1]: Finished systemd-remount-fs.service - Remount Root and Kernel File Systems.
[    6.863434] systemd[1]: Activating swap swap.img.swap - /swap.img...
[    6.864559] systemd[1]: Starting multipathd.service - Device-Mapper Multipath Device Controller...
[    6.864869] systemd[1]: systemd-hwdb-update.service - Rebuild Hardware Database was skipped because of an unmet condition check (ConditionNeedsUpdate=/etc).
[    6.864913] systemd[1]: systemd-pstore.service - Platform Persistent Storage Archival was skipped because of an unmet condition check (ConditionDirectoryNotEmpty=/sys/fs/pstore).
[    6.865998] systemd[1]: Starting systemd-random-seed.service - Load/Save OS Random Seed...
[    6.866017] systemd[1]: systemd-tpm2-setup.service - TPM2 SRK Setup was skipped because of an unmet condition check (ConditionSecurity=measured-uki).
[    6.866336] systemd[1]: Finished systemd-tmpfiles-setup-dev-early.service - Create Static Device Nodes in /dev gracefully.
[    6.866475] systemd[1]: systemd-sysusers.service - Create System Users was skipped because no trigger condition checks were met.
[    6.867409] systemd[1]: Starting systemd-tmpfiles-setup-dev.service - Create Static Device Nodes in /dev...
[    6.871608] systemd[1]: Finished lvm2-monitor.service - Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling.
[    6.874585] Adding 16777212k swap on /swap.img.  Priority:-2 extents:12 across:18128892k SS
[    6.874849] systemd[1]: Activated swap swap.img.swap - /swap.img.
[    6.875219] systemd[1]: Finished keyboard-setup.service - Set the console keyboard layout.
[    6.875474] systemd[1]: Reached target swap.target - Swaps.
[    6.877514] systemd[1]: Finished systemd-sysctl.service - Apply Kernel Variables.
[    6.880280] systemd[1]: Finished systemd-tmpfiles-setup-dev.service - Create Static Device Nodes in /dev.
[    6.882538] systemd[1]: Starting systemd-udevd.service - Rule-based Manager for Device Events and Files...
[    6.883148] systemd[1]: Finished systemd-random-seed.service - Load/Save OS Random Seed.
[    6.903200] nvme nvme0: using unchecked data buffer
[    6.904239] systemd[1]: Started multipathd.service - Device-Mapper Multipath Device Controller.
[    6.914893] systemd[1]: Mounted proc-fs-nfsd.mount - NFSD configuration filesystem.
[    6.919019] systemd[1]: Started systemd-journald.service - Journal Service.
[    6.951544] systemd-journald[622]: Received client request to flush runtime journal.
[    6.962889] systemd-journald[622]: /var/log/journal/c8a82ac6119c430e8201416ac2b5d339/system.journal: Journal file uses a different sequence number ID, rotating.
[    6.962898] systemd-journald[622]: Rotating system journal.
[    6.969346] loop0: detected capacity change from 0 to 126776
[    6.969643] loop1: detected capacity change from 0 to 8
[    6.974475] loop2: detected capacity change from 0 to 126688
[    6.974886] loop3: detected capacity change from 0 to 480920
[    6.977733] loop4: detected capacity change from 0 to 1132440
[    6.980016] loop5: detected capacity change from 0 to 480896
[    6.980744] loop6: detected capacity change from 0 to 357512
[    6.983068] loop7: detected capacity change from 0 to 187776
[    6.984969] loop8: detected capacity change from 0 to 452992
[    6.987187] loop10: detected capacity change from 0 to 452992
[    7.271437] cx7-pcie-hotplug MTKP0001:00: PCIe hotplug driver initialized successfully
[    7.305533] CPPC Cpufreq:Enabling auto_sel_mode (autonomous selection mode)
[    7.313760] processor cpu0: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.317796] processor cpu1: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.323690] processor cpu2: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.325399] processor cpu3: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.326214] cx7-pcie-hotplug MTKP0001:00: Hotplug enabled
[    7.327523] processor cpu4: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.338767] usbcore: registered new device driver onboard-usb-dev
[    7.340138] processor cpu5: EM: created perf domain
[    7.353184] arm_spe_pmu arm,spe-v1: probed SPEv1.2 for CPUs 0-19 [max_record_sz 64, align 64, features 0xd7]
[    7.353212] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.3.auto: option mask 0x0
[    7.353292] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[    7.353444] Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[    7.353512] Loaded X.509 cert 'wens: 61c038651aabdcf94bd0ac7ff06c7248db18c600'
[    7.354529] processor cpu10: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.357061] processor cpu11: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.358245] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.3.auto: Registered PMU @ 0x0000000013802000 using 32 counters with Global(Counter0) filter settings
[    7.358417] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.4.auto: option mask 0x0
[    7.359018] processor cpu12: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.360071] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.4.auto: Registered PMU @ 0x0000000013842000 using 16 counters with Global(Counter0) filter settings
[    7.360284] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.5.auto: option mask 0x0
[    7.360655] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.5.auto: Registered PMU @ 0x0000000013862000 using 16 counters with Global(Counter0) filter settings
[    7.361174] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.6.auto: option mask 0x0
[    7.361618] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.6.auto: Registered PMU @ 0x0000000013882000 using 16 counters with Global(Counter0) filter settings
[    7.361749] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.7.auto: option mask 0x0
[    7.363414] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.7.auto: Registered PMU @ 0x00000000138a2000 using 16 counters with Global(Counter0) filter settings
[    7.363848] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.8.auto: option mask 0x0
[    7.365734] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.8.auto: Registered PMU @ 0x00000000138c2000 using 16 counters with Global(Counter0) filter settings
[    7.366182] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.9.auto: option mask 0x0
[    7.366845] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.9.auto: Registered PMU @ 0x00000000138e2000 using 16 counters with Global(Counter0) filter settings
[    7.366967] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.10.auto: option mask 0x0
[    7.367906] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.10.auto: Registered PMU @ 0x0000000013002000 using 32 counters with Global(Counter0) filter settings
[    7.368182] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.11.auto: option mask 0x0
[    7.368759] sbsa-gwdt sbsa-gwdt.0: Initialized with 10s timeout @ 1000000000 Hz, action=1. [enabled]
[    7.368805] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.11.auto: Registered PMU @ 0x0000000013042000 using 16 counters with Global(Counter0) filter settings
[    7.368999] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.12.auto: option mask 0x0
[    7.370003] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.12.auto: Registered PMU @ 0x0000000013062000 using 16 counters with Global(Counter0) filter settings
[    7.370107] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.13.auto: option mask 0x0
[    7.370157] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.13.auto: Registered PMU @ 0x0000000013082000 using 16 counters with Global(Counter0) filter settings
[    7.370184] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.14.auto: option mask 0x0
[    7.370531] cdc_acm 1-1.1:1.1: probe with driver cdc_acm failed with error -22
[    7.371099] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.14.auto: Registered PMU @ 0x00000000130a2000 using 16 counters with Global(Counter0) filter settings
[    7.371643] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.15.auto: option mask 0x0
[    7.371868] usbcore: registered new interface driver cdc_acm
[    7.371870] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
[    7.373714] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.15.auto: Registered PMU @ 0x00000000130c2000 using 16 counters with Global(Counter0) filter settings
[    7.376028] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.16.auto: option mask 0x0
[    7.376764] processor cpu13: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.377447] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.16.auto: Registered PMU @ 0x00000000130e2000 using 16 counters with Global(Counter0) filter settings
[    7.377760] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.17.auto: option mask 0x0
[    7.377911] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.17.auto: Registered PMU @ 0x0000000014902000 using 32 counters with Global(Counter0) filter settings
[    7.377958] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.18.auto: option mask 0x0
[    7.378012] processor cpu14: EM: CPUs of 0-4,10-14 must have the same capacity
[    7.378269] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.18.auto: Registered PMU @ 0x0000000014942000 using 16 counters with Global(Counter0) filter settings
[    7.379036] processor cpu15: EM: CPUs of 15-19 must have the same capacity
[    7.379104] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.19.auto: option mask 0x0
[    7.379447] Loading iSCSI transport class v2.0-870.
[    7.379993] processor cpu16: EM: CPUs of 15-19 must have the same capacity
[    7.379996] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.19.auto: Registered PMU @ 0x0000000014962000 using 16 counters with Global(Counter0) filter settings
[    7.380933] processor cpu17: EM: CPUs of 15-19 must have the same capacity
[    7.383634] processor cpu18: EM: CPUs of 15-19 must have the same capacity
[    7.385432] processor cpu19: EM: CPUs of 15-19 must have the same capacity
[    7.397431] iscsi: registered transport (iser)
[    7.419325] nvidia-nvlink: Nvlink Core is being initialized, major device number 500

[    7.423251] nvidia 000f:01:00.0: vgaarb: VGA decodes changed: olddecodes=io+mem,decodes=none:owns=none
[    7.426649] input: NVIDIA HDMI/DP,pcm=3 as /devices/platform/NVDA2014:00/sound/card0/input3
[    7.428082] input: NVIDIA HDMI/DP,pcm=7 as /devices/platform/NVDA2014:00/sound/card0/input4
[    7.428301] RPC: Registered rdma transport module.
[    7.428305] RPC: Registered rdma backchannel transport module.
[    7.428681] Bluetooth: Core ver 2.22
[    7.429292] NET: Registered PF_BLUETOOTH protocol family
[    7.429296] Bluetooth: HCI device and connection manager initialized
[    7.429301] Bluetooth: HCI socket layer initialized
[    7.429304] Bluetooth: L2CAP socket layer initialized
[    7.429313] Bluetooth: SCO socket layer initialized
[    7.431050] mt7925e 0009:01:00.0: enabling device (0000 -> 0002)
[    7.432036] input: NVIDIA HDMI/DP,pcm=8 as /devices/platform/NVDA2014:00/sound/card0/input5
[    7.437590] mt7925e 0009:01:00.0: ASIC revision: 79250000
[    7.443355] input: NVIDIA HDMI/DP,pcm=9 as /devices/platform/NVDA2014:00/sound/card0/input6
[    7.463443] Bluetooth: hci0: HW/SW Version: 0x00000000, Build Time: 20251210093205
[    7.466113] usbcore: registered new interface driver btusb
[    7.513130] mt7925e 0009:01:00.0: HW/SW Version: 0x8a108a10, Build Time: 20251210092928a

[    7.861127] mt7925e 0009:01:00.0: WM Firmware Version: ____000000, Build Time: 20251210093025
[    7.884439] audit: type=1400 audit(1779270259.957:2): apparmor="STATUS" operation="profile_load" profile="unconfined" name="1password" pid=1296 comm="apparmor_parser"
[    7.884444] audit: type=1400 audit(1779270259.957:3): apparmor="STATUS" operation="profile_load" profile="unconfined" name="evolution" pid=1314 comm="apparmor_parser"
[    7.884446] audit: type=1400 audit(1779270259.957:4): apparmor="STATUS" operation="profile_load" profile="unconfined" name="element-desktop" pid=1312 comm="apparmor_parser"
[    7.884448] audit: type=1400 audit(1779270259.957:5): apparmor="STATUS" operation="profile_load" profile="unconfined" name="crun" pid=1309 comm="apparmor_parser"
[    7.884450] audit: type=1400 audit(1779270259.957:6): apparmor="STATUS" operation="profile_load" profile="unconfined" name="QtWebEngineProcess" pid=1299 comm="apparmor_parser"
[    7.884452] audit: type=1400 audit(1779270259.957:7): apparmor="STATUS" operation="profile_load" profile="unconfined" name="balena-etcher" pid=1301 comm="apparmor_parser"
[    7.884453] audit: type=1400 audit(1779270259.957:8): apparmor="STATUS" operation="profile_load" profile="unconfined" name="Discord" pid=1297 comm="apparmor_parser"
[    7.884455] audit: type=1400 audit(1779270259.957:9): apparmor="STATUS" operation="profile_load" profile="unconfined" name="desktop-icons-ng" pid=1310 comm="apparmor_parser"
[    7.884457] audit: type=1400 audit(1779270259.957:10): apparmor="STATUS" operation="profile_load" profile="unconfined" name="ch-run" pid=1306 comm="apparmor_parser"
[    7.884458] audit: type=1400 audit(1779270259.957:11): apparmor="STATUS" operation="profile_load" profile="unconfined" name="buildah" pid=1303 comm="apparmor_parser"
[    8.098221] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[    8.098229] Bluetooth: BNEP filters: protocol multicast
[    8.098234] Bluetooth: BNEP socket layer initialized
[    8.129403] NOTICE: Automounting of tracing to debugfs is deprecated and will be removed in 2030
[    8.129409] NOTICE: Automounting of tracing to debugfs is deprecated and will be removed in 2030
[    8.257268] NET: Registered PF_QIPCRTR protocol family
[    8.443798] mt7925e 0009:01:00.0 wlP9s9: renamed from wlan0
[    8.867173] mlx5_core 0002:01:00.0 enP2p1s0f0np0: Link down
[    9.195620] mlx5_core 0002:01:00.1 enP2p1s0f1np1: Link up
[    9.199438] enP7s7: 0xffff800085ac0000, 4c:bb:47:2f:fb:97, IRQ 337
[    9.199493] mlx5_core 0002:01:00.1 roceP2p1s0f1: Port: 1 Link ACTIVE
[    9.409549] mlx5_core 0000:01:00.0 enp1s0f0np0: Link down
[    9.422681] Bluetooth: hci0: Device setup in 1914455 usecs
[    9.422689] Bluetooth: hci0: HCI Enhanced Setup Synchronous Connection command is advertised, but not supported.
[    9.509486] Bluetooth: hci0: AOSP extensions version v1.00
[    9.509493] Bluetooth: hci0: AOSP quality report is supported
[    9.509719] Bluetooth: MGMT ver 1.23
[    9.515410] NET: Registered PF_ALG protocol family
[    9.658474] mlx5_core 0000:01:00.1 enp1s0f1np1: Link up
[    9.660816] mlx5_core 0000:01:00.1 rocep1s0f1: Port: 1 Link ACTIVE
[    9.662521] asix 1-1.3:1.0 enx000ec645a2b4: configuring for phy/internal link mode
[    9.741451] warning: `lldpd' uses wireless extensions which will stop working for Wi-Fi 7 hardware; use nl80211
[    9.756496] netfs: FS-Cache loaded
[    9.828949] NFS: Registering the id_resolver key type
[    9.828961] Key type id_resolver registered
[    9.828962] Key type id_legacy registered
[    9.869104] NFSD: Using nfsdcld client tracking operations.
[    9.869107] NFSD: no clients to reclaim, skipping NFSv4 grace period (net effffff9)
[   10.407686] NVRM: loading NVIDIA UNIX Open Kernel Module for aarch64  580.142  Release Build  (dvs-builder@U22-I3-H10-02-1)  Tue Mar  3 19:08:06 UTC 2026
[   10.420830] nvidia-modeset: Loading NVIDIA UNIX Open Kernel Mode Setting Driver for aarch64  580.142  Release Build  (dvs-builder@U22-I3-H10-02-1)  Tue Mar  3 18:57:53 UTC 2026
[   10.676915] Initializing XFRM netlink socket
[   10.695246] bridge: filtering via arp/ip/ip6tables is no longer available by default. Update your scripts to load br_netfilter if you need this.
[   12.269710] [drm] [nvidia-drm] [GPU ID 0x000f0100] Loading driver
[   12.269879] [drm] Initialized nvidia-drm 0.0.0 for 000f:01:00.0 on minor 0
[   12.645422] Bluetooth: RFCOMM TTY layer initialized
[   12.645433] Bluetooth: RFCOMM socket layer initialized
[   12.645440] Bluetooth: RFCOMM ver 1.11
[   12.709524] evm: overlay not supported
[   12.764629] docker0: port 1(veth4ee4a3f) entered blocking state
[   12.764635] docker0: port 1(veth4ee4a3f) entered disabled state
[   12.764639] veth4ee4a3f: entered allmulticast mode
[   12.764670] veth4ee4a3f: entered promiscuous mode
[   12.772243] eth0: renamed from vethb9aeb9e
[   12.773093] docker0: port 1(veth4ee4a3f) entered blocking state
[   12.773098] docker0: port 1(veth4ee4a3f) entered forwarding state
[   13.614214] r8127: enP7s7: link up
[   14.276723] rfkill: input handler disabled
[   15.238979] wlP9s9: authenticate with f0:99:bf:09:9f:ce (local address=f8:3d:c6:56:9a:8c)
[   15.356526] wlP9s9: send auth to f0:99:bf:09:9f:ce (try 1/3)
[   15.358791] wlP9s9: authenticated
[   15.360561] wlP9s9: associate with f0:99:bf:09:9f:ce (try 1/3)
[   15.383890] wlP9s9: RX AssocResp from f0:99:bf:09:9f:ce (capab=0x1411 status=0 aid=62)
[   15.425687] wlP9s9: associated
[   15.428939] wlP9s9: Limiting TX power to 20 (20 - 0) dBm as advertised by f0:99:bf:09:9f:ce
[   18.567444] kauditd_printk_skb: 149 callbacks suppressed
[   18.567449] audit: type=1400 audit(1779270270.640:161): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache_systemd_detect_virt" pid=3342 comm="systemd-detect-" capability=38  capname="perfmon"
[   18.569015] audit: type=1400 audit(1779270270.642:162): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache//cloud_id" pid=3299 comm="cloud-id" capability=38  capname="perfmon"
[   83.955753] systemd-journald[622]: /var/log/journal/c8a82ac6119c430e8201416ac2b5d339/user-1000.journal: Journal file uses a different sequence number ID, rotating.
[  322.102491] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  323.197808] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[  620.122600] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  622.213446] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[  661.136989] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  663.262498] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[  910.077329] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  911.179649] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[  915.320953] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  916.424123] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[  946.167257] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[  947.271687] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1094.772228] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1095.865757] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1132.423571] mlx5_core 0000:01:00.0: mlx5_core_test_wc:383:(pid 23459): Write combining is not supported
[ 1132.437817] mlx5_core 0000:01:00.1: mlx5_core_test_wc:383:(pid 23459): Write combining is not supported
[ 1132.452316] mlx5_core 0002:01:00.0: mlx5_core_test_wc:383:(pid 23459): Write combining is not supported
[ 1132.462800] mlx5_core 0002:01:00.1: mlx5_core_test_wc:383:(pid 23459): Write combining is not supported
[ 1132.554970] nvidia 000f:01:00.0: Using 40-bit DMA addresses
[ 1368.254244] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1369.337539] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1513.203355] mlx5_core 0002:01:00.1 enP2p1s0f1np1: Link down
[ 1513.203368] mlx5_core 0000:01:00.1 enp1s0f1np1: Link down
[ 1513.205872] mlx5_core 0002:01:00.1 roceP2p1s0f1: Port: 1 Link DOWN
[ 1513.206894] mlx5_core 0000:01:00.1 rocep1s0f1: Port: 1 Link DOWN
[ 1519.166359] mlx5_core 0000:01:00.1: Port module event: module 1, Cable unplugged
[ 1519.166373] mlx5_core 0002:01:00.1: Port module event: module 1, Cable unplugged
[ 1519.194233] mlx5_core 0000:01:00.0: E-Switch: Unload vfs: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1519.207388] mlx5_core 0000:01:00.0: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1522.690320] mlx5_core 0000:01:00.0: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1523.205170] mlx5_core 0000:01:00.0: E-Switch: cleanup
[ 1523.536983] mlx5_core 0000:01:00.1: E-Switch: Unload vfs: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1523.548295] mlx5_core 0000:01:00.1: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1526.172152] mlx5_core 0002:01:00.0: Port module event: module 0, Cable plugged
[ 1526.568162] mlx5_core 0002:01:00.0 enP2p1s0f0np0: Link up
[ 1526.569210] mlx5_core 0002:01:00.0 roceP2p1s0f0: Port: 1 Link ACTIVE
[ 1528.831323] mlx5_core 0000:01:00.1: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1529.401401] mlx5_core 0000:01:00.1: E-Switch: cleanup
[ 1529.733310] mlx5_core 0002:01:00.0: E-Switch: Unload vfs: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1529.757199] mlx5_core 0002:01:00.0: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1533.678229] mlx5_core 0002:01:00.0: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1534.351723] mlx5_core 0002:01:00.0: E-Switch: cleanup
[ 1534.670853] mlx5_core 0002:01:00.1: E-Switch: Unload vfs: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1534.691124] mlx5_core 0002:01:00.1: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1539.529128] mlx5_core 0002:01:00.1: E-Switch: Disable: mode(LEGACY), nvfs(0), necvfs(0), active vports(0)
[ 1540.044013] mlx5_core 0002:01:00.1: E-Switch: cleanup
[ 1540.497688] cx7-pcie-hotplug MTKP0001:00: Cable removal
[ 1540.497794] pcieport 0000:00:00.0: AER: Multiple Correctable error message received from 0000:00:00.0
[ 1540.497817] pcieport 0000:00:00.0: PCIe Bus Error: severity=Correctable, type=Physical Layer, (Receiver ID)
[ 1540.497817] pcieport 0002:00:00.0: AER: Correctable error message received from 0002:00:00.0
[ 1540.497820] pcieport 0000:00:00.0:   device [10de:22ce] error status/mask=00000001/0000e000
[ 1540.497822] pcieport 0000:00:00.0:    [ 0] RxErr                  (First)
[ 1540.497830] pcieport 0002:00:00.0: PCIe Bus Error: severity=Correctable, type=Physical Layer, (Receiver ID)
[ 1540.497833] pcieport 0002:00:00.0:   device [10de:22ce] error status/mask=00000001/0000e000
[ 1540.497835] pcieport 0002:00:00.0:    [ 0] RxErr                  (First)
[ 1543.510054] cx7-pcie-hotplug MTKP0001:00: Cable plugin
[ 1545.210279] pcieport 0000:00:00.0: AER: Multiple Correctable error message received from 0000:ff:1f.7 (no details found
[ 1545.210289] pcieport 0000:00:00.0: AER: Multiple Uncorrectable (Fatal) error message received from 0000:ff:1f.7 (no details found
[ 1545.221008] pcieport 0002:00:00.0: AER: Multiple Correctable error message received from 0002:ff:1f.7 (no details found
[ 1545.221024] pcieport 0002:00:00.0: AER: Multiple Uncorrectable (Fatal) error message received from 0002:ff:1f.7 (no details found
[ 1546.512342] pci 0000:01:00.0: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[ 1546.512602] pci 0000:01:00.0: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[ 1546.512633] pci 0000:01:00.0: ROM [mem 0x00000000-0x000fffff pref]
[ 1546.513510] pci 0000:01:00.0: PME# supported from D3cold
[ 1546.513858] pci 0000:01:00.0: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[ 1546.513860] pci 0000:01:00.0: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[ 1546.515290] pci 0000:01:00.0: Adding to iommu group 7
[ 1546.515366] pcieport 0000:00:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1546.515417] pci 0000:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[ 1546.515760] pci 0000:01:00.1: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[ 1546.516002] pci 0000:01:00.1: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[ 1546.516032] pci 0000:01:00.1: ROM [mem 0x00000000-0x000fffff pref]
[ 1546.516629] pci 0000:01:00.1: PME# supported from D3cold
[ 1546.516969] pci 0000:01:00.1: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[ 1546.516970] pci 0000:01:00.1: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[ 1546.517833] pci 0000:01:00.1: Adding to iommu group 8
[ 1546.517879] pcieport 0000:00:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1546.517938] pci 0000:01:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1546.517988] pci 0000:01:00.1: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[ 1546.518139] pci 0000:01:00.0: BAR 0 [mem 0x7500000000-0x7501ffffff 64bit pref]: assigned
[ 1546.518190] pci 0000:01:00.1: BAR 0 [mem 0x7502000000-0x7503ffffff 64bit pref]: assigned
[ 1546.518240] pci 0000:01:00.0: ROM [mem 0x67100000-0x671fffff pref]: assigned
[ 1546.518242] pci 0000:01:00.0: VF BAR 0 [mem 0x7504000000-0x75047fffff 64bit pref]: assigned
[ 1546.518272] pci 0000:01:00.1: ROM [mem 0x67200000-0x672fffff pref]: assigned
[ 1546.518273] pci 0000:01:00.1: VF BAR 0 [mem 0x7504800000-0x7504ffffff 64bit pref]: assigned
[ 1546.521031] mlx5_core 0000:01:00.0: enabling device (0000 -> 0002)
[ 1546.521159] mlx5_core 0000:01:00.0: firmware version: 28.45.4028
[ 1546.521179] mlx5_core 0000:01:00.0: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[ 1546.894479] mlx5_core 0000:01:00.0: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[ 1546.895186] mlx5_core 0000:01:00.0: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[ 1546.897354] mlx5_core 0000:01:00.0: Flow counters bulk query buffer size increased, bulk_query_len(8)
[ 1546.900659] mlx5_core 0000:01:00.0: mlx5_pcie_event:322:(pid 23529): PCIe slot power capability was not advertised.
[ 1546.909837] mlx5_core 0000:01:00.0: mlx5e: IPSec ESP acceleration enabled
[ 1547.037989] mlx5_core 0000:01:00.0: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[ 1547.040760] mlx5_core 0000:01:00.0 enp1s0f0np0: renamed from eth0
[ 1547.058663] mlx5_core 0000:01:00.1: enabling device (0000 -> 0002)
[ 1547.058987] mlx5_core 0000:01:00.1: firmware version: 28.45.4028
[ 1547.059041] mlx5_core 0000:01:00.1: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[ 1547.561216] mlx5_core 0000:01:00.1: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[ 1547.561631] mlx5_core 0000:01:00.1: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[ 1547.564610] mlx5_core 0000:01:00.1: Flow counters bulk query buffer size increased, bulk_query_len(8)
[ 1547.570192] mlx5_core 0000:01:00.1: Port module event: module 1, Cable unplugged
[ 1547.570481] mlx5_core 0000:01:00.1: mlx5_pcie_event:322:(pid 402): PCIe slot power capability was not advertised.
[ 1547.572404] mlx5_core 0000:01:00.0 enp1s0f0np0: Link down
[ 1547.578336] mlx5_core 0000:01:00.1: mlx5e: IPSec ESP acceleration enabled
[ 1547.723793] mlx5_core 0000:01:00.1: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[ 1547.725428] mlx5_core 0000:01:00.1 enp1s0f1np1: renamed from eth0
[ 1547.742046] pci 0002:01:00.0: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[ 1547.742327] pci 0002:01:00.0: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[ 1547.742375] pci 0002:01:00.0: ROM [mem 0x00000000-0x000fffff pref]
[ 1547.743465] pci 0002:01:00.0: PME# supported from D3cold
[ 1547.743951] pci 0002:01:00.0: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[ 1547.743953] pci 0002:01:00.0: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[ 1547.745651] pci 0002:01:00.0: Adding to iommu group 10
[ 1547.745712] pcieport 0002:00:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1547.745769] pci 0002:01:00.0: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[ 1547.746130] pci 0002:01:00.1: [15b3:1021] type 00 class 0x020000 PCIe Endpoint
[ 1547.746438] pci 0002:01:00.1: BAR 0 [mem 0x00000000-0x01ffffff 64bit pref]
[ 1547.746491] pci 0002:01:00.1: ROM [mem 0x00000000-0x000fffff pref]
[ 1547.747280] pci 0002:01:00.1: PME# supported from D3cold
[ 1547.747625] pci 0002:01:00.1: VF BAR 0 [mem 0x00000000-0x000fffff 64bit pref]
[ 1547.747626] pci 0002:01:00.1: VF BAR 0 [mem 0x00000000-0x007fffff 64bit pref]: contains BAR 0 for 8 VFs
[ 1547.748697] pci 0002:01:00.1: Adding to iommu group 11
[ 1547.748734] pcieport 0002:00:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1547.748803] pci 0002:01:00.0: Max Payload Size set to  512/ 512 (was  512), Max Read Rq  512
[ 1547.748890] pci 0002:01:00.1: Max Payload Size set to  512/ 512 (was  128), Max Read Rq  512
[ 1547.749069] pci 0002:01:00.0: BAR 0 [mem 0x3d00000000-0x3d01ffffff 64bit pref]: assigned
[ 1547.749138] pci 0002:01:00.1: BAR 0 [mem 0x3d02000000-0x3d03ffffff 64bit pref]: assigned
[ 1547.749213] pci 0002:01:00.0: ROM [mem 0x5d100000-0x5d1fffff pref]: assigned
[ 1547.749214] pci 0002:01:00.0: VF BAR 0 [mem 0x3d04000000-0x3d047fffff 64bit pref]: assigned
[ 1547.749245] pci 0002:01:00.1: ROM [mem 0x5d200000-0x5d2fffff pref]: assigned
[ 1547.749247] pci 0002:01:00.1: VF BAR 0 [mem 0x3d04800000-0x3d04ffffff 64bit pref]: assigned
[ 1547.751348] mlx5_core 0002:01:00.0: enabling device (0000 -> 0002)
[ 1547.751473] mlx5_core 0002:01:00.0: firmware version: 28.45.4028
[ 1547.751496] mlx5_core 0002:01:00.0: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[ 1548.245916] mlx5_core 0002:01:00.0: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[ 1548.246516] mlx5_core 0002:01:00.0: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[ 1548.250224] mlx5_core 0002:01:00.0: Flow counters bulk query buffer size increased, bulk_query_len(8)
[ 1548.260761] mlx5_core 0000:01:00.1 enp1s0f1np1: Link down
[ 1548.262023] mlx5_core 0002:01:00.0: mlx5_pcie_event:322:(pid 23507): PCIe slot power capability was not advertised.
[ 1548.262816] mlx5_core 0002:01:00.0: mlx5e: IPSec ESP acceleration enabled
[ 1548.264562] mlx5_core 0000:01:00.0: Port module event: module 0, Cable plugged
[ 1548.264665] mlx5_core 0000:01:00.0: mlx5_pcie_event:326:(pid 23507): Detected insufficient power on the PCIe slot (27W).
[ 1548.264722] mlx5_core 0002:01:00.0: Port module event: module 0, Cable plugged
[ 1548.264742] mlx5_core 0000:01:00.1: mlx5_pcie_event:326:(pid 466): Detected insufficient power on the PCIe slot (27W).
[ 1548.264856] mlx5_core 0002:01:00.0: mlx5_pcie_event:326:(pid 23397): Detected insufficient power on the PCIe slot (27W).
[ 1548.403138] mlx5_core 0002:01:00.0: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[ 1548.404429] mlx5_core 0002:01:00.0 enP2p1s0f0np0: renamed from eth0
[ 1548.426857] mlx5_core 0002:01:00.1: enabling device (0000 -> 0002)
[ 1548.427195] mlx5_core 0002:01:00.1: firmware version: 28.45.4028
[ 1548.427244] mlx5_core 0002:01:00.1: 126.028 Gb/s available PCIe bandwidth (32.0 GT/s PCIe x4 link)
[ 1548.722684] mlx5_core 0000:01:00.0 enp1s0f0np0: Link up
[ 1548.927690] mlx5_core 0002:01:00.1: Rate limit: 127 rates are supported, range: 0Mbps to 195312Mbps
[ 1548.928162] mlx5_core 0002:01:00.1: E-Switch: Total vports 10, per vport: max uc(128) max mc(2048)
[ 1548.929573] mlx5_core 0002:01:00.1: Flow counters bulk query buffer size increased, bulk_query_len(8)
[ 1548.943248] mlx5_core 0002:01:00.1: mlx5e: IPSec ESP acceleration enabled
[ 1548.943933] mlx5_core 0002:01:00.1: Port module event: module 1, Cable unplugged
[ 1548.944194] mlx5_core 0002:01:00.1: mlx5_pcie_event:326:(pid 402): Detected insufficient power on the PCIe slot (27W).
[ 1548.951863] mlx5_core 0002:01:00.0 enP2p1s0f0np0: Link up
[ 1548.952202] mlx5_core 0002:01:00.0 enP2p1s0f0np0: Link up
[ 1548.953751] mlx5_core 0000:01:00.0 rocep1s0f0: Port: 1 Link ACTIVE
[ 1548.954428] mlx5_core 0002:01:00.0 roceP2p1s0f0: Port: 1 Link ACTIVE
[ 1549.100897] mlx5_core 0002:01:00.1: MLX5E: StrdRq(1) RqSz(8) StrdSz(2048) RxCqeCmprss(0 enhanced)
[ 1549.102703] mlx5_core 0002:01:00.1 enP2p1s0f1np1: renamed from eth0
[ 1549.580959] mlx5_core 0002:01:00.1 enP2p1s0f1np1: Link down
[ 1650.936890] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1652.023711] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1692.983774] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1694.068797] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1949.043831] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1950.125743] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 1992.106415] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 1993.209917] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 2054.112498] mlx5_core 0000:01:00.0: mlx5_core_test_wc:383:(pid 23872): Write combining is not supported
[ 2054.137096] mlx5_core 0000:01:00.1: mlx5_core_test_wc:383:(pid 23872): Write combining is not supported
[ 2054.148338] mlx5_core 0002:01:00.0: mlx5_core_test_wc:383:(pid 23872): Write combining is not supported
[ 2054.163703] mlx5_core 0002:01:00.1: mlx5_core_test_wc:383:(pid 23872): Write combining is not supported
[ 2227.757145] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 2228.833155] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 2346.600756] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 2347.696645] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 2591.408332] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 2592.507654] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 2784.040717] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 2785.155120] asix 1-1.3:1.0 enx000ec645a2b4: Link is Down
[ 3174.304987] asix 1-1.3:1.0 enx000ec645a2b4: Link is Up - 100Mbps/Full - flow control rx/tx
[ 3608.520517] audit: type=1400 audit(1779273861.274:163): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache_systemd_detect_virt" pid=24044 comm="systemd-detect-" capability=38  capname="perfmon"
[ 3608.521891] audit: type=1400 audit(1779273861.275:164): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache//cloud_id" pid=24039 comm="cloud-id" capability=38  capname="perfmon"
[ 7208.506814] audit: type=1400 audit(1779277461.217:165): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache_systemd_detect_virt" pid=43713 comm="systemd-detect-" capability=38  capname="perfmon"
[ 7208.507984] audit: type=1400 audit(1779277461.218:166): apparmor="DENIED" operation="capable" class="cap" profile="ubuntu_pro_esm_cache//cloud_id" pid=43708 comm="cloud-id" capability=38  capname="perfmon"
```

