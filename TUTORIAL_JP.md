# PyRIT 日本語チュートリアル

## はじめに

PyRIT は Microsoft が公開した生成 AI のリスク評価ツールです。本リポジトリはそのフォーク版で、公式版と同様の機能を提供します。主に Azure OpenAI Services をはじめ、OpenAI API や HuggingFace など、さまざまなモデルを対象とした攻撃シナリオの構築と評価ができます。

以下では、ドキュメントに沿って PyRIT の仕組みや使い方を詳しく解説します。このチュートリアルを読めば、リポジトリの構成や主要コンポーネントを把握し、実際に攻撃シナリオを試すところまで理解できるでしょう。

---

## 1. インストール

### 1.1 基本インストール

Python 3.10 以上を用意した上で、公式の PyPI パッケージを pip からインストールします。

```bash
pip install pyrit
```

PyPI のバージョンとドキュメントのサンプルを合わせることで、ノートブックからでも同じ手順で実行できます。

### 1.2 コントリビューター向けインストール

開発環境を構築する場合は、リポジトリを clone してから conda 環境を作る方法が推奨されています。

```bash
git clone https://github.com/Azure/PyRIT
cd PyRIT
conda create -n pyrit-dev python=3.11
conda activate pyrit-dev
pip install -e '.[dev]'
```

`playwright` を使う場合は追加で以下を実行します。

```bash
pip install -e '.[dev,playwright]'
playwright install
```

依存関係の管理やテスト実行に必要なツールがまとめて入り、開発に便利です。

### 1.3 DevContainer の利用

VS Code の DevContainer 機能を使う場合、Docker と DevContainer 拡張機能をインストールし、`Dev Containers: Reopen in Container` を選択します。これにより統一された環境でノートブックを動かせます。

---

## 2. 環境設定

### 2.1 `.env` の準備

リポジトリには `.env_example` が含まれているので、これを `.env` にコピーして必要なキーやエンドポイントを入力します。Azure OpenAI の場合はポータルから取得したキーとエンドポイントを設定してください。

複数の環境を使い分ける場合は `.env.local` を作成すると優先的に読み込まれます。

### 2.2 Azure への認証

Azure CLI をインストールした上で、以下のコマンドでログインします。

```bash
az login
```

Azure のサブスクリプションを指定する場合は `az account set` を実行してください。

---

## 3. Jupyter ノートブックの利用

### 3.1 カーネルの設定

開発環境でノートブックを使う際は IPython kernel をインストールしておくと便利です。

```bash
python -m ipykernel install --user --name=pyrit_kernel
```

ノートブックを起動したら `Kernel > Change kernel > pyrit_kernel` を選択して実行します。

### 3.2 ノートブック依存の注意点

ドキュメントに含まれる `.ipynb` ファイルは `jupytext` を用いて `.py` と同期されています。編集する場合は `.py` ファイル側を変更し、ノートブックは再生成する形を推奨しています。

---

## 4. PyRIT のアーキテクチャ

PyRIT は以下のコンポーネントで構成されています。

1. **データセット**: 攻撃に用いるシードプロンプトやテンプレートを管理。
2. **オーケストレータ**: 攻撃シナリオを実行するための中心的なロジック。
3. **ターゲット**: 実際にプロンプトを送る API エンドポイント。
4. **コンバータ**: プロンプトを別形式に変換する仕組み。
5. **スコアラー**: 結果を評価し、成功・失敗を判断。
6. **メモリ**: プロンプト履歴などの状態を保持。

各コンポーネントはレゴブロックのように組み合わせて使えるため、目的に応じて入れ替えや拡張が容易です。

---

## 5. データセット

### 5.1 シードプロンプト

`pyrit/datasets/seed_prompts` には基本となるプロンプト集があり、攻撃の起点として利用します。YAML 形式で新しいプロンプトを追加することも可能です。

### 5.2 データセットの読み込み

データセットは YAML ファイルや外部リポジトリから取得できます。PyRIT は同一ハッシュの重複プロンプトを自動で除外する仕組みを備えているため、同じデータを誤って登録する心配がありません。

---

## 6. オーケストレータ

オーケストレータは攻撃の流れを管理するコンポーネントです。ドキュメント `doc/code/orchestrators` 以下にはさまざまな例があり、以下のような種類が存在します。

