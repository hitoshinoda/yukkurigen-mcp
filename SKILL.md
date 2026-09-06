# YukkuriGen — ゆっくり動画を作る (Agent Skill)

テーマから「ゆっくり解説/劇場」動画の **YMM4プロジェクト(.ymmp)** を生成するためのスキル。
あなた（AI）が台本を書き、YukkuriGen が .ymmp に変換してダウンロードURLを返します。
ユーザーはそれを YMM4 で開いて仕上げ・書き出しできます。

## 接続

- 発行: ユーザーに `https://app.yukkurigen.com/settings/api-keys` でAPIキーを発行してもらう。
  **無料プランでもここまで使える**（無料枠10クレジット）:
  - `.ymmp` の生成・書き出し（1回3クレジット＝3回ぶん）
  - **MP4 プレビュー**（`render_mp4` + `preview`。1クレジット・640×360・最大20秒＝10回ぶん）
    ——まず動くものを見せたいときはこれを使う。
  - 本番の MP4 レンダー（全編・本番解像度・5クレジット）だけは有料プラン
    〈スタンダード以上〉または買い切りライセンスが必要。`402 plan_required` が返る
    ——そのときは下の「支払いが要るとき」に従って購入まで案内する。
- MCP: `claude mcp add --transport http yukkurigen https://app.yukkurigen.com/api/mcp --header "Authorization: Bearer <KEY>"`
- REST: `Authorization: Bearer <KEY>` を付けて `https://app.yukkurigen.com/api/v1/...` を叩く。

## ツール（MCP）/ エンドポイント（REST）

| やること | MCP tool | REST |
|---|---|---|
| キャラ一覧 | `list_characters` | `GET /api/v1/characters` |
| 残高確認 | `get_credits` | `GET /api/v1/credits` |
| 台本→.ymmp を一括生成 | `create_yukkuri_video` | `POST /api/v1/agent/generate` |
| 複数本をまとめて作る（最大20） | `create_yukkuri_videos_batch` | `POST /api/v1/agent/generate/batch` |
| プロジェクト一覧 | `list_projects` | `GET /api/v1/projects` |
| 台本を読み返す（行番号つき） | `get_project` | `GET /api/v1/projects/{id}` |
| 指定した行だけ直す | `update_lines` | `PATCH /api/v1/projects/{id}/lines` |
| 音声を作る（MP4の前に必須） | `generate_audio` | `POST /api/v1/projects/{id}/audio/generate` |
| 既存プロジェクトを書き出し | `export_ymmp` | `POST /api/v1/projects/{id}/export/ymmp` |
| 書き出し状態/再DL | `get_export` | `GET /api/v1/exports/{exportId}` |
| 書き出し一覧（拾い直し） | `list_exports` | `GET /api/v1/exports` |
| MP4 レンダー開始(非同期) | `render_mp4` | `POST /api/v1/projects/{id}/render` |
| レンダー進捗/出力URL | `get_render` | `GET /api/v1/projects/{id}/render/{renderId}/progress` |
| 立ち絵/BGMパスの保存・取得 | — | `PUT` / `GET /api/v1/ymm4-assets` |
| 決済リンクを出す（402のとき） | `create_checkout` | `POST /api/v1/billing/checkout-link` |
| 支払いが済んだか確認 | `get_checkout` | `GET /api/v1/billing/checkout-link/{sessionId}` |

`list_projects` / `get_project` / `update_lines` / `generate_audio` は**消費なし**。
確認用プレビュー（`render_mp4` + `preview`）は**1クレジット**。課金は
書き出し（`export_ymmp` / `create_yukkuri_video`）とレンダー（`render_mp4`）
だけで起きる。直すこと自体にお金がかからないので、ユーザーが納得するまで
何度でも直してよい。

## 手順

1. **キャラを選ぶ**: `list_characters` で `id` と音声エンジンを確認。
   **1本の動画に出せるのは同じ一座のキャラだけ**（立ち絵の系統が違うと画面に出せない）:
   - ゆっくり（東方）: `reimu`（霊夢）, `marisa`（魔理沙）
   - 東北勢: `zundamon`（ずんだもん）, `metan`（四国めたん）, `tsumugi`（春日部つむぎ）, `anko`（あんこもん）

   混ぜると `400 cast_mismatch` を返す（課金なし）。応答の `casts` に一座の一覧が入っている。
   テンプレートは台本から自動で選ぶので、`templateId` は指定しなくてよい。
