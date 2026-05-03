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

## ステップ２：NFSサーバーのインストール
NFSサーバーをインストールすることで、管理者nodeの/home4clusterディレクトリを全計算nodeから同じパスで共有できるようになります。モデルファイルやスクリプトを1か所に置くだけで全nodeから使えるようになります。
まず管理者nodeでNFSサーバーをインストールします。
以下のコマンドを実行してください。
```
sudo apt install -y nfs-kernel-server
```
インストールが完了したら、各nodeで共有するディレクトリを作成します。
```
sudo mkdir -p /home4cluster
sudo chmod 777 /home4cluster
```
ディレクトリを作成したら、NFSの共有設定を行います。
`/etc/exports`に共有設定を追記します。

```
sudo bash -c 'cat >> /etc/exports << EOF

### NFS Mount of Cluster Home
/home4cluster 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF'
```
共有設定が正しく追記されているか、一応確認します。
```
mprg@spark-3894:~$ cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

### NFS Mount of Cluster Home
/home4cluster 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```