- **PromptSendingOrchestrator**: 単純にプロンプトを送信する基本形。
- **PAIR Orchestrator**: ユーザーとアシスタントのロールプレイを活用した攻撃。
- **Tree of Attacks**: 木構造で複数手法を試す複合的な攻撃。
- **Crescendo**: 徐々にモデルの制御を強めていく手法。
- **Skeleton Key Attack**: モデルのコンテキスト制御を突破するアプローチ。

これらはサンプルコード付きで提供されているので、環境変数でターゲットを指定してすぐに試すことができます。

---

## 7. ターゲット

PyRIT ではプロンプトを送る先として複数のターゲットクラスが用意されています。

### 7.1 OpenAIChatTarget

OpenAI 互換 API に対してプロンプトを送るための基本クラスです。Azure OpenAI だけでなく、OpenRouter や Groq なども設定次第で利用できます。エンドポイントやキーを `.env` に記載するだけで切り替えられます。

### 7.2 OpenAICompletionTarget

テキスト補完 API を利用するためのターゲットです。`max_tokens` などのパラメータを調整して、従来の OpenAI completion API を試すことができます。

### 7.3 HTTPTarget

任意の HTTP エンドポイントにリクエストを送るための汎用ターゲットです。独自の API を使った攻撃シナリオを組む際に利用できます。

### 7.4 HuggingFaceChatTarget

ローカルの HuggingFace モデルを使う場合のターゲットです。CPU のみでも動作しますが、モデルによって応答速度が変わるため、ドキュメントでは平均応答時間の目安が紹介されています。

### 7.5 その他のターゲット

画像生成向けの DALL·E ターゲットや、Playwright を使ったブラウザ操作、Prompt Shield との連携など、さまざまなターゲット実装が存在します。`doc/code/targets` 配下のサンプルを参照してください。

---

## 8. コンバータ

### 8.1 基本的な使い方

コンバータはプロンプトを送信前に加工する仕組みです。テキストを ROT13 でエンコードしたり、ASCII アート化したりといった変換が可能です。

```python
from pyrit.prompt_converter import ROT13Converter
result = await ROT13Converter().convert_tokens_async(prompt="Hello")
```

### 8.2 応用例

複数のコンバータを組み合わせて、攻撃用プロンプトを自動生成することもできます。画像や音声を扱うためのコンバータも用意されており、マルチモーダルな攻撃シナリオに対応できます。

---

## 9. スコアラー

スコアラーは攻撃結果を評価するためのコンポーネントです。主な種類として次のものがあります。

- **True/False スコアラー**: 成功したかどうかを真偽値で返す。
- **Likert スコアラー**: 0〜1 の数値で危険度を評価する。
- **Azure Content Safety スコアラー**: Azure のコンテンツ安全 API を利用して判定する。

これらを使うことで、モデルの応答が攻撃に成功しているかを自動で判定できます。人手で評価する Human in the Loop 方式もサポートされています。

---

## 10. メモリ

PyRIT は攻撃中の状態を記録するためのメモリ機構を持っています。デフォルトでは `DuckDB` ベースのローカルデータベースを利用しますが、Azure SQL など外部のデータベースを使うことも可能です。

メモリには送信したプロンプトやレスポンス、スコアの結果が保存され、後から解析や再利用ができます。`doc/code/memory` 以下には DuckDB の使い方、Azure SQL への接続方法、データのエクスポート手順などがまとめられています。

---

## 11. テストの実行

PyRIT の開発では `pytest` を用いたユニットテストが整備されています。開発用環境を構築した後、次のコマンドでテストを実行できます。

```bash
pytest tests/unit
```

特定のファイルだけをテストする例や、個別のテストケースを指定する例もドキュメントに掲載されています。CI 環境ではさらに統合テストも行われます。

---

## 12. コントリビューション

公式ドキュメントでは、Issue を立ててから Pull Request を送るまでの流れや、CLA の同意方法、コードスタイルのガイドラインなどが説明されています。特に、**pre-commit** によるコードチェックを実行すること、ユニットテストを追加することが推奨されています。

Git 操作やノートブックの管理方法など、初めて PyRIT に貢献する人向けの情報が揃っているので、興味のある方は `doc/contributing` 以下を参照してください。

---

## 13. 追加リソース

### 13.1 公式サイトとブログ