2. **台本を書く**: 話者(`speaker`=キャラid)とセリフ(`text`)の配列を作る。掛け合い形式が「ゆっくり」らしい。
   - 漢字の読み間違いを避けたい箇所は `reading`（発音かな）を付ける。
   - 冒頭の挨拶 → 本題（結論→理由→具体例）→ まとめ、の構成が定番。
   - **独自の意見・体験を1つ以上入れる**と、量産テンプレに埋もれない動画になる。
3. **背景を決める**: `backgroundImageUrl`（https のURL）を渡す。
   **省略すると無地の背景になり、画面の大半が空いた動画になる。** 場面転換を付けたい
   ときは、その行の `backgroundImageUrl` に別のURLを入れる（以降の行へ引き継がれる）。
4. **カメラを置く**: 盛り上がり・感情のピーク・オチの行に `cameraMode: "dynamic"` を
   付けると、**背景ごとカメラがその話者に寄る**。立ち絵を消したい行（場面の要約など）は
   `"summary"`。
   **2〜4回に留めること。全行に付けると寄りっぱなしになり、寄りが効かなくなる。**
   MP4 レンダーにのみ効き、`.ymmp` には現れない。
5. **生成する**: `create_yukkuri_video`（または `POST /agent/generate`）に `{ output: "ymmp", title, backgroundImageUrl, script }` を渡す。
6. **受け取る**: `downloadUrl`（24時間有効）を返すので、ユーザーに渡す。期限切れなら `get_export` で再取得。

## 直す（ユーザーと会話しながら）

このサービスに編集画面を開いてもらう必要はない。**ユーザーの「ここを変えて」を
そのまま受けて、あなたが直す。**

1. **読む**: `get_project` で今の台本を行番号つきで受け取る。何行目を直すのかを
   ユーザーの言葉から特定する。
2. **直す**: `update_lines` に**変える行だけ**を渡す。渡さなかった行は一切変わらない
   ので、ユーザーが気に入っている部分を壊す心配がない。
   ```json
   { "edits": [ { "index": 3, "text": "実は理由はもっと単純なのだ" },
                { "index": 5, "backgroundImageUrl": "https://example.com/bg2.jpg" } ] }
   ```
3. **音声を作り直す**: `text` / `speaker` / `reading` を変えた行は、**その行の音声が
   無効化される**（応答の `audioInvalidated` に行番号が入る）。古い音声を残すと
   新しい字幕と違うことを喋る動画になるため、こちらで必ず消している。
   `generate_audio` を呼んで作り直す。
4. **安く確かめる**: `render_mp4` に `preview: { fromLine, toLine }` を付けると、
   **1クレジット**・低解像度で**その範囲だけ**焼ける。直した箇所を見るのはこちら。
5. **出す**: `preview` を付けずに `render_mp4`（5クレジット・全編・本番解像度）。

台本を作り直す（`create_yukkuri_video` をもう一度呼ぶ）のは最後の手段。
ユーザーが良いと言った行まで消えるうえ、書き出しに3クレジットかかる。

## MP4 は音声を自動生成しない

`render_mp4` は**行に載っている音声を再生するだけ**で、音声を作りはしない。
音声が無い行があるまま呼ぶと、**課金せずに `409 missing_audio`** を返して止まる
（無音の動画に5クレジット払わせないため）。応答の `missingAudioLines` に対象行が
入っているので、`generate_audio` を呼んでから再実行する。

`create_yukkuri_video` が作る `.ymmp` にはこの手順は要らない。YMM4 が開いたときに
音声を作り直すため。**MP4 を出すときだけ** `generate_audio` が必要になる。

## 最小例（REST）

```bash
curl -X POST https://app.yukkurigen.com/api/v1/agent/generate \
  -H "Authorization: Bearer $YUKKURIGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "output": "ymmp",
    "title": "身近な熱力学のはなし",
    "script": [
      { "speaker": "reimu",  "text": "今日は熱力学第二法則を解説するわ" },
      { "speaker": "marisa", "text": "エントロピーってやつだな" },
      { "speaker": "reimu",  "text": "そう。散らかった部屋が勝手に片付かないのと同じ理屈よ", "cameraMode": "dynamic" }
    ]
  }'
```

