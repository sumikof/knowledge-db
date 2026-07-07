# ドキュメント駆動ナレッジグラフ構築システム 仕様書(v0.4)

**ステータス**: 設計確定(残タスクは §11 参照)
**作成日**: 2026-07-05 / **更新日**: 2026-07-07

**v0.4 での決定事項**(ヒアリング結果の反映):
- 実行環境: **GPUサーバー(VRAM 24GB以上)** → LLMサービングは **vLLM で確定**(llama-server はCPU環境向けの代替構成に格下げ)。取込スループットNFR(100頁/10分)は維持
- 規模: ドキュメント **〜1,000件**(現想定通り)、Phase 2 コードは **1〜数リポジトリ・〜数十万行**
- ドキュメントソース: Phase 1 は**ファイルシステム + Git のみ**。Confluence / SharePoint は `SourceConnector` 抽象のみ用意し **Phase 3 で実装**
- 更新トリガー: **定期スキャン主体**(cron + ハッシュ差分)。削除検知も同一スキャンで実施
- 削除ドキュメント: **論理削除**(`archived` フラグ + 既定検索から除外 + 定期パージ。期間は90日を仮置き)(§5.7)
- バックアップ: **定期停止 dump(週次目安)+ SQLite バックアップ**。全再構築(`kg rebuild`)は最終手段(§12.1)
- 同義語辞書の反映: **増分マージコマンド `kg dict apply`**(LLM再抽出なしでノード統合・エッジ付け替え)(§12.2)
- MCP トランスポート: **Streamable HTTP に一本化**(localhost 利用含む)。stdio は廃止(内部ポート非公開の原則との矛盾を解消)(§7.1)
- 品質基準: M1 の手動レビュー結果を**ゴールドセット化**し、実測値ベースで目標確定(M4)(§10)
- オントロジー: サンプルドキュメント未提供のため **§4.2 の9タイプで確定スタート**、`proposed_type` 退避で実データ適合
- 帳票型Excel: **段階対応**。Phase 1 はセル値のベストエフォートテキスト化。代替案として「シート画像化 → ローカルVLM解析」を明記し M4 実測で要否判断(§5.1)
- MCP 利用規模: **個人〜数名(同一チーム)**。静的 Bearer トークンで運用(§7.5)
- 外部テレメトリの明示的遮断を構成要件に追加(§3.2, §12.4)
- エンティティ埋め込みの永続化と専用 vector index(§5.1)、日本語全文検索(CJKアナライザ + RRF)(§7.6)、取込の並行制御(§5.8)を追加

**v0.3 での決定事項**:
- LLM推論: **ローカル完結**(OpenAI互換サーバー)。外部LLM APIは使用しない
- LLM接続: LangChain `ChatOpenAI`(`base_url` 切替)でバックエンドを透過的に扱う(§5.6)
- モデル・イメージ取得(Hugging Face等)は構築手順の範疇とし、本仕様の外部接続要件から除外

**v0.2 での決定事項**:
- Graph Store: **Neo4j Community** で確定(ベクトルインデックス内蔵機能を利用、別途ベクトルDBは持たない)
- パイプライン: **LangGraph**(骨格・チェックポイント)+ **LangChain**(LLM呼び出し抽象化・構造化出力のみの薄い利用)
- Neo4j への読み書き: 公式 `neo4j` Python ドライバで直接 Cypher(冪等 MERGE + provenance プロパティ)。高レベル抽象(LLMGraphTransformer / PropertyGraphIndex 等)は不採用
- MCPサーバー実装言語: Python(FastMCP)— パイプラインとコードベースを統一

---

## 1. 目的・背景

組織内ドキュメント(設計書・仕様書・運用手順・議事録等)から知識を構造化したナレッジグラフ(KG)を自動構築し、AIエージェントが MCP (Model Context Protocol) 経由で参照・探索できる基盤を提供する。

将来的にはソースコードを解析対象に加え、「ドキュメント ⇔ コード」の対応関係をグラフ上で表現することで、以下を可能にする:

- 仕様変更時の影響範囲分析(このドキュメントの変更はどのコードに影響するか)
- コード理解支援(この関数はどの仕様に基づくか)
- ドキュメント鮮度検知(コードは変わったがドキュメントが未更新の箇所の検出)

単純な RAG(ベクトル検索)との差別化ポイントは **多段の関係探索(multi-hop)・構造的な問い合わせ・出所(provenance)の追跡可能性** にある。

## 2. スコープとフェーズ定義

| フェーズ | スコープ | 成果物 |
|---|---|---|
| **Phase 1** | ドキュメント取り込み(ファイルシステム / Git)→ KG構築 → MCPサーバー公開 | 動作するE2Eパイプライン + MCPサーバー(読み取り系ツール) |
| **Phase 2** | コード解析(静的解析)・コードグラフ構築・ドキュメントとの紐づけ | コードレイヤー追加、クロスリンク生成、影響分析ツール |
| **Phase 3** | 運用強化 | 差分同期の自動化(定期スキャンのスケジューラ化)、Confluence / SharePoint コネクタ、品質評価、鮮度検知、書き込み系ツール |

**Phase 1 のスコープ外(明示)**: リアルタイム同期、マルチテナント、権限制御(将来検討)、ドキュメント編集機能、Confluence / SharePoint 等の API コネクタ(`SourceConnector` 抽象のみ Phase 1 で定義。API 取得が必要なドキュメントは当面エクスポートしてファイル配置で代替)。

### 2.1 ユースケース

ユースケース定義とオペレーションフローは別紙 `kg-usecases.md` に定義する。運用系 UC-01〜07(CLI経由)と利用系 UC-10〜15(MCP経由のエージェント標準フロー)で構成し、CLIコマンド体系(`kg init/ingest/status/resume/review/dict/rebuild/ingest-code`)は同別紙 §5 を正とする。

> **注**: 別紙は本リポジトリに未収載。入手次第 `docs/superpowers/specs/kg-usecases.md` として追加すること。

### 2.2 対象ドキュメント形式(Phase 1)

- Markdown(最優先。構造抽出が容易)
- Word (.docx)、PDF、Excel(表形式仕様書。帳票型は段階対応 §5.1)
- HTML(社内Wiki エクスポート等)

### 2.3 対象言語(Phase 2 コード解析)

- 第一優先: Java(業務システム想定)、Python
- 拡張: TypeScript/JavaScript、SQL(DDL)
- 規模想定: 1〜数リポジトリ、〜数十万行(クロスリンク候補は数千〜数万ペア規模であり、§6 の3段階絞り込みで対応可能)

