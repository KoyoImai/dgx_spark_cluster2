# LLMの学習（???+LoRA）

## モデルのダウンロード
管理者nodeで以下のコマンドを実行してください．
```
mkdir -p /home4cluster/models

docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/pytorch:25.05-py3 \
  python3 -c "
from huggingface_hub import snapshot_download
snapshot_download(
    repo_id='Qwen/Qwen2.5-7B-Instruct',
    local_dir='/home4cluster/models/Qwen2.5-7B-Instruct'
)
print('Download complete')
"
```
