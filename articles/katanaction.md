---
title: "kintoneで反応速度ゲームを作ってみた"
emoji: "⚔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kintone"]
published: true
---

## はじめに

サイボウズが提供するクラウドサービス [kintone](https://kintone.cybozu.co.jp/) は業務システムが作成できるプラットフォームです。本来は仕事で使うツールとして、

- 案件管理
- 顧客管理
- 日報
- タスク管理

等に使われるケースがほとんどですが、今回はそんな真面目な使い方ではなく、kintoneを **ゲーム** として使ってみた記事です。

**~~不真面目~~ ネタ**記事としてお楽しみください😏

## 作ったもの

- プレイ動画

https://twitter.com/BB_File/status/1369249452238671876

- イメージ写真

![](https://storage.googleapis.com/zenn-user-upload/luptbqr2t4v06sabcg1yyme1ptkz)

その名も **KATANACTION!!** カタナでアクションするゲームです⚔

## 使ったもの

- カタナのおもちゃ × 2本
- obniz(M5StickC版) × 2つ
- obniz(ブロック倒す用) × 1つ
- 磁気センサ × 2つ
- 磁石 × 2つ
- 発泡ブロック × 6つ
- サーボモータ × 2つ


総額15,000円くらい。
**カタナのおもちゃ** はダイソーで売っていたものを分解して中にobnizを埋め込んでいます。
刀身に磁気センサを埋め込み、鞘側に磁石を付けています。

![](https://docs.google.com/drawings/d/e/2PACX-1vTW0Eq7prW8aQGu1yztvEhT2Yej3UtTKsSPuYZLxb0Rgu_8rai4dejxcUBIrY9WFZfeqRMq4uUa8WV7/pub?w=768&amp;h=401)

**倒れる物体** は発泡ブロックを重ねていて、下の段にサーボモータを埋め込んでいます。
最初ソレノイドを使って押し上げる方法でやろうとしましたが、パワーが足りずモータで。
![](https://docs.google.com/drawings/d/e/2PACX-1vTsZRliueevJOwUu-ToTQjzVvLFIqGOia0arbjKlr-17b5_G3Mox1LmTfpkO-oEmNHDAtskT8WDrYdo/pub?w=926&amp;h=520)

わりとちゃんと動く！
https://twitter.com/BB_File/status/1369249780715511811

## システム構成図

kintoneのカスタマイズビュー（HTMLが自由に埋め込める場所）にVue.jsで作成したゲーム画面を埋め込んでいます。obnizを動かすJavaScriptプログラムはkintone上にアップロードしています。
![](https://docs.google.com/drawings/d/e/2PACX-1vTGb01hm8MTj64WQRhECcc7W6P4Wm9LEi12kMoGmofh9s7SH9WI3P8HgStal-BYPjED4XBcU57DssMa/pub?w=927&amp;h=603)

## ソースコード

全部載せると膨大になるので一部のみ抜粋します。

### カタナ

磁気センサはボタンと同じ扱いなので、[obnizのボタン操作のプログラム](https://obniz.com/ja/sdk/parts/Button/README.md)が使えます。

```javascript:katana_sample.js
const katana1 = new Obniz.M5StickC('XXXX-XXXX'); // M5StickC用のメソッド
let count1 = 0;

katana1.onconnect = async function() {
  const sensor1 = katana1.wired('Button', {signal: 26, pull: '3v'});
  sensor1.onchange = async state => {
    if (!state) {
      count1++;
      // カウント数に応じてサーボモータを動かす処理を追記
    }
  };
};
```

鞘に磁石をつけることで「鞘を抜き差しすると磁気センサが反応してカウントアップ」されていきます。あとはカウント数に応じて処理を追記するだけです。

## 倒れる物体

[obnizのサーボモータを操作するプログラム](https://obniz.com/ja/sdk/parts/ServoMotor/README.md)がそのまま使えます。

```javascript:servo_sample.js
const block = new Obniz('XXXX-XXXX');

block.onconnect = async function() {
  servo1 = block.wired('ServoMotor', {gnd:0, vcc:1, signal:2});
  servo2 = block.wired('ServoMotor', {gnd:3, vcc:4, signal:5});
  servo1.angle(0);
  servo2.angle(0);

  // カタナの処理を組み合わせて servo?.angle(90)にする
};
```

## カタナと倒れる物体のコードを合体

```javascript:sample.js
const katana1 = new Obniz.M5StickC('XXXX-XXXX'); // Player1のカタナ用
const katana2 = new Obniz.M5StickC('XXXX-XXXX'); // Player2のカタナ用
const block = new Obniz('XXXX-XXXX'); // 倒れる物体用

let count1 = 0;
let count2 = 0;
let sensor1, sensor2, servo1, servo2;

// obnizのコネクション
katana1.onconnect = () => {
  sensor1 = katana1.wired('Button', {signal: 26, pull: '3v'});
};
katana2.onconnect = () => {
  sensor2 = katana2.wired('Button', {signal: 26, pull: '3v'});
};
block.onconnect = () => {
  servo1 = obniz.wired("ServoMotor", {gnd: 0, vcc: 1, signal: 2});
  servo2 = obniz.wired("ServoMotor", {gnd: 3, vcc: 4, signal: 5});
};

// カタナ1を抜き差ししたときのイベント
sensor1.onchange = async state => {
  if (!state) {
    count1++;
    if (count1 === 1) {
      // Player1が反応した

      if (count2 < 1) {
        // count1が1のときにcount2が1未満なら、Player1が先に動かしたことになる
        // => Player1の勝ち
        servo1.angle(90);
      }
    }
  }
};

// カタナ2を抜き差ししたときのイベント
sensor2.onchange = async state => {
  if (!state) {
    count2++;
    if (count2 === 1) {
      // Player2が反応した

      if (count1 < 1) {
        // count2が1のときにcount1が1未満なら、Player2が先に動かしたことになる
        // => Player2の勝ち
        servo2.angle(90);
      }
    }
  }
};
```

## kintone側

kintone側はわりと無駄遣いをしていて、

- ゲーム画面：ただのHTML画面として利用
- ランキング画面：ただのHTML画面として利用（裏側でランキングデータをkintoneに格納）

といった感じで **「別にkintoneじゃなくても良くね？」** 感満載ですw

ただ、kintoneをフロントにすることで、[kintone JavaScript API](https://developer.cybozu.io/hc/ja/articles/360000361686) が使えるのでkintone内のデータ操作はめちゃくちゃ楽です。
今回だと、サーボモータを `servo?.angle(90)` にした後、各Playerの反応速度をkintoneのDBへ格納するためにkintone JavaScript APIを使っています。

```javascript:post_kintone_sample.js
const startTime = new Date().getTime();
let endTime1, endTime2;

// カタナ1を抜き差ししたときのイベント
sensor1.onchange = async state => {
  if (!state) {
    count1++;
    if (count1 === 1) {
      // Player1が反応した
      // スタートしてから反応までの時間
      endTime1 = (new Date().getTime() - startTime) / 1000;

      if (count2 < 1) {
        // count1が1のときにcount2が1未満なら、Player1が先に動かしたことになる
        // => Player1の勝ち
        servo1.angle(90);

        // kintoneへデータを登録する処理
        const postParams = {
          app: 'XXX',
          record: {
            winner: { value: 'Player1' },
            player1_time: { value: endTime1 },
            player2_time: { value: endTime2 },
          }
        };
        await kintone.api(kintone.api.url('/k/v1/record'), 'POST', postParams);
      }
    }
  }
};
```

## おわりに

kintoneはビジネスツールですが、APIが搭載されたWeb DBなのでビジネス用途以外にもいろいろな使い方ができます。ぜひいろいろなものに使ってみてください！
[無料の開発環境](https://developer.cybozu.io/hc/ja/articles/200720464)もありますよ〜