## 3. 全体アーキテクチャ

推論(LLM・埋め込み)を含む全コンポーネントが Docker Compose 内で完結し、**恒常的なデータの外部送信は存在しない**。境界を越える接続は「ドキュメント/コードの取得(Out)」と「MCPクライアント(In)」のみ。

```
                        【外部/ネットワーク境界の外】
        ┌──────────────────────────┐   ┌───────────────────────┐
        │ ドキュメントソース          │   │ (Phase 2)              │
        │ 社内Git / ファイル共有      │   │ 社内Gitリポジトリ(コード)│
        │ (Phase 3: Confluence等)    │   │                        │
        └────────────▲─────────────┘   └───────────▲───────────┘
                     │ E1: 定期スキャン時のみ(取得)   │ E3: 取込時のみ
═════════════════════╪═══════════════════════════════╪═══════════
                     │ 【本システム: Docker Compose 内】
   ┌─────────────────┴─────────────────────────────────────────┐
   │      Ingestion Layer: 取込CLI (`ingest`, SourceConnector)   │
   │   LangGraph: parse→chunk→extract→resolve→validate→load     │
   └──┬──────────────┬──────────────────┬───────────────┬──────┘
      │ bolt(7687)   │ HTTP              │ HTTP           │ ファイルI/O
      ▼              ▼ /v1/embeddings    ▼ /v1/chat/…     ▼
 ┌─────────┐  ┌──────────────┐  ┌──────────────────┐  ┌─────────────┐
 │ Neo4j    │  │ `embedding`   │  │ `llm`             │  │ Metadata DB  │
 │Community │  │ vLLM          │  │ vLLM(確定)        │  │ (SQLite,     │
 │graph+vec │  │ + ruri-v3     │  │ ※CPU代替:         │  │ 共有ボリューム)│
 └────▲────┘  └──────▲───────┘  │  llama-server     │  └──────▲──────┘
      │ bolt         │ HTTP      └────────▲─────────┘         │
      │              │                   │ HTTP(任意機能のみ)  │
   ┌──┴──────────────┴──────────────────┴────────────────────┴───┐
   │  MCP Server (`app`, FastMCP, Streamable HTTP)                │
   │  ※既定ではLLMを呼ばない(§3.2)                                │
   └───────────────────────────▲─────────────────────────────────┘
                               │ E2: Streamable HTTP + Bearer
═══════════════════════════════╪═════════════════════════════════
          【利用者側】          │
   ┌───────────────────────────┴─────────────────────────────────┐
   │ AIエージェント (Claude Code / Claude Desktop / 自作Agent)      │
   │  ・ローカル/チーム共有ともに Streamable HTTP(localhost 可)     │
   └─────────────────────────────────────────────────────────────┘
```

※ ruri-v3 モデル重みやコンテナイメージの取得(Hugging Face 等)は初期構築手順の範疇であり、稼働時の外部接続要件には含めない。稼働時は `HF_HUB_OFFLINE=1` を設定する(§12.4)。

### 3.1 外部接続要件一覧

| # | 接続先 | 方向 | プロトコル/ポート | 発生タイミング | 送信されるデータ | エアギャップ時の代替 |
|---|---|---|---|---|---|---|
| E1 | ドキュメントソース(社内Git、ファイル共有。Phase 3 で Confluence 等API) | Out(取得) | Git(SSH/HTTPS)、SMB/NFS(Phase 3: HTTPS API) | 定期スキャン/取込ジョブ実行時のみ | なし(取得のみ)。認証情報はSecret/環境変数で注入 | 対象ファイルを共有ボリュームに配置し完全ローカル参照 |
| E2 | MCPクライアント(AIエージェント) | **In** | Streamable HTTP+TLS(ポートは配備時決定) | 常時 | 応答としてグラフ内容・チャンク原文が**クライアント側へ渡る**(クライアント環境の管理は本システム外、§7.5) | —(内部利用) |
| E3 | (Phase 2) 社内Gitリポジトリ(コード) | Out(取得) | Git(SSH/HTTPS) | コード取込時のみ | なし(取得のみ) | E1と同様 |

**内部接続(境界内で完結)**: `app`/`ingest` → Neo4j(bolt/7687)、→ `embedding`(HTTP)、→ `llm`(HTTP、OpenAI互換)、SQLite(共有ボリューム上のファイル)。すべて Docker 内部ネットワークに閉じ、ホストへのポート公開は MCP の HTTP ポートと、運用者向け Neo4j Browser(任意・要認証)に限定する。

### 3.2 外部接続に関する設計判断

- **LLM・埋め込み推論はローカル完結**のため、ドキュメント本文・コード断片がネットワーク境界を越えることはない。外部送信ポリシー審査の対象接続はゼロ。
- **依存ライブラリ・ランタイムが暗黙に行い得る外部送信を明示的に遮断する**: LangSmith トレーシング(`LANGCHAIN_TRACING_V2=false`)、vLLM 匿名利用統計(`VLLM_NO_USAGE_STATS=1`)、Hugging Face Hub 自動アクセス(`HF_HUB_OFFLINE=1`)。Compose の既定環境変数として設定し、チェックリストを §12.4 に置く。
- **MCPサーバーは既定でLLMを呼ばない**(検索・探索は Neo4j + embedding で完結)。クエリ拡張・結果要約などLLMを使う付加機能はフラグ制御とし、有効化しても接続先は内部の `llm` コンテナのみ。
- Outbound(E1/E3)はプロキシ環境向けに `HTTPS_PROXY` / `NO_PROXY` を尊重し、接続先は設定ファイルに集約する。

### 3.3 コンポーネント責務

| コンポーネント | 責務 | 技術(確定) |
|---|---|---|
| 取込パイプライン | ソース取得(SourceConnector)、パース、チャンク化、LLM抽出、エンティティ解決、グラフ書込 | Python + LangGraph(骨格)+ LangChain(LLM抽象化のみ) |
| Graph Store + Vector Index | ノード・エッジ永続化、Cypher問合せ、埋め込み検索、日本語全文検索 | Neo4j Community(vector index / full-text index 内蔵機能を利用) |
| Metadata DB | 取込履歴、差分管理、ジョブ状態(排他制御含む)、パイプラインチェックポイント、同義語辞書 | SQLite(詳細は §5.3) |
| Embedding | チャンク・エンティティのベクトル生成 | **vLLM + cl-nagoya/ruri-v3-310m**(OpenAI互換 /v1/embeddings、詳細は §5.5) |
| MCP Server | エージェント向けAPI | Python (FastMCP)、Streamable HTTP。取込パイプラインと同一コードベース |
| LLM | 抽出・エンティティ解決・リンク判定 | **ローカルLLM: vLLM(確定)**(OpenAI互換 /v1/chat/completions)。LangChain `ChatOpenAI` 経由(§5.6)。CPU環境向け代替として llama-server 構成も保持 |
| パーサー | 形式別のドキュメント解析 | Docling / unstructured / python-docx / openpyxl 等(ライブラリ、常駐なし) |
| 静的解析(Phase 2) | コード構造抽出 | tree-sitter(ライブラリ、常駐なし) |

