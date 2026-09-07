# YukkuriGen — AI から使う「ゆっくり解説」動画生成

台本（話者＋セリフの配列）を渡すと、**MP4** が返る。こちらで音声を合成してから
本番レンダーを開始する（5クレジット・有料プラン）。`output:"preview"` なら
1クレジット・低解像度・20秒で、**無料プランでも動くものが見られる**。
人が編集画面で作ることは想定していない——**あなたが自分の AI に頼み、その AI が
ここを呼ぶ。**

```
あなた「ゆっくり解説で、日本の年金制度の動画を10本作って」
  → AI が create_yukkuri_videos_batch を呼ぶ
  → MP4 が10本返る
あなた「3本目、もう少しゆっくり喋らせて」
  → AI が update_lines を呼び直す
```

直すのもこのサイトではなく、AI との会話で行う。

## つなぐ

まず鍵を取る（無料・月10クレジット）:
https://app.yukkurigen.com/settings/api-keys

### Claude Code

```bash
claude mcp add --transport http yukkurigen https://app.yukkurigen.com/api/mcp \
  -H "Authorization: Bearer $YUKKURIGEN_API_KEY"
```

チームで共有するなら、このリポジトリの `.mcp.json` をプロジェクトに置く。
鍵は環境変数 `YUKKURIGEN_API_KEY` から読む（ファイルに鍵を書かない）。

### Codex

`~/.codex/config.toml` に足す:

```toml
[mcp_servers.yukkurigen]
url = "https://app.yukkurigen.com/api/mcp"
bearer_token_env_var = "YUKKURIGEN_API_KEY"
```

```bash
export YUKKURIGEN_API_KEY=yg_live_...
```

鍵は環境変数から読むので、設定ファイルに書かない。

### Gemini CLI

```bash
export YUKKURIGEN_API_KEY=yg_live_...
gemini extensions install https://github.com/hitoshinoda/yukkurigen-mcp
```

このリポジトリの `gemini-extension.json` が読まれる。鍵は環境変数から読むので、
ファイルに書かない。

### その他の MCP クライアント

エンドポイントは1つだけ:

```
POST https://app.yukkurigen.com/api/mcp
Authorization: Bearer <API key>
```

JSON-RPC 2.0 over HTTP。`tools/list` でツール一覧が取れる。
`server.json` は MCP レジストリ用のマニフェスト。

### REST で直接叩く

```bash
curl -X POST https://app.yukkurigen.com/api/v1/agent/generate \
  -H "Authorization: Bearer $YUKKURIGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"テスト","script":[{"speaker":"reimu","text":"こんにちは"},{"speaker":"marisa","text":"よろしくだぜ"}]}'
```

全項目の定義: https://app.yukkurigen.com/openapi.json

## AI に読ませるもの

**[SKILL.md](./SKILL.md)** が手順書のすべて。接続、最小例、立ち絵、カメラ、
クレジット、402 を受けたときの購入導線、冪等キー、バッチ生成、コールバックまで。
`app.yukkurigen.com/skill.md` と同一のファイルで、CI で一致を検査している。

## できること / かかるもの

| | クレジット | 無料枠で |
|---|---|---|
| 台本を作る・読む・直す・音声を作る | 0 | ○ |
| MP4 プレビュー（低解像度・指定行の周辺だけ） | 1 | ○（月10回） |
| MP4 本番レンダー | 5 | × 有料プラン |

無料枠は月10クレジット。402 を受けたら `create_checkout` で決済リンクを出せる
——会話を止めずに購入まで進める。

同時に走らせられるレンダーはプランごとに上限がある（無料1 / スタンダード3 /
プロ10）。超えると `429 too_many_concurrent_renders`（課金なし）。

## 正直に言っておくこと

- **`create_yukkuri_video` から MP4 まで一息に通す経路は本番未検証**（2026-09-07 時点）。
  台本の作成・音声の合成・レンダーの各段は個別に動いているが、1コールで最後まで
  通した実績がまだ無い。**まず少数の行で試して、`get_render` が `completed` を
  返すことを確かめてほしい。** 途中で止まる場合は段ごとに切り分けられる
  （`get_project` → `generate_audio` → `render_mp4`）。不具合として報告してほしい。
- **`.ymmp`（YMM4 プロジェクト）の書き出しは 2026-09-07 に撤去した。** 音声も
  立ち絵も相手の YMM4 が作る形で、こちらの音声合成を一度も通らなかった——
  代替が容易なわりに、保守する面だけが増えていた。MP4 一本にした。
- **OAuth 認可サーバは提供していない。** 鍵は画面で発行する API キー。
  RFC 9728 の Protected Resource Metadata は出しているが、`authorization_servers`
  は載せていない（無いものを広告しないため）。
- MP4 の完了通知（`callbackUrl`）は、こちら側が「終わったこと」を確定させた
  時点で送る。誰もポーリングしていない場合は掃除の巡回まで待つ。急ぐなら
  `get_render` を1〜2回叩けばその場で確定する。

## リンク

- [ヘルプセンター](https://help.yukkurigen.com/)
- [自分の AI につなぐ](https://help.yukkurigen.com/connect-your-ai)
- [料金](https://yukkurigen.com/pricing)
- [OpenAPI](https://app.yukkurigen.com/openapi.json)
- [llms.txt](https://app.yukkurigen.com/llms.txt)

## ライセンス

[MIT](./LICENSE)。**このリポジトリの中身（接続情報・マニフェスト・文書）に
対するライセンスであって、YukkuriGen のサービス本体には及ばない。**
サービスの利用条件は[利用規約](https://yukkurigen.com/terms)による。
