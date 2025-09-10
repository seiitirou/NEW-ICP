
# NEW-ICP
絶対的基準馬から導き出した馬の地力と、今回の馬が、どれだけの実力を発揮できるかを予想の軸とした予想
📘 ICP × GMax 予測システム – 仕様書 v5.0
🔎 概要

目的：AIを用いた競馬予測システム

基本式：

予測スコア = 地力スコア × 出力率


最終出力：TARGET互換のCSV

各馬のICPスコア

損益分岐オッズ

🏇 地力スコア（Base Power）
コンセプト

馬の「絶対能力（MAXポテンシャル）」を数値化。
基準馬（G-Horse）＝100。

入力データ

レース日付・開催・コース（芝／ダート、距離、右左）

レースクラス（G1〜条件戦）

走破タイム

着差（馬身 → 秒換算）

斤量

ラップ（前後半／可能ならハロン毎）

通過順位

当日バイアス（逃げ／差し／内外／時計傾向）

計算手順

基準馬との差（タイム差算出）

着差補正（1馬身＝0.2秒換算）

斤量補正（0.5kg差＝0.2秒相当）

ペース補正（標準ラップとの差分補正）

当日バイアス補正（有利なら減点／不利なら加点）

レースレベル補正（クラス基準＋上位馬平均で収束）

時系列重み付け（新しいレースほど高い比重）

最終値算出（上位3走の加重平均 → G-Horse基準にスケーリング）

⚡ 出力率（Performance Output Rate）
コンセプト

「その日の発揮度合い」を表す。

要素

距離適性（±200mまで許容、それ以上は減点）

ラップ適性（脚質型とコース標準ラップの一致度）


未経験コースは「近似コースの好走率」で推定

例：東京1600m好走馬 → 阪神1800m・福島2000mでも高評価

A-index（競馬場適性指数）

内外回り・直線の長さ・坂の有無を考慮

展開補正（当日のバイアスと脚質の一致度）

XFER（交流補正）

地方→中央、海外→国内の水準差を調整

斤量補正（当日）

軽量 → 減点（恵まれ）

重量 → 加点（不利克服）

📊 勝率への変換

ソフトマックス正規化

𝑃
𝑖
=
𝑒
𝑠
𝑖
∑
𝑗
𝑒
𝑠
𝑗
P
i
	​

=
∑
j
	​

e
s
j
	​

e
s
i
	​

	​


キャリブレーション（JRA方式）

実測オッズと比較し、ロジスティック回帰で補正

💰 損益分岐オッズ

定義：

損益分岐オッズ = 1 / 勝率


例：勝率25% → 4.0倍

📂 出力仕様（TARGET互換）
出力ファイル
race_{日付}_{競馬場}{R番号}.csv
例: race_20250819_TOKYO11R.csv

CSV列構造
枠	馬番	馬名	ICPスコア	損益分岐オッズ

文字コード：Shift-JIS（TARGET対応のため）

📁 ディレクトリ構造
/runs/
  └── 20250819/
       ├── race_20250819_TOKYO11R.csv
       ├── race_20250819_TOKYO10R.csv
       ├── race_20250819_HAKODATE12R.csv
       └── day_20250819.csv   # 日次まとめ（任意）

🛠 開発フロー

データ取得（JRA-VAN, SaraD）

地力スコア算出（補正＋時系列）

出力率算出（距離・ラップ・A-index・馬場補完・斤量・XFER）

勝率化（Softmax＋Calibration）

損益分岐オッズ計算

TARGET互換CSV出力

🚀 拡張性

血統適性（父系・母父系ごとの傾向）

騎手・厩舎補正

休養明け補正（叩き2走目効果など）

機械学習によるパラメータ最適化

✅ まとめ

地力スコア = 馬の絶対能力

出力率 = 当日の発揮度合い

ICPスコア = 両者の積

損益分岐オッズ = 1 / 勝率

出力 = TARGET互換CSV（レースごと）

# 🐎 ICP×GMax 統合版 競馬予測AI 仕様書（超詳細版）

---

## 1. 基本コンセプト

- 本システムは **「馬の地力（BasePower） × 出力率（OutputRate）」** によって、レース当日のパフォーマンスを数値化する。  
- 出力は **TARGET互換形式** で表示し、以下を必ず出力する：  
  - **ICPスコア（最終予測値）**  
  - **損益分岐オッズ（その馬を買える最低限のオッズライン）**  
- データは **データベースソフト SARAD** から取得し、内部で学習・推定処理を行う。  

---

## 2. データソース

