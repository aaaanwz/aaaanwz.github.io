---
title: "データサイエンティスト　タスクカタログ"
date: 2024-01-09
draft: true
categories:
- データサイエンス
---

## 仮説検定

### 非劣性検定
同質な群からランダムにサンプリングしてt検定を行うとp値が一様分布に従う。
→テスト/コントロール群からランダムにサンプリングしてp値算出を繰り返し、p値が一様分布になるか適合度検定する

## 因果推論

### A/B
特徴量X0~Xnと目的変数Yがある時、X1とYの因果を分析する。
X1~XnのRMSEが最小になるようにX0でセグメンテーションを行い、セグメント毎のYを算出する

### Off-Policy Evaluation

1. Direct Method
未観測の部分を予測で埋める

2. Inverse Propensity Score

3. Doubly Robust
1と2のアンサンブル

### 傾向スコアマッチング

### DID法

## サブスクリプションLTV推定

Cox比例ハザードモデル?

## 離反確率予測

BG(Beta-Geometric)モデル


## イベント回数予測

NBD(Negative Binomial Distribution)モデル

## ボリューム予測
一回あたりの購入金額など

Gamma-Gammaモデル