### 3.4 設計原則

1. **オンプレ可搬性**: 全コンポーネントを Docker Compose 一式で起動可能にする。**外部SaaS依存なし**(推論を含め全てローカル)。
2. **Storage抽象化**: `GraphStoreAdapter` インターフェースを定義し、Neo4j 以外(FalkorDB、SQLiteベース簡易実装等)へ差替可能にする。同様に取得元は `SourceConnector` インターフェース(list / fetch / メタデータ取得)で抽象化し、Phase 1 はファイルシステム実装と Git 実装のみ提供する。
3. **Provenance第一**: すべての抽出結果は「どのドキュメントのどのチャンクから、どの手法・モデルで、いつ抽出されたか」を保持する。エージェントが根拠を提示できることが信頼性の要。
4. **冪等な再取込**: 同一ドキュメントの再取込は差分のみ反映され、グラフが重複・肥大化しない。

## 4. データモデル(オントロジー)

### 4.1 レイヤー構造

グラフは3レイヤーで構成する。構造レイヤーは決定的に生成され(LLM不使用)、知識レイヤーはLLM抽出、コードレイヤーは静的解析で生成される。

```
[構造レイヤー]  Document ─ Section ─ Chunk        ← パーサーが決定的に生成
[知識レイヤー]  Concept / System / Requirement 等  ← LLMが抽出、Chunkから MENTIONS で接続
[コードレイヤー] Repository ─ File ─ Class ─ Method ← 静的解析(Phase 2)
```

### 4.2 ノードタイプ(Phase 1)

サンプルドキュメントによる事前検証は行わず、本セットを初期オントロジーとして確定する。実データとの不適合は `proposed_type` 退避(§4.6)で吸収し、レビューを経てタイプを追加する。

| ラベル | 説明 | 主要プロパティ |
|---|---|---|
| `Document` | 取込単位 | `id`, `title`, `path`, `format`, `hash`, `updated_at`, `archived`(論理削除フラグ, §5.7) |
| `Section` | 見出し単位 | `id`, `title`, `level`, `order` |
| `Chunk` | 抽出・検索の最小単位 | `id`, `text`, `embedding`, `token_count` |
| `Concept` | ドメイン概念・用語 | `id`, `name`, `aliases[]`, `definition`, `name_embedding` |
| `System` | システム・サービス・外部連携先 | `id`, `name`, `type`, `name_embedding` |
| `Component` | 機能・画面・バッチ・API等 | `id`, `name`, `kind`, `name_embedding` |
| `Requirement` | 要件・仕様項目・制約 | `id`, `text`, `req_type`(機能/非機能/制約), `name_embedding` |
| `Process` | 業務プロセス・運用手順 | `id`, `name`, `name_embedding` |
| `Actor` | 役割・チーム・利用者区分 | `id`, `name`, `name_embedding` |

`name_embedding` は「名前+定義文」を `トピック: ` プレフィックスで埋め込んだベクトル(§5.5)。エンティティ解決の増分突合(§5.1)と `kg_search` のノード名検索に使う。**`embedding` / `name_embedding` プロパティは MCP レスポンス・`kg_query` 結果から常に除外する**(768次元 float がエージェントのコンテキストを圧迫するため。§7.1)。

### 4.3 エッジタイプ(Phase 1)

| タイプ | 方向 | 生成方法 |
|---|---|---|
| `HAS_SECTION` / `HAS_CHUNK` | Document→Section→Chunk | 決定的 |
| `REFERENCES` | Document→Document(リンク・参照記述) | 決定的 + LLM補完 |
| `MENTIONS` | Chunk→(知識ノード) | LLM |
| `DEFINES` | Chunk→Concept(定義箇所) | LLM |
| `DEPENDS_ON` | System/Component 間依存 | LLM |
| `PART_OF` | Component→System 等の包含 | LLM |
| `CONSTRAINS` | Requirement→Component/Process | LLM |
| `RELATED_TO` | 汎用関連(低信頼の受け皿) | LLM |

`REFERENCES` の参照先が未取込ドキュメントの場合は、`Document {status: "placeholder"}` ノードを作成してエッジを張る。参照先が後から取り込まれた時点で placeholder を実体にマージする(doc_id はパスから決定的に生成されるため自然に一致する)。

### 4.4 ノードタイプ・エッジタイプ(Phase 2: コードレイヤー)

ノード: `Repository`, `SourceFile`, `Module`, `Class`, `Method`, `ApiEndpoint`, `DbTable`, `ConfigKey`
エッジ(コード内): `DECLARES`, `CALLS`, `IMPORTS`, `EXTENDS`, `IMPLEMENTS`, `READS_WRITES`(→DbTable)

クロスリンク(ドキュメント⇔コード):

| タイプ | 意味 | 生成方法 |
|---|---|---|
| `DESCRIBES` | Chunk→コードノード(記述している) | 識別子一致 + 埋め込み類似 + LLM判定 |
| `IMPLEMENTED_BY` | Requirement→Method/Class | LLM判定(高信頼のみ) |
| `TESTED_BY` | Requirement/Component→テストコード | 命名規約 + アノテーション解析 |

### 4.5 共通エッジプロパティ(Provenance)

すべてのLLM生成エッジに必須:

```
source_chunk_id: 抽出元チャンク
method: "llm_extract" | "identifier_match" | "embedding" | "manual" | "deterministic"
model: 抽出に使用したモデルID(LLM時)
confidence: 0.0–1.0
extracted_at: timestamp
pipeline_version: 抽出プロンプト・ロジックのバージョン
```

`confidence` の閾値運用: 0.8以上を既定でエージェントに公開、それ未満は `include_low_confidence` オプション時のみ返却。

