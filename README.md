# HDS Black-Box Translator
````md
# HDS Black-Box Translator（Chess / Shogi）

> **再現性（決定性）を最優先**した「説明レイヤー」。  
> エンジンを置き換えず、**エンジンの出力ログ**を **HDS（Objective → Axes → Alternatives → Risk(EVR) → Choice）** で翻訳・可視化します。

---

## Capability & Pledge（能力と誓約）
私は、モデル/エンジンの内部挙動（評価・探索・表現）を外部I/Oから推定する実装を **行い得ます**。  
しかし本リポジトリは **「翻訳（HDSによる構造化）」** に限定し、**内部推定・復元・模倣に繋がる実装や手順は、可能であっても意図的に含めません／公開しません**。

- 本コードは **正規の出力ログ（候補手・評価・PV等）** を対象とします
- 特定エンジンの競争優位を剥がす目的（逆解析/模倣）では利用しません
- 内部推定に直結するPR・issueは方針として対象外です

---

## スコープ（ブラックボックス可視化の定義）
本リポジトリが指す「可視化」は、次の範囲に限定します。

- **BB-L1（外形の構造化）**：出力（候補手/評価/ログ）をHDSで整理し、人間が追える形にする  
- **BB-L2（ログ根拠つき翻訳）**：PV/深さ推移/比較手など “ログ根拠” を添えて説明を強化する  
- **BB-L3（内部因果）**：NN内部表現や探索機構の因果まで説明する（**対象外**）

---

## 概要（2段階デモ）
このリポジトリは、次の **2段階**で「HDSでブラックボックスを翻訳できる」ことを示します。

### Phase 1（Chess）：決定性の担保と全体像
- **エンジン不要でも動く**（軽量な特徴量評価で）  
- 同一入力 → 同一出力（コメント・差分）を保証し、**“説明器の器”**を証明します

### Phase 2（Shogi）：AIログ接続による翻訳
- 将棋エンジンのログ（候補手＋評価＋（任意でPV等））を入力  
- HDS 5層ログ、要約、簡易可視化を出し、**“AIの手を翻訳して教える”**を実演します

---

## デモ1：Chess Move Explainer（決定的な説明器）
### 何をする？
- 入力：`FEN` + `UCI`（例：`e2e4`）
- 盤面を **解釈可能な特徴量**（例：material / king safety / activity）で評価
- **手を指す前→後の差分（delta）**を計算し、短いコメントを生成

### 目的
- 「強いチェスエンジン」を作るのではなく  
- 「HDS型の説明フローが **決定的に動く**」ことを見せる

---

## デモ2：Shogi Engine Decision Visualization（ログ接続で翻訳）
### 何をする？
- 入力：`SFEN` + エンジンの候補手ログ（例：MultiPV）
- 候補手テーブル、HDSログ（5層）、日本語要約を生成
- 盤面＋評価バーを可視化（デモは簡易盤面）

### 目的
- 「なぜその一手？」を、**ログ根拠（候補手/評価/PV等）**に沿って翻訳する  
- 同一入力で必ず同一出力になる（決定性チェックで保証）

---

## 入力フォーマット
### Chess
- `fen`: FEN文字列
- `move_uci`: UCI手（例：`e2e4`）
- `perspective`: `"white"` / `"black"`

### Shogi（ログ取り込み用の最小スキーマ例）
```json
{
  "position_id": "any_id",
  "sfen": "....",
  "side_to_move": "black",
  "candidate_moves": [
    { "move": "7g7f", "eval": 300, "policy": 0.45, "label": "best", "pv": "...", "desc": "任意" },
    { "move": "2g2f", "eval": 250, "policy": 0.30, "label": "second", "pv": "...", "desc": "任意" }
  ]
}
````

* `eval` はセンチポーン等のスカラーを想定（符号の向きは運用で固定）
* `pv` / `policy` / `desc` は任意（あるほどBB-L2の忠実性が上がる）

---

## 再現性（決定性）

* 同一入力 → 同一テーブル / 同一HDSログ / 同一サマリー
* 外部API不要（オフラインで完結）
* 乱数を使う場合はseed固定（推奨）
* `Determinism Check`（複数回実行で一致検証）を用意する設計

---

## 使い方（例）

### Chess

```python
comment, deltas = explainer.explain_move(fen_string, best_move_uci, perspective="white")
print(comment)
print(deltas)
```

### Shogi

```python
df, hds_log, summary = explain_position(sample_position_dict)
print(hds_log)
print(summary)
```

---

## 参考：将棋AI 知識蒸留データセット（任意）

> ※以下は配布元の説明を **要約・転記** したものです。利用条件（ライセンス/規約）は各配布元に従ってください。

本リポジトリ自体はデータセット必須ではありませんが、将棋側の「ログ接続デモ」を作る際に、評価値付き局面データがあると便利です。

* 知識蒸留済みデータセット（約1.1億局面）
* `shogi_suisho5_depth9` をベースに、Kanadeで指し手と評価値を書き換え
* シャッフル済みで学習に使用可能
* `Eval_Coef=285` でDLモデルのvalueと評価値を変換
* 3ファイル構成：Qsearch適用psv / Qsearch不適用psv / Qsearch不適用hcpe
* バグがある可能性、品質保証なし

リンク：

* [https://huggingface.co/datasets/nodchip/shogi_suisho5_depth9](https://huggingface.co/datasets/nodchip/shogi_suisho5_depth9)
* [https://fuuppi.booth.pm/items/7108913](https://fuuppi.booth.pm/items/7108913)

---

## ライセンス

* `LICENSE` を参照（未設定の場合は追加してください）

## 免責

* 本プロジェクトは研究/デモ用途です。実運用は自己責任でお願いします。

````

```md
# HDS Black-Box Translator (Chess / Shogi)

