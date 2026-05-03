# DGX Spark Cluster

## 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QSFPケーブル2本 \
・LANケーブル6本 \
・[USB-Cハブ](https://www.ankerjapan.com/products/a8352?srsltid=AfmBOoq95ZKB998T5GecohoCODQpk4HWPhwSNI8mhbB-wakpWkvt89U1)（管理者nodeのRJ45増設用）


## 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \
・管理者nodeのみ研究室インターネットに接続しユーザーがログイン可能

## クラスタ構築
**[ステップ１：ipアドレスの固定](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step1)**

**[ステップ２：NFSサーバーのインストール](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step2)**

**[ステップ３：NAT（インターネット接続の共有）の設定](https://github.com/KoyoImai/dgx_spark_cluster2/blob/main/Step3/README.md)**

**[ステップ４：QSFPの2台接続]()**