### 2.1 SARADから取得するデータ
- 出走馬情報（馬名・馬番・父母系統・年齢・性別）  
- レース情報（開催・コース・距離・馬場状態・天候・RLS関連情報）  
- 過去成績（着順、着差、通過順、ラップ、上がり3F、対戦相手）  
- 斤量、騎手、調教師など付帯情報  

### 2.2 データ前処理
- SARADのレースタイム・ラップを標準化（距離・コース差を吸収）  
- RLS（Race Level Score）は、出走馬の過去実績と照合して相対評価  
- 各馬の過去走ごとに「標準化タイム＋補正値」を計算して学習データ化  

---

## 3. 地力スコア（BasePower）

### 3.1 定義
- 架空の **絶対基準馬（どの条件でも常に100点）** を物差しに算出する。  
- 「その馬が持つ最大能力」を表現。  

### 3.2 計算式（raw値）
1. **個別レースパフォーマンス**
   \[
   P_{i,r} = 標準化タイム_{i,r} - 補正（斤量・展開・ラップ適応）
   \]

2. **最大能力抽出**
   \[
   BasePower_{i,raw} = \max_{r \in 過去レース} f(P_{i,r}, RLS_r)
   \]

- 高RLSレースは重みを大きくする（学習で最適化）。  

### 3.3 補正要素
- **斤量補正**：0.5kg差 ≈ ±補正pt（学習で係数最適化）  
- **展開補正**：位置取りや隊列長による影響を加点/減点  
- **ラップ補正**：  
  - 得意ラップから離れていて好走 → ＋補正  
  - 得意ラップ条件で凡走 → −補正  
  - 補正量はディープラーニングで学習  

### 3.4 ラップ適応力の導入
- 補正結果から「ラップズレに対する適応力」を学習  
- 適応力が高い馬は、幅広い展開でも地力スコアが安定  
- 適応力が低い馬は、理想条件を外れると地力スコアが減衰  

### 3.5 基準化処理
- **raw値**（内部値）は100点を超える場合もそのまま保存  
- **公開値**は必ず100点以下に収める  

処理手順：  
1. レース内または期間内で最大値 M を求める  
2. M > 100 の場合、全馬を以下でスケーリング：  
   \[
   BasePower_{i,out} = BasePower_{i,raw} \times \frac{100}{M}
   \]  
3. 公開用は out値、学習用は raw値を使用  

---

## 4. 出力率（OutputRate）

### 4.1 定義
- 当日の「どれくらい地力を発揮できるか」を表す係数  
- 0〜1の範囲を指数変換してスコア化  

\[
OutputRate_i = \exp(\Delta o_{course,i}+\Delta o_{form,i}+\Delta o_{bias,i}+\Delta o_{pace,i})
\]

### 4.2 各補正項目

1. **コース・距離適性補正（Δo_course）**  
   - 馬ごとに「得意コース・距離」を学習  
   - 初出走は類似条件＋統計で補完  

2. **近走フォーム補正（Δo_form）**  
   - 直近のパフォーマンス傾向を指数化  
   - 連続好走 → ＋補正、凡走続き → −補正  

3. **当日バイアス補正（Δo_bias）**  
   - 脚質有利・馬場傾向を統計的に検出  
   - 脚質・馬場特性とマッチすれば＋補正  

4. **展開補正（Δo_pace）**  
   - 各馬の理想ラップを「高RLS勝利時ラップ」から初期化  
   - 学習で latent vector として洗練  
   - 当日の予測ラップとの差分に応じて補正  
   - ズレ耐性（σ）は馬ごとに学習  

---

## 5. 最終スコア（ICPスコア）

\[
ICPScore_i = BasePower_{i,out} \times OutputRate_i
\]

---

## 6. 損益分岐オッズ（Break-even Odds）

1. 勝率計算（ソフトマックス）  
\[
WinProb_i = \frac{e^{ICPScore_i}}{\sum_j e^{ICPScore_j}}
\]

2. 損益分岐点（100%回収ライン）  
\[
BreakEvenOdds_i = \frac{1}{WinProb_i}
\]

---

## 7. 学習フロー（ディープラーニング）

1. **入力特徴量**
   - レース情報（コース、距離、天候、RLS）  
   - 馬情報（過去ラップ系列、地力履歴、脚質）  
   - 当日予測（想定ラップ、オッズ）  

2. **モジュール**
   - BasePower Head：最大能力抽出  
   - OutputRate Head：補正統合  
   - LapHead：当日ラップ予測  
   - IdealLap Encoder：理想ラップ潜在表現の学習  

3. **損失関数**
   - 予測勝率 vs 実結果（交差エントロピー）  
   - 予測ラップ vs 実ラップ（Huber損失）  
   - 加重和で最適化  

4. **最適化**
   - AdamW + 勾配クリッピング  

---

## 8. 出力形式（TARGET互換）

