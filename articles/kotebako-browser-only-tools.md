---
title: "業務ファイルを他人のサーバーに上げたくないので、ブラウザだけで完結する道具を作った"
emoji: "🧰"
type: "tech"
topics: ["javascript", "個人開発", "githubpages", "フロントエンド"]
published: true
---

「顧客名簿のCSVが文字化けした」「この画像、資料に貼るには重すぎる」。

こういうとき、検索すると変換サイトが山ほど出てきます。でも**業務データをどこの誰が運用しているか分からないサーバーにアップロードする**のは、正直かなり気持ちが悪い。プライバシーポリシーを読んでも「アップロードされたファイルは1時間後に削除されます」としか書いていない。それを検証する手段はありません。

なので、**一切送信しない**道具を作りました。

https://kotebako.github.io/

小手箱（こてばこ）といいます。手箱＝身の回りの小物をしまう箱、その小さい方です。

## 何が違うのか

処理が全部ブラウザの中で終わります。**ページを開いたあとに Wi-Fi を切っても動きます。** これが一番分かりやすい証明だと思っています。

サーバーサイドのコードは存在しません。GitHub Pages に静的ファイルを置いているだけです。リポジトリは公開してあるので、何をしているかは全部読めます。

https://github.com/kotebako/kotebako.github.io

## 今ある道具

### 画像を一括リサイズ・圧縮

何枚でもまとめて縮小します。`canvas` で完結。

実装で気を使ったのは3点です。

**1. 段階的に縮小する**

1200px から 400px へ一気に `drawImage` すると、間引きになってジャギーが出ます。半分ずつ落としてから最終サイズにすると滑らかになります。

```js
var cw = sw, ch = sh;
while (cw / 2 > dw) {
  var next = document.createElement('canvas');
  next.width  = Math.max(1, Math.round(cw / 2));
  next.height = Math.max(1, Math.round(ch / 2));
  var nx = next.getContext('2d');
  nx.imageSmoothingQuality = 'high';
  nx.drawImage(cur, 0, 0, next.width, next.height);
  cur = next; cw = next.width; ch = next.height;
}
```

**2. 透過PNGをJPEGにすると背景が黒くなる**

canvas の初期状態は透明で、JPEG は透明を表現できないため黒く潰れます。先に白で塗ります。

```js
if (type === 'image/jpeg') {
  ctx.fillStyle = '#fff';
  ctx.fillRect(0, 0, dw, dh);
}
```

**3. 変換して大きくなったら元を返す**

小さいPNGを再エンコードすると膨らむことがあります。サイズを比較して、負けていたら元のBlobをそのまま渡します。

余談ですが、canvas を経由すると Exif が落ちます。位置情報や撮影日時が消えるので、SNSに上げる画像としてはむしろ安全です。

### CSVの文字化けを直す

Excel で開くと `譁?蟄怜喧縺・` になるやつです。

原因ははっきりしていて、**Windows版Excelは、BOMが無いUTF-8ファイルをShift_JISだと思い込んで開く**からです。Web系のシステムが吐くCSVはたいてい BOM なし UTF-8 なので、ほぼ確実に化けます。

文字コードの判定は、外部ライブラリを使わずに `TextDecoder` を総当たりして書きました。デコード結果を採点して、一番日本語として成立しているものを採用します。

```js
function score(text) {
  if (text.indexOf('�') !== -1) return -1;   // 置換文字が出たら即失格

  var jp = 0, junk = 0, ascii = 0, total = text.length;
  for (var i = 0; i < total; i++) {
    var c = text.charCodeAt(i);
    if (c < 0x80) { ascii++; continue; }
    if ((c >= 0x3040 && c <= 0x30FF) ||     // かな
        (c >= 0x4E00 && c <= 0x9FFF)) {     // 漢字
      jp++;
    } else if (c >= 0x0080 && c <= 0x02FF) {
      junk += 2;                             // ラテン拡張の氾濫＝典型的な誤デコード
    } else {
      junk++;
    }
  }
  var nonAscii = total - ascii;
  if (nonAscii === 0) return 0.5;
  return (jp - junk) / nonAscii;
}
```

肝は `U+FFFD`（置換文字）が1つでも出たら即失格にすることです。`TextDecoder` は `fatal: false` だと壊れたバイト列を握り潰して置換文字にしてしまうので、そこを検出に使えます。

実際に Shift_JIS / EUC-JP / UTF-8 で同じCSVを作って通したところ、3種類とも正しく判定・復元できました。

### 文字数カウント

原稿用紙の枚数と読み上げ時間が出ます。5分のプレゼン原稿が何字なのか毎回調べていたので作りました（300字/分で1,500字です）。

サロゲートペアを1文字として数えたかったので `Array.from` を使っています。`String.prototype.length` はUTF-16のコード単位を返すため、絵文字や一部の異体字が2文字になります。

```js
Array.from('𩸽と😀').length   // => 3
'𩸽と😀'.length                // => 5
```

## 構成

ビルド工程はありません。外部ライブラリもCDNも使っていません。素のHTML/CSS/JSだけです。

```
index.html            トップ
image-resize/         画像リサイズ
csv-mojibake/         CSV文字コード変換
moji-count/           文字数カウント
assets/               共通CSSと各ツールのJS
```

`index.html` をローカルでダブルクリックしても普通に動きます。npm install も webpack も要りません。この手の小さな道具に、モダンなフロントエンド一式は過剰だと思っています。

## なぜ無料なのか

広告も課金もありません。単に、自分が困って作ったものを置いてあるだけです。

ただ、この「送らない」という性質はもっと当たり前になっていいと思っています。ブラウザは十分に強くなっていて、画像処理も文字コード変換もクライアントだけで足ります。**わざわざサーバーに送る理由がない処理を、送らずに済ませる**。それだけの話です。

道具は少しずつ増やしていきます。要望があれば教えてください。

https://kotebako.github.io/
