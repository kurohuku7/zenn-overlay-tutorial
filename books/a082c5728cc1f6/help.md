---
title: "その他の情報"
free: false
---


## OpenVR リポジトリ
https://github.com/ValveSoftware/openvr/wiki/API-Documentation


Wiki に書かれていない詳細な情報は C++ のヘッダファイルにコメントで書かれていることが多いです。
こんなことできないかな？と思ったら、とりあえずヘッダファイルの関数を探してみるのが良いと思います。
https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h

## SteamVR Unity Plugin リポジトリ
OpenVR には含まれていない Unity 向けのユーティリティが入っていたりします。
https://github.com/ValveSoftware/steamvr_unity_plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/index.html

## その他リポジトリ
SteamVR オーバーレイアプリの中にはオープンソースで公開されているものもあります。

例えば SteamVR ユーザならお馴染みの OVR Advanced Settings はオープンソースで開発されています。
https://github.com/OpenVR-Advanced-Settings/OpenVR-AdvancedSettings

OVRDrop の前身 OpenVRDesktopDisplayPortal も Unity のプロジェクトがオープンソースとして公開されています。
https://github.com/Hotrian/OpenVRDesktopDisplayPortal

色々あるので探してみてください。

## その他チュートリアル
[VaNiiMenu](https://sabowl.sakura.ne.jp/gpsnmeajp/unity/vaniimenu/) や [EVMC4U](https://gpsnmeajp.github.io/EasyVirtualMotionCaptureForUnity-documents/)、[Virtual Motion Tracker](https://gpsnmeajp.github.io/VirtualMotionTrackerDocument/) 等を開発されている Segment 氏が公開されている記事は特に参考になりました。
https://qiita.com/gpsnmeajp/items/421e3853465df3b1520b

余談ですが [VR 酔い対策ツール](https://store.steampowered.com/app/1393780/)を作った際に、最初は OpenVR のリポジトリを見て C++ で開発しようとしたものの、サンプルを動かすだけでも予想以上に手間取って「これは先が長いな...」と思っていた矢先、Segment 氏が公開されていた Unity のサンプルを見つけて試しに動かしてところ、あっさりと動かせてしまったので Unity で開発することにした経緯があります。

## 3D オブジェクトを表示したい
両目に視差のある画像を表示することで、オーバーレイで立体物を表示できます。
例えば VR ペイントアプリの [Vermillion](https://store.steampowered.com/app/1608400/Vermillion__VR_Painting/) は、オーバーレイアプリとしてオブジェクトを立体で表示できるモードがあり、任意の VR ゲームで画材を持ち込めるようにしています。
https://youtu.be/udc1i97KPLY?si=pZ_Y5mJG2LyYVo9w

### 手法1: Stereo Overlay
VROverlayFlags
VROverlayFlags_StereoPanorama

### 手法2: Stereo Panorama


### 手法3: Projection Overlay
Projection Overlay を使うと左目または右目だけに表示されるオーバーレイを作れます。
目の位置を取りたいときは `GetEyeToHeadTransform()` で目から HMD への変換行列が取得できます。
Origin → HMD → Eye と変換していくと目の座標が計算できます。
`SetOverlayTransformProjection()` あたりを調べてみてください。
https://x.com/kurohuku7/status/1566307113697423360

### 手法4: 平面の組み合わせ
単純な形状であれば、力技で複数のオーバーレイを組み合わせて立体物を表示する方法もあります。
6 枚のオーバーレイを使って Cube を作ったり、`SetOverlayCurvature()` を使ってオーバーレイを湾曲させたり、筒状に丸めることも可能です。
[OVR Locomotion Effect](https://store.steampowered.com/app/1393780/) では、グリッドの空間を表示するために 6 枚の巨大なオーバーレイでプレイヤーを閉じ込める方法で、立体的な空間を作っています。
また、風のエフェクトは円筒状に丸めたオーバーレイの中に HMD が入るように配置することで、立体的に見えるような風を表現しています。
https://youtu.be/vv-e_6-vjiE?si=JMYUDWER3vI6ujZm

立体視については、`openvr.h` 内を `stereo` で検索すると手がかりが出てきます。

## HMD に表示されている画像を取得したい
`OpenVR.Compositor.GetMirrorTextureD3D11()` や `GetMirrorTextureGL()` で取得できます。

