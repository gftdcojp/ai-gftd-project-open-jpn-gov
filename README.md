# ai-gftd-project-open-jpn-gov

> **日本政府オープンディレクトリ + e-Gov 法令API プロキシ** on Cloudflare Workers.
> 1府11省 + 主要庁/独立機関/外局を path-based DID で宣言し、e-Gov 法令API を XRPC でラップ。

[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](./LICENSE)
[![Status: MVP](https://img.shields.io/badge/status-MVP-orange.svg)]()

## なぜ

日本政府の組織図と法令は公開情報ですが、**機械可読な DID 直リンク + 型付き API** にはなっていません。このリポは:

1. 省庁・庁・独立機関・主要外局を **path-based DID** で識別可能にする (`did:web:open-jpn-gov.gftd.ai:ministry:mof` 等)。
2. e-Gov 法令API を **XRPC (AT Protocol) query** でラップし、Edge cache で叩き放題にする。
3. コード1ファイル (+ roster 1ファイル) で全部入り。**DB なし、認証なし、公開データのみ**。

## API (5 XRPC methods)

全て `GET /xrpc/{NSID}`. NSID prefix: `ai.gftd.apps.openJpnGov.*`

| NSID | 説明 |
|---|---|
| `listMinistries` | 省庁 roster 一覧 (`category` で filter: cabinet / ministry / agency / independent) |
| `getMinistry` | `code` または `did` で単体取得 (+配下 agencies の DID list) |
| `listAgencies` | `parentCode` 配下の外局・委員会一覧 |
| `searchLaws` | e-Gov 法令一覧を `query` で絞り込み |
| `getLaw` | `lawNumOrId` で法令本文取得 (`format=plain\|xml`) |

全 schema: [`lexicons/ai/gftd/apps/openJpnGov/`](./lexicons/ai/gftd/apps/openJpnGov)

### Example

```bash
# 財務省の情報
curl 'https://open-jpn-gov.gftd.ai/xrpc/ai.gftd.apps.openJpnGov.getMinistry?code=mof'
# → {"code":"mof","did":"did:web:open-jpn-gov.gftd.ai:ministry:mof","nameJa":"財務省",
#    "nameEn":"Ministry of Finance","category":"ministry","statuteJa":"財務省設置法",
#    "website":"https://www.mof.go.jp/","agencies":["did:web:...:agency:nta"]}

# 経産省の外局
curl 'https://open-jpn-gov.gftd.ai/xrpc/ai.gftd.apps.openJpnGov.listAgencies?parentCode=meti'

# 「個人情報」を含む法令
curl 'https://open-jpn-gov.gftd.ai/xrpc/ai.gftd.apps.openJpnGov.searchLaws?query=個人情報&limit=10'

# 法令本文
curl 'https://open-jpn-gov.gftd.ai/xrpc/ai.gftd.apps.openJpnGov.getLaw?lawNumOrId=平成十五年法律第五十七号'
```

## DID パターン

```
did:web:open-jpn-gov.gftd.ai:{category}:{code}

# 例
did:web:open-jpn-gov.gftd.ai:cabinet:cao             # 内閣府
did:web:open-jpn-gov.gftd.ai:ministry:mof            # 財務省
did:web:open-jpn-gov.gftd.ai:ministry:meti           # 経産省
did:web:open-jpn-gov.gftd.ai:agency:digital          # デジタル庁
did:web:open-jpn-gov.gftd.ai:agency:nta              # 国税庁 (財務省外局)
did:web:open-jpn-gov.gftd.ai:independent:jinji       # 人事院
```

## Roster (約35 entries)

**Cabinet (3)**: 内閣官房 / 内閣法制局 / 内閣府
**省 (11)**: 総務 / 法務 / 外務 / 財務 / 文科 / 厚労 / 農水 / 経産 / 国交 / 環境 / 防衛
**庁 (3)**: デジタル / 復興 / こども家庭
**独立機関 (2)**: 人事院 / 会計検査院
**主要外局・委員会 (17)**: 警察庁 / 金融庁 / 消費者庁 / 文化庁 / スポーツ庁 / 国税庁 / 気象庁 / 海上保安庁 / 観光庁 / 中小企業庁 / 特許庁 / 資源エネルギー庁 / 林野庁 / 水産庁 / 原子力規制委員会 / 出入国在留管理庁 / 公安調査庁 / 消防庁 / 中央労働委員会

全文は [`worker/src/roster.ts`](./worker/src/roster.ts).

## DoDAF v2 / BPMN / DMN / Forms

本リポは **DoDAF v2.02 準拠**のアーキテクチャ記述 + 実行成果物を同梱します。

| DoDAF view | File |
|---|---|
| AV-1 Overview & Summary | [`dodaf/AV-1.json`](./dodaf/AV-1.json) |
| OV-1 High-Level Operational Concept | [`dodaf/OV-1.json`](./dodaf/OV-1.json) |
| OV-5b Operational Activity Model | [`dodaf/OV-5b.json`](./dodaf/OV-5b.json) |
| OV-6a Operational Rules Model | [`dodaf/OV-6a.json`](./dodaf/OV-6a.json) |
| CV-2 Capability Taxonomy | [`dodaf/CV-2.json`](./dodaf/CV-2.json) |
| SV-1 Systems Interface Description | [`dodaf/SV-1.json`](./dodaf/SV-1.json) |

| BPMN 2.0 | File | Process key |
|---|---|---|
| Search Law | [`bpmn/search-law.bpmn`](./bpmn/search-law.bpmn) | `searchLaw` |
| Resolve Ministry | [`bpmn/resolve-ministry.bpmn`](./bpmn/resolve-ministry.bpmn) | `resolveMinistry` |

| DMN 1.3 | File | Decision key |
|---|---|---|
| Law Type Classification | [`dmn/law-type-classification.dmn`](./dmn/law-type-classification.dmn) | `openJpnGov.lawTypeClassification` |

| Camunda Form | File | Form key |
|---|---|---|
| Search Law | [`forms/searchLaw.form.json`](./forms/searchLaw.form.json) | `openJpnGov.searchLaw.v1` |
| Resolve Ministry | [`forms/resolveMinistry.form.json`](./forms/resolveMinistry.form.json) | `openJpnGov.resolveMinistry.v1` |

**Runtime**: `GET /dodaf`, `/dodaf/{viewId}`, `/forms`, `/forms/{formKey}` で閲覧可能。PDS service binding がある時は `ai.gftd.dodafv2.deployView` + `ai.gftd.form.register` に自動投入。

## 非目標 (MVP)

- 47 都道府県 / 市区町村 roster
- e-Stat (政府統計) proxy
- 官報 (Kampō) ingestion
- 電子調達 / 入札
- マイナンバー / 法人番号 lookup
- 国会議員 / 政党 roster

Contributions welcome.

## 上流データ / 免責

- 省庁 roster: 内閣官房組織図 + 各省設置法 (public)
- 法令本文: [e-Gov 法令API v2](https://laws.e-gov.go.jp/api/2/) — デジタル庁提供
- **このリポは非公式で、政府とは一切関係ありません** (`magatama.jsonld` profile に `[AI Agent — unofficial]` disclaimer を付与)

## Running Locally

```bash
git clone https://github.com/gftdcojp/ai-gftd-project-open-jpn-gov
cd ai-gftd-project-open-jpn-gov/worker
npm i -g wrangler
wrangler dev --local    # http://127.0.0.1:8787
```

## License

[Apache License 2.0](./LICENSE). Copyright © 2026 gftd.co.jp.
