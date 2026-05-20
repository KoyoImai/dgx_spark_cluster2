# ステップ11：QSFPスイッチの速度低下原因調査
QSFPスイッチ接続時に，通信速度（nccl-test）が低下する原因を調査する．

## 



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
```
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --set-fec enp1s0f0np0 encoding rs
mprg@spark-fb97:~/nccl-tests$ sudo ethtool --show-fec enp1s0f0np0
FEC parameters for enp1s0f0np0:
Supported/Configured FEC encodings: RS
Active FEC encoding: RS
```
```

```
