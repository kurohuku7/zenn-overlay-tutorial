---
title: "Unity プロジェクトの準備"
free: false
---

## Unity プロジェクトの作成
Unity で新規プロジェクトを作成します。
プロジェクトテンプレートは 3D を選択してください。
![](https://storage.googleapis.com/zenn-user-upload/df7746e620dc-20240227.png)

## SteamVR Plugin のインストール
Unity Asset Store で SteamVR Plugin をマイアセットに追加します。
https://assetstore.unity.com/packages/tools/integration/steamvr-plugin-32647

そのまま、Asset Store の「Unityで開く」ボタンを押すと Package Manager が開くので、パッケージをインポートします。
![](https://storage.googleapis.com/zenn-user-upload/14df39868604-20240122.png)

ダイアログが表示されるので、指示に従って Unity を再起動しておきます。
![](https://storage.googleapis.com/zenn-user-upload/735f69eb776b-20240122.png)

プロジェクトに Assets/SteamVR フォルダが作られていれば OK です。
![](https://storage.googleapis.com/zenn-user-upload/5089910653b4-20240227.png)

## プロジェクトの設定
Edit > Project Setttings を開きます。
左側の XR Plug-in Management で Initialize XR on Startup から OpenVR Loader にチェックを入れます。
左側の OpenVR から Application Type を Overlay に設定します。

:::message
上記のプロジェクト設定はやらなくてもオーバーレイは動作しました。
:::

## シーンを空にする
わかりやすくするため、デフォルトでシーンに配置されている Main Camera と Directional Light を削除して、シーンを空の状態にしておきます。