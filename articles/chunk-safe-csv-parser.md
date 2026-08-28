---
title: "分割して読み込むと壊れるCSVパーサーを、壊れないようにする"
emoji: "🔪"
type: "tech"
topics: ["javascript", "csv", "個人開発", "フロントエンド", "パーサー"]
published: true
---

ブラウザで200MBのCSVを開くツールを作りました。外部ライブラリは使っていません。

一番苦労したのは構文解析そのものではなく、**「ファイルを分割して読んでも、一括で読んだときと同じ結果になる」**を保証する部分でした。ここを雑にやると、数十万行に1行だけ壊れるという最悪のバグになります。

## なぜ分割して読む必要があるのか

`FileReader.readAsText()` でファイル全体を一度に読むと、200MBのファイルが**文字列として丸ごとメモリに乗ります**。JavaScriptの文字列はUTF-16なので、実際には400MB前後を消費します。そこからさらに配列に変換するので、タブが落ちます。

なので `File.slice()` で4MBずつ読みます。

```js
function readSlice(file, start, end) {
  return new Promise(function (resolve, reject) {
    var r = new FileReader();
    r.onload = function () { resolve(new Uint8Array(r.result)); };
    r.onerror = function () { reject(r.error); };
    r.readAsArrayBuffer(file.slice(start, end));
  });
}
```

ここから2つの厄介な問題が出ます。

## 問題1：マルチバイト文字がチャンクの境目にまたがる

4MBちょうどの位置が、たまたま「あ」の1バイト目と2バイト目の間だったらどうなるか。

```js
// これはダメ
new TextDecoder('utf-8').decode(chunk1);   // 末尾が壊れる
new TextDecoder('utf-8').decode(chunk2);   // 先頭が壊れる
```

両方に文字化けが出ます。しかも**同じデコーダを使い回してもダメ**です。

正解は `stream: true` です。

```js
var decoder = new TextDecoder('utf-8');   // チャンクをまたいで使い回す
// ...
parser.push(decoder.decode(bytes, { stream: true }));
```

こう書くと、デコーダは**中途半端なバイト列を内部に保持して、次のチャンクの先頭とくっつけてから**文字にしてくれます。標準機能なので自前でバッファを持つ必要はありません。

ただし、デコーダのインスタンスを使い回すことが前提です。チャンクごとに `new TextDecoder()` すると、この状態が捨てられて壊れます。**私は最初これを踏みました。**

## 問題2：引用符の中の改行がチャンクの境目にまたがる

こちらの方が厄介です。CSVはこういう行を許します。

```csv
id,note
1,"これは
複数行にわたるメモです"
2,plain
```

`"これは` までで4MBに達したらどうなるか。素朴に「チャンクごとに行分割してパースする」実装だと、**そこで行が切れたものとして扱われて壊れます**。

解決策は、パーサーを**状態機械にして、状態をチャンクをまたいで保持する**ことです。

```js
function DelimitedParser(delim, onRow) {
  this.delim = delim;
  this.onRow = onRow;
  this.row = [];          // 組み立て中の行
  this.field = '';        // 組み立て中のフィールド
  this.quoted = false;    // 引用符の中にいるか
  this.afterQuote = false;// 閉じ引用符を見た直後か
  this.lastCR = false;    // 直前が CR か（CRLF の判定用）
}
```

`push()` は1文字ずつ状態を進めるだけで、行が完成したときに `onRow` を呼びます。チャンクが尽きても状態はインスタンスに残るので、次のチャンクが来たら続きから再開できます。

```js
DelimitedParser.prototype.push = function (text) {
  for (var i = 0; i < text.length; i++) {
    var ch = text[i];

    if (this.afterQuote) {
      this.afterQuote = false;
      if (ch === '"') { this.field += '"'; continue; }  // "" はリテラルの "
      this.quoted = false;
      // ここで抜けて、ch を通常処理に流す
    }

    if (this.quoted) {
      if (ch === '"') this.afterQuote = true;
      else this.field += ch;
      continue;
    }
    // ... 以下、区切り文字と改行の処理
  }
};
```

`afterQuote` が要るのは、`""`（エスケープされた引用符）と「引用の終わり」を**1文字先を見ないと区別できない**からです。そしてその1文字が次のチャンクにあるかもしれない。だから状態として持ちます。

## これをどうテストするか

「一括で読んだ結果」と「N文字ずつ読んだ結果」が一致することを、複数のNで確認します。

```js
function parseAll(text, delim, chunkSize) {
  const rows = [];
  const p = new DelimitedParser(delim, r => rows.push(r.slice()));
  if (chunkSize) {
    for (let i = 0; i < text.length; i += chunkSize) {
      p.push(text.slice(i, i + chunkSize));
    }
  } else {
    p.push(text);
  }
  p.end();
  return rows;
}

const tricky = 'id,name,note\n1,"Smith, John","He said ""ok""\nnext line"\n2,Plain,simple\n';
const whole = parseAll(tricky);
for (const size of [1, 2, 3, 7, 13, 64]) {
  assert.deepEqual(parseAll(tricky, ',', size), whole);
}
```

**1文字ずつ流し込んでも一致する**ことが確認できれば、あらゆる境界パターンを踏んだことになります。4MBずつだと境界は数十回しか訪れませんが、1文字ずつなら全ての位置が境界になります。

このテストで、実際に2つのバグが見つかりました。

## おまけ：区切り文字の判定

CSVかTSVかセミコロン区切りかは、ファイルには書かれていません。素朴な方法は「一番多く出現する文字」ですが、これは**本文にカンマが多い場合に外します**。

```csv
name,note
Bob,"Hello, world, again"
Amy,"Yes, indeed, truly"
```

カンマの出現数は7、しかし正しい区切り文字はカンマで列数は2。もし本文がセミコロンだらけだったら誤判定します。

なので、**列数の一貫性**で選びます。

```js
for (var i = 0; i < DELIMS.length; i++) {
  var d = DELIMS[i];
  var counts = lines.map(function (l) { return splitLine(l, d).length; });
  var first = counts[0];
  if (first < 2) continue;
  var consistent = counts.filter(function (c) { return c === first; }).length;
  var s = consistent / counts.length * 100 + Math.min(first, 50);
  if (s > bestScore) { bestScore = s; best = d; }
}
```

「その文字で切ったとき、全行が同じ列数になるか」を見る。正しい区切り文字なら一貫し、間違っていればバラバラになります。一貫性を主、列数を従にして採点します。

## 計測

17MB・20万行のCSV（引用符内の改行が約4万行、エスケープ引用符が約4万行）で:

```
文字コード判定      4 ms
解析              385 ms   (45 MB/s)
型推論              1 ms
全文フィルタ        61 ms
```

ブラウザ側は仮想スクロールにしてあるので、**5万行を読み込んでもDOMに存在する行は30**です。スクロールしても増えません。

## できたもの

https://kotebako.com/loupe/

CSVやJSONをドロップすると開きます。処理は全部ブラウザの中なので、ファイルはどこにも送られません。ページを読み込んだあとに機内モードにしても動きます。

コードはこちらです。3ファイル、依存ゼロです。

https://github.com/kotebako/loupe

---

本番のCSVをオンラインの変換サイトに貼る、という習慣が広く存在します。「1時間後に削除します」と書いてあっても、それを検証する方法はありません。**送らない実装にすれば、約束ではなく構造の話になります。** そういうものが増えるといいと思って作りました。
