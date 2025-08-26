---
title: "はじめに"
free: false
---

これは kurohuku が VR 酔い対策ツール [OVR Locomotion Effect](https://store.steampowered.com/app/1393780/OVR_Locomotion_Effect__AntiVR_Sickness/) や、ダッシュボード UI アセット [OVRLE UI](https://assetstore.unity.com/packages/tools/gui/ovrle-ui-dashboard-ui-kit-for-steamvr-270636) の開発から得たオーバーレイアプリ開発の知見を体系的にまとめたものです。

## 作るもの

このチュートリアルでは、サンプルとして VR ゲーム内で使用できる簡単な腕時計のオーバーレイアプリを作ります。

時計はデフォルトで非表示状態となっていて、特定のボタンを押すと数秒だけ時刻が表示されるものを作ります。
![](/images/3sec-display.gif)

SteamVR のダッシュボード上に、左右どちらの手に表示するか選択できる設定画面も作成します。
![](/images/switch-hand.gif)

## 動作環境

このチュートリアルでは、下記の環境でプログラムの動作確認をしています。

- SteamVR beta 2.5.2
- Unity 2022.3.21
- SteamVR Unity Plugin v2.8.0 （OpenVR v2.0.10）
- Meta Quest 3 (ファームウェア v63)
- Virtual Desktop v1.30.5
- Windows 11

SteamVR が使用できる環境であれば HMD や PC との接続方法が異なっていても問題ありません。

## 読む順番

チュートリアルは最初から順番に読み進めることを想定しています。
Unity の新規プロジェクトを作成して、少しずつソースコードを追加していく形式になっています。

## 設定言語

各ツールの画面は、言語を英語に設定した場合の表記になっています。
日本語に設定している場合とメニュー名などが異なる可能性があるため、必要に応じて読み替えてください。

## 情報の正確性・連絡先

公式ドキュメントに書かれていない情報を手探りで調べている部分もあるため、不正確な情報が含まれているかもしれません。何か気づいたことがあれば [@kurohuku7](https://twitter.com/kurohuku7) へご連絡ください。

このチュートリアルの全文は GitHub リポジトリとして公開しているので、誤字脱字なども含めて修正の提案が可能です。
https://github.com/kurohuku7/zenn-overlay-tutorial

## Dashboard UI ライブラリ

筆者が開発した Overlay ダッシュボード UI ライブラリ [OVRLE UI](https://assetstore.unity.com/packages/tools/gui/ovrle-ui-dashboard-ui-kit-for-steamvr-270636) を Asset Store で販売中です！
オーバーレイツールの UI をドラッグアンドドロップで簡単に構築できるので、ぜひお試し下さい。

[![](/images/ovrle-ui.png)](https://assetstore.unity.com/packages/tools/gui/ovrle-ui-dashboard-ui-kit-for-steamvr-270636)
