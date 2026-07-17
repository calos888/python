---
description: 外付けSSD上のThunderbirdメール(t-it.co.jp)を分析し、秘書のように業務状況と次アクションを整理する
---

外付けSSD上のThunderbird(t-it.co.jp業務アカウント)のメールを分析し、秘書として業務整理・次アクション提案を行う。

対象日数: `$ARGUMENTS` が数値ならその日数、指定が無ければ直近3日分を対象にする。

## 手順

1. **プロファイルの場所を特定する**（Macによってボリューム名や接続状態が違うため、毎回動的に調べる。決め打ちしない）
   ```bash
   cat ~/Library/Thunderbird/profiles.ini
   ```
   `Path=` の行がプロファイルの絶対パス。存在しない/読めない場合は「外付けSSDがマウントされていません」とユーザーに伝えて終了する。

2. **Gloda検索用DBをスクラッチパッドにコピーする**（Thunderbirdが起動中だとDB本体がロックされているため、コピーしてから読む。-wal/-shmは存在しない場合もあるのでエラーは無視してよい）
   ```bash
   PROFILE="<手順1で取得したパス>"
   DB="$PROFILE/global-messages-db.sqlite"
   cp "$DB" <scratchpad>/gloda.sqlite
   cp "$DB-wal" <scratchpad>/gloda.sqlite-wal 2>/dev/null
   cp "$DB-shm" <scratchpad>/gloda.sqlite-shm 2>/dev/null
   ```

3. **対象フォルダを動的に解決する**（IDをハードコードしない。年が変わってArchivesフォルダが増えても自動で拾えるように、名前パターンで都度検索する）
   ```sql
   select id from folderLocations
   where folderURI like '%pop.t-it.co.jp%'
     and name not in ('Junk','Trash');
   ```
   ※ 過去に `imap.t-it.co.jp` のIMAP INBOXは容量が非常に大きく(数十GB)、Gloda検索インデックス対象外(indexingPriority=-1)だったことがある。対象フォルダが0件で見つからない場合はその可能性を伝える。

4. **直近N日分のメールを本文スニペット付きで抽出する**
   ```sql
   select m.id, datetime(m.date/1000000,'unixepoch') as dt, fl.name,
          c.c3author, c.c4recipients, c.c1subject, substr(c.c0body,1,1200)
   from messages m
   join folderLocations fl on fl.id = m.folderID
   join messagesText_content c on c.docid = m.id
   where fl.id in (<手順3のID一覧>)
     and m.date > <(今日 - N日) をマイクロ秒epochに変換した値>
     and m.deleted = 0
   order by m.date desc;
   ```
   `date -v-Nd +%s` (macOS) で日数からepoch秒を計算し、1,000,000倍してマイクロ秒に変換する。

5. **内容を読んで秘書としてまとめる**。以下の3分類で日本語のMarkdownとして出力する:
   - 🔴 至急対応が必要（相手を待たせている/期限が近い/クレーム・不良品対応など）
   - 🟡 進行中案件のステータス整理（案件・取引先ごとに現状と次アクションを表形式で）
   - ⚪ 情報共有のみ（対応不要なメルマガ・展示会案内など）

   各項目は「誰が・何を待っているか」「次に田沼さんが取るべき具体的なアクション」を明記する。件名や差出人の生の羅列で終わらせず、必ず「何をすればよいか」まで踏み込む。

6. コピーした一時DBファイルはスクラッチパッドに置いたままで良い(セッション終了時に自動的に片付く)。分析結果だけをチャットに出力する。
