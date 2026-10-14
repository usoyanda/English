---
name: study-english-qb
description: 英会話レッスンメモ（チャットログ・文字起こし・ホワイトボード写真）を受け取り、復習ノートと index.html 用のQB（問題バンク）を作成してリポジトリに直接追加する。「レッスンメモ」「QB作って」「復習ノート」「アプリに追加して」のような依頼があれば使う。
---

# study-english-qb

英会話スクール（B1〜B2レベル）の復習用クイズアプリ `index.html`（単一HTML、GitHub Pages運用）に、レッスンメモから復習ノートとQB（問題バンク）を作って追加するスキル。

以前はclaude.ai Projectでチャット経由（HTMLをやり取りして人間が手動で上書き）で行っていた作業を、このリポジトリ内で直接完結させる。index.html を直接編集し、検証し、コミットする。

## 全体フロー

1. レッスンメモを受け取る
2. `index.html` を読み、現在地を把握する（下記）
3. 復習ノートを作成し、チャットで提示する
4. QBブロックを作成する
5. `index.html` に直接追記する（PATS_DEF追加分 → QB配列）
6. 構文・整合性を機械チェックする
7. `git commit`（必要なら `git push`）する
8. 何が変わったかを簡潔に報告する

ユーザーの作業は「メモを貼る」だけにする。ファイルのやり取りやコピペは発生させない。

## 1. 現在地の把握（毎回まずやる）

作業前に `index.html` を読んで、以下を確認する。

```bash
grep -o "lesson:'[^']*'" index.html | tail -5      # 直近の回番号
grep -o "key:'[a-z_]*'" index.html | sort -u        # 既存のpatキー一覧
grep -c "{id:'" index.html                          # 現在の総問題数
```

QB最終ブロックの回番号（丸数字）が、次に振る番号の基準になる。講師メモ側に書かれた番号（④YUTAレッスン、など）は使わない。アプリの通し番号だけを正とする。

## 2. 入力の扱い

| 種類 | 扱い |
|------|------|
| 講師チャットログ | 主な入力。そのまま貼り付けで受け取る |
| 音声の文字起こし | テキストとして受け取る |
| 写真（ホワイトボード等） | 画像として受け取り、読み取った内容を使う。判読不能な箇所は推測せず確認する |

ログに `(X)` `(O)` `(自然)` の記法がある場合は、誤文と訂正文の実データとして最優先で使う。

レッスン日付・講師名・レッスン名が示されない場合は、ログから読み取れる範囲で埋め、不明な点のみ確認する。

## 3. 出力① 復習ノート

チャット本文に表示する。構成は以下で固定。

1. 見出し：`## ㉖ Lesson / 10月X日` ＋ テーマ1行
2. 項目別の解説（語彙・パターン・文法ポイント）。表と📌補足を使用
3. 語彙・表現整理（表）
4. このレッスンのポイントまとめ（3〜5項目）

作成ルール：

- 補足の例文・関連語は追加してよい。ただし意味・用法の正確さを優先する。
- 講師メモのタイポ（大文字小文字、アポストロフィ、ハイフン抜け等）は正しい形に直す。
- **他のレッスンへの言及は禁止**。今回貼られた内容以外を参照しない。
- 講師メモの訳や説明に誤りがあれば、訂正して記載する。

## 4. 出力② QBブロック

### 4.1 採番

| 項目 | ルール | 例 |
|------|--------|-----|
| 回番号 | QB最終回 ＋1（丸数字） | ㉕の次は㉖ |
| `lesson` | 回番号＋日付。日付不明時は「月」 | `'㉖10/15'` / `'㉖10月'` |
| `month` | YYMM形式 | `'2610'` |
| `id` | prefix ＋ 回番号2桁 ＋ 連番2桁 | `v2601` / `fr2602` |
| ブロックコメント | 月・回番号・日付・テーマ | `// ===== 2610（10月）㉖10/15 テーマ =====` |

prefix は `v`=vocab、`f`=fill、`g`=grammar、`t`=translate、`fr`=free。

### 4.2 問題数の配分

| type | 標準 | 内容が薄い回 |
|------|------|-------------|
| vocab | 5 | 3〜5 |
| fill | 5 | 3〜5 |
| grammar | 3〜5 | 2〜3 |
| translate | 4〜5 | 3〜4 |
| free | 2 | 1〜2 |

レッスン内容に根拠のない問題で数を水増ししない。

### 4.3 必須フィールド

| type | 必須 | 任意 |
|------|------|------|
| vocab | `id` `pat` `month` `type` `q` `a` `choices[4]` `explain` `lesson` | — |
| fill | `id` `pat` `month` `type` `q` `blank` `choices[4]` `hint` `lesson` | — |
| grammar | `id` `pat` `month` `type` `q` `optA` `optB` `a` `explain` `lesson` | `deep` |
| translate | `id` `pat` `month` `type` `q` `a` `hint` `lesson` | — |
| free | `id` `pat` `month` `type` `q` `a` `hint` `lesson` | — |

### 4.4 pat の割り当て