## 立ち絵を付ける（任意）

既定では音声＋字幕のみ。**動く立ち絵**を付けたい場合は、ユーザー自身のローカル立ち絵フォルダを
`tachieDirs` で渡す（キャラid → フォルダ。中に 眉/目/口/体 のサブフォルダがある構成）。

```json
{ "script": [...], "tachieDirs": { "zundamon": "D:\\素材\\ずんだもん" } }
```

素材はユーザーの環境のものを参照するだけで、こちらから配布はしない。

立ち絵にフォルダ（パーツ方式）を指定する場合、パーツのファイル名は既定で
`<フォルダ>/<部位>/00.png` を仮定する。実際の配布パックは名前が異なることが多いので
（例: `眉/02.png`, `口/デフォルト.png`）、分かる場合は `tachieParts` で明示する:

```json
{
  "script": [...],
  "tachieDirs":  { "zundamon": "D:\\pack" },
  "tachieParts": { "zundamon": { "eyebrow": "D:\\pack\\眉\\02.png",
                                 "mouth":   "D:\\pack\\口\\デフォルト.png" } }
}
```

キーは `eyebrow` / `eye` / `mouth` / `body` / `hair` / `complexion` のみ（他は400）。
`tachieDirs` / `tachieFaces` / `bgmPath` と同じく `PUT /api/v1/ymm4-assets` に保存でき、
保存すれば以降の書き出しに自動適用される。

**パスは絶対パスで渡すこと**（`C:\\...` などのドライブパス、または UNC）。
`/images/...` のようなサイト絶対パスやURLは、YMM4がローカルファイルとして解決できないため
**無視される**（壊れた「ファイルが無い」クリップを出さないため）。

**表情を切り替えたい場合は、立ち絵を「1枚画像」で指定すること。**
`tachieFaces`（キャラid → 表情 → 画像パス）＋ 各行の `emotion` で顔が切り替わるが、
これは立ち絵に**画像ファイル**を指定したキャラにのみ効く。フォルダ（パーツ方式）を
指定したキャラでは表情指定は無視される。

```json
{
  "tachieDirs":  { "zundamon": "D:\\素材\\ずんだもん\\normal.png" },
  "tachieFaces": { "zundamon": { "angry": "D:\\素材\\ずんだもん\\angry.png" } },
  "script": [{ "speaker": "zundamon", "text": "むむむ", "emotion": "angry" }]
}
```

**一度設定すれば毎回自動で付く**: ユーザーに `PUT /api/v1/ymm4-assets` で
`{ "tachieDirs": {...}, "bgmPath": "...", "tachieFaces": {...} }` を保存してもらえば、
以降の書き出しはリクエストで指定しなくても立ち絵とBGMが入る（リクエスト指定が優先）。
**PUT は全置換**なので、送らなかったキーは消える。部分更新したい場合は先に `GET` して
マージしてから送ること。

## クレジットとエラー

- 消費: `.ymmp` 書き出し = 3、MP4 レンダー = 5、AI台本生成 = 1。
- 不足時: `code: "insufficient_credits"` / `code: "plan_required"`。**諦めずに購入まで案内すること**（下の「支払いが要るとき」）。
- 権限不足: `code: "insufficient_scope"`。キーのスコープを確認。
- 認証まわり: `code: "unauthenticated"`（401、キーが無効/失効）、
  `code: "forbidden"`（403、他人のプロジェクトなど対象への権限が無い）、
  `code: "not_found"`（404、projectId / renderId / exportId が無い）。
  いずれもクレジットは消費されない。
- 台本を作るとき（`create_yukkuri_video`）: `code: "cast_mismatch"`（400、一座をまたぐ話者。
  声は出るが画面に映らない動画になるため、**課金せずに**中断する。`undrawableSpeakers` と
  `casts` を見て、どちらかの一座に揃える）。
- 台本を直すとき（`update_lines`）: `code: "line_not_found"`（404、その行番号が無い
  ——`get_project` で今の行番号を取り直す）、`code: "duplicate_index"`（400、同じ
  行番号を2回指定した——1回にまとめる）、`code: "unknown_speaker"`（400、知らない
  話者id——応答の `knownSpeakers` から選ぶ）。
