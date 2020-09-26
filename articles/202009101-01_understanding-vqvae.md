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

損失値と勾配伝搬

```python
# 量子化前後の特徴マップの近づける
e_latent_loss = F.mse_loss(quantize.detach(), inputs         )
q_latent_loss = F.mse_loss(quantize,          inputs.detach())

# commitmentの係数を設ける
loss = q_latent_loss + commitment * e_latent_loss

# Decoder側の量子化された特徴マップに対する勾配をそのままEncoder側の入力に渡す
# 逆伝搬用の計算グラフを構築しないようにdetach()を使用する
quantize = inputs + (quantize - inputs).detach()
```

EMA

```python
# 1つ1つの空間的位置の数だけ存在する、辞書へのIndexをOneHotベクトル化する
# embedding_onehot: [N, ] --> [N, K]
embedding_onehot = F.onehot(embedding_idx, num_classes=num_emb)

# 各辞書ベクトルへの参照回数の合計を計算する
# reference_counts = [N, K] --> [K, ]
reference_counts = torch.sum(embedding_onehot, dim=0)

# 指数移動平均(EMA)を計算する
# 計算前にEMA用の参照回数を初期化する: cluster_size = torch.zeros(num_emb)
cluster_size = beta * cluster_size
               + (1 - beta) * reference_counts

# EMA用の参照回数の合計を計算する
n = cluster_size.sum()

# ラプラス平滑化を行い、参照回数が0の場合でも計算できるようにしておく
cluster_size = ((cluste_size + eps) / (n + cluster_size * eps)) * n

# OneHotベクトルを使って、参照している各辞書ベクトルの合計を計算する
# dw: [D, N] x [N, K] --> [D, K]
dw = flatten.transpose(0, 1) @ embedding_onehot

# 特徴ベクトルに対して指数移動平均を計算する
ema_embedding = beta * ema_embedding + (1 - beta) * dw

# 参照回数を正規化する
# [D, K] / [1, K]
embedding = ema_embedding / cluster_size.unsqueeze(0)
```