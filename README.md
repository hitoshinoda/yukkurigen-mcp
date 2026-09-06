# YukkuriGen — AI から使う「ゆっくり解説」動画生成

台本（話者＋セリフの配列）を渡すと、**YMM4 プロジェクト(.ymmp)** か **MP4** が返る。
人が編集画面で作ることは想定していない——**あなたが自分の AI に頼み、その AI が
ここを呼ぶ。**

```
あなた「ゆっくり解説で、日本の年金制度の動画を10本作って」
  → AI が create_yukkuri_videos_batch を呼ぶ
  → .ymmp が10本返る
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

### Gemini CLI

```bash
export YUKKURIGEN_API_KEY=yg_live_...
gemini extensions install https://github.com/<owner>/<repo>
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
| 台本を読む・直す・音声を作る | 0 | ○ |
| `.ymmp` 書き出し | 3 | ○（月3回） |
| MP4 プレビュー（低解像度・指定行の周辺だけ） | 1 | ○ |
| MP4 本番レンダー | 5 | × 有料プラン |

無料枠は月10クレジット。402 を受けたら `create_checkout` で決済リンクを出せる
——会話を止めずに購入まで進める。

同時に走らせられるレンダーはプランごとに上限がある（無料1 / スタンダード3 /
プロ10）。超えると `429 too_many_concurrent_renders`（課金なし）。

## 正直に言っておくこと

- **生成される .ymmp は実機の YMM4 では未検証。** 自前のスキーマと、公開されて
  いる .ymmp から読み取った型名に照らして検査しているだけで、「YMM4 で開いて
  再生できた」ことは確認していない。特に AquesTalk（ゆっくり霊夢・魔理沙）の
  `VoiceParameter` の型名は他の実装と食い違っている。**まず少数の行で試して
  ほしい。** 開けなければ生成側の不具合として報告してほしい。
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