- 出力は **テキスト形式**（TARGETに取り込める形）。  
- 項目：馬番・馬名・ICPスコア・損益分岐オッズ。  

例：

```
馬番  馬名            ICPスコア    損益分岐オッズ
 1   サンプルホース     112.3        3.5
 2   テストホース       105.7        5.8
```

---

## 9. 実装メモ（Python例）

```python
# BasePower正規化
raw_scores = [108.2, 97.5, 85.3]
M = max(raw_scores)
if M > 100:
    base_power_out = [s * 100 / M for s in raw_scores]
else:
    base_power_out = raw_scores

# OutputRateとの統合
scores = [bp * orate for bp, orate in zip(base_power_out, output_rates)]

# 勝率と損益分岐点
exp_scores = [math.exp(s) for s in scores]
win_probs = [es / sum(exp_scores) for es in exp_scores]
breakeven_odds = [1/p for p in win_probs]

# TARGET形式出力
for num, name, sc, odds in zip(numbers, names, scores, breakeven_odds):
    print(f"{num:2d}  {name:12s}  {sc:.1f}   {odds:.1f}")
```

---

## 10. 今後の拡張
- 他券種（複勝・馬連・三連系）への対応  
- 強化学習による「回収率最大化」学習  
- 調教時計・馬体重・外厩情報など追加因子の
# 🐎 ICP×GMax 統合版 競馬予測AI 仕様書（超詳細版）

---

## 1. 基本コンセプト

- 本システムは **「馬の地力（BasePower） × 出力率（OutputRate）」** によって、レース当日のパフォーマンスを数値化する。  
- 出力は **TARGET互換形式** で表示し、以下を必ず出力する：  
  - **ICPスコア（最終予測値）**  
  - **損益分岐オッズ（その馬を買える最低限のオッズライン）**  
- データは **データベースソフト SARAD** から取得し、内部で学習・推定処理を行う。  

---

## 2. データソース

### 2.1 SARADから取得するデータ
- 出走馬情報（馬名・馬番・父母系統・年齢・性別）  
- レース情報（開催・コース・距離・馬場状態・天候・RLS関連情報）  
- 過去成績（着順、着差、通過順、ラップ、上がり3F、対戦相手）  
- 斤量、騎手、調教師など付帯情報  

### 2.2 データ前処理
- SARADのレースタイム・ラップを標準化（距離・コース差を吸収）  
- RLS（Race Level Score）は、出走馬の過去実績と照合して相対評価  
- 各馬の過去走ごとに「標準化タイム＋補正値」を計算して学習データ化  

---

## 3. 地力スコア（BasePower）

### 3.1 定義
- 架空の **絶対基準馬（どの条件でも常に100点）** を物差しに算出する。  
- 「その馬が持つ最大能力」を表現。  

### 3.2 計算式（raw値）
1. **個別レースパフォーマンス**
   \[
   P_{i,r} = 標準化タイム_{i,r} - 補正（斤量・展開・ラップ適応）
   \]

2. **最大能力抽出**
   \[
   BasePower_{i,raw} = \max_{r \in 過去レース} f(P_{i,r}, RLS_r)
   \]

- 高RLSレースは重みを大きくする（学習で最適化）。  

### 3.3 補正要素
- **斤量補正**：0.5kg差 ≈ ±補正pt（学習で係数最適化）  
- **展開補正**：位置取りや隊列長による影響を加点/減点  
- **ラップ補正**：  
  - 得意ラップから離れていて好走 → ＋補正  
  - 得意ラップ条件で凡走 → −補正  
  - 補正量はディープラーニングで学習  

### 3.4 ラップ適応力の導入
- 補正結果から「ラップズレに対する適応力」を学習  
- 適応力が高い馬は、幅広い展開でも地力スコアが安定  
- 適応力が低い馬は、理想条件を外れると地力スコアが減衰  

### 3.5 基準化処理
- **raw値**（内部値）は100点を超える場合もそのまま保存  
- **公開値**は必ず100点以下に収める  

処理手順：  
1. レース内または期間内で最大値 M を求める  
2. M > 100 の場合、全馬を以下でスケーリング：  
   \[
   BasePower_{i,out} = BasePower_{i,raw} \times \frac{100}{M}
   \]  
3. 公開用は out値、学習用は raw値を使用  

---

## 4. 出力率（OutputRate）

### 4.1 定義
- 当日の「どれくらい地力を発揮できるか」を表す係数  
- 0〜1の範囲を指数変換してスコア化  

\[
OutputRate_i = \exp(\Delta o_{course,i}+\Delta o_{form,i}+\Delta o_{bias,i}+\Delta o_{pace,i})
\]

