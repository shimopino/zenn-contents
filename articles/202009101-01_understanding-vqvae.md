---
title: "VQVAEの概要"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["機械学習"]
published: false
---


# VQVAE

## Feature Quanrization部分

```python
# D(=1つの特徴ベクトルの次元)=64, K(=保存する特徴ベクトルの数)=10
emb_dim, num_emb = 64, 10

# Feature Quantization Moduleには、畳み込み層の出力である特徴マップを使用する
# inputs: [B, C=D, H, W] --> [B, H, W, C=D]
inputs = inputs.permute(0, 2, 3, 1).contiguous()

# 特徴マップを1次元に変換する
# flatten inputs feature map: [B, H, W, D] --> [N(=BxHxW), D]
flatten = inputs.view(-1, emb_dim)

# 辞書数分格納する特徴ベクトルごとにパラメータを設定
# embedding weight: [D, K]
embedding = torch.randn(emb_dim, num_emb)

# 入力された特徴マップと、辞書の特徴ベクトルとの距離を計算する
# flatten:   [N, D]
# embedding: [D, K]
# distance: [N, K] --> 特徴マップの1つ1つの空間的位置に対する辞書ベクトルの距離
distance = (
    flatten.pow(2).sum(dim=1, keepdim=True)
    -2 * flatten @ embedding
    + embeddings.pow(2).sum(dim=0, keepdim=True)
)

# distanceから、1つ1つの空間的位置に対して、最も近い辞書ベクトルを計算する
# embedding_idx: [N, K] --> [N, ]
embedding_idx = torch.argmin(distance, dim=1)

# 辞書ベクトルの次元を入力された特徴マップに合わせる
# embedding_idx: [N, ] --> [B, H, W, ]
embedding_idx = embedding_idx.view(*inputs.size()[:-1])

# 1つ1つの空間的位置に対応する辞書Indexに対応するベクトルを抽出する
# 辞書Indexは[0, K-1]の値をとる
# quantize: [B, H, W, ] x [K, D] --> [B, H, W, D]
quantize = F.embedding(embedding_idx, embedding.transpose(0, 1))

# 量子化したベクトルを入力された特徴マップに合わせる
# quantize: [B, H, W, D] --> [B, H, W, D=C]
quantize = quantize.view(*inputs.size())
```