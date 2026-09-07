# YukkuriGen — ゆっくり動画を作る (Agent Skill)

テーマから「ゆっくり解説/劇場」動画の **MP4** を生成するためのスキル。
あなた（AI）が台本を書き、YukkuriGen が音声を合成してレンダーし、動画のURLを返します。
ユーザーはそれをそのまま投稿できます。

## 接続

- 発行: ユーザーに `https://app.yukkurigen.com/settings/api-keys` でAPIキーを発行してもらう。
  **無料プランでもここまで使える**（無料枠10クレジット）:
  - 台本の作成・読み返し・修正（`create_yukkuri_video` のプロジェクト作成部分、
    `get_project`、`update_lines`、`generate_audio`）——**すべて消費なし**
  - **MP4 プレビュー**（`output:"preview"` または `render_mp4` + `preview`。
    1クレジット・640×360・最大20秒＝10回ぶん）
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
| 台本→MP4 を一括生成 | `create_yukkuri_video` | `POST /api/v1/agent/generate` |
| 複数本をまとめて作る（最大20） | `create_yukkuri_videos_batch` | `POST /api/v1/agent/generate/batch` |
| プロジェクト一覧 | `list_projects` | `GET /api/v1/projects` |
| 台本を読み返す（行番号つき） | `get_project` | `GET /api/v1/projects/{id}` |
| 指定した行だけ直す | `update_lines` | `PATCH /api/v1/projects/{id}/lines` |
| 音声を作る（MP4の前に必須） | `generate_audio` | `POST /api/v1/projects/{id}/audio/generate` |
| 見た目の一覧 | `list_templates` | `GET /api/v1/agent/templates` |
| チャンネルの一覧 | `list_channels` | `GET /api/v1/channels` |
| 既存プロジェクトを MP4 に | `render_mp4` | `POST /api/v1/projects/{id}/render` |
| レンダー進捗/出力URL | `get_render` | `GET /api/v1/projects/{id}/render/{renderId}/progress` |
| 決済リンクを出す（402のとき） | `create_checkout` | `POST /api/v1/billing/checkout-link` |
| 支払いが済んだか確認 | `get_checkout` | `GET /api/v1/billing/checkout-link/{sessionId}` |

`list_projects` / `get_project` / `update_lines` / `generate_audio` は**消費なし**。
確認用プレビュー（`output:"preview"` / `render_mp4` + `preview`）は**1クレジット**。
課金はレンダーだけで起きる。直すこと自体にお金がかからないので、ユーザーが
納得するまで何度でも直してよい。

## 手順

1. **キャラを選ぶ**: `list_characters` で `id` と音声エンジンを確認。
   **1本の動画に出せるのは同じ一座のキャラだけ**（立ち絵の系統が違うと画面に出せない）:
   - ゆっくり（東方）: `reimu`（霊夢）, `marisa`（魔理沙）
   - 東北勢: `zundamon`（ずんだもん）, `metan`（四国めたん）, `tsumugi`（春日部つむぎ）, `anko`（あんこもん）

   混ぜると `400 cast_mismatch` を返す（課金なし）。応答の `casts` に一座の一覧が入っている。

   **見た目（テンプレート）は人が決める。あなたは選ぶだけ。**
   `list_templates` で一覧を取り、返った `id` を `templateId` に渡す。
   利用者が「いつもの見た目で」と言ったら、`list_templates` の `source: "mine"`
   ——その人がエディタで作ったもの——から選ぶこと。**新しく作ろうとしないこと。**
   省略すれば台本の話者から自動で選ぶので、指定は必須ではない。

   チャンネル（テーマ・想定視聴者・既定の指示）を使うなら `list_channels` で
   `id` を取り、`channelId` に渡す。

   **返るのは MP4。** こちらで音声を合成してから本番レンダーを開始し、
   `renderId` を返す（5クレジット・有料プラン）。**`output:"preview"` なら
   1クレジット・低解像度・20秒**で、無料プランでも動くものが見られる。
   どちらも `get_render` でポーリングする。
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
   立ち絵が画面に出ている行にだけ効く。
5. **生成する**: `create_yukkuri_video`（または `POST /agent/generate`）に `{ title, backgroundImageUrl, script }` を渡す。
   既定で MP4 を焼く。試すだけなら `output: "preview"` を足す（1クレジット・20秒）。
6. **受け取る**: `renderId` が返るので `get_render` でポーリングする。
   **完了の判定は `done === true` かつ `outputFile` が非空**（下の「レンダーの成否判定」）。
   その `outputFile` をユーザーに渡す。`callbackUrl` を渡しておけば、完了時に
   こちらから通知する（そちらの本文は `downloadUrl`）。

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
ユーザーが良いと言った行まで消えるうえ、レンダーにもう一度課金される。

## MP4 は音声を自動生成しない

`render_mp4` は**行に載っている音声を再生するだけ**で、音声を作りはしない。
音声が無い行があるまま呼ぶと、**課金せずに `409 missing_audio`** を返して止まる
（無音の動画に5クレジット払わせないため）。応答の `missingAudioLines` に対象行が
入っているので、`generate_audio` を呼んでから再実行する。