### 4.2 各補正項目

1. **コース・距離適性補正（Δo_course）**  
   - 馬ごとに「得意コース・距離」を学習  
   - 初出走は類似条件＋統計で補完  

2. **近走フォーム補正（Δo_form）**  
   - 直近のパフォーマンス傾向を指数化  
   - 連続好走 → ＋補正、凡走続き → −補正  

3. **当日バイアス補正（Δo_bias）**  
   - 脚質有利・馬場傾向を統計的に検出  
   - 脚質・馬場特性とマッチすれば＋補正  

4. **展開補正（Δo_pace）**  
   - 各馬の理想ラップを「高RLS勝利時ラップ」から初期化  
   - 学習で latent vector として洗練  
   - 当日の予測ラップとの差分に応じて補正  
   - ズレ耐性（σ）は馬ごとに学習  

---

## 5. 最終スコア（ICPスコア）

\[
ICPScore_i = BasePower_{i,out} \times OutputRate_i
\]

---

## 6. 損益分岐オッズ（Break-even Odds）

1. 勝率計算（ソフトマックス）  
\[
WinProb_i = \frac{e^{ICPScore_i}}{\sum_j e^{ICPScore_j}}
\]

2. 損益分岐点（100%回収ライン）  
\[
BreakEvenOdds_i = \frac{1}{WinProb_i}
\]

---

## 7. 学習フロー（ディープラーニング）

1. **入力特徴量**
   - レース情報（コース、距離、天候、RLS）  
   - 馬情報（過去ラップ系列、地力履歴、脚質）  
   - 当日予測（想定ラップ、オッズ）  

2. **モジュール**
   - BasePower Head：最大能力抽出  
   - OutputRate Head：補正統合  
   - LapHead：当日ラップ予測  
   - IdealLap Encoder：理想ラップ潜在表現の学習  

3. **損失関数**
   - 予測勝率 vs 実結果（交差エントロピー）  
   - 予測ラップ vs 実ラップ（Huber損失）  
   - 加重和で最適化  

4. **最適化**
   - AdamW + 勾配クリッピング  

---

## 8. 出力形式（TARGET互換）

- 出力は **テキスト形式**（TARGETに取り込める形）。  
- 項目：馬番・馬名・ICPスコア・損益分岐オッズ。  

例：

```
馬番  馬名            ICPスコア    損益分岐オッズ
 1   サンプルホース     112.3        3.5
 2   テストホース       105.7        5.8
```

---

## 9. 実装メモ（Python例）

```python
# BasePower正規化
raw_scores = [108.2, 97.5, 85.3]
M = max(raw_scores)
if M > 100:
    base_power_out = [s * 100 / M for s in raw_scores]
else:
    base_power_out = raw_scores

# OutputRateとの統合
scores = [bp * orate for bp, orate in zip(base_power_out, output_rates)]

# 勝率と損益分岐点
exp_scores = [math.exp(s) for s in scores]
win_probs = [es / sum(exp_scores) for es in exp_scores]
breakeven_odds = [1/p for p in win_probs]

# TARGET形式出力
for num, name, sc, odds in zip(numbers, names, scores, breakeven_odds):
    print(f"{num:2d}  {name:12s}  {sc:.1f}   {odds:.1f}")
```

---

## 10. 今後の拡張
- 他券種（複勝・馬連・三連系）への対応  
- 強化学習による「回収率最大化」学習  
- 調教時計・馬体重・外厩情報など追加因子の導入  

# 🐎 ICP × GMax – 次世代競馬予測AI
**ICP × GMax** は、競走馬の「地力（絶対能力）」と「当日の発揮度（出力率）」を数値化し、  
TARGET互換の予想データを出力する競馬予測アプリです。  

---

## 🎯 目的
- 馬の **最大能力（地力スコア）** と **当日発揮度（出力率）** を掛け合わせて、予測スコアを算出。  
- **勝率・損益分岐オッズ** を出力し、「買える馬」「買えない馬」を明確にする。  
- 出力は **TARGET互換CSV** 形式。

---

## 📊 データ（SARADから取得）
- 馬情報：馬名・馬番・性齢・血統
- レース条件：開催場・距離・コース形態・馬場状態
- 過去成績：タイム・着差・ラップ（200mごと）・通過順位・上がり
- 騎手・斤量・厩舎情報

---

## 🏇 レース区間分析（E・M・L）
レースを3区間に分割して200m単位でラップを分析。

- **E（Early）**：スタート〜1角  
  - スタートダッシュ・位置取り・先行力
- **M（Middle）**：中間  
  - ペース持続力・中だるみ適性
- **L（Late）**：3-4角〜ゴール  
  - 末脚・坂適性・差し/追い込み力

