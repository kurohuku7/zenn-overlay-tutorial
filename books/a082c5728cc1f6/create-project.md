---
title: "Unity プロジェクトの準備"
free: false
---

## プロジェクトの新規作成
任意の名前で Unity の新規プロジェクトを作成します。
プロジェクトテンプレートは 3D を選択します。
![](https://storage.googleapis.com/zenn-user-upload/df7746e620dc-20240227.png)

## SteamVR Plugin のインストール
Unity Asset Store で SteamVR Plugin をMyアセットに追加します。
https://assetstore.unity.com/packages/tools/integration/steamvr-plugin-32647

そのまま Asset Store 上で「Unityで開く」ボタンを押すと Unity の Package Manager が開くので、パッケージをインポートします。
![](https://storage.googleapis.com/zenn-user-upload/14df39868604-20240122.png)

ダイアログが表示されたら、指示に従って OK をクリック後、Unity を再起動します。
![](https://storage.googleapis.com/zenn-user-upload/735f69eb776b-20240122.png)

プロジェクトに Assets/SteamVR フォルダが作られていれば、インストールは完了です。
![](https://storage.googleapis.com/zenn-user-upload/5089910653b4-20240227.png)

## プロジェクトの設定
メニューから Edit > Project Setttings を開きます。
左側の一覧から XR Plug-in Management を選択します。
Initialize XR on Startup のチェックを外します。
![](/images/turn-off-xr-plugin-management.png)

## シーンを空にする
デフォルトの SampleScene 内のオブジェクトを削除して、シーンを空にしておきます。
![](/images/empty-scene.png)