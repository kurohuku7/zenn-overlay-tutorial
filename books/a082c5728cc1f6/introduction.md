---
title: "はじめに"
free: false
---

このチュートリアルについての注意書きです。

## 読む順番
このチュートリアルは、腕時計のオーバーレイアプリを作成することを題材に、順を追って OpenVR Overlay API について解説しています。
まっさらな状態から少しずつスクリプトを継ぎ足していくため、基本的に各ページを順番に読み進めてください。

## 動作環境
下記の環境でコードの動作を確認しています。

- SteamVR beta 2.5.1
- Unity 2022.3.20
- SteamVR Unity Plugin v2.8.0
- OpenVR v2.5.1

## 情報の正確性
間違った情報が含まれている可能性があります。OpenVR のドキュメントやソースコードに書かれていない情報もあるため、結構手探りで API を使っていたりします。

## 連絡先
なにか気づいたことがあれば [@kurohuku7](https://twitter.com/kurohuku7) へご連絡ください。
このチュートリアルのテキストや画像は GitHub の公開リポジトリに入っているので、修正のプルリクエストを送信することもできます。

## 設定言語
各ツールのメニュー名などは、設定言語を英語にした場合の表記になっています。

## OpenVR のファイルについて
SteamVR Unity Plugin に含まれる OpenVR のファイルを使用しています。
プラグイン内の他のファイルを導入したくない場合は、OpenVR のリポジトリから必要なファイルをダウンロードして、プロジェクト内に直接配置してください。

- openvr.cs
- openvr.dll

https://github.com/ValveSoftware/openvr

## エラー処理について
コードの読みやすさのため、エラー発生時に throw を使用している箇所が多いです。

```cs:例外処理を使用したコード例
private void InitOpenVR()
{
   var initError = EVRInitError.None;
   OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
   if (initError != EVRInitError.None)
   {
       throw new Exception("OpenVR の初期化に失敗しました: " + initError);
   }
}
```

return でエラーコード等を返したい場合は、下記のようにコードを読み替えてください。
```cs:エラーコードを使用したコード例1
// 初期化に成功したら EVRInitError.None を返す
private bool InitOpenVR()
{
   var initError = EVRInitError.None;
   OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
   if (initError != EVRInitError.None)
   {
       Debug.LogError("OpenVR の初期化に失敗しました: " + initError);
   }
   return initError;
}
```
```cs:エラーコードを使用したコード例2
// 初期化に成功したら true, 失敗したら false を返す
private bool InitOpenVR()
{
   var initError = EVRInitError.None;
   OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
   if (initError != EVRInitError.None)
   {
       Debug.LogError("OpenVR の初期化に失敗しました: " + initError);
       return true;
   }
  return false;
}
```