- 購入導線（`purchaseUrl` を開いた先）: `code: "price_unavailable"`（503）。その商品が
  現在購入できない設定になっている。利用者に伝え、別のプランを案内するか時間を置く。
- MP4 を出すとき（`render_mp4`）: `code: "missing_audio"`（409）。音声が無い行が
  あるため**課金せずに**止めた。`missingAudioLines` の行を `generate_audio` で
  作ってから、もう一度 `render_mp4` を呼ぶ。
- 同時に走らせすぎ: `code: "too_many_concurrent_renders"`（429）。**課金されない。**
  同時に実行できるレンダーはプランごとに上限がある（無料1 / スタンダード3 / プロ10）。
  `running` と `limit` と `retryAfter` が返るので、進行中のものが終わってから投げ直す。
  大量に作るときは、投げっぱなしにせず `get_render` で終わりを見てから次を出すこと。
- コールバックを頼んだのに受け付けられないとき: `code: "callback_unavailable"`（503）。
  **課金されない。** `callbackUrl` を外して投げ直し、`get_render` のポーリングで
  完了を待つこと（通知が使えないだけで、レンダー自体は普通に走る）。
- サーバ側の失敗（500）: `code` で分岐すること。

## 支払いが要るとき（402 を受けたら）

**402 で会話を終わらせないこと。** ユーザーはブラウザを見ていないので、
あなたが決済リンクを渡さないと先に進めない。

402 の応答には**買うべきものが1つだけ**入っている。選択肢を並べる必要はない。

```json
{ "code": "insufficient_credits", "shortfall": 3,
  "purchase": { "priceKey": "credit_pack_50", "label": "クレジットパック 50",
                "amountJpy": 2500, "credits": 50,
                "reason": "クレジットが 3 足りません。このパックで足ります（購入分は期限なし）。" } }
```

手順:

1. `create_checkout` に `purchase.priceKey` をそのまま渡す（`POST /api/v1/billing/checkout-link`）
2. 返った `url` をユーザーに渡す。`purchase.label` と `amountJpy` を添えて、
   何をいくらで買うのかを先に伝えること
3. `get_checkout` に `sessionId` を渡して確認する
   （`GET /api/v1/billing/checkout-link/{sessionId}`）
4. `status: "paid"` になったら、**元の操作を同じ `Idempotency-Key` で再試行する**
   （402 は鍵を解放するので、同じ鍵で通る）
5. 15分ほど `pending` のままなら打ち切り、「支払いが確認できませんでした」と伝えて止まる。
   **無限に待たないこと**

`plan_required`（無料プランで MP4）のときは、パックではなく**プラン**が返る。
無料プランは残高があっても MP4 を出せないので、パックを買わせても解決しない。

`get_checkout` は支払いの有無を直接返す。**残高の増減から推測しないこと**——
残高は月次リセットや他の操作でも動く。
  - `export_failed` / `render_start_failed` … そのまま再試行してよい。
    予約したクレジットは返される。ただし返金はサーバ側の後処理なので、
    **応答を受け取った時点の残高には反映されていないことがある**
    （その場での返金が落ちた場合は、後追いの掃除処理が拾う。返金と、
    その記録の**両方**の書き込みが落ちた場合はサーバログにのみ残るので、
    残高が戻らないときは問い合わせること）。
    残高を当てにする処理を続ける場合は、少し置いてから
    `GET /api/v1/credits` で確認すること。
  - `render_incomplete` … レンダーは**開始済み**で、完了を確認できなかっただけ。
    同梱の `renderId` で進捗を確認すること。結果が出る前に再投入すると
    二重に課金される（既に返金されている場合もあるので、まず進捗を見る）。
  - `progress_unavailable` … 進捗が読めなかっただけ。レンダーは継続している可能性があるので
    打ち切らず再試行する。
  - `internal_error` … 一覧取得など、課金を伴わない読み取りが失敗した。
    クレジットは動かない。そのまま再試行してよい。
