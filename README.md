# DGX Spark Cluster

## 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QSFPケーブル2本 \
・LANケーブル6本 \
・[USB-Cハブ](https://www.ankerjapan.com/products/a8352?srsltid=AfmBOoq95ZKB998T5GecohoCODQpk4HWPhwSNI8mhbB-wakpWkvt89U1)（管理者nodeのRJ45増設用） \
・QSFPケーブル対応スイッチ（4台までQSFPで接続可能）


## 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \
・管理者nodeのみ研究室インターネットに接続しユーザーがログイン可能

## クラスタ構築
**[ステップ001：ipアドレスの固定](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step001)**

**[ステップ002：NFSサーバーのインストール](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step002)**

**[ステップ003：NAT（インターネット接続の共有）の設定](https://github.com/KoyoImai/dgx_spark_cluster2/blob/main/Step003)**

**[ステップ004：QSFPの2台接続](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step004)**

**[ステップ005：QSFPの4台接続（2ペア＋クロス接続）](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step005)**

## LLMの学習＆推論

**[ステップ101：Dockerなどの環境構築](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step101)**

**[ステップ102：LLMの学習（Qwen2.5-7B-Instruct+LoRA+tatsu-lab/alpaca）](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step102)**

**[ステップ103：LLMの学習（Qwen2.5-7B-Instruct+フルファインチューニング+tatsu-lab/alpaca）](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step103)**

**[ステップ104：LLMの学習（Qwen2.5-7B-Instruct+LoRA+tatsu-lab/alpaca）-バッチサイズ変更-](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step104)**

**[ステップ105：LLMの学習（Qwen2.5-7B-Instruct+LoRA+tatsu-lab/alpaca）-LoRAランク変更-](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/Step105)**

**[ステップN：LLMの推論](https://github.com/KoyoImai/dgx_spark_cluster2/tree/main/StepN)**