例（東京1600m）：200m × 8本ラップを E/M/L に集約し、区間ごとに評価。

---

## ⚡ 地力スコア（Base Power）

### 定義
「その馬が本来持っている最大能力」  
基準馬（G-Horse）＝100

### 計算手順
1. 標準化タイムを作成  
   - 着差補正（1馬身＝0.2秒換算）  
   - 斤量補正（0.5kg＝0.2秒）  
   - ペース補正（標準ラップとの差）

2. 区間ごとにパフォーマンスを算出  
   \[
   P_E = \text{標準Eラップ} - \text{実Eラップ}
   \]

3. 区間統合  
   \[
   BasePower = w_E \cdot P_E + w_M \cdot P_M + w_L \cdot P_L
   \]

4. 基準化（最大値を100に揃える）

---

## 🔥 出力率（Output Rate）

### 定義
「当日どのくらい走れるか（0〜1）」

### 計算式
\[
OutputRate = \exp(\Delta o_{course} + \Delta o_{form} + \Delta o_{bias} + \Delta o_{pace})
\]

### 補正項目
- **Δo_course**：コース適性（右回り/左回り・距離）
- **Δo_form**：近走調子（好走続き＝＋、凡走続き＝−）
- **Δo_bias**：当日馬場傾向（前残り・差し有利）
- **Δo_pace**：E/M/L区間のズレ耐性  
  \[
  Δo_{pace} = α \cdot sim + β \cdot \exp\left(-\frac{d^2}{2σ^2}\right)
  \]  
  - `sim`：理想ラップとコース標準ラップの類似度  
  - `σ`：ズレ耐性

---

## 🏆 最終スコア
\[
ICPScore = BasePower \times OutputRate
\]

- **勝率（Softmax）**  
\[
WinProb_i = \frac{e^{ICPScore_i}}{\sum_j e^{ICPScore_j}}
\]

- **損益分岐オッズ**  
\[
Odds_i = \frac{1}{WinProb_i}
\]

---

## 📂 出力形式（TARGET互換CSV）
- 文字コード：Shift-JIS
- 列：枠, 馬番, 馬名, ICPスコア, 損益分岐オッズ