> **注(較正)**: LLM が自己申告する confidence は較正されていない。閾値 0.8 は仮置きであり、M4 でゴールドセットに対する実測 precision と突き合わせて調整する(例: confidence 帯域ごとの実測 precision を測り、目標 precision を満たす帯域を公開閾値とする)。

### 4.6 スキーマ管理

- オントロジー(許可ノード・エッジタイプと制約)は YAML で宣言的に定義し、抽出プロンプトとバリデーションの単一情報源とする。
- LLMが未知のタイプを提案した場合は `RELATED_TO` + `proposed_type` プロパティに退避し、定期レビューでオントロジーに昇格させる運用とする(スキーマの無秩序な増殖を防止)。

## 5. 取込パイプライン仕様(Phase 1)

LangGraph でステートマシンとして実装。各ノードは失敗時に再開可能(チェックポイントを Metadata DB に永続化)。

```
ingest → parse → chunk → extract_entities → extract_relations
      → resolve_entities → validate → load_graph → index_vectors → finalize
```

### 5.1 各ステップ仕様

**parse**: 形式別パーサーで統一中間表現(見出しツリー + ブロック列)へ変換。表はMarkdownテーブルとして保持。PDF は unstructured / Docling 系、docx は python-docx 系を想定。

Excel は形状別に扱う:

| 形状 | Phase 1 の扱い |
|---|---|
| 一覧表型(1行=1項目) | シート=Section、行→Markdownテーブル化(openpyxl) |
| 複数シート構成(1ブック=1機能・複数観点) | ブック=Document、シート=Section にマッピング |
| 帳票型(セル結合・レイアウト重視) | **段階対応**。Phase 1 はセル値を読み取り順でテキスト化するベストエフォート(レイアウト意味の部分欠落を許容)。M4 で抽出品質を実測し、不十分な場合は「シートを画像化 → ローカルVLM(視覚対応モデル、例: Qwen2.5-VL)で構造解釈」を追加する。VLM は `llm` コンテナと同様に vLLM でサービング可能(VRAM 24GB 内でのモデル同居/切替は M4 時点で設計) |

**chunk**: セクション境界を尊重した意味的チャンク化。目安 300–800 トークン、見出しパスをメタデータとして付与(例: `設計書 > 3. 外部IF > 3.2 送信仕様`)。

**extract_entities / extract_relations**: チャンク単位でLLMに構造化出力(JSON Schema強制)させる。オントロジーYAMLからプロンプトを自動生成。few-shot 例をドメインごとに差替可能にする。1チャンク1呼び出しを基本とし、vLLM への並列リクエスト数はスループットNFR(§8)と `llm` コンテナの処理能力に合わせて調整する(§5.8)。

**resolve_entities(エンティティ解決)**: 重複統合の品質がKG全体の品質を決める最重要ステップ。**突合対象は今回バッチ内に閉じず、既存グラフ上の全エンティティを含む**(増分取込で同一エンティティが再登場するのが通常ケースのため)。

1. 正規化(全角半角、大小文字、既知の同義語辞書)による決定的マージ — エンティティIDが「正規化名+タイプ」から決定的に生成されるため、既存ノードとは MERGE で自然に一致する
2. 埋め込み類似度で候補ペア生成(閾値例: cosine > 0.92)— 新規エンティティの「名前+定義文」を `トピック: ` プレフィックスで埋め込み、**エンティティ専用 vector index(`name_embedding`、§5.5)に対して検索**することで既存グラフ全体と突合する
3. 候補ペアをLLMで同一性判定(定義文・出現文脈を提示)
4. マージ時は `aliases[]` に統合、エッジは付け替え。判定結果は `resolution_log`(§5.3)に記録

**validate**: オントロジー制約チェック(不正なタイプ、自己ループ、孤立ノード)、confidence 下限チェック。

**load_graph**: 冪等な upsert(§5.2)。Document 単位のトランザクションでコミットする(§5.8)。

**index_vectors**: チャンク埋め込み(`検索文書: `)とエンティティ埋め込み(`トピック: `)を生成し、それぞれの vector index に投入。

### 5.2 差分更新(冪等性)

- Document 単位で `hash` を保持。変更時はチャンク単位で diff を取り、変更チャンクのみ再抽出。
- チャンク由来のエッジ・ノードには `source_chunk_id` があるため、削除チャンク由来の要素を特定・削除できる(他チャンクからも参照されるノードは残し、エッジのみ削除)。
- エンティティIDは「正規化名 + タイプ」から決定的に生成し、再取込でIDが揺れないようにする。
- **チャンク境界のカスケードは許容する**: 文書前方への挿入等でそれ以降のチャンク境界がずれた場合、後続チャンクの `chunk_hash` はすべて変わり、そのドキュメント内の広範囲が再抽出になる。これは設計上の許容事項であり(正しさを優先)、「変更チャンクのみ再抽出」はチャンク境界が保存される編集(セクション内の修正等)での最適化と位置づける。

### 5.3 Metadata DB(SQLite)の役割と設計

SQLite は「知識」ではなく**運用データ**を担当する。Neo4j は知識ストアに徹し、SQLite + 元ドキュメントがあれば Neo4j は再構築可能、という関係を保つ(ただし再構築の意味論は §12.1 参照 — LLM の非決定性により同一グラフの厳密再現は保証しない。日常の復旧手段はバックアップである)。

**用途1: LangGraph チェックポイント永続化**
`SqliteSaver`(langgraph-checkpoint-sqlite)をそのまま使用。パイプライン実行中の状態スナップショットを保持し、LLM呼び出し途中の失敗からステップ単位で再開できる。テーブルはライブラリ管理のため自前設計不要。

**用途2: 取込台帳・差分管理(自前スキーマ)**

```sql
documents (
  doc_id TEXT PRIMARY KEY,      -- パスから決定的に生成
  path TEXT, format TEXT,
  source TEXT,                  -- SourceConnector 識別子(fs / git / …)
  content_hash TEXT,            -- 変更検知用
  last_ingested_at TEXT,
  last_seen_at TEXT,            -- 直近スキャンで存在確認された時刻(削除検知用, §5.7)
  pipeline_version TEXT,
  status TEXT                   -- pending / ingested / failed / archived
);
chunks (
  chunk_id TEXT PRIMARY KEY,
  doc_id TEXT REFERENCES documents,
  chunk_hash TEXT,              -- チャンク単位diffの基準
  heading_path TEXT, ord INTEGER
);
ingest_jobs (
  job_id TEXT PRIMARY KEY,
  started_at TEXT, finished_at TEXT,
  status TEXT, error TEXT,      -- running / done / failed(running 行が排他ロックを兼ねる, §5.8)
  llm_input_tokens INTEGER, llm_output_tokens INTEGER  -- 負荷・処理量の観測
);
```