`create_yukkuri_video` は**中で音声まで作る**ので、この手順は要らない。
`get_project` → `update_lines` で直したあと `render_mp4` を呼ぶときだけ、
自分で `generate_audio` を挟む必要がある。

## 最小例（REST）

```bash
curl -X POST https://app.yukkurigen.com/api/v1/agent/generate \
  -H "Authorization: Bearer $YUKKURIGEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "身近な熱力学のはなし",
    "backgroundImageUrl": "https://example.com/bg.jpg",
    "script": [
      { "speaker": "reimu",  "text": "今日は熱力学第二法則を解説するわ" },
      { "speaker": "marisa", "text": "エントロピーってやつだな" },
      { "speaker": "reimu",  "text": "そう。散らかった部屋が勝手に片付かないのと同じ理屈よ", "cameraMode": "dynamic" }
    ]
  }'
```

## 立ち絵とBGM

立ち絵は**テンプレートが自動で置く**。キャラを選べば画面に出る——別途の指定は要らない。

BGM は既定のものが入る。差し替えたい場合はエディタで設定する
（`POST /api/v1/projects/{id}/bgm`）。

## クレジットとエラー

- 消費: MP4 レンダー = 5、プレビュー = 1、AI台本生成 = 1。台本の作成・修正・音声生成は 0。
- 不足時: `code: "insufficient_credits"` / `code: "plan_required"`。**諦めずに購入まで案内すること**（下の「支払いが要るとき」）。
- 権限不足: `code: "insufficient_scope"`。キーのスコープを確認。
- 認証まわり: `code: "unauthenticated"`（401、キーが無効/失効）、
  `code: "forbidden"`（403、他人のプロジェクトなど対象への権限が無い）、
  `code: "not_found"`（404、projectId / renderId が無い）。
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
  - `generate_failed` / `render_start_failed` … そのまま再試行してよい。
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
  - `PATCH /projects/{id}/lines`: **そのプロジェクトの行が使っている character_id**
    ＋ 自分で作ったカスタムキャラ。
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

課金される操作（`create_yukkuri_video` / `render_mp4`）には
`Idempotency-Key` ヘッダ（MCP では `idempotencyKey` 引数）を付けられる。
レンダーは5クレジットと一番高いので、特に付けること。同じ鍵で再投入すると、最初に成功した
ときの応答がそのまま返り、課金は起きない。有効期間は24時間。

応答を落とした（タイムアウト、接続断、プロセス再起動）ときは、**同じ鍵で
そのまま投げ直せばよい**。鍵を変えると別の操作として扱われ、もう一度課金される。

鍵を控えていない場合は `list_projects` でプロジェクトを見つけ、
`GET /api/v1/projects/renders`（`?projectId=` で絞れる）でそのレンダーの
`renderId` を拾い直してから `get_render` で出力URLを得る。
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
| `500` `render_start_failed` / `generate_failed` | 解放される | そのまま同じ鍵で再試行してよい。課金分は返金済みか、返金待ちとして記録済み（両方の書き込みが落ちた場合のみサーバログにのみ残る） |
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

- `platform` / `outputFormat`: 縦横と用途（youtube / tiktok / shorts）。
- `fontSize` / `titleFontSize`: 字幕とタイトルの文字サイズ。
- `voicePlaybackRate`: 読み上げ速度（0.5〜2.0）。
- `bgmVolume`: BGM 音量（0〜1）。
- `materialImageFilter`: 素材画像のフィルタ（`blur` / `top-dark`）。
- `showOutroCard`: 終了カードの有無。

MCP のツール定義と openapi は同じ集合を公開している。片方にしか無い
項目があれば、それは不具合として扱ってよい。

## 注意

- **音声はこちらで合成する。** 相手側に YMM4 や VOICEVOX を入れてもらう必要はない。
- 音声エンジンはキャラごとに決まっている（`list_characters` の `voiceEngine`）。
  ゆっくり霊夢・魔理沙は AquesTalk、ずんだもん等は VOICEVOX。
- **初回は `output: "preview"` で試すこと**（1クレジット・20秒）。本番の
  5クレジットを払う前に、キャラ・背景・カメラが意図どおりか確かめられる。
- **`create_yukkuri_video` から MP4 まで一息に通す経路は本番未検証**（2026-09-07 時点）。
  台本の作成・音声の合成・レンダーの各段は個別に動いているが、1コールで
  最後まで通した実績がまだ無い。**少数の行で試し、`get_render` が
  `completed` を返すことを確かめてから本番の台本を流すこと。**
  途中で止まる場合は `get_project` で台本を、`generate_audio` で音声を、
  `render_mp4` でレンダーを、と段ごとに切り分けられる。不具合として報告してほしい。
- MP4 の音量は行ごとに測って揃えている（実測 -14.4 LUFS、真のピーク
  -2.0 dBTP）。YouTube の基準（-14 前後）とほぼ同じなので、そのまま
  投稿して音量で不利になることはない。
- 機械可読な定義: `https://app.yukkurigen.com/openapi.json`