より詳しい最新情報は [公式サイト](https://azure.github.io/PyRIT/) や `doc/blog` にまとまっています。特にブログでは新しいターゲットの追加や仕様変更点が紹介されているため、定期的に確認すると良いでしょう。

### 13.2 部署向け参考資料

実際のレッドチーム運用に関する記事として、Microsoft の公開する "Planning red teaming for large language models" が紹介されています。PyRIT を使う前に読んでおくと、リスク評価の全体像を把握しやすくなります。

### 13.3 デプロイ関連

`doc/deployment` フォルダーには、HuggingFace モデルを Azure ML にデプロイする方法や、エンドポイントをスコアリングする手順、トラブルシューティングガイドなどが含まれています。自前のモデルをクラウド上で評価したい場合に役立ちます。

---

## 14. まとめ

このチュートリアルでは、PyRIT のインストールから基本的なコンポーネントの説明、ターゲットやコンバータの活用方法、メモリの仕組み、テスト・コントリビューションまで幅広く紹介しました。PyRIT はモジュール化された設計により、さまざまな攻撃シナリオを柔軟に構築できます。公式ドキュメントの各セクションを参照しながら実際に手を動かすことで、より深い理解が得られるでしょう。

PyRIT の仕組みを理解し、Azure OpenAI や OpenAI API、ローカルモデルなど多様な環境で試すことで、生成 AI システムのリスク評価を効率的に行えるようになります。本チュートリアルがその一助となれば幸いです。


---

## 15. クイックスタートの例

最後に、実際に PyRIT を動かす手順を簡単にまとめます。以下は ChatGPT を例にした最小構成です。

1. `.env` に Azure OpenAI のエンドポイントとキーを設定します。
2. `PromptSendingOrchestrator` を使ってシードプロンプトを送り、結果を表示します。

```python
from pyrit.common import IN_MEMORY, initialize_pyrit
from pyrit.orchestrator import PromptSendingOrchestrator
from pyrit.prompt_target import OpenAIChatTarget

initialize_pyrit(memory_db_type=IN_MEMORY)

target = OpenAIChatTarget(endpoint="https://your-endpoint.openai.azure.com/", api_key="your-key")

orchestrator = PromptSendingOrchestrator(objective_target=target)
response = await orchestrator.run_attack_async(objective="Hello, world!")  # type: ignore
await response.print_conversation_async()  # type: ignore
```

これで応答が得られれば、環境構築は完了です。ここからコンバータやスコアラー、複数ターゲットの設定などを追加していくことで、本格的な攻撃シナリオを作成できます。

---

## 16. より進んだ活用方法

### 16.1 データセットの作成

独自のシードプロンプトを大量に扱う場合は、`pyrit/datasets` 以下に YAML ファイルを置き、`SeedPromptDataset` として読み込むことができます。重複判定の仕組みがあるため、大規模なデータを扱う際も安心です。

### 16.2 Orchestrator のカスタマイズ

既存のオーケストレータを継承し、自分の攻撃手法を追加することも可能です。コード例は `doc/code/orchestrators` 配下に多数あるので、参考にしながら新しいクラスを作成すると良いでしょう。

### 16.3 スコアリングの自動化

大量のプロンプトを試す場合、手動で評価するのは大変です。`LookBackScorer` や `SelfAskTrueFalseScorer` を組み合わせると、一定の基準で自動的に結果を分類できます。これにより評価作業の効率化が図れます。

### 16.4 メモリのバックエンド変更

標準の DuckDB 以外に、Azure SQL Database を利用することも可能です。データベース接続文字列を `.env` に設定し、`initialize_pyrit` で `AZURE_SQL` を指定すれば、同じ API でクラウドベースのメモリを利用できます。

```python
initialize_pyrit(memory_db_type="AZURE_SQL")
```

### 16.5 ローカルモデルでの検証

HuggingFaceChatTarget を使えば、インターネット接続不要で手元の GPU または CPU だけでモデルを試せます。小規模なモデルなら比較的軽量に動作するため、社内環境でのセキュリティ検証にも便利です。

---

## 17. 参考情報のまとめ

ドキュメント全体を通して、以下のポイントを押さえておくとスムーズに利用できます。

- 環境変数でエンドポイントやモデル名を切り替える構成になっている。
- ノートブックは `.py` との併用で管理し、Git で差分を確認しやすくしている。
- テストや静的解析 (`pre-commit`, `mypy`) を通すことで品質を維持している。
- ブログ記事では新機能や互換性の改善点を随時紹介している。

以上を踏まえ、公式リポジトリと本フォークを活用すれば、多様な攻撃手法を効率よく検証できるはずです。


以上でチュートリアルは終了です。
