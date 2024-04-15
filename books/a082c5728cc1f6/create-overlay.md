---
title: "オーバーレイの作成"
free: false
---

## key と name
オーバーレイの作成時に `key` と `name` の 2 つの文字列を設定します。
`key` は他のオーバーレイと重複しない一意な文字列です。
`name` は表示用の任意の文字列です。
ここでは、それぞれ `"WatchOverlayKey"` と `"WatchOverlay"` とします。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

+   var key = "WatchOverlayKey";
+   var name = "WatchOverlay";
}
```

※ 実際のアプリでは、重複を避けるため `key` に `Application.companyName` や `Application.productName` などを組み込むとよいかと思います。

## オーバーレイハンドル
ファイルをファイルハンドルで操作するように、オーバーレイはオーバーレイハンドルで操作します。
オーバーレイハンドルを保存する変数を作成します。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

    var key = "WatchOverlayKey";
    var name = "WatchOverlay";
+   var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
}
```

初期値の `OpenVR.k_ulOverlayHandleInvalid` はオーバーレイが作成できていないことを表す値です。

## オーバーレイの作成とハンドルの取得
オーバーレイの作成には [CreateOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateOverlay_System_String_System_String_System_UInt64__) を使います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateOverlay) を参照）
引数に、先ほど作成した `key`, `name` と `overlayHandle` の参照を渡します。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

    var key = "WatchOverlayKey";
    var name = "WatchOverlay";
    var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
+   var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
}
```

`overlayHandle` に、作成されたオーバーレイのハンドルが保存されます。
作成時のエラーは関数の戻り値として取得できます。

## エラー処理
エラー処理を追加します。
```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

    var key = "WatchOverlayKey";
    var name = "WatchOverlay";
    var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
    var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの作成に失敗しました: " + error);
+   }
}
```
OpenVR の初期化と同じように、エラーが発生しなければ EVROverlayError.None が返ります。
オーバーレイ関連のエラーは [EVROverlayError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVROverlayError.html) に定義されています。

## オーバーレイのクリーンアップ
作成したオーバーレイを、アプリケーション終了時に破棄します。

### overlayHandle を移動
overlayHandle を Steam() 内から、クラスのメンバに移動します。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
+   private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();

-       var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
        var key = "WatchOverlayKey";
        var name = "WatchOverlay";
        var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
    }

    ～省略～
}
```

これは OnDestroy() から、overlayHandle を使用するためです。

### オーバーレイの破棄
オーバーレイの破棄に使用するのは `DestroyOverlay()` です。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::DestroyOverlay) を参照）
OnDestroy() 内で破棄します。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();

        var key = "WatchOverlayKey";
        var name = "WatchOverlay";
        var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
    }

    private void OnDestroy()
    {
+       if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
+       {
+           OpenVR.Overlay.DestroyOverlay(overlayHandle);
+       }

        ShutdownOpenVR();
    }

    ～省略～
}
```

### オーバーレイの確認
SteamVR に同梱されている Overlay Viewer を使って、実際にオーバーレイが作られているか確認してみましょう。

SteamVR のウィンドウのメニューから Developer > Overlay Viewer を選択します。
![](/images/overlay-viewer-menu.png)

Overlay Viewer では、SteamVR 上に作成されているオーバーレイの一覧を確認できます。
![](/images/overlay-list.png)

この状態でプログラムを実行してみましょう。
Overlay Viewer の左上の一覧に、先ほど指定したキー `WatchOverlayKey` が追加されます。
クリックすると、左下にオーバーレイの詳細が表示されますね。
![](/images/overlay-viewer-created.png)

右側の灰色はオーバーレイの描画内容が表示されますが、今は何も描画していない状態です。

:::message
Overlay Viewer の実行ファイルは、SteamVR のディレクトリに入っています。
デフォルトは `C:\Program Files (x86)\Steam\steamapps\common\SteamVR\bin\win32\overlay_viewer.exe` です。
開発中は何度も起動することになるので、ショートカットを作成しておくと便利です。
:::

## コード整理
ここまでのコードを関数に分けつつ整理しておきます。

### オーバーレイの作成
オーバーレイの作成は `CreateOverlay()` という関数に分けておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
       InitOpenVR();


-      var key = "WatchOverlayKey";
-      var name = "WatchOverlay";
-      var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
-      if (error != EVROverlayError.None)
-      {
-          throw new Exception("オーバーレイの作成に失敗しました: " + error);
-      }
+      overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    }

    ～省略～

+   private ulong CreateOverlay(string key, string name) {
+       var handle = OpenVR.k_ulOverlayHandleInvalid;
+       var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイの作成に失敗しました: " + error);
+       }
+       return handle;
+   }
}
```

### オーバーレイの破棄
オーバーレイの破棄は `DestroyOverlay()` という関数に分けておきます。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void OnDestroy()
    {
-       if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
-       {
-           OpenVR.Overlay.DestroyOverlay(overlayHandle);
-       }
+       DestroyOverlay(overlayHandle);
        ShutdownOpenVR();
    }

    ～省略～

+   private void DestroyOverlay(ulong handle) {
+       if (handle != OpenVR.k_ulOverlayHandleInvalid)
+       {
+           OpenVR.Overlay.DestroyOverlay(handle);
+       }
+   }
}
```

## 最終的なコード
```cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    }

    private void OnDestroy()
    {
        DestroyOverlay(overlayHandle);
        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System != null) return;

        var error = EVRInitError.None;
        OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
        if (error != EVRInitError.None)
        {
            throw new Exception("OpenVRの初期化に失敗しました: " + error);
        }
    }

   private void ShutdownOpenVR()
   {
       if (OpenVR.System != null)
       {
           OpenVR.Shutdown();
       }
   }

    private ulong CreateOverlay(string key, string name) {
        var handle = OpenVR.k_ulOverlayHandleInvalid;
        var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
        return handle;
    }

    private void DestroyOverlay(ulong handle) {
        if (handle != OpenVR.k_ulOverlayHandleInvalid)
        {
            OpenVR.Overlay.DestroyOverlay(handle);
        }
    }
}
```
