# 法務デューデリジェンスAIツール 実装計画（Box + Gemini 2.5 Pro）

本ドキュメントは、「法務デューデリジェンスAIツール 要求仕様 詳細化（2025.10.07版）」を、Box と Google Gemini 2.5 Pro を中核に据えて実装するためのアーキテクチャおよび開発計画を整理したものです。

## 1. 全体コンセプト
- **目的**: Box 上に五月雨式にアップロードされるDD関連文書を即時に取り込み、Gemini 2.5 Pro を用いて網羅的な論点抽出・整合性チェック・レポートドラフト生成までを自動化/半自動化します。
- **スコープ**: 非上場/上場株式譲渡、TOB、カーブアウト案件など典型論点集がカバーする全スキーム。
- **成果物**: 
  - DDレビューのリアルタイム可視化ダッシュボード
  - リスク/不整合の自動ラベリング
  - Word/PPT テンプレート準拠のレポート草稿

## 2. 全体アーキテクチャ

```
Box VDR -> Ingestion Lambda -> Processing Queue -> Analysis Workers (Gemini API)
                                         |-> Metadata DB (PostgreSQL)
                                         |-> Vector Store (Vertex AI Matching Engine)
                                         |-> Orchestration (Temporal/Cloud Tasks)
                                     Frontend (React/Next.js) <- API Gateway <- Backend (FastAPI)
                                                        |
                                                        -> Reporting Service (Docx/PPTX generator)
```

### 2.1 データ投入
1. **Box VDR**: 対象案件ごとにフォルダを作成。文書アップロードイベントを Box Webhook で検知。
2. **Ingestion Lambda (GCP Cloud Functions でも可)**: 
   - Webhook を受信し、Box API でファイルメタデータ+バイナリを取得。
   - バージョン情報・フォルダパスを含むインデックスエントリを生成。
   - 文書本文抽出（Box View API + OCR）。
   - メタデータを PostgreSQL、本文を Cloud Storage、Embedding を Vector Store に格納。

### 2.2 解析パイプライン
- **キュー管理**: Cloud Pub/Sub または AWS SQS を使用し、優先度（例：主要契約＞補足資料）をメッセージ属性に付与。
- **Analysis Workers**:
  - Gemini 2.5 Pro API（法務向け Safety 設定 + System Instruction）を利用。
  - プロンプトに典型論点集の YAML/JSON 仕様を与え、網羅的抽出指示。
  - 出力は構造化 JSON（論点ID、該当条項、抜粋テキスト、根拠ページ等）。
  - ハルシネーション低減: Retrieval-Augmented Generation (RAG) で文書本文の関連セクションを 4K トークン以内にまとめ、引用元を付与。
- **横断分析**:
  - Vector Store から関連文書の embedding を検索し、Gemini の Function Calling 機能で比較分析テンプレートを実行。
  - 例: ガバナンス整合性 → 登記簿 vs 議事録 vs 名簿のキーデータ（就任日、決議日等）を正規化テーブル化し、ルールエンジン（dbt/BigQuery）で検証。
  - スタンドアローン・イシュー → 経理/人事/IT契約の依存項目をタグ抽出し、Knowledge Graph に格納。

### 2.3 ワークフロー管理
- **Backend**: FastAPI + PostgreSQL。案件、文書、タスク、リスクをエンティティとして管理。
- **Orchestration**: Temporal や Airflow で「文書解析 → 横断分析 → レポート反映」をジョブ化。
- **リアルタイム通知**: 
  - 解析ステータスは WebSocket (Ably/Pusher) でフロントに push。
  - Slack/Microsoft Teams に解析完了やリスク検出を通知。

### 2.4 フロントエンド
- **Stack**: Next.js + TypeScript + Chakra UI。
- **機能**:
  - 文書処理キューの可視化（未着手/解析中/要確認/完了）。
  - タスク割当（ドラッグ＆ドロップ）。
  - リスクダッシュボード（重要度、論点ID、根拠文書リンク）。
  - スタンドアローン・イシュー一覧とTSA検討ステータス。

## 3. 主要機能別の詳細設計