再取込時は `content_hash` → 変更ありなら `chunk_hash` を比較し、変更チャンクのみ再抽出する(§5.2)。この diff 判定の正本は SQLite 側に置く。

**用途3: エンティティ解決の辞書・レビューキュー(自前スキーマ)**

```sql
synonyms (canonical TEXT, alias TEXT, source TEXT);  -- 決定的マージ用の同義語辞書
resolution_log (merged_from TEXT, merged_to TEXT, method TEXT, confidence REAL, at TEXT);
proposed_types (name TEXT, example_chunk_id TEXT, status TEXT);  -- オントロジー昇格待ち(§4.6)
```

同義語辞書は人手メンテの対象であり、グラフ再構築を跨いで生き残る必要があるため SQLite に置く。辞書更新の既存グラフへの反映は `kg dict apply`(§12.2)。

**Neo4j と SQLite の境界(判断基準)**: エージェントが問い合わせる対象 → Neo4j。パイプラインの運用・再実行・監査に必要な情報 → SQLite。両方に跨るID(doc_id, chunk_id)は決定的生成により一致を保証する。

### 5.4 ランタイム構成

Docker Compose で以下の構成とする:

| コンテナ/プロセス | 内容 | 常駐 |
|---|---|---|
| `neo4j` | Graph Store + Vector Index + Full-text Index | ○ |
| `app` | MCPサーバー(FastMCP, Streamable HTTP) | ○ |
| `ingest`(同一イメージ) | 取込CLI。手動/cron から実行 | ×(ジョブ実行時のみ) |
| `embedding` | vLLM(ruri-v3-310m 専用、OpenAI互換 /v1/embeddings) | ○ |
| `llm` | ローカルLLM推論(**vLLM で確定**、OpenAI互換 /v1/chat/completions)。embedding とは別コンテナで分離(リソース競合・バージョン制約の分離)。CPU環境向け代替として llama-server 構成も Compose プロファイルで提供 | ○(取込時必須。MCP付加機能を使わない運用では取込時のみ起動も可) |

SQLite はファイル(`app`/`ingest` がボリューム共有)であり、コンテナは不要。それ以外に必要なのは Python ライブラリ群(LangGraph / LangChain / neo4jドライバ / Docling / openpyxl / FastMCP、Phase 2 で tree-sitter)のみで、追加のミドルウェアは発生しない。

### 5.5 埋め込み仕様(確定: vLLM + ruri-v3)

**モデル**: `cl-nagoya/ruri-v3-310m`(次元 **768**、最大系列長 8192 トークン)。モデル変更は全ベクトル再生成を意味するため、`pipeline_version` に埋め込みモデルIDを含めて追跡する。

**vector index は2本作成する**:

| index | 対象 | 用途 |
|---|---|---|
| `chunk_embedding_idx` | `Chunk.embedding`(`検索文書: ` で埋め込み) | `kg_search` の本文検索 |
| `entity_name_idx` | 知識ノードの `name_embedding`(`トピック: ` で「名前+定義文」を埋め込み) | エンティティ解決の増分突合(§5.1)、`kg_search` のノード名検索 |

いずれも `dimensions: 768, similarity: cosine`。

**プレフィックス規約(最重要)**: Ruri v3 は用途別プレフィックスを前提とした非対称埋め込みモデルであり、OpenAI互換APIはプレフィックスを付与しない。`EmbeddingClient` 側で必ず付与する:

| 用途 | プレフィックス |
|---|---|
| チャンク・ドキュメントのインデックス時 | `検索文書: ` |
| `kg_search` のクエリ埋め込み | `検索クエリ: ` |
| エンティティ解決の候補生成(名前+定義文の対称比較) | `トピック: `(両側同一) |
| 汎用の意味類似 | 空プレフィックス(両側同一) |

インデックス側とクエリ側の組み合わせを誤ると検索品質が大きく劣化するため、プレフィックス付与は `EmbeddingClient` に閉じ込め、呼び出し側は用途enum(`INDEX_DOC` / `QUERY` / `TOPIC` / `STS`)のみを指定するインターフェースとする。

**サービング**: vLLM の OpenAI 互換サーバー(`/v1/embeddings`)。encoder系embeddingモデルのサポートはvLLMの比較的新しい機能であり、起動オプション(旧 `--task embed` → 新 `--runner pooling`)がバージョンで変わるため、Compose内でvLLMバージョンを固定する。310m は小さいため、LLM用vLLMとは**別コンテナ**で分離する(リソース競合・バージョン制約の分離)。

**チャンク長との関係**: 最大8192トークンによりチャンク(300–800トークン)には十分な余裕があり、将来セクション単位の粗粒度埋め込みを追加する選択肢も残る。

**既知のリスク(Phase 2 で評価)**: ruri-v3 は日本語特化のため、英語識別子・英語docstring主体のコード側埋め込み(§6 クロスリンク候補生成)では品質が落ちる可能性がある。Phase 2 では (a) コード要素の「日本語要約文」をLLM生成してから埋め込む、(b) コード側のみ別モデル、(c) 埋め込み候補生成を弱めLLM判定の比重を上げる、の3案を比較評価する。(a) を第一候補とする(埋め込み空間を1つに保てるため)。

### 5.6 LLM接続仕様(確定: ローカルLLM + LangChain ChatOpenAI)

**サービング(v0.4 確定)**: 実行環境は GPU サーバー(VRAM 24GB 以上)のため、**vLLM を採用**する。7B〜32Bクラス(量子化含む)のinstructモデルを想定し、continuous batching による並列処理でスループットNFR(§8)を満たす。llama-server 構成(GGUF量子化・CPU可)は、GPU が確保できない配備先向けの代替として Compose プロファイルで保持する(その場合スループットNFRは緩和が必要 → 配備時に個別合意)。

**接続方式**: vLLM / llama-server はともに OpenAI 互換 API を提供するため、LangChain の `ChatOpenAI`(`langchain-openai`)に `base_url` を内部エンドポイントへ向けて接続する(`api_key` はダミー値)。バックエンドの差は `base_url` と `model` の設定値のみで吸収され、パイプラインコードは無変更。

```python
llm = ChatOpenAI(
    base_url=settings.llm_base_url,   # http://llm:8000/v1 等
    model=settings.llm_model,
    api_key="local",
    temperature=0,
)
```