例：
```csv
枠,馬番,馬名,ICPスコア,損益分岐オッズ
3,5,サンプルホース,112.3,3.5
6,11,テストホース,105.7,5.8
わかった。ここで**“完全準備”をゼロから順番にやります。
各ステップにチェック方法**を付けるので、1つずつ進めてね。迷子になったら、そのステップの「チェック」だけ見ればOK。

⸻

ステップ1｜いま開いてるアプリを閉じる（衝突防止）
	•	Excel・メモ帳・Pythonウィンドウを閉じる（CSVが開いたままだと、更新に失敗することがあります）。

チェック：何も開いていない。

⸻

ステップ2｜フォルダ構成を固定する（迷子防止）

エクスプローラーで D:\ を開き、以下を作ります（あればそのままでOK）。

D:\IC-App\
 ├─ config\
 ├─ icp\
 ├─ runs\
 │   ├─ sample\
 │   ├─ _sanitized\
 │   └─ _logs\
 └─ scripts\

チェック：上の6フォルダが見える。runs\_sanitized と _logs がある。

⸻

ステップ3｜クリック実行の“守衛”を置く（規格ズレの自動修正）

この3ファイルを“丸ごと”貼り替えます。貼り替えたら保存。

3-1) D:\IC-App\scripts\run_icp.bat（全置換）

どのバッチを実行中か、自分のパスを最上段に出す／入力をガードで正規化してから計算／ログ保存／SJIS出力。

@echo off
setlocal ENABLEDELAYEDEXPANSION
title IC-App (Guarded Run)

REM ==== SELF CHECK ====
echo BAT    : "%~f0"

REM ==== ROOT & PY ====
set "ROOT=D:\IC-App"
set "PY=python"
where %PY% >nul 2>nul || set "PY=py"
where %PY% >nul 2>nul || (
  echo [NG] Python not found. Install Python and retry.
  pause & exit /b 1
)

REM ==== FOLDERS ====
if not exist "%ROOT%\runs" mkdir "%ROOT%\runs"
if not exist "%ROOT%\runs\sample" mkdir "%ROOT%\runs\sample"
if not exist "%ROOT%\runs\_sanitized" mkdir "%ROOT%\runs\_sanitized"
if not exist "%ROOT%\runs\_logs" mkdir "%ROOT%\runs\_logs"

REM ==== TIMESTAMP ====
for /f %%t in ('powershell -NoProfile -Command "Get-Date -Format yyyyMMdd_HHmmss"') do set TS=%%t

REM ==== INPUT (raw) ====
set "RAW_IN=%ROOT%\runs\sample\race_input.csv"
if not exist "%RAW_IN%" if exist "%ROOT%\runs\sample\race_input.txt" set "RAW_IN=%ROOT%\runs\sample\race_input.txt"
if not exist "%RAW_IN%" (
  echo [INFO] create sample: %ROOT%\runs\sample\race_input.csv
  powershell -NoProfile -Command ^
    "$p='%ROOT%\runs\sample\race_input.csv';" ^
    "$c=@'umaban,umaname,odds,speed_rating,class_rating,margin_rating,distance_fit,pace_fit
1,アルファ,3.1,97,0.95,0.10,1.10,1.05
2,ブラボー,8.6,100,0.90,0.00,0.98,0.95
3,チャーリー,4.9,95,1.00,0.15,1.05,1.02'@;" ^
    "Set-Content -Path $p -Value $c -Encoding utf8BOM"
)

REM ==== LOG ====
set "LOGFILE=%ROOT%\runs\_logs\run_%TS%.log"

REM ==== GUARD ====
echo [RUN] Guard input: "%RAW_IN%" > "%LOGFILE%"
"%PY%" "%ROOT%\scripts\icp_guard.py" --in "%RAW_IN%" --outdir "%ROOT%\runs\_sanitized" --contract "%ROOT%\config\io_contract.yaml" >> "%LOGFILE%" 2>&1
if errorlevel 1 (
  echo [NG] Guard failed. Opening log...
  start notepad.exe "%LOGFILE%"
  pause & exit /b 1
)
set "SAN_TXT=%ROOT%\runs\_sanitized\last_sanitized.txt"
if not exist "%SAN_TXT%" (
  echo [NG] Guard did not produce pointer file: "%SAN_TXT%"
  start notepad.exe "%LOGFILE%"
  pause & exit /b 1
)
set /p INCSV=<"%SAN_TXT%"

REM ==== OUTPUT ====
set "OUTCSV=%ROOT%\runs\sample\race_icp_%TS%_sjis.csv"

REM ==== SHOW PLAN ====
echo ROOT   : "%ROOT%"
echo PYTHON : "%PY%"
echo RAW_IN : "%RAW_IN%"
echo IN     : "%INCSV%"
echo OUT    : "%OUTCSV%"
echo SCRIPT : "%ROOT%\scripts\recalc_icp.py"
echo LOG    : "%LOGFILE%"

REM ==== RUN ====
"%PY%" "%ROOT%\scripts\recalc_icp.py" --race_csv "%INCSV%" --out_csv "%OUTCSV%" >> "%LOGFILE%" 2>&1
if errorlevel 1 (
  echo [NG] Python failed. See log.
  start notepad.exe "%LOGFILE%"
  pause & exit /b 1
)

REM ==== VERIFY ====
if not exist "%OUTCSV%" (
  echo [NG] Output not found: "%OUTCSV%"
  start notepad.exe "%LOGFILE%"
  pause & exit /b 1
)
for %%I in ("%OUTCSV%") do set SZ=%%~zI
if "!SZ!"=="0" (
  echo [NG] Output is zero bytes: "%OUTCSV%"
  start notepad.exe "%LOGFILE%"
  pause & exit /b 1
)

echo [OK] Done. File created: "%OUTCSV%"
powershell -NoProfile -Command "Get-Content -Path '%OUTCSV%' -TotalCount 2 | ForEach-Object { Write-Output ('[HEAD] ' + $_) }"
start "" "%ROOT%\runs\sample"
echo (Log saved) %LOGFILE%
pause
endlocal

3-2) D:\IC-App\scripts\icp_guard.py（新規・全貼り）

入力の文字コード/区切り/列名ゆれを自動で正規化し、安全な中間CSV（UTF-8+BOM, カンマ）を作ります。

# -*- coding: utf-8 -*-
from __future__ import annotations
"""
icp_guard.py
- 入力CSV/TXTを「読む→検証→正規化→中間CSVへ」する守衛。
- 文字コード: utf-8-sig/utf-8/utf-16(le/be)/cp932 を自動判定（依存無し）
- 区切り   : カンマ/タブ 自動判定 → カンマに統一
- 列名     : io_contract.yaml の同義語を正準名へ
- 必須列   : 欠ければ即エラー（ログに不足名を明記）
- 出力     : UTF-8(BOM) のカンマCSV。パスを last_sanitized.txt に書出
"""
import os, sys, argparse, datetime
import pandas as pd

def sniff_encoding(path: str) -> str:
    with open(path, 'rb') as f:
        head = f.read(4)
    if head.startswith(b'\xef\xbb\xbf'): return 'utf-8-sig'
    if head.startswith(b'\xff\xfe'):     return 'utf-16'
    if head.startswith(b'\xfe\xff'):     return 'utf-16'
    try:
        with open(path, 'r', encoding='utf-8') as _:
            return 'utf-8'
    except Exception:
        try:
            with open(path, 'r', encoding='cp932') as _:
                return 'cp932'
        except Exception:
            return 'utf-8'

def sniff_sep(path: str, enc: str) -> str:
    try:
        with open(path, 'r', encoding=enc, errors='ignore') as f:
            head = ''.join([next(f) for _ in range(3)])
        return '\t' if head.count('\t') >= head.count(',') else ','
    except Exception:
        return ','

def load_contract(path: str) -> dict:
    import yaml
    if not os.path.exists(path):
        return {
            "required": ["umaban","umaname","odds"],
            "numeric":  ["umaban","odds","speed_rating","class_rating","margin_rating","distance_fit","pace_fit","track_bias","draw_bias","weight_adj","inside_eff","front_eff"],
            "aliases": {
              "umaban":   ["枠番","馬番","番号","uma_no","uma_num"],
              "umaname":  ["馬名","name","horse_name","uma_name"],
              "odds":     ["単勝","odds_win","o"]
            }
        }
    with open(path, 'r', encoding='utf-8') as f:
        import yaml
        return yaml.safe_load(f)

def normalize_columns(cols, aliases):
    norm = []
    for c in cols:
        c0 = str(c).strip()
        lc = c0.lower()
        hit = None
        for k, syn in aliases.items():
            if lc == k.lower() or lc in [s.lower() for s in syn]:
                hit = k; break
        norm.append(hit if hit else lc)
    return norm

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--in", dest="inp", required=True)
    ap.add_argument("--outdir", required=True)
    ap.add_argument("--contract", required=True)
    a = ap.parse_args()

    enc = sniff_encoding(a.inp)
    sep = sniff_sep(a.inp, enc)
    try:
        df = pd.read_csv(a.inp, encoding=enc, sep=sep, engine="python")
    except Exception as e:
        print(f"[GUARD][ERROR] read failed: enc={enc} sep={'TAB' if sep=='\\t' else 'COMMA'} path={a.inp}")
        print(repr(e)); sys.exit(1)

    contract = load_contract(a.contract)
    required = [c.lower() for c in contract.get("required", [])]
    numeric  = [c.lower() for c in contract.get("numeric", [])]
    aliases  = contract.get("aliases", {})

    df.columns = normalize_columns(df.columns, aliases)
    missing = [r for r in required if r not in df.columns]
    if missing:
        print(f"[GUARD][ERROR] missing required columns: {missing}")
        print(f"[GUARD][INFO] columns found: {list(df.columns)}")
        sys.exit(2)

    for c in numeric:
        if c in df.columns:
            df[c] = pd.to_numeric(df[c], errors="coerce")

    if len(df) == 0:
        print("[GUARD][ERROR] input has 0 rows"); sys.exit(3)

    os.makedirs(a.outdir, exist_ok=True)
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    out_path = os.path.join(a.outdir, f"race_input_%s_sanitized.csv" % ts)
    df.to_csv(out_path, index=False, encoding="utf-8-sig")

    with open(os.path.join(a.outdir, "last_sanitized.txt"), "w", encoding="utf-8") as f:
        f.write(out_path)

    print(f"[GUARD][OK] enc={enc} sep={'TAB' if sep=='\\t' else 'COMMA'} -> {out_path}")

if __name__ == "__main__":
    main()

3-3) D:\IC-App\config\io_contract.yaml（新規・全貼り）

必須列・数値列・同義語マッピング（列名ゆれの吸収）を定義。

required:
  - umaban
  - umaname
  - odds

numeric:
  - umaban
  - odds
  - speed_rating
  - class_rating
  - margin_rating
  - distance_fit
  - pace_fit
  - track_bias
  - draw_bias
  - weight_adj
  - inside_eff
  - front_eff

aliases:
  umaban:   ["枠番","馬番","番号","uma_no","uma_num"]
  umaname:  ["馬名","name","horse_name","uma_name"]
  odds:     ["単勝","odds_win","o"]

  speed_rating:  ["speed","spd","speed_index"]
  class_rating:  ["class","cls","class_index"]
  margin_rating: ["margin","mg","margin_sec"]
  distance_fit:  ["dist_fit","dfit"]
  pace_fit:      ["pacefit","pfit"]
  track_bias:    ["t_bias","tbias"]
  draw_bias:     ["d_bias","dbias","frame_bias"]
  weight_adj:    ["w_adj","weightadj"]

  inside_eff:    ["inside_bias","inside_today","in_eff"]
  front_eff:     ["front_bias","front_today","fr_eff"]

チェック：
	•	scripts\run_icp.bat を開き、1行目に IC-App (Guarded Run)、冒頭に echo BAT : "%~f0" がある。
	•	scripts\icp_guard.py がある。
	•	config\io_contract.yaml がある。

⸻

ステップ4｜スコア計算ロジック一式を最新に（差し替え済みならそのまま）

以下がすでに動いている最新なら触らなくてOK。もし古いか不明なら、前回渡した差し替え版を丸ごと上書きしてください。
	•	D:\IC-App\scripts\recalc_icp.py（先頭に sys.path.insert(0, ROOT) があること／出力は cp932）
	•	D:\IC-App\icp\base_power.py（レースレベル補正 λ×(Top3平均−100) を実装済み）
	•	D:\IC-App\icp\output_rate.py（相互作用 pack×slow, string×fast、当日効きを合成）
	•	D:\IC-App\icp\icp_core.py（ICP→p_win→BE/EV、人気帯校正＆buy_flag）

クイック確認（必ずあるべき行）：
	•	recalc_icp.py：
	•	sys.path.insert(0, ROOT)
	•	out.to_csv(... encoding="cp932"
	•	icp_core.py：
	•	out["RaceLevel"] = race_level
	•	out["EV"] = EV と out["buy_flag"] = (out["ICP_rank"] <= 3) & buy_flag

チェック：上の“あるべき行”がファイル内にある。

⸻

ステップ5｜設定ファイル（config.yaml）を置く

D:\IC-App\config\config.yaml を、前に渡した学習・運用用の内容で保存（温度や重み）。
※ 迷ったら、そのままでOK。後からDLで上書きできます。

チェック：config.yaml が存在する（無くても recalc_icp.py はデフォルトで動きます）。

⸻

ステップ6｜初回テスト実行（自動サンプルでOK）
	1.	何も置いていない状態で、D:\IC-App\scripts\run_icp.bat をダブルクリック。
	2.	黒画面の最上段に BAT : "D:\IC-App\scripts\run_icp.bat" と出る。
	3.	途中に RAW_IN や IN、OUT、LOG のパスが出る。
	4.	最後に
	•	[OK] Done. File created: "…race_icp_<時刻>_sjis.csv"
	•	[HEAD] umaban,umaname,odds,...,BE_odds
	•	[HEAD] 1,アルファ,3.1,...
が表示され、runs\sample\ フォルダが開く。

チェック：
	•	runs\sample\race_icp_＜今の時刻＞_sjis.csv ができている。
	•	Excelで開くと馬名が正しく日本語表示。
	•	runs\_logs\run_＜時刻＞.log がある。

⸻

ステップ7｜あなたの入力データで実行
	1.	D:\IC-App\runs\sample\race_input.csv を上書き（ヘッダ名は日本語でもOK。ガードが直します）。
	2.	もう一度 run_icp.bat をダブルクリック。
	3.	新しい時刻のCSVが出る → Excelで確認。

チェック：
	•	ICP_rank が 1,2,3 と並ぶ。
	•	p_win の合計 ≈ 1.0。
	•	BE_odds = (1 - 0.20) / p_win の関係が合う。
	•	buy_flag がTRUEの馬がいるか（Top3＆EV閾値クリア）。

⸻

トラブル早見表（ここだけ見れば直せる）
	•	古いbatを実行していた
→ 画面の最上段 BAT : “…” を見る。D:\IC-App\scripts\run_icp.bat じゃなければ、そのファイルを上書き。
	•	[GUARD][ERROR] missing required columns
→ 入力ヘッダが不足。io_contract.yaml の aliases にある同義語に合わせるか、列を追加。
（最低限：umaban, umaname, odds）
	•	[NG] Output is zero bytes / CSV中身が「[OK] Done -」
→ 旧バッチの不具合。今回の run_icp.bat に必ず置換すれば解決。
	•	ModuleNotFoundError: icp
→ recalc_icp.py 先頭に
sys.path.insert(0, ROOT) があるか確認。
	•	日本語が文字化け
→ 出力は cp932 固定。Excelで「データ→テキスト/CSVから」ではなく、普通にダブルクリックで開く。
	•	Python not found
→ すでに動いているなら不要。もし出たら、Pythonをインストール後に再実行。

⸻

ここまでできたら “完全準備 完了”
	•	以後は race_input.csv を置く → run_icp.bat クリックだけ。
	•	入力の規格ズレ・文字化け・古いbat誤実行・空出力は守衛がすべて防止します。

この手順で進めて、どのステップで引っかかったかだけ教えてくれれば、そこをピンポイントで直すよ。