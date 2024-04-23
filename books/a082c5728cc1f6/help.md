---
title: "その他の情報"
free: false
---

更に詳しい情報がほしいと気にどこを見るべきか、ざっくりですが思いつくところをまとめておきます。

## OpenVR リポジトリ
OpenVR の公式リポジトリが最も重要になります。

https://github.com/ValveSoftware/openvr

まずは Wiki を読むところから始めてみると良いと思います。
https://github.com/ValveSoftware/openvr/wiki/API-Documentation


Wiki に書かれていない詳細な情報は C++ のヘッダファイルにコメントで書かれていることが多いです。こういう機能ないかな？と思ったら、とりあえずヘッダファイルを探ってみるのが良いと思います。
[https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h](https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h)

その他、わからないことがあれば issue を探ってみてください。
https://github.com/ValveSoftware/openvr/issues

## SteamVR Unity Plugin リポジトリ
Unity 向けのユーティリティが入っていたりします。
https://github.com/ValveSoftware/steamvr_unity_plugin

こちらがドキュメントです。
https://valvesoftware.github.io/steamvr_unity_plugin/api/index.html

## その他リポジトリ
[OVR Advanced Settings](https://store.steampowered.com/app/1009850/OVR_Advanced_Settings/) などオープンソースで開発されているプロジェクトは、参考になる部分が多いです。
https://github.com/OpenVR-Advanced-Settings/OpenVR-AdvancedSettings

:::details OVRAS 有料化について
丁度、このチュートリアルを書いている最中の 2024-04-15 に、今後の継続的な開発を目的として上記の OVR Advanced Setting は Steam 上で有料化されました。公開されているソースコードが今後どうなるのかアナウンスが見つからなかったのですが、もしかすると時期によっては公開状況などが変わっているかもしれないので、各自でご確認いただければと思います。
:::

## その他チュートリアル
[VaNiiMenu](https://sabowl.sakura.ne.jp/gpsnmeajp/unity/vaniimenu/) や [EVMC4U](https://gpsnmeajp.github.io/EasyVirtualMotionCaptureForUnity-documents/)、[Virtual Motion Tracker](https://gpsnmeajp.github.io/VirtualMotionTrackerDocument/) などを開発されている Segment 氏が公開されている技術記事は、個人的に特に参考にさせて頂き、とても助かりました。OpenVR 関連の色々なソースコードや技術記事を公開されています。
https://qiita.com/gpsnmeajp/items/421e3853465df3b1520b

（余談ですが VR 酔い対策ツールを作り始めたときに、最初は OpenVR のリポジトリを見て C++ で開発しようとしたものの、公式のサンプルを動かすだけでも想像以上に手間取って「これは先が長いな...」と思っていた矢先、Segment 氏が公開されていた Unity のサンプルを見つけて試しに動かしてみたら、あまりにもあっさりと動かせてしまったので Unity に切り替えて開発することにした経緯があります。）

## 3D オブジェクトを表示したい
両目にそれぞれ視差のある画像を表示することで、オーバーレイで立体物を表示できます。
例えば VR ペイントアプリの [Vermillion](https://store.steampowered.com/app/1608400/Vermillion__VR_Painting/) は、オーバーレイとして 3D オブジェクトを表示できる機能があり、VR ゲームへ画材を持ち込めるようになっています。
https://youtu.be/udc1i97KPLY?si=pZ_Y5mJG2LyYVo9w

十分な情報は準備できていないのですが、今知っている手法をざっくりまとめておきます。
※下記はしっかり検証していないものもあるので、間違いを含む可能性が高めです。

### 手法1: SideBySide
`SetOverlayFlags()` で `VROverlayFlags_SideBySide_Parallel` を指定することで、SideBySide 画像を表示できます。
`VROverlayFlags_SideBySide_Parallel` を使用したオーバーレイは、画像の半分を左目用に、もう半分を右目用に使用することで、立体視を行います。
Unity 上で左目用と右目用のカメラを 2 つ作成して、それぞれの映像を 1 枚のテクスチャに並べて描画することで立体をオーバーレイで表示できます。

### 手法2: Stereo Panorama
`SetOverlayFlag()` で `VROverlayFlags_StereoPanorama` を指定すると Stereo Panorama が使えます。
こちらも 1 枚の画像を左目用と右目用の領域に分けて立体視を実現できます。
上で紹介した [Vermillion はこれで実装したらしい](https://x.com/thmsvdberg/status/1655997759160287232)です。

### 手法3: Overlay Projection
`SetOverlayTransformProjection()` を使うと左目だけ、または右目だけに表示されるオーバーレイを作れます。
目の位置を取りたいときは `GetEyeToHeadTransform()` が使えます。
Origin → HMD → Eye と変換していくと目の位置が計算できます。
https://x.com/kurohuku7/status/1566307113697423360

### 手法4: 平面の組み合わせ
単純な形状であれば、力技で複数のオーバーレイを組み合わせて立体物を表示する方法もあります。
6 枚のオーバーレイを使って Cube を作ったり、`SetOverlayCurvature()` を使ってオーバーレイを湾曲させたり、筒状に丸めることも可能です。
[OVR Locomotion Effect](https://store.steampowered.com/app/1393780/) では、グリッドの空間を表示するために 6 枚の巨大なオーバーレイでプレイヤーを閉じ込める方法で、立体的な空間を作っています。
また、風のエフェクトは円筒状に丸めたオーバーレイの中に HMD が入るように配置することで、立体的に見える風を表現しています。
https://youtu.be/vv-e_6-vjiE?si=JMYUDWER3vI6ujZm

立体視については、`openvr.h` 内を `stereo` や `sidebyside` で検索すると手がかりが出てきます。

## HMD に表示されている画像を取得したい
`OpenVR.Compositor.GetMirrorTextureD3D11()` や `GetMirrorTextureGL()` で取得できます。