**構造化出力の方式(要注意点)**: 抽出パイプラインは JSON Schema 準拠の出力(§5.1)に依存する。`with_structured_output()` の既定はツール呼び出し(function calling)ベースだが、ローカルモデルの function calling 対応はモデル・サーバーの組み合わせ依存で信頼性が揺れる。本システムでは **`method="json_schema"`(サーバー側の制約付きデコーディング)を第一方式**とする:

- vLLM: guided decoding(`response_format: json_schema` / `guided_json`)対応
- llama-server: GBNF 文法による `response_format: json_schema` 対応

両者ともスキーマ準拠をサーバー側で強制できるため、出力パースの失敗率を大きく下げられる。フォールバックとして「プロンプト指示 + Pydantic バリデーション + リトライ(最大2回)」を実装し、`method` は設定で切替可能にする。

**モデル要件**: 日本語の技術文書からの関係抽出・同一性判定が主タスクのため、(1) 日本語instruct性能、(2) JSON Schema制約下での生成品質、(3) 取込スループット(§8)を満たす推論速度、の3点で選定する。VRAM 24GB で動作する候補(例: 7B〜32B級の量子化モデル)の比較評価は M1 の前提タスクとする(§11-1)。エンティティ解決の同一性判定(§5.1)は抽出より軽いタスクのため、将来的に小型モデルへの分離も検討余地がある。

**運用上の注意**: `pipeline_version` に LLMモデルID・量子化設定を含める(抽出結果の再現性追跡)。パイプラインの並列リクエスト数は vLLM の実効同時処理能力(モデルサイズ・KVキャッシュ量に依存)と整合させる(§5.8)。

### 5.7 定期スキャンと削除ドキュメントの扱い

**スキャン**: 更新検知は定期スキャン主体(cron 等から `kg ingest --scan` を起動)。`SourceConnector.list()` で対象を全列挙し、SQLite 台帳と突合する:

- 台帳に無いパス → 新規取込
- `content_hash` 変化 → 差分再取込(§5.2)
- 台帳にあるがソースに無い → **削除検知**
- 存在確認できたものは `last_seen_at` を更新

**削除の扱い(論理削除)**:

1. 削除検知した Document は Neo4j 上で `archived: true` を設定(SQLite 側 `status` も `archived`)。チャンク・エッジは即時削除しない
2. `archived` なドキュメント由来の要素は、既定の検索・探索(`kg_search` / `kg_traverse` 等)から除外する。`kg_list_documents` では `archived` として明示し、`include_archived` オプションで参照は可能(エージェントが「廃止済み文書由来」と説明できる)
3. **定期パージ**: archived から一定期間(既定 90日、設定可能)経過したものを物理削除(チャンク・エッジのカスケード削除、他チャンクから参照されるノードは残置)。誤削除・一時的なファイル移動からの復旧猶予を確保する

### 5.8 並行実行制御と読み取り一貫性

- **取込ジョブは同時1実行**: `ingest_jobs` の `running` 行を排他ロックとして使用し、多重起動をエラーにする(`kg status` で実行中ジョブを確認、`kg resume` で再開)。SQLite への書き込みが単一ジョブに限定されるため、ロック競合も回避される
- **パイプライン内部の並列度**: LLM呼び出し(チャンク単位)はジョブ内で並列化する。並列数は設定値とし、vLLM の実効スループットに合わせて調整する
- **読み取り一貫性**: グラフ書込は Document 単位のトランザクションでコミットする。取込中の MCP 読み取りは「コミット済みドキュメントまで反映された状態」を見る(ドキュメント単位の一貫性)。取込ジョブ全体のアトミック性は保証しない(1,000件規模の全件取込が数日に及ぶため、部分反映を許容する設計とする)

## 6. コード解析仕様(Phase 2 概要)

- **静的解析基盤**: tree-sitter(多言語統一)を第一候補。Java は必要に応じ JavaParser 等で補強。LLMは構造抽出には使わない(決定的・高速・正確な静的解析で足りるため)。抽出対象: 宣言、呼び出し関係、import、アノテーション、Javadoc/docstring。
- **粒度**: Method レベルまで。行番号・シグネチャ・docstring をプロパティとして保持。
- **規模前提**: 1〜数リポジトリ・〜数十万行(§2.3)。クロスリンク候補は3段階絞り込みで数千〜数万ペアに収まり、現行設計で対応可能。
- **クロスリンク生成(3段階)**:
  1. **決定的一致**: ドキュメント中のクラス名・メソッド名・APIパス・テーブル名の識別子マッチ(高信頼、confidence=0.95)
  2. **埋め込み類似**: Requirement/Chunk の埋め込み × Method の「シグネチャ+docstring+要約」埋め込みの類似候補
  3. **LLM判定**: 候補ペアに対し「この仕様はこのコードを記述しているか」を判定させ confidence を付与
- **鮮度検知(Phase 3)**: コード側ノードの更新(コミット)時刻 > リンク先ドキュメント更新時刻 のとき `stale_link` フラグを立て、エージェントが警告を返せるようにする。

## 7. MCPサーバー仕様

### 7.1 基本方針

- ツール名は `kg_` プレフィックスで統一(発見性)。
- Phase 1 は **読み取り専用**(全ツール `readOnlyHint: true`)。書き込み系(ノート追加・リンク修正)は Phase 3。
- トランスポート: **Streamable HTTP に一本化**(v0.4)。ローカル利用も localhost への HTTP 接続とする。stdio は提供しない(stdio はクライアントプロセスから Neo4j / embedding への直接接続を要求し、内部ポート非公開の原則 §3.1 と矛盾するため)。
- レスポンスは構造化JSON(`structuredContent`)+ 人間可読なMarkdownサマリの併記。
- すべての結果に provenance(出典ドキュメント・チャンク・confidence)を含める。
- **`embedding` / `name_embedding` プロパティはすべてのレスポンスから除外する**(`kg_query` の結果も含む。§4.2)。
- ページネーション必須(既定 limit=20、`cursor` 方式)。コンテキスト圧迫を避けるため、既定レスポンスは要約フィールドのみ、`detail=true` で全文返却。
- `archived` ドキュメント由来の要素は既定で除外、`include_archived` で参照可(§5.7)。

### 7.2 ツール一覧(Phase 1)