- キャラid の打ち間違い: `code: "validation_error"` と **`validSpeakers`（そのリクエストで使えるid一覧）** が返る。
  未知のidは課金前に400になる（黙って別キャラや立ち絵なしで出力してクレジットだけ
  消費する、ということはしない）。照合される集合はリクエストによって違う:
  - `POST /agent/generate`: `speaker` はシステムキャラ（`GET /api/v1/characters`）。
    `tachieDirs` / `tachieFaces` / `tachieParts` のキーは**その台本に出てくる話者**。
  - `POST /projects/{id}/export/ymmp`: **そのプロジェクトの行が使っている character_id**。
  - `PUT /ymm4-assets`: システムキャラ ＋ **自分で作ったカスタムキャラ**。
  返ってきた `validSpeakers` をそのまま使えばよい（システムキャラ一覧だけを見て
  判断すると、カスタムキャラを誤って除外する）。
- レート上限: `code: "rate_limited"` と **`retryAfter`（秒）**、同値の `Retry-After` ヘッダ。その秒数だけ待って再試行する。
- **200 が返っても完了とは限らない**: `waitForCompletion: true` で呼んでも、
  レンダーが始まったあとに完了確認そのものが失敗した場合は
  `status: "rendering"` と `renderId` を返す（`"completed"` とは限らない）。
  この場合の課金は正当で、レンダーは走っている——進捗で完了を追うこと。
  **`status` を見ずに `outputUrl` を読むと undefined になる。**
- レンダーの成否判定: 進捗（`get_render` / `GET .../render/{renderId}/progress`）は
  **`done` だけ見ると誤る**。
  - 成功: `done === true` **かつ** `outputFile` が非空
  - 失敗: `fatalErrorEncountered === true`、または `done === true` なのに
    `outputFile` が空（チャンク欠落、および後追いで失敗が確定した分）
  **失敗でも `done` が立つことがある**ので、必ず失敗判定を先に行うこと。
  どちらの失敗でもクレジットは自動返金される。`errors` は原因の配列だが
  リトライ予定のものも含むので、判定には `fatalErrorEncountered` を使うこと。
- **打ち切りの目安**: Lambda 側が強制終了されると進捗が固まり、`done` も
  `fatalErrorEncountered` も永久に立たないことがある。`overallProgress` が
  20分以上まったく動かない場合は失敗として扱い、ポーリングを止めてよい。
  失敗が確定した分のクレジットは返金される。ただし返金はサーバ側の後処理で
  行われるため、**打ち切った直後の残高には反映されていない**（進捗が
  失敗を返した場合はその時点、進捗が凍ったまま止まった場合は後追いの
  掃除処理の後）。残高は少し置いてから `GET /api/v1/credits` で確認すること。
  なお、打ち切ったあとにサーバ側が**動画は出来ていた**と判定して完了に
  変わることがある（その場合は課金されたまま）。打ち切りは「もう待たない」
  という意味であって最終結果ではないので、後で `get_render` を一度
  確認するか、`GET /api/v1/projects/renders` で結果を拾い直すとよい。

## 再投入しても二重課金しない方法

課金される操作（`create_yukkuri_video` / `export_ymmp` / `render_mp4`）には
`Idempotency-Key` ヘッダ（MCP では `idempotencyKey` 引数）を付けられる。
レンダーは5クレジットと一番高いので、特に付けること。同じ鍵で再投入すると、最初に成功した
ときの応答がそのまま返り、課金は起きない。有効期間は24時間。

応答を落とした（タイムアウト、接続断、プロセス再起動）ときは、**同じ鍵で
そのまま投げ直せばよい**。鍵を変えると別の操作として扱われ、もう一度課金される。

鍵を控えていない場合は `list_exports` / `GET /api/v1/exports`（`?projectId=` で
絞れる）で一覧を取り、`exportId` を拾い直してから `get_export` で
ダウンロードURLを得る。
返金済みのものは `status: "failed"` で返る。

同じ鍵の処理がまだ走っている間に投げ直すと `409` と `code: "in_flight"` が返る。
少し待って同じ鍵で再試行すること（別の鍵に変えると二重に課金される）。

**どの場合も、鍵は新しくしないこと。** 新しい鍵にすると、実は最初の呼び出しが
通っていた場合に二重課金になる。応答ごとの扱いは次のとおり:

