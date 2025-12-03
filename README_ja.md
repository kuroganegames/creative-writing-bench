# クリエイティブライティングベンチマーク v3

🎨 クリエイティブライティングベンチマーク v3 リポジトリへようこそ！このベンチマークは、大規模言語モデルの創作力をハイブリッドなルーブリック評価と Elo スコアリングで測定し、特に上位モデル間の差異をより明確に捉えられるよう設計されています。この仕組みは [EQ-Bench.com](https://eqbench.com/creative_writing.html) のクリエイティブラ イティング・リーダーボードで使われているものです。

## ベンチマークの流れ

評価プロセスは次のステップで進みます。

1. **生成:** テスト対象モデルは 32 個の異なる執筆プロンプトに対して 3 回ずつ応答を生成します（合計 96 件）。創造性を促しつつ一貫性を保つため、temperature=0.7、min_p=0.1 を使用します。
2. **ルーブリック評価:** 生成された各作品を、包括的なルーブリックに基づきジャッジモデル（リーダーボード同等性のため Anthropic Claude Sonnet 4 を推奨）が個別に採点します。
3. **初期 Elo 推定:** 集計したルーブリックスコアから、既存モデルとの相対的な初期 Elo レーティングを推定します。
4. **疎なペアワイズ対戦:** リーダーボード上の近接モデルとペアで比較し、ジャッジが複数の観点で優劣を判定します。スコア差は `+` 記号の数で表されます。
5. **Glicko 計算:** ペアワイズ比較の勝ち差（`+` の数）を組み込んだ Glicko-2 方式で Elo を計算します。このプロセスを、順位が収束するまで繰り返します。
6. **包括的なペアワイズ対戦:** 最終的な近接モデルとさらに詳細なペア比較を行います。
7. **最終 Elo 計算:** すべての比較結果を用いて決定的なリーダーボード Elo スコアを算出します。
8. **正規化:** 特定モデル（例: `deepseek/deepseek-r1` を 1500、`mistralai/ministral-3b` を 200）をアンカーにして生の Elo スコアを正規化し、長期的に比較可能なスケールに整えます。

詳しくはこちら: [https://eqbench.com/about.html#creative-writing-v3](https://eqbench.com/about.html#creative-writing-v3)

## 主な特徴

* **ハイブリッド評価:** ルーブリックの個別採点と、より識別力の高いペアワイズ Elo 比較を組み合わせています。
* **Glicko-2 システム:** 勝敗の不確実性と変動性を考慮し、勝ち差を重み付けする形で適応した堅牢なレーティング方式を採用。
* **識別力の高いプロンプト:** ユーモア、ロマンス、空間把握、独自視点などでモデルを試すよう設計されたプロンプト群。
* **バイアス低減:** 既知の LLM ジャッジ・バイアス（長さ、配置、冗長性、詩的な破綻）を抑えるための工夫を組み込み。
* **反復ベース:** 生成のばらつきを考慮し、各プロンプトで複数回（推奨は 3 回）実行します。

## 使い方

### 前提条件

* Python 3.x
* テストモデルとジャッジモデル用の API キー（OpenAI/OpenRouter 互換形式）
* 必要な Python パッケージ

### セットアップ

1. **リポジトリを取得:**
    ```bash
    git clone https://github.com/EQ-bench/creative-writing-bench.git
    cd creative-writing-bench
    ```

2. **依存関係をインストール:**
    ```bash
    pip install -r requirements.txt
    # もしくは手動で:
    # pip install requests python-dotenv numpy scipy tqdm glicko2 nltk joblib
    ```
    NLTK データのダウンロードも必要です:
    ```python
    import nltk
    nltk.download('punkt')
    nltk.download('cmudict')
    ```

3. **API キーを設定:**
    * サンプル環境ファイルをコピー: `cp .env.example .env`
    * `.env` を編集し、テストモデルとジャッジモデルの API キーとエンドポイント URL を記入します。タイムアウトやリトライ設定もここで調整できます。

### ベンチマークの実行

望むパラメータでメインスクリプトを実行します。リーダーボードと互換性のあるスコアを得るには、推奨ジャッジモデルと提供された runs ファイルを使用してください。

```bash
python3 creative_writing_bench.py \
    --test-model "your-model-provider/your-model-name" \
    --judge-model "anthropic/claude-sonnet-4" \
    --runs-file "creative_bench_runs.json" \
    --creative-prompts-file "data/creative_writing_prompts_v3.json" \
    --run-id "my_model_run_1" \
    --threads 500 \
    --verbosity "INFO" \
    --iterations 3
```

**重要な引数:**

* `--test-model`: 評価したいモデルの識別子。
* `--judge-model`: ジャッジモデルの識別子（リーダーボード用には `anthropic/claude-sonnet-4` を使用）。
* `--runs-file`: 実行データを保存する JSON ファイルへのパス。**EQ-Bench リーダーボードと比較可能な Elo スコアを得るには、このリポジトリにある `creative_bench_runs.json` を必ず使用してください。** 既存の履歴データが含まれています。ここから始め、以降の実行で更新されます。
* `--iterations`: 各プロンプトでの生成回数（デフォルトかつ推奨: 3）。
* `--run-id`: この実行を識別する一意のプレフィックス。同じモデルを何度か試す場合の整理に役立ちます。
* `--threads`: 生成とジャッジの並列スレッド数（API レート制限や環境に応じて調整）。500 のような大きな値は寛容なレート制限を想定しています。
* `--verbosity`: ログレベル（例: `DEBUG`, `INFO`）。


### 正準リーダーボード結果

リーダーボード結果は `creative_bench_runs.zip` と `elo_results.zip` に保存されています。これらをルートディレクトリに解凍し、デフォルトの実行ファイルパスを用いることで、ELO 対戦に結果が組み込まれ、リーダーボードと比較可能なスコアを得られます。

これらの正準 zip は常に最新とは限りません。最新結果が必要な場合は contact@eqbench.com までご連絡ください。

リーダーボード再現の簡単な例:

```bash
# .env を API 用に設定済みであることを確認した上で:
unzip creative_bench_runs.zip
unzip elo_results.zip
python3 creative_writing_bench.py \
    --test-model "your-model-provider/your-model-name" \
    --judge-model "anthropic/claude-sonnet-4" \
    --runs-file "creative_bench_runs.json" \
    --run-id "my_model_run_1" \
    --iterations 3
```



### 出力の見方

* 進捗はコンソールにログ出力されます。
* 生成テキストやジャッジスコアを含む詳細な実行データは、指定した `--runs-file`（例: `creative_bench_runs.json`）に保存されます。
* ペア比較や最終レーティングを含む Elo 解析結果は `elo_results.json` に保存されます。
* テストモデルの最終正規化 Elo スコア（`elo_norm`）が最後に出力され、`elo_results.json` に保存されます。これが EQ-Bench リーダーボードと比較可能なスコアです。

## ベンチマークの考え方

創作評価は本質的に主観的です。このベンチマークは以下を通じて信頼できる**相対的**順位付けを目指します。

* 文芸的評価が堅実なジャッジモデル（Sonnet 4）を利用。
* ルーブリック単体より識別力の高いペアワイズ比較を採用。
* モデルの弱点を意図的に露呈させるプロンプトを選び、評価勾配を急に設定。
* 既知の LLM ジャッジ・バイアスを認識し、低減を試みる。

ただし完璧なベンチマークは存在しません。サンプル出力を読み、自身の判断でも確認することを推奨します。

## スコアリング: ルーブリック vs. Elo

* **ルーブリックスコア:** 各作品を詳細なルーブリックに沿って個別採点した集計値。特定の観点での洞察を与えますが、性能が高い領域では飽和しがちです。
* **Elo スコア:** 他モデルとのペア比較に基づく相対レーティング。特に上位で識別力が高い一方、比較するモデル群に左右されます。

両者は異なる側面を測っており、ジャッジ方法や基準の違いにより一致しない場合があります。リーダーボードの主要指標は **正規化 Elo スコア（`elo_norm`）** です。

## バイアス低減

ペアワイズ LLM ジャッジでよく見られるバイアスを、次のように抑制しています。

* **長さバイアス:** 出力を 4000 文字に切り詰めて軽減。
* **位置バイアス:** A/B と B/A の両順序で比較し平均化。
* **冗長性・詩的破綻バイアス:** 過剰・破綻した文体を減点する特定の評価項目を設定。

明示的に制御していないバイアスには、ジャッジ自身の嗜好、ポジティブ/ネガティブ偏向、NSFW 忌避（スムットバイアス）、文体の好み、陳腐な表現を好む "slop" バイアスなどがあります。結果を解釈する際はこれらを念頭に置いてください。

## 制約

* **主観性:** 創作の質は主観的で、ジャッジの評価が人間の好みと一致しないことがあります。
* **ジャッジの限界:** Sonnet 4 は優秀ですが万能ではなく、人間が捉えるニュアンスを見逃す場合があります。
* **ロールプレイ評価ではない:** 会話型ロールプレイの能力は対象外です。
* **英語のみ:** 現在は英語の文章のみを評価します。
* **コスト:** ベンチマーク実行には API コストがかかります（ジャッジに Sonnet 4 を使うとモデルあたり約 $10）。

**ベンチマークスコアは絶対ではなく指標に過ぎません。必ずサンプル出力も確認してください！**

## 引用

このベンチマークを研究等で利用する場合は、以下を引用してください。

```bibtex
@misc{creative-writing-bench-v3,
  author = {Samuel J Paech},
  title = {EQ-Bench Creative Writing Benchmark v3},
  year = {2025},
  publisher = {GitHub},
  journal = {GitHub repository},
  howpublished = {\url{https://github.com/EQ-bench/creative-writing-bench}}
}
```
