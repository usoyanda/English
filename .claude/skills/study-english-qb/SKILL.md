---
name: study-english-qb
description: 英会話レッスンの復習ノート作成と、index.html へのQB（問題バンク）追加を行う。モードA：レッスンメモ（チャットログ・文字起こし・写真）を貼られたら復習ノートを作り notes/ に保存する。モードB：「先月分をQBにして」「月初めの追加」と言われたら、前月ぶんのノートを読んで全レッスンのQBを作り index.html に追記・検証・push する。「レッスンメモ」「復習ノート」「QB作って」「先月分をアプリに追加」などの依頼で使う。
---

# study-english-qb

英会話スクール（B1〜B2レベル）の復習用クイズアプリ `index.html`（単一HTML、GitHub Pages運用、公開リポジトリ）の運用スキル。流れは2段階に分かれる。

```
講師メモ ──(モードA：レッスンごと)──▶ notes/YYMM.md に復習ノートを保存・commit
                                              │
                          (モードB：月初め)    ▼
                     前月のノートを読む ──▶ QBを一括作成 ──▶ index.html に追記・検証・push
```

- **モードA** はレッスンごとに、ユーザーが講師メモを貼って実行する（手動）。
- **モードB** は月初めに「先月分をQBにして」の一言で実行する。**ノートはリポジトリに保存済みなので、貼り直しは不要**。

Claude Codeのセッションは使い捨てなので、作ったノートは必ず commit・push して残す（push しないと次のセッションで消える）。

---

## 共通：現在地の把握（毎回最初にやる）

```bash
grep -o "lesson:'[^']*'" index.html | sort -u | tail -5   # QBに入っている直近の回
ls notes/ 2>/dev/null                                      # 保存済みのノート
grep -h "^<!-- lesson:" notes/*.md 2>/dev/null | tail -5   # ノート側の直近の回
grep -o "key:'[a-z_]*'" index.html | sort -u               # 既存のpatキー
grep -c "{id:'" index.html                                 # 現在の総問題数
```

**回番号の基準**は、`index.html` のQB最終回と `notes/` の最終回のうち、大きい方。次に振る番号はその +1（丸数字）。講師メモ側に書かれた番号（④YUTAレッスン等）は使わない。アプリの通し番号だけが正。

---

## モードA：復習ノートを作って保存する

トリガー：ユーザーがレッスンメモを貼った。

### A-1. 入力の扱い

| 種類 | 扱い |
|------|------|
| 講師チャットログ | 主な入力。そのまま貼り付けで受け取る |
| 音声の文字起こし | テキストとして受け取る |
| 写真（ホワイトボード等） | 画像として受け取り、読み取った内容を使う。判読不能な箇所は推測せず確認する |

ログに `(X)` `(O)` `(自然)` の記法がある場合は、誤文と訂正文の実データとして最優先で使う。日付・講師名・テーマが不明なら、ログから読み取れる範囲で埋め、不明な点だけ確認する。

### A-2. ノートの構成（固定）

```markdown
<!-- lesson: ㉖10/15 | month: 2610 -->
## ㉖ Lesson / 10月15日
**テーマ：…**

（項目別の解説。表と📌補足を使う）

### 🔤 語彙・表現整理
（表）

### 📌 このレッスンのポイントまとめ
（3〜5項目）
```

1行目の `<!-- lesson: … | month: … -->` は機械可読のマーカー。モードBが読むので**必ず付ける**。

### A-3. 作成ルール

- 補足の例文・関連語は追加してよい。ただし意味・用法の正確さを優先する。
- 講師メモのタイポ（大文字小文字、アポストロフィ、ハイフン抜け等）は正しい形に直す。
- 講師メモの訳や説明に誤りがあれば訂正して記載する。
- **他のレッスンへの言及は禁止**。今回貼られた内容以外を参照しない。
- **リポジトリは公開**。他の生徒の名前、個人が特定できる情報、勤務先などはノートに書かない。講師は名前を伏せるか first name のみ。

### A-4. 保存

1. `notes/YYMM.md`（例：`notes/2610.md`）の末尾にノートを追記する。ファイルが無ければ作る。月をまたいだら新しいファイルにする。
2. `git add notes/ && git commit && git push`
3. チャットにはノート本文を表示し、追記した回番号と保存先を1行で伝える。

`index.html` は**モードAでは触らない**。

---

## モードB：前月ぶんのQBを一括作成する

トリガー：ユーザーが「先月分をQBにして」「月初めの追加」など。または月初めの定期実行。

### B-1. 対象の特定

1. 対象月は「今日の日付の前月」（例：10/3 なら `notes/2609.md`）。ユーザーが月を指定したらそれに従う。
2. `notes/YYMM.md` 内の各ノート（`<!-- lesson: ㉖10/15 | … -->`）について、`index.html` に `lesson:'㉖` が**既にあるか**を調べる。あれば処理済みなのでスキップ。無いものが対象。
3. 前月より古いノートで未処理のものがあれば、それも対象に含める。
4. 対象が無ければ「追加するものは無い」と伝えて終了する。ファイルは変更しない。

### B-2. QBブロックの作成

