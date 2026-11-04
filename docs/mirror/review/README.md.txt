# 朝検収の運用（review/）

夜間に生成・分析された成果物を、朝に NO3 が検収するための置き場と手順の正本。

## パス構成

```
review/
  README.md                    ← 本ファイル（運用の正本）
  pending/
    YYYY-MM-DD.md              ← その日の未検収リスト。MUSE が朝に読むのは当日分だけ
  approved/
    YYYY-MM-DD.md              ← OK 済み（投稿待ち・投稿済みの記録）
  rejected/
    YYYY-MM-DD.md              ← 却下
  items/
    YYYY-MM-DD/
      {task_id}.md             ← 成果物の本文・詳細（プレビューの本体）
```

## 生成側の追記ルール

1. 成果物を `review/items/YYYY-MM-DD/{task_id}.md` に書く
2. 同じ日の `review/pending/YYYY-MM-DD.md` に、下の固定ヘッダの1行を追記する
3. ヘッダが無ければ、先にヘッダを書く
4. 同じ task_id を二重に書かない

## pending の固定ヘッダと欄の意味

```markdown
| task_id | media | stage | conf | needs_review | summary | preview_path | status |
```

| 欄 | 内容 |
|----|------|
| task_id | 作業ID。一意（例：task-YYYYMMDD-NNN） |
| media | threads / instagram / note など |
| stage | ① / ② / ③ / 不明 |
| conf | 0.0〜1.0 |
| needs_review | true のとき検収で優先する（要確認） |
| summary | 40字以内。原文の言い換えにしない、判断材料としての要約 |
| preview_path | 本文・詳細のパス（review/items/YYYY-MM-DD/{task_id}.md） |
| status | 追記時は pending_review。検収・投稿に応じて更新する |

## items 詳細ファイルの最小テンプレ

```markdown
# task-YYYYMMDD-NNN

- media:
- stage_estimate:
- confidence:
- needs_human_review:
- source: （原文パスまたは引用）
- created:

## body
（投稿案・分析結果の本文）

## notes
（任意。生成時の補足）
```

同じテンプレが `review/items/YYYY-MM-DD/_template.md` にも置いてある。

## MUSE 朝ルーチン手順

目的：その日の未検収を読み、NO3 がワンタップで返せる短いサマリーを出す。投稿は検収後のみ。

1. **リストを開く**：`review/pending/YYYY-MM-DD.md`（当日分）のみを読む。無い、または行がヘッダだけなら「未検収0件」と通知して終了。未処理の前日分が残っていれば併記してよい
2. **行を分類する**：needs_review = true を「要確認」、それ以外を「通常」。status が pending_review 以外の行は無視する
3. **Claude を呼ぶか判定する**：下の「Claude を呼ぶ条件」に当てはまるときだけ呼ぶ。当てはまらなければサマリーへ直行。条件に当たる場合は、`goals/meta-routing/CLAUDE_STATE.md` に要照合フラグを立て、`goals/meta-routing/CALL.md` に記録する
4. **NO3 向けサマリーを出す**：下の形式。本文全文は載せない。詳細が必要なときだけ preview_path を開く旨を1行で書いてよい。サマリーには「Claude 呼び出し：あり／なし」を1行入れる
5. **NO3 の返信を処理する**：OK → approved へ、却下 → rejected へ、保留 → pending_review のまま。解釈できない返信は聞き返し、状態は変えない
6. **投稿（許可されている場合のみ）**：approved かつ未投稿のものを投稿する。実行後 published に更新。失敗時は failed と理由を机に書き、NO3 に通知する
7. **締めの通知**：検収結果の件数（OK／保留／却下）、投稿の成功・失敗件数、残 pending 件数

### NO3 向けサマリーの形式

```
【検収】YYYY-MM-DD 未検収 N件（要確認 M件）
Claude 呼び出し：あり／なし

■ 要確認
1. task-... | media | stage | 要約
2. …

■ 通常
3. task-... | 要約
…

返信: 1OK 2保留 3却下
まとめて: 全部OK / 要確認以外OK
```

## Claude を呼ぶ条件と、渡す最小セット

呼ぶ条件（いずれか）：
- 要確認（needs_review: true）が一定件数以上（目安：3件以上）
- 同じテーマで却下が続いた
- NO3 が「照合して」と明示した
- ルール変更直後の初回バッチ

渡すもの（最小）：
- 当日 pending のうち要確認の行だけ
- 該当するルールの断片だけ（GOAL 全文は渡さない）
- 聞きたいこと1つ（例：優先順位／ルールとのずれの有無）

Claude は常時監視しない。全量読みもしない。呼ばれるのは必要なときだけ。

## 検収後の扱い

| 返信・結果 | status | リストの扱い |
|-----------|--------|--------------|
| OK | approved | pending から approved 側へ（行の移動または status 更新） |
| 保留 | pending_review | pending に残す |
| 却下 | rejected | pending から rejected 側へ |
| 投稿成功 | published | approved 側に記録 |
| 投稿失敗 | failed | 理由を机に残し、NO3 に通知 |

## 投稿区分（確定事項 D-007）

- アイダ：連携済み。NO3 の承認後に、MUSE が公開できる
- フラ：Muse 未連携のため、MUSE の投稿実行の対象外
- 無検収のまま投稿しない

## 禁止事項

- 朝ルーチンで `review/pending`（当日分）以外の大量のファイルを読まない
- GOAL・履歴の全量を Claude に渡さない
- キーなどの秘密情報をサマリーに出さない
- status が pending_review でないものを勝手に投稿しない