- 既存の `PATS_DEF` に該当するものがあれば、そのキーを使う（1章で取得した一覧から選ぶ）。
- 純粋な単語問題は `__vocab__` とする。
- 該当がない場合のみ新規パターンを作る。1回あたり最大3つを目安とし、既存で代用できる場合は作らない。
- 新規パターンは `{cat:'<8カテゴリのいずれか>', key:'<new_key>', t:'<表示名>', ex:'<説明>'}` の形で、PATS_DEF配列の該当カテゴリブロック末尾に追加する。8カテゴリは `freq` `time` `want` `feel` `ask` `req` `comp` `desc`。

### 4.5 各タイプの作成基準

**vocab**
- `explain` は「意味 → バッククォートで例文 → 関連語や注意点」の順で書く。
- ダミー選択肢は同じ品詞・同じ分野で、紛らわしいものにする。

**fill**
- 空欄は1つのみ。
- `choices` に正解（`blank`）を必ず含める。形の違い（原形・-ing・過去分詞等）をダミーにすると効果が高い。

**grammar**
- 誤文は、ログに `(X)` があればそれを使う。ない場合は学習者が実際にしやすい誤りを作る。
- 正解位置（A/B）は偏らせない。
- `explain` には「なぜ正しいか」と「なぜ逆が誤りか」の両方を書く。
- `deep` は、紛らわしい用法の整理や時制比較など、深掘りの価値がある問題にのみ付ける（1回あたり0〜2問）。

**translate**
- 複数の正解がありうる場合は ` / ` 区切りで併記する（自動採点で全候補と比較される）。
- 採点は3文字以下の単語を完全一致で判定するため、短縮形と非短縮形（It's / It is）は両方併記する。

**free**
- 模範解答の末尾に「など」を付け、自己採点の幅を持たせる。

### 4.6 記述上の注意

- 文字列にアポストロフィを含む場合は、ダブルクォートで囲む（`q:"\"I'm sorry...\""` のように）。シングルクォートの文字列中でアポストロフィをエスケープすると `\n` のような解釈違いが起きやすいので避ける。
- 講師メモのタイポは直して採用する。タイポ自体が学習ポイントになる場合（例：polish / Polish）は問題化してよい。

## 5. index.html への追記

1. `PATS_DEF` 追加分がある場合、`const PATS_DEF = [` の直後に挿入する。
2. QBブロックは、QB配列の閉じ `];` の直前（最後の問題オブジェクトの後ろ）に、コメント行＋問題オブジェクト群を挿入する。
3. Editツールで直接 `index.html` を書き換える。別ファイルに書き出して後で人間が貼り直す運用はしない。

## 6. 検証（追記後、コミット前に必ず実行）

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const scripts=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
const stub='var document={getElementById:function(){return{style:{},addEventListener:function(){},classList:{add:function(){},remove:function(){}}}},querySelectorAll:function(){return[]},querySelector:function(){return null},addEventListener:function(){},createElement:function(){return{style:{},appendChild:function(){},setAttribute:function(){}}},body:{appendChild:function(){}}};var window={innerWidth:800,addEventListener:function(){},localStorage:{getItem:function(){return null},setItem:function(){}}};var localStorage=window.localStorage;var navigator={};';
scripts.forEach((s,i)=>{ try{ new Function(stub+s); console.log('script',i,'OK'); } catch(e){ console.log('script',i,'ERROR:',e.message); } });
const m=html.match(/const QB = \[([\s\S]*?)\n\];/);
const QB=eval('['+m[1]+']');
const ids=QB.map(q=>q.id);
console.log('QB総数:',QB.length,'ID重複:',ids.length-new Set(ids).size);
const m2=html.match(/const PATS_DEF = \[([\s\S]*?)\n\];/);
const PATS=eval('['+m2[1]+']');
const patKeys=new Set(PATS.map(p=>p.key).concat(['__vocab__']));
const bad=QB.filter(q=>!patKeys.has(q.pat));
console.log('pat不整合:',bad.length, bad.map(q=>q.id));
QB.forEach(q=>{
  if(q.choices){const a=q.type==='vocab'?q.a:q.blank; if(!q.choices.includes(a))console.log('答え不一致:',q.id); if(new Set(q.choices).size!==4)console.log('選択肢重複:',q.id);}
  if(q.type==='grammar'&&!['A','B'].includes(q.a))console.log('grammar解答不正:',q.id);
});
"
```

エラーが出た場合は、出力前チェックに戻って修正する。意味・訳・文法説明に誤りがないか、他レッスンへの言及がないかも目視で確認する。

## 7. コミット・反映

検証が通ったら、このリポジトリで直接コミットする。

```bash
git add index.html
git commit -m "<回番号><日付>分のQB追加（<問題数>問）

- テーマ: <ひとこと>
- 新規pat: <あれば列挙、なければ省略>"
git push
```

GitHub Pagesは `index.html` を直接配信しているため、pushすれば即座に反映される（ビルド工程なし）。

## 8. 完了報告

チャットでは以下を簡潔に報告する。

- 作成した回番号・問題数
- 新規追加したpatキー（あれば）
- コミットハッシュ／pushの成否
- 気になった点（講師メモの誤り訂正、他レッスンへの不要な言及の削除など）があれば1〜2行で

ファイルの受け渡しは発生しない。index.html は既にリポジトリにコミット済みの状態で完了する。