#### 採番

| 項目 | ルール | 例 |
|------|--------|-----|
| `lesson` | ノートのマーカーと同じ値 | `'㉖10/15'` |
| `month` | YYMM形式 | `'2610'` |
| `id` | prefix ＋ 回番号2桁 ＋ 連番2桁 | `v2601` / `fr2602` |
| ブロックコメント | 月・回番号・日付・テーマ | `// ===== 2610（10月）㉖10/15 テーマ =====` |

prefix は `v`=vocab、`f`=fill、`g`=grammar、`t`=translate、`fr`=free。複数レッスンは回番号順に並べる。

#### 問題数の配分（1レッスンあたり）

| type | 標準 | 内容が薄い回 |
|------|------|-------------|
| vocab | 5 | 3〜5 |
| fill | 5 | 3〜5 |
| grammar | 3〜5 | 2〜3 |
| translate | 4〜5 | 3〜4 |
| free | 2 | 1〜2 |

ノートに根拠のない問題で数を水増ししない。

#### 必須フィールド

| type | 必須 | 任意 |
|------|------|------|
| vocab | `id` `pat` `month` `type` `q` `a` `choices[4]` `explain` `lesson` | — |
| fill | `id` `pat` `month` `type` `q` `blank` `choices[4]` `hint` `lesson` | — |
| grammar | `id` `pat` `month` `type` `q` `optA` `optB` `a` `explain` `lesson` | `deep` |
| translate | `id` `pat` `month` `type` `q` `a` `hint` `lesson` | — |
| free | `id` `pat` `month` `type` `q` `a` `hint` `lesson` | — |

#### pat の割り当て

- 既存の `PATS_DEF` に該当するものがあれば、そのキーを使う。
- 純粋な単語問題は `__vocab__`。
- 該当が無い場合のみ新規パターンを作る。**1レッスンあたり最大3つ**を目安とし、既存で代用できるなら作らない。
- 新規パターンは `{cat:'<8カテゴリのいずれか>', key:'<new_key>', t:'<表示名>', ex:'<説明>'}` の形で、`PATS_DEF` 配列の該当カテゴリブロック末尾に追加する。8カテゴリは `freq` `time` `want` `feel` `ask` `req` `comp` `desc`。

#### タイプ別の作成基準

**vocab**：`explain` は「意味 → バッククォートで例文 → 関連語や注意点」の順。ダミー選択肢は同じ品詞・同じ分野で紛らわしいものにする。

**fill**：空欄は1つのみ。`choices` に正解（`blank`）を必ず含める。形の違い（原形・-ing・過去分詞等）をダミーにすると効果が高い。

**grammar**：誤文はノートに誤→正の実データがあればそれを使い、無ければ学習者がしやすい誤りを作る。正解位置（A/B）は偏らせない。`explain` には「なぜ正しいか」と「なぜ逆が誤りか」の両方を書く。`deep` は紛らわしい用法の整理や時制比較など深掘りの価値がある問題にだけ付ける（1レッスンあたり0〜2問）。

**translate**：正解が複数ありうる場合は ` / ` 区切りで併記する（自動採点で全候補と比較される）。3文字以下の単語は完全一致で判定されるため、短縮形と非短縮形（It's / It is）は両方併記する。

**free**：模範解答の末尾に「など」を付け、自己採点の幅を持たせる。

#### 記述上の注意

- 文字列にアポストロフィを含む場合は、ダブルクォートで囲む。シングルクォート文字列の中で `\'` を使うと崩れやすいので避ける。
- 誤りや不自然な点がノートにあれば、問題化する前に正しい形へ直す。タイポ自体が学習ポイント（polish / Polish など）なら問題化してよい。

### B-3. index.html への追記

1. `PATS_DEF` 追加分があれば、`const PATS_DEF = [` の直後に挿入する。
2. QBブロックは、QB配列の閉じ `];` の直前に、コメント行＋問題オブジェクト群を挿入する。
3. Editツールで直接 `index.html` を書き換える。別ファイルに書き出して人間が貼り直す運用はしない。

### B-4. 検証（コミット前に必ず実行）

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

エラーが出たら修正して再実行する。意味・訳・文法説明に誤りがないか、他レッスンへの言及がないかも目視で確認する。**検証が通るまでコミットしない**。

### B-5. コミット・反映

```bash
git add index.html
git commit -m "<対象月>分のQB追加（<回の範囲>、<問題数>問）"
git push
```

GitHub Pagesは `index.html` を直接配信しているので、pushすれば反映される（ビルド工程なし）。

### B-6. 完了報告（簡潔に）

- 処理した回番号と問題数、総問題数
- 新規追加した pat（あれば）
- コミットハッシュと push の成否
- ノートに誤りがあって直した点（あれば1〜2行）

---

## 補足

- 月が変わったのにノートが無い（その月にレッスンが無かった）場合は、追加なしで終了してよい。
- QB化済みかどうかは `index.html` の `lesson:'<回番号>` の有無だけで判定する。状態管理用のフラグは持たない。
- 既存のノート本文は、QB化の後も書き換えない（履歴として残す）。