| ツール | 概要 | 主要パラメータ |
|---|---|---|
| `kg_schema` | オントロジー(ノード・エッジタイプ一覧と説明)を返す。エージェントの探索起点 | なし |
| `kg_search` | ハイブリッド検索(ベクトル+キーワード+ノード名)。ノードとチャンクを横断(§7.6) | `query`, `node_types[]`, `limit` |
| `kg_get_node` | ノード詳細 + 隣接ノード一覧 + 出典チャンク | `node_id`, `include_neighbors`, `depth` |
| `kg_traverse` | 指定ノードから関係タイプを絞って多段探索 | `start_id`, `edge_types[]`, `direction`, `max_depth` |
| `kg_find_path` | 2ノード間の最短パス(関係の連鎖を説明) | `from_id`, `to_id`, `max_hops` |
| `kg_query` | 読み取り専用Cypher実行(上級用途、タイムアウト・行数上限付き) | `cypher`, `params` |
| `kg_get_source` | チャンク原文と前後文脈、ドキュメント内位置を返す | `chunk_id`, `context_window` |
| `kg_list_documents` | 取込済みドキュメント一覧・鮮度情報(`archived` 含む) | `filter`, `cursor` |

### 7.3 ツール追加(Phase 2)

| ツール | 概要 |
|---|---|
| `kg_code_for_doc` | ドキュメント/要件に対応するコード要素を返す(confidence付き) |
| `kg_doc_for_code` | コード要素(FQN指定)を記述するドキュメント箇所を返す |
| `kg_impact` | ノード起点の影響範囲分析(DEPENDS_ON/CALLS/IMPLEMENTED_BY を辿る) |
| `kg_stale_docs` | コード更新に追随していない可能性のあるドキュメントを列挙(Phase 3) |

### 7.4 エラー設計

- 該当なしは空結果 + 「次に試すべきツール・クエリ」の提案文を返す(エージェントが行き詰まらないため)。
- `kg_query` の構文エラーは行位置と修正例を返す。書き込みCypher(CREATE/MERGE/DELETE等)は静的に拒否した上で、**read-only トランザクションでの実行を強制**する(静的チェックをすり抜けるプロシージャ経由の書き込みもドライバ層で遮断)。

### 7.5 セキュリティ

- **認証**: HTTP公開時は静的 Bearer トークン(環境変数管理)。利用規模が個人〜数名(同一チーム)のため、トークン発行台帳・失効管理は設けない(トークンローテーションは手動)。Phase 1 では全ユーザー同一権限(ドキュメント単位ACLはスコープ外と明記)。
- **TLS**: リバースプロキシ(nginx / Caddy 等)での終端を推奨構成とし、Compose にプロファイルとして同梱する。同一ホスト内の localhost 利用時は TLS なしを許容。
- **`kg_query` サンドボックス**: read-only トランザクション、タイムアウト5秒、返却1000行上限。
- **間接プロンプトインジェクション(注意喚起)**: 本システムは取り込んだドキュメント原文をそのままエージェントに返す。ドキュメント内に悪意ある指示文が含まれる場合、それがエージェントのコンテキストに入るリスクは MCP の構造上排除できない。信頼できる社内ドキュメントのみを取込対象とすること、およびクライアント側エージェントの権限設計(本システムは読み取り専用であり、被害はクライアント側権限に依存)を利用者向けドキュメントに明記する。

### 7.6 検索仕様(`kg_search` ハイブリッド検索)

3系統の検索を実行し、**RRF(Reciprocal Rank Fusion)** でスコア統合する:

| 系統 | 実現方式 |
|---|---|
| ベクトル検索(チャンク) | `chunk_embedding_idx` にクエリ埋め込み(`検索クエリ: `)で検索 |
| ベクトル検索(ノード名) | `entity_name_idx` にクエリ埋め込み(`トピック: `)で検索 |
| キーワード検索 | Neo4j full-text index(Lucene)。**日本語対応のため CJK アナライザ(bigram)でインデックスを作成する**(デフォルトアナライザは日本語を単語分割できないため必須)。対象: `Chunk.text`、知識ノードの `name` / `aliases` / `definition` |

RRF の k 値・系統別重みは設定値とし、M4 の検索品質評価で調整する。`node_types[]` 指定時はノード系統をラベルでフィルタする。

## 8. 非機能要件

| 項目 | 要件 |
|---|---|
| 可搬性 | Docker Compose 一式でオンプレ・エアギャップ環境に配置可能。推論(LLM・埋め込み)は完全ローカルで、稼働時のデータ外部送信なし(テレメトリ遮断 §12.4) |
| ハードウェア | GPU: VRAM 24GB 以上 ×1(`llm` が主に使用。`embedding` は同居または CPU)。RAM: 32GB 以上目安(Neo4j heap 4–8GB + vLLM ホスト側 + OS)。ディスク: モデル重み・Neo4j データ・dump 保管で 200GB 目安 |
| 規模想定 | Phase 1: ドキュメント 〜1,000件 / ノード 〜10万 / エッジ 〜50万(確定)。初回全件取込は数日規模を許容(再開可能であること) |
| 検索応答 | `kg_search` p95 < 2秒 |
| 取込スループット | 100ページ級ドキュメントを10分以内(GPU + vLLM 並列前提。概算: 100頁 ≒ 200–300チャンク × 2呼び出し = 400–600 LLM呼び出し/600秒 → 実効 1–1.5秒/呼び出しを continuous batching で達成)。llama-server(CPU)代替構成ではこのNFRは適用外とし配備時に個別合意 |
| 再現性 | 抽出プロンプト・オントロジーをバージョン管理し、`pipeline_version` で追跡可能 |
| 観測性 | 取込ジョブの進捗・失敗・LLM処理量(トークン数)を Metadata DB に記録、CLI/簡易ダッシュボードで確認 |
| バックアップ | §12.1 参照(週次停止dump + SQLiteバックアップ) |

## 9. 技術選定比較(Graph Store)

| 候補 | 長所 | 短所 | 判定 |
|---|---|---|---|
| **Neo4j Community** | Cypher標準、ベクトルインデックス内蔵、実績・情報量最大 | JVM常駐でやや重い、Communityはクラスタ不可・オンラインバックアップ不可(§12.1で運用対処) | **第一候補** |
| FalkorDB | 軽量(Redis系)、Cypher互換、Docker一発 | エコシステム小 | 軽量構成の代替 |
| SQLite自作(property graph) | 依存ゼロ、可搬性最強 | 多段トラバースの実装・性能負担 | PoC/小規模のみ |
| Memgraph | 高速、Cypher互換 | ライセンス要確認(BSL) | 保留 |