### 3.1 個別文書の深層分析
- **プロンプトテンプレート**:
  - System: 「あなたは日本のM&A法務DD専門家。典型論点集（JSON）を遵守し、事実関係の根拠抜粋付きで網羅的に抽出。」
  - User: 「以下は {document_type}。該当論点IDで事実を抽出し、該当箇所の引用を返せ。」
  - Tool: Retrieval で抜粋を提供。
- **Gemini Function Schema**:
  ```json
  {
    "type": "object",
    "properties": {
      "document_id": {"type": "string"},
      "issues": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "issue_id": {"type": "string"},
            "facts": {"type": "string"},
            "evidence_refs": {"type": "array", "items": "string"},
            "confidence": {"type": "number"}
          },
          "required": ["issue_id", "facts", "evidence_refs"]
        }
      }
    },
    "required": ["document_id", "issues"]
  }
  ```
- **ポストプロセス**: evidence_refs を利用して Box ファイル内のページ/段落にハイライトを設定（Box Annotations API）。

### 3.2 文書間整合性検査
- **データ正規化**: Gemini で抽出した構造化データを BigQuery にロードし、dbt モデルで整合性ルールを SQL 化。
- **例: ガバナンス検証**
  ```sql
  select board.member_name, board.start_date, registry.registered_until
  from registry
  left join board_minutes board on registry.member_name = board.member_name
  where board.start_date > registry.registered_until;
  ```
- **アラート生成**: 検出結果をメッセージングキューに載せ、Gemini に説明文生成を依頼。重要度ラベルは事前定義ルール + Gemini のスコアリングで算出。

### 3.3 スタンドアローン・イシュー抽出
- Gemini に対し、契約/人事/IT文書から依存要素を抽出する専用プロンプトを用意し、Knowledge Graph（Neo4j）に格納。Graph クエリで依存関係を可視化。

## 4. レポート作成ワークフロー

1. **レビュー結果統合**
   - 各担当弁護士のコメントはフロントから API 経由で収集し、PostgreSQL に保存。
   - Gemini に「統合ブリーフ」を指示し、論点ごとに人コメントとAI抽出をマージ。

2. **Word レポート自動生成**
   - Python-docx で雛形を読み込み、Gemini 出力を挿入。
   - リスト系論点は pandas -> tabulate -> docx テーブルに変換。
   - 生成物は Box に自動アップロード。

3. **PowerPoint サマリー生成**
   - python-pptx でテンプレートを加工。
   - 重要論点は Gemini にスコアリングさせ、上位N件をサマリーに配置。
   - ページ参照は Word レポート生成時に付与したアンカーを参照し自動記載。

## 5. セキュリティ・ガバナンス
- **権限管理**: Box のグループ/ロールとアプリ内ロールを SSO (Azure AD / Okta) で連携。
- **監査ログ**: すべてのAPI呼出・Geminiプロンプト/レスポンスを Cloud Logging + SIEM で保管。
- **データ保護**: 
  - Gemini には client-side encryption + Data Residency を設定。
  - Box Shield で異常アクセスを検知。

## 6. 開発・運用計画
- **フェーズ1 (3ヶ月)**: 基本機能（文書取込、個別論点抽出、ダッシュボードMVP）。
- **フェーズ2 (3ヶ月)**: 横断分析、スタンドアローン・イシュー、レポート自動生成。
- **フェーズ3 (継続)**: 精度改善、追加スキーム対応、TSA提案自動化。
- **MLOps**: Vertex AI Pipeline で継続的評価、Gemini プロンプト最適化の A/B テストを実施。

## 7. 実装に向けた前提と次ステップ
1. 典型論点集・レポート雛形のデジタル化（構造化データ化）。
2. Box テナントへのアプリ登録、Webhook 設定、API スコープ承認。
3. Gemini 2.5 Pro の Enterprise 契約と Data Governance 設定。
4. プロトタイプ実装 → 代表案件でパイロット運用 → フィードバック反映。

---
本計画により、Box と Gemini 2.5 Pro を組み合わせて要求仕様を満たす実装が可能です。継続的なプロンプト/ルールチューニングと、法務チームとの協働により品質を高めていくことを推奨します。