> A reproducibility-first explanation layer.  
> This does NOT replace an engine. It translates **official engine outputs** into **HDS (Objective → Axes → Alternatives → Risk(EVR) → Choice)**.

---

## Capability & Pledge
I can implement external I/O–based inference of internal engine/model behavior (evaluation, search, representations).  
However, this repository is intentionally limited to **translation (HDS-based structuring)**, and therefore **will not include or publish implementations/procedures that enable internal recovery, reconstruction, or imitation**, even if technically feasible.

- This code consumes **official/standard output logs** (candidate moves, evals, PVs, etc.)
- It is not intended for reverse engineering or extracting competitive advantages from specific engines
- PRs/issues that move toward internal recovery are out of scope by policy

---

## Scope (What “black-box visualization” means here)
This repository defines “visualization” as:

- **BB-L1 (structured outputs)**: organize engine outputs (candidates/evals/logs) into an HDS trace  
- **BB-L2 (log-grounded translation)**: strengthen explanations with PV, depth trends, comparisons, etc.  
- **BB-L3 (internal causality)**: explaining NN/search internals causally (**out of scope**)

---

## Overview (Two-Phase Demo)
This repo demonstrates HDS-based translation in two phases:

### Phase 1 (Chess): determinism + overall pipeline
- Can run without a strong engine (uses small interpretable features)
- Proves **deterministic explanation behavior** (same input → same output)

### Phase 2 (Shogi): translation by plugging real engine logs
- Input shogi engine logs (candidate moves + eval + optional PV, etc.)
- Produces HDS 5-layer logs, summaries, and simple visualizations
- Demonstrates “AI move translation” in a verifiable form

---

## Demo 1: Chess Move Explainer (Proof of Determinism)
### What it does
- Input: `FEN` + `UCI` move (e.g., `e2e4`)
- Evaluate the position via interpretable features (e.g., material / king safety / activity)
- Compute **before→after deltas** and generate a short comment

### Goal
- Not to build a chess engine
- But to show a minimal, deterministic explanation workflow

---

## Demo 2: Shogi Engine Decision Visualization (Translation via Logs)
### What it does
- Input: `SFEN` + engine candidate-move logs (e.g., MultiPV)
- Produce a candidate table + HDS 5-layer logs + a Japanese executive summary
- Visualize the position (simple board) and eval bars

### Goal
- Translate “why this move?” into an HDS-structured narrative grounded in logs (candidates/eval/PV…)
- Guarantee determinism: same input → same output (with a determinism check)

---

## Input Formats
### Chess
- `fen`: FEN string
- `move_uci`: UCI move (e.g., `e2e4`)
- `perspective`: `"white"` / `"black"`

### Shogi (minimal schema for log ingestion)
```json
{
  "position_id": "any_id",
  "sfen": "....",
  "side_to_move": "black",
  "candidate_moves": [
    { "move": "7g7f", "eval": 300, "policy": 0.45, "label": "best", "pv": "...", "desc": "optional" },
    { "move": "2g2f", "eval": 250, "policy": 0.30, "label": "second", "pv": "...", "desc": "optional" }
  ]
}
````

* `eval` is a scalar score (centipawn-like); keep sign conventions consistent in your pipeline
* `pv` / `policy` / `desc` are optional (more fields → stronger BB-L2 faithfulness)

---

## Reproducibility (Determinism)

* Same input → same table / same HDS log / same summary
* No external APIs (offline-ready)
* If you add randomness, fix seeds (recommended)
* A “Determinism Check” verifies identical outputs across repeated runs

---

## Usage (Examples)

### Chess

```python
comment, deltas = explainer.explain_move(fen_string, best_move_uci, perspective="white")
print(comment)
print(deltas)
```

### Shogi

```python
df, hds_log, summary = explain_position(sample_position_dict)
print(hds_log)
print(summary)
```

---

## Reference: Knowledge-Distilled Shogi Dataset (Optional)

> Note: The following is a summarized/quoted description from the distributors. Please follow the license/terms on the original sources.

This repo does not require a dataset, but eval-annotated positions can help when building log-driven shogi demos.

* Knowledge-distilled dataset (~110 million positions)
* Based on `shogi_suisho5_depth9`, with moves/evals rewritten using Kanade
* Shuffled and ready for training usage
* Uses `Eval_Coef=285` to convert between DL value and evaluation score
* Three variants: Qsearch-on psv / Qsearch-off psv / Qsearch-off hcpe
* May contain bugs; no quality warranty

Links:

* [https://huggingface.co/datasets/nodchip/shogi_suisho5_depth9](https://huggingface.co/datasets/nodchip/shogi_suisho5_depth9)
* [https://fuuppi.booth.pm/items/7108913](https://fuuppi.booth.pm/items/7108913)

---

## License

* See `LICENSE` (add one if missing)

## Disclaimer

* Research/demo purposes only. Use at your own risk.