→ **Neo4j Community で確定**(v0.2)。`GraphStoreAdapter` 抽象化は保険として維持するが、vector index 等 Neo4j 固有機能の利用を優先し、抽象化のために機能を犠牲にしない。

### 9.1 Python フレームワーク選定(確定)

- **LangGraph**: パイプライン骨格(グラフトポロジー、SqliteSaver によるチェックポイント、リトライ)
- **LangChain**: LLM呼び出し抽象化(`with_structured_output` による JSON Schema 強制、ローカルLLMバックエンド間の切替)とドキュメントローダー程度の薄い利用に限定
- **不採用**: LlamaIndex(PropertyGraphIndex)、LangChain LLMGraphTransformer — provenance 付与・差分再抽出・決定的ID生成の要件が高レベルAPIの守備範囲外であり、抽象化の内側改造より自前実装が合理的なため
- **Neo4j I/O**: 公式 `neo4j` ドライバで直接 Cypher。読み取り系(`kg_query`)も同ドライバで統一

## 10. 開発マイルストーン案

| M | 内容 | 完了条件 |
|---|---|---|
| M0(M1前提) | ローカルLLMモデル比較評価(VRAM 24GB / vLLM 前提、日本語関係抽出 × JSON Schema 準拠 × 速度) | 採用モデル決定、`pipeline_version` 初版確定 |
| M1 | オントロジーYAML確定 + Markdown取込→KG構築のE2E(単一形式) | サンプル設計書10件でグラフ生成、手動レビューで抽出妥当性確認。**レビュー結果をゴールドセットとして保存**(M4 の評価基準の種) |
| M2 | エンティティ解決(増分突合含む)+ 差分更新 + 定期スキャン/論理削除 + docx/PDF/Excel(一覧表・複数シート型)対応 | 再取込で重複が増えない、変更差分のみ再抽出される、削除検知→archived 遷移が機能する |
| M3 | MCPサーバー(Phase 1ツール8本、Streamable HTTP + Bearer)+ Claude Code から利用確認 | mcp-builder流の評価QAセット10問で正答確認 |
| M4 | 評価・チューニング(抽出精度、検索品質、confidence較正、帳票型Excel品質実測→VLM要否判断) | M1ゴールドセットに対する実測値で目標確定・達成。RRFパラメータ調整完了 |
| M5 | Phase 2: コードグラフ + クロスリンク | Javaリポジトリ1件でdoc⇔code往復クエリが機能 |

品質基準の決め方(確定): 事前に仮目標を置かず、M1 の手動レビュー結果(アノテーションは開発者自身が実施)をゴールドセット化し、実測 precision を基に「維持+改善」の形で M4 の目標値を確定する。

## 11. 残タスク(オープンクエスチョン)

v0.3 §11 のオープンクエスチョン6件はヒアリング(2026-07-06)で全件クローズした(冒頭の v0.4 決定事項参照)。残るのは以下:

1. **ローカルLLMのモデル選定**(M0): VRAM 24GB / vLLM 前提での候補比較評価。日本語関係抽出品質 × guided decoding 品質 × スループットの3軸
2. **論理削除のパージ期間**: 90日を仮置き。運用開始後に見直し
3. **帳票型Excel の VLM 解析の要否**(M4 で判断): ベストエフォートテキスト化の実測品質次第。採用時は VLM モデル選定と VRAM 配分設計が必要
4. **別紙 `kg-usecases.md` のリポジトリ収載**(§2.1)
5. **confidence 公開閾値の較正**(M4): 仮値 0.8 を実測で調整
6. **RRF パラメータ**(M4): k 値・系統別重みの調整

## 12. 運用仕様

### 12.1 バックアップ / リストア

- **Neo4j**: Neo4j Community はオンラインバックアップ非対応のため、**定期(週次目安)のメンテナンス窓で停止 → `neo4j-admin database dump` → 起動**をスクリプト化して運用する(1,000件規模のDBなら停止時間は数分)。dump は世代管理(直近4世代目安)し、SQLite バックアップと同時に取得して整合を保つ。
- **SQLite(台帳・辞書・チェックポイント)**: dump と同タイミングでファイルコピー(WALチェックポイント後)。同義語辞書・resolution_log は人手作業の蓄積であり、**喪失不可**のため必須。
- **リストア**: dump 復元 → SQLite 復元 → 復元時点以降の差分を定期スキャンで再取込。
- **全再構築(`kg rebuild`)**: dump 喪失時の最終手段。SQLite + 元ドキュメントから全件再取込する。所要は数日 + GPU 占有。**LLM 抽出の非決定性により再構築後のグラフは元と厳密には一致しない**(同義語辞書と決定的ID生成により主要エンティティは安定するが、エッジ集合は揺れる)。この理由から日常の復旧手段はあくまでバックアップとする。

### 12.2 同義語辞書の反映(`kg dict apply`)

辞書更新(エンティティ統合の追加)は **増分マージコマンド**で既存グラフへ即時反映する:

1. `synonyms` テーブルの未適用エントリを検出(適用済み管理は `resolution_log` と突合)
2. 対象2ノードをグラフ上でマージ: エッジを canonical 側へ付け替え、`aliases[]` を統合、被マージ側ノードを削除
3. `resolution_log` に `method: "manual"` で記録

LLM再抽出は不要。逆方向(誤マージの分割)は Phase 1 では手動 Cypher + 対象ドキュメントの再取込で対処し、専用コマンド化は Phase 3 で検討する。

### 12.3 定期パージ

`kg purge`(または定期スキャンの後処理)で、`archived` から既定90日経過した Document とその配下(チャンク・エッジ、他から参照されないノード)を物理削除する(§5.7)。

### 12.4 テレメトリ遮断チェックリスト(Compose 既定値)

| 設定 | 目的 |
|---|---|
| `LANGCHAIN_TRACING_V2=false`(かつ LangSmith API キーを設定しない) | LangSmith への実行トレース送信の遮断 |
| `VLLM_NO_USAGE_STATS=1` / `DO_NOT_TRACK=1` | vLLM 匿名利用統計の遮断 |
| `HF_HUB_OFFLINE=1` | Hugging Face Hub への自動アクセス遮断(モデルはビルド時に取得済み) |
| Neo4j: `dbms.usage_report.enabled=false` | Neo4j の利用状況レポート送信の遮断 |

新規ライブラリ導入時は同種のテレメトリ有無を確認し、本表に追記することを開発規約とする。