| 応答 | 鍵の状態 | 次にすること |
|---|---|---|
| `402` insufficient_credits / plan_required | 解放される | 購入・アップグレード後、**同じ鍵**でそのまま投げ直す |
| `409` in_flight | 別のリクエストが保持中 | 少し待って**同じ鍵**で再試行する |
| `503` unavailable（判定できなかった） | 取られていない | **同じ鍵**でそのまま再試行してよい。課金は起きていない |
| `503` unavailable（クレジット確保に失敗） | 保持されたまま | **課金されたか確定していない**。`GET /api/v1/credits` で残高を確認し、しばらく置いてから同じ鍵で再試行する（鍵は65分で自然に解ける） |
| `500` `render_start_failed` / `export_failed` | 解放される | そのまま同じ鍵で再試行してよい。課金分は返金済みか、返金待ちとして記録済み（両方の書き込みが落ちた場合のみサーバログにのみ残る） |
| `500` `render_incomplete` | 解放される | **すぐに再投入しないこと。** レンダーは開始済みで、走っている／既に出来ている場合がある。同梱の `renderId` で進捗を先に確認する（そのまま投げ直すと、同じ動画をもう一度 5 クレジットで作る） |

クレジット確保の失敗だけ鍵を保持するのは、Firestore のトランザクションが
コミット後に失敗しうるためで、そこで解放すると再試行が二度目の課金になる。

失敗した応答は記録しないので、500 が返った場合の再試行は通常どおり実行される
（一時的な障害が鍵に焼き付いて恒久化することはない）。**逆に言えば、鍵は
「もう一度実行してよいか」を判定しない**——`render_incomplete` のように
「実行は済んでいる」種類の 500 では、鍵ではなく `code` を見て判断すること。

## 大量に作る

1本ずつ20往復する必要はない。**`create_yukkuri_videos_batch`（`POST
/api/v1/agent/generate/batch`）に最大20件まとめて渡せる。**

```json
{
  "idempotencyKeyPrefix": "run-2026-09-06-a",
  "items": [
    { "title": "1本目", "script": [{ "speaker": "reimu", "text": "ここが1本目です" }] },
    { "title": "2本目", "script": [{ "speaker": "marisa", "text": "ここが2本目だぜ" }] }
  ]
}
```

各要素は `create_yukkuri_video` と**まったく同じ形**。中では1件ずつ順に
処理され、認証・レート制限・クレジットはそれぞれに効く（まとめても
安くならない）。応答は 200 固定で、`results` に1件ずつの結果が index 順に並ぶ:

- 途中の1件が失敗しても、**成功した分の `projectId` は必ず返る**。捨てないこと。
- 残高切れ（`insufficient_credits` / `plan_required`）とレート制限
  （`rate_limited`）を受けた時点で**そこで打ち切る**。`stoppedAtIndex` /
  `stopReason` / `notAttempted` が付く。
- **`notAttempted` は「失敗した」ではなく「まだ作っていない」。** 失敗と
  混同して作り直すと、成功した分をもう一度作って二重に払うことになる。
  残りだけを投げ直すこと。

`idempotencyKeyPrefix` を付けると、各件へ `<prefix>:<index>` が冪等キーとして
渡る。**同じ prefix で投げ直せば、既に作られた分は二重課金されない。**
打ち切られたあとの再投入は、同じ prefix のまま全件投げ直してよい。

レート制限は1分あたりの呼び出し回数（無料5 / スタンダード10 / プロ20）で、
バッチの中の1件も1回と数える。無料プランで20件投げると6件目で打ち切られる
——これは仕様どおりで、`notAttempted` を見て1分後に残りを投げ直せばよい。

## 完了を待たずに済ませる（コールバック）

MP4 レンダーは数分かかる。`render_mp4` に `callbackUrl` を付けると、
**完了/失敗したときにこちらから POST する**ので、進捗を見るためだけに
起きている必要がなくなる。

`render_mp4` の引数に `"callbackUrl": "https://example.com/hooks/yukkurigen"`
を足すだけでよい。

- `callbackUrl` は **https のみ・内部アドレス不可**。使えない URL は
  **課金せずに 400** で断る（黙って無視はしない）。
- 開始の応答に `callbackSecret` が1度だけ入る（利用者ごとに固定）。
  これを保存して署名を検証すること。

届く本文:

```json
{
  "event": "render.completed",
  "projectId": "PROJECT_ID",
  "renderId": "RENDER_ID",
  "status": "completed",
  "downloadUrl": "https://...(署名付き・6時間有効)",
  "occurredAt": "2026-09-06T00:00:00.000Z"
}
```

