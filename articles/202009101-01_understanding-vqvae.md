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
embeddings = torch.randn(emb_dim, num_emb)
```