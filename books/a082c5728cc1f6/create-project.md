---
title: "Unity プロジェクトの準備"
free: false
---

## プロジェクトの新規作成
Unity で新規プロジェクトを作成します。
プロジェクトテンプレートは 3D を選択します。
![](/images/create-project.png)

## SteamVR Plugin のインストール
Unity Asset Store で SteamVR Plugin を**Myアセットに追加**します。
https://assetstore.unity.com/packages/tools/integration/steamvr-plugin-32647

![](/images/add-to-my-assets.png)

そのまま Asset Store 上で**Unityで開く**ボタンを押します。
Unity 上で Package Manager が開くので、SteamVR Plugin をインポートします。

![](/images/open-in-unity.png)
![](/images/import-package.png)

ダイアログが表示されたら OK をクリック後、Unity を再起動します。
![](/images/restart-dialog.png)

プロジェクトに `Assets/SteamVR` フォルダが作られていれば、インストール完了です。
![](/images/steamvr-folder.png)

:::details SteamVR Plugin を使わない場合
このチュートリアルでは SteamVR Plugin に含まれている OpenVR のファイルを使用して開発を行いますが、OpenVR のファイルのみを別途ダウンロードして使用することも可能です。
最小限のファイルだけで開発したい場合や、最新の OpenVR を使いたい場合は、OpenVR の GitHub リポジトリから

- **headers/openvr_api.cs**
- **bin/win64/openvr_api.dll**
（他のプラットフォームで動かす場合は、対応する dll ファイル）

をダウンロードして `Assets` フォルダ内に配置してください。

https://github.com/ValveSoftware/openvr
:::

## プロジェクトの設定
メニューから **Edit > Project Setttings** を開きます。
左側の **XR Plug-in Management** を選択します。
**Initialize XR on Startup** のチェックを外します。
![](/images/turn-off-xr-plugin-management.png)

## シーンを空にする
シーン内の全てのオブジェクトを削除して、空の状態にします。
![](/images/empty-scene.png)

これで Unity プロジェクトの準備は完了です。