検証の手順（`X-Yukkurigen-Signature: t=<unix秒>,v1=<hex>`）:

1. `t` が現在時刻から**5分以上離れていたら捨てる**。これをやらないと、
   一度盗まれた本文と署名の組を永久に使い回される。
2. `HMAC-SHA256(callbackSecret, "<t>.<受け取った本文そのまま>")` を計算し、
   `v1` と**定数時間で**比較する。本文は再シリアライズせず、生のまま使うこと。

注意:

- 通知は**1回しか送らない**（届いたことは確認しない）。受け取り損ねたときは
  `list_renders`（`GET /api/v1/projects/renders`）で拾い直せる。
- 転送（3xx）には従わない。宛先はリダイレクトさせず直接受けること。
- 通知が送れない状態のときは `code: "callback_unavailable"`（503、課金なし）。
  `callbackUrl` を外して投げ直し、`get_render` で待つこと。
- **通知の遅さについて（未確認の点あり）**: 完了は Lambda ではなく、こちら側が
  「終わったこと」を確定させた時点で送る。誰かが進捗を見ていれば即座だが、
  誰も見ていない場合は掃除の巡回で拾う。**巡回は10分おき**なので、
  最悪でも10分ちょっとで届く。急ぐ場合は `get_render` を1回叩けば
  その場で確定する（叩いた側の分はその時点で片付く）。

## 調整できること

台本以外の指定はすべて任意で、省略すれば既定値になる。openapi.json に
全項目があるが、よく使うものは:

- `fps`: 30（既定）か 60。**`create_yukkuri_video` と `export_ymmp` だけ**
  で指定できる（.ymmp 側の設定なので、`render_mp4` には無い——送っても
  黙って無視され、エラーにもならない）。動きの多い動画は 60 が滑らか。
- `platform` / `outputFormat`: 縦横と用途（youtube / tiktok / shorts）。
- `fontSize` / `titleFontSize`: 字幕とタイトルの文字サイズ。
- `voicePlaybackRate`: 読み上げ速度（0.5〜2.0）。
- `bgmVolume`: BGM 音量（0〜1）。
- `materialImageFilter`: 素材画像のフィルタ（`blur` / `top-dark`）。
- `showOutroCard`: 終了カードの有無。

MCP のツール定義と openapi は同じ集合を公開している。片方にしか無い
項目があれば、それは不具合として扱ってよい。

## 注意

- 生成される .ymmp は音声を「YMM4で開いたとき再生成」する前提（軽量）。音声そのものは含まれない。
- キャラの立ち絵・BGM等の素材は、ユーザーのYMM4環境/ライセンスに依存する場合がある。
- **.ymmp は実機の YMM4 では未検証**。生成物は自前のスキーマと、公開されている
  .ymmp から読み取った型名に照らして検査しているが、「実際に YMM4 で開いて
  再生できた」ことは確認していない。特に次は推定を含む:
  - VOICEVOX の `Pronounce` は `null` を出し、YMM4 側に読みを再生成させている
    （その `$type` が公開ファイルに現れないため）。
  - **AquesTalk（ゆっくり霊夢・魔理沙など）の `VoiceParameter` の型名**は、
    実際に YMM4 プロジェクトを生成している OSS が出している値と食い違っている。
    こちらは派生型（`AquesTalk1VoiceParameter`）、向こうは素の
    `VoiceParameter`。**AquesTalk のキャラを使う場合は特に、まず少数の行で
    開けるか確かめること。** VOICEVOX のキャラ（ずんだもん等）にはこの
    食い違いは無い。
  - 立ち絵は `tachieDirs` を指定したときだけ入る。パーツ画像の命名規約は
    こちらの想定に基づく。
  そのため、初回は少数の行で試し、YMM4 で開けることを確かめてから本番の
  台本を流すこと。開けない場合は生成側の不具合として報告してほしい。
- MP4 の音量は行ごとに測って揃えている（実測 -14.4 LUFS、真のピーク
  -2.0 dBTP）。YouTube の基準（-14 前後）とほぼ同じなので、そのまま
  投稿して音量で不利になることはない。
- 機械可読な定義: `https://app.yukkurigen.com/openapi.json`
