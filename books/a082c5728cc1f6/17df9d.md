---
title: "困ったときは（参考資料）"
free: false
---

## 困ったらどこを見ればいい？
### OpenVR リポジトリ
Valve の OpenVR 公式リポジトリが重要になります。
https://github.com/ValveSoftware/openvr

基本的なことは Wiki にまとめられています。
https://github.com/ValveSoftware/openvr/wiki/API-Documentation

Wiki に書かれていない詳細な情報は C++ のヘッダファイルにコメントで書かれていたりします。Wiki には載っていない関数が結構あるので、実際に細かいことをやろうとするとヘッダファイルを見に行くことが多くなります。
https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h

それでも載っていない情報は多いので、詳しいところを触ろうとすると、色々手探りで探していくことになると思います。

### SteamVR Unity Plugin リポジトリ
Unity で作る場合は、同じく Valve の SteamVR Unity Plugin リポジトリも役立ちます。
https://github.com/ValveSoftware/steamvr_unity_plugin

C# から OpenVR の dll を叩くためのラッパーが含まれるのですが、OpenVR 単体には Unity 用の便利機能が追加されていたりします。「こういうのないかな？」と思ったら意外とあったりします。
https://valvesoftware.github.io/steamvr_unity_plugin/api/index.html

例のごとく、ドキュメントには乗っていない関数があるので、実際には Assets 内のソースを見に行くことが多めです。

### その他リポジトリ
SteamVR オーバーレイアプリの中にはオープンソースで公開されているものもあります。

例えば SteamVR ユーザであればお馴染みの OVR Advanced Settings のソースコードは役に立ちました。OVR Advanced Setting がやっていることであれば、ソースを見れば呼び出すべき関数などがわかるので非常にありがたいです。C++ で書かれているため Unity で開発する場合には、一部読み替えが必要になります。
https://github.com/OpenVR-Advanced-Settings/OpenVR-AdvancedSettings

OVRDrop の前身 OpenVRDesktopDisplayPortal はオープンソースとして公開されており、Unity で作られたプロジェクトです。
https://github.com/Hotrian/OpenVRDesktopDisplayPortal

### その他のチュートリアル
VaniiMenu作者のSegment氏が公開されているチュートリアルは非常に参考になりました。
https://qiita.com/gpsnmeajp/items/421e3853465df3b1520b

最初は OpenVR リポジトリを見て C++ で書いていこうと思ったのですが、サンプルをビルドして動かすだけでも結構手間取って、「これは先が長いな...」と思っていた矢先、氏が公開されていたサンプルを見つけて Unity で動かしてみたら、あっさりと動いたので、簡単そうな Unity で作っていくことにしました。

