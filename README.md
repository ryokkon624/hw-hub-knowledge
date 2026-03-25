# Housework Hub (HwHub)

![Java](https://img.shields.io/badge/Java-21-orange)
![SpringBoot](https://img.shields.io/badge/SpringBoot-4.x-brightgreen)
![Vue](https://img.shields.io/badge/Vue-3-42b883)
![Terraform](https://img.shields.io/badge/Terraform-managed-blue)
![Coverage](https://img.shields.io/badge/coverage-95%25-brightgreen)
---

## Overview

Housework Hub（HwHub）は、家庭内の家事・買い物・メンバー管理を協調的に行うためのアプリケーションです。  
複数のおうち（Household）をサポートし、家事タスクのテンプレート化、定期実行、担当者割当、履歴管理などを提供します。  
ユーザーからの問い合わせには Claude API を活用した AI 自動返信機能を備えており、解決できない場合はサポートスタッフへのエスカレーションも可能です。

本リポジトリ群は以下の構成で成り立っています。

- **hw-hub-backend** : メインAPI（Spring Boot / MyBatis / MySQL）
- **hw-hub-batch** : 定期バッチ処理（Spring Batch / ECS Fargate）
- **hw-hub-frontend** : フロントエンド（Vue 3 + Vite + TypeScript）
- **hw-hub-database** : DBスキーマ・Flywayマイグレーション管理
- **hw-hub-infra** : AWSインフラ（Terraform）
- **hw-hub-knowledge** : AIサポートナレッジ（S3同期）

---

## Architecture

- Backend / Batch は AWS ECS Fargate 上で稼働
- DB は Amazon RDS (MySQL)
- ファイル保存は S3
- 認証は JWT
- フロントエンドは S3 + CloudFront によりホスティング
- バッチは EventBridge Scheduler により起動
- インフラは Terraform により管理

### High-level Flow

Online (Frontend + Backend)

```mermaid
flowchart LR
    Browser["Browser"]
    CloudFront["CloudFront"]
    S3["S3 (Frontend SPA)"]
    ALB["ALB"]
    ECS["ECS (Backend API)"]
    RDS["RDS (MySQL)"]
    Browser --> CloudFront
    CloudFront --> S3
    S3 --> ALB
    ALB --> ECS
    ECS --> RDS
```

Batch Processing

```mermaid
flowchart LR
    Scheduler["EventBridge Scheduler"]
    ECSBatch["ECS (Batch Task)"]
    RDS["RDS (MySQL)"]
    S3Knowledge["S3 (Knowledge)"]
    ClaudeAPI["Anthropic Claude API"]
    Scheduler --> ECSBatch
    ECSBatch --> RDS
    ECSBatch --> S3Knowledge
    ECSBatch --> ClaudeAPI
```

---

## Support Flow（問い合わせサポートフロー）

ユーザーが問い合わせを起票してからクローズするまでのフローです。  
AI による自動返信を一次対応とし、解決できない場合はサポートスタッフにエスカレーションします。  
ナレッジベースは `hw-hub-knowledge` リポジトリで管理し、main へのマージで S3 に自動同期されます。

### シーケンス図

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant API as Backend API
    participant DB as RDS (MySQL)
    participant Batch as AI返信バッチ
    participant S3 as S3 (Knowledge)
    participant Claude as Anthropic Claude API
    actor Staff as サポートスタッフ

    User->>API: 問い合わせ作成 (POST /api/inquiries)
    API->>DB: t_inquiry 登録 (status=OPEN)
    API-->>User: 201 Created

    Note over Batch: EventBridge Scheduler<br/>毎時45分に起動

    Batch->>DB: OPEN の問い合わせを取得
    Batch->>S3: faq.md / howto.md を取得
    Batch->>Claude: 問い合わせ内容 + ナレッジを送信
    Claude-->>Batch: AI回答テキスト
    Batch->>DB: t_inquiry_message 登録 (sender_type=AI)
    Batch->>DB: t_inquiry 更新 (status=AI_ANSWERED)
    Batch->>DB: t_notification 登録（ユーザーへ通知）

    User->>API: 問い合わせ詳細確認
    API->>DB: メッセージスレッド取得
    API-->>User: AI回答を表示

    alt AI回答で解決した場合
        User->>API: クローズ (POST /api/inquiries/{id}/close)
        API->>DB: t_inquiry 更新 (status=CLOSED)
        API-->>User: 200 OK
    else AI回答で解決しなかった場合
        User->>API: エスカレーション (POST /api/inquiries/{id}/escalate)
        API->>DB: t_inquiry 更新 (status=PENDING_STAFF)
        API-->>User: 200 OK

        Staff->>API: 対応待ち一覧確認 (GET /api/admin/inquiries)
        API->>DB: PENDING_STAFF 一覧取得
        API-->>Staff: 問い合わせ一覧

        Staff->>API: スタッフ返信 (POST /api/admin/inquiries/{id}/reply)
        API->>DB: t_inquiry_message 登録 (sender_type=STAFF)
        API->>DB: t_inquiry 更新 (status=STAFF_ANSWERED)
        API->>DB: t_notification 登録（ユーザーへ通知）
        API-->>Staff: 200 OK

        User->>API: 返信確認 → クローズ
        API->>DB: t_inquiry 更新 (status=CLOSED)
        API-->>User: 200 OK
    end
```

### ステータス遷移

```mermaid
stateDiagram-v2
    direction LR

    [*] --> OPEN : 問い合わせ作成

    OPEN --> AI_ANSWERED : AI返信バッチ実行

    AI_ANSWERED --> CLOSED : ユーザーがクローズ
    AI_ANSWERED --> PENDING_STAFF : ユーザーがエスカレーション

    PENDING_STAFF --> STAFF_ANSWERED : スタッフが返信
    STAFF_ANSWERED --> CLOSED : ユーザーがクローズ
    STAFF_ANSWERED --> PENDING_STAFF : ユーザーが追加返信

    CLOSED --> [*]
```

### ナレッジ管理フロー

```mermaid
flowchart LR
    Repo["hw-hub-knowledge\n(faq.md / howto.md)"]
    Actions["GitHub Actions\n(main push)"]
    S3["S3\n(hwhub-knowledge)"]
    Batch["AI返信バッチ"]

    Repo -->|"aws s3 sync"| Actions
    Actions --> S3
    S3 -->|"GetObject"| Batch
```

---

## Tech stack

### Backend
- Java 21
- Spring Boot 4.0.X ※3.5.Xからバージョンアップ済み
- MyBatis + MyBatis Generator
- Flyway
- MySQL

### Frontend
- Vue 3 + Composition API
- TypeScript
- Pinia
- Tailwind CSS
- vue-i18n

### Infrastructure
- AWS ECS Fargate
- Application Load Balancer
- Amazon RDS (MySQL)
- Amazon S3
- CloudFront
- EventBridge Scheduler
- CloudWatch / SNS
- **Terraform**

---

## Repository Structure

| Repository | Role |
|------------------------------------------------------------------|-----------------------------|
| [hw-hub-backend](https://github.com/ryokkon624/hw-hub-backend)   | REST API / authentication / business logic |
| [hw-hub-batch](https://github.com/ryokkon624/hw-hub-batch)       | scheduled batch processing |
| [hw-hub-frontend](https://github.com/ryokkon624/hw-hub-frontend) | Web UI |
| [hw-hub-database](https://github.com/ryokkon624/hw-hub-database) | Flyway database schema |
| [hw-hub-infra](https://github.com/ryokkon624/hw-hub-infra) | Terraform infrastructure |
| [hw-hub-knowledge](https://github.com/ryokkon624/hw-hub-knowledge) | AI support knowledge base (S3 sync) |

---

## Repository Relationship

```mermaid
flowchart LR

Frontend["hw-hub-frontend"]
Backend["hw-hub-backend"]
Batch["hw-hub-batch"]
DB["hw-hub-database"]
Infra["hw-hub-infra"]
Knowledge["hw-hub-knowledge"]

Frontend --> Backend
Backend --> DB
Batch --> DB
Knowledge --> Batch
Infra --> Backend
Infra --> Batch
```

---

## CI / CD 概要

GitHub Actions により CI/CD を構築しています。

main への push で以下を実行：

- テスト
- カバレッジ生成
- Docker build & push (ECR)
- ECS TaskDefinition 更新
- ECS Service / Scheduler 反映

---

## Coverage Report
- Backend: [GitHub Pages](https://ryokkon624.github.io/hw-hub-backend/)
- Batch: [GitHub Pages](https://ryokkon624.github.io/hw-hub-batch/)
- Frontend: [GitHub Pages](https://ryokkon624.github.io/hw-hub-frontend/)

---

## Infrastructure as Code

**Terraform**を使用しています。

Managed resources include:

- ECS services
- ECS batch tasks
- networking configuration
- monitoring and alerts
- scheduled jobs

Some existing AWS resources are referenced rather than managed:

- ALB
- RDS
- CloudFront
- S3
- SNS

---

## Development

各リポジトリにそれぞれの詳細を記載したREADMEファイルがあります。

- [backend_README.md](https://github.com/ryokkon624/hw-hub-backend/blob/main/backend_README.md)
- [batch_README.md](https://github.com/ryokkon624/hw-hub-batch/blob/main/batch_README.md)
- [frontend_README.md](https://github.com/ryokkon624/hw-hub-frontend/blob/main/frontend_README.md)
- [database_README.md](https://github.com/ryokkon624/hw-hub-database/blob/main/database_README.md)
- [infra_README.md](https://github.com/ryokkon624/hw-hub-infra/blob/main/infra_README.md)

---

## Versioning Strategy

HwHub は [Semantic Versioning](https://semver.org/) に基づいてバージョンを管理しています。

### バージョン形式

| 種別 | 形式 | 例 | 説明 |
|------|------|----|------|
| PRD リリース | `vMAJOR.MINOR.PATCH` | `v1.2.0` | git tag がそのままバージョンになる |
| STG ビルド | `vMAJOR.MINOR.PATCH-stg.N` | `v1.2.0-stg.3` | 直前のタグから N コミット後のビルド |
| ローカル開発 | `local` | `local` | 環境変数未設定時のデフォルト値 |

### バージョンアップルール

| 変更種別 | 上げるバージョン | 例 |
|----------|------------------|----|
| 破壊的変更（API非互換・DB大規模変更） | MAJOR | `v1.0.0` → `v2.0.0` |
| 機能追加（feature リリース） | MINOR | `v1.0.0` → `v1.1.0` |
| バグ修正のみ | PATCH | `v1.0.0` → `v1.0.1` |

### リリースフロー

```mermaid
flowchart LR
    Commit["main への push"]
    STG["STG deploy\n(stg-SHA タグ)"]
    Check["STG 動作確認"]
    Tag["git tag vX.Y.Z"]
    PRD["PRD deploy\n(prd-vX.Y.Z タグ)"]

    Commit --> STG
    STG --> Check
    Check --> Tag
    Tag --> PRD
```

### リポジトリ別バージョン管理

各リポジトリのタグは独立して管理されます。  
PATCH リリースではフロントエンドのみ・バックエンドのみのデプロイが発生するため、  
アプリ情報画面ではフロントとAPIのバージョンを個別に表示しています。

| リポジトリ | タグ管理 | CI/CDトリガー |
|------------|----------|---------------|
| hw-hub-frontend | 独立タグ | main push → STG / git tag → PRD |
| hw-hub-backend | 独立タグ | main push → STG / git tag → PRD |
| hw-hub-batch | 独立タグ | main push → STG / git tag → PRD |

### バージョン情報の伝搬

```mermaid
flowchart LR
    Tag["git tag / git describe"]
    Actions["GitHub Actions"]
    Backend["Backend\nAPP_VERSION 環境変数"]
    Batch["Batch\nAPP_VERSION 環境変数"]
    Frontend["Frontend\nVITE_APP_VERSION\nビルド時埋め込み"]
    API["GET /actuator/info"]
    UI["アプリ情報画面"]

    Tag --> Actions
    Actions --> Backend
    Actions --> Batch
    Actions --> Frontend
    Backend --> API
    API --> UI
    Frontend --> UI
```

### ECR イメージタグ規則

| 環境 | タグ形式 | 例 |
|------|----------|----|
| STG（コミット単位） | `stg-${GITHUB_SHA}` | `stg-abc1234` |
| STG（最新） | `stg-latest` | `stg-latest` |
| PRD（バージョン単位） | `prd-vX.Y.Z` | `prd-v1.2.0` |
| PRD（最新） | `prd-latest` | `prd-latest` |

---

## Future Roadmap

Planned improvements:

- mobile application (Capacitor)
- analytics dashboard

---

## Project Status

- architecture established
- CI/CD pipeline implemented
- AI-powered inquiry support implemented (Claude API)
- role-based admin panel implemented
- high test coverage achieved
- infrastructure managed via Terraform
