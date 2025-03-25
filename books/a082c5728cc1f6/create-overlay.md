---
title: "オーバーレイの作成"
free: false
---

## key と name の準備

オーバーレイの作成時に `key` と `name` の 2 つの文字列を設定します。
`key` はオーバーレイを識別するときに使われる一意な文字列です。
`name` は表示用の任意の文字列です。
ここでは、それぞれ `"WatchOverlayKey"` と `"WatchOverlay"` として作成します。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

+   var key = "WatchOverlayKey";
+   var name = "WatchOverlay";
}
```

## オーバーレイハンドルの準備

オーバーレイハンドルを保存する変数を作成します。
ファイルをファイルハンドルで操作するように、オーバーレイはオーバーレイハンドルで操作します。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

    var key = "WatchOverlayKey";
    var name = "WatchOverlay";
+   var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
}
```

初期値の [OpenVR.k_ulOverlayHandleInvalid](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html?q=k_ulOverlayHandleInvalid#Valve_VR_OpenVR_k_ulOverlayHandleInvalid) はオーバーレイが作成できていないことを表しています。ハンドルは `ulong` 型です。

## オーバーレイの作成とハンドルの取得

オーバーレイの作成には [CreateOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateOverlay_System_String_System_String_System_UInt64__) を使います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateOverlay) を参照）
先ほど作成した `key`, `name` と `overlayHandle` の参照を引数として渡します。

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
戻り値はオーバーレイ作成時のエラー情報です。

## エラー処理

オーバーレイの作成に失敗したときのエラー処理を追加します。

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

エラーがなければ `EVROverlayError.None` が返ります。
オーバーレイ関連のエラー内容は [EVROverlayError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVROverlayError.html) で定義されています。

## オーバーレイのクリーンアップ

アプリケーション終了時にオーバーレイを破棄するコードを追加します。

### overlayHandle を移動

`overlayHandle` を `Start()` 内から、クラスのメンバ変数に移動します。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
+   private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();

        var key = "WatchOverlayKey";
        var name = "WatchOverlay";
-       var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
        var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
    }

    ～省略～
}
```

### オーバーレイの破棄

オーバーレイの破棄に使用するのは [DestroyOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_DestroyOverlay_System_UInt64_) です。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::DestroyOverlay) を参照）
`OnApplicationQuit()` を作成して、オーバーレイを破棄するコードを追加します。

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

+   private void OnApplicationQuit()
+   {
+       if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
+       {
+           var error = OpenVR.Overlay.DestroyOverlay(overlayHandle);
+           if (error != EVROverlayError.None)
+           {
+               throw new Exception("オーバーレイの破棄に失敗しました: " + error);
+           }
+       }
+   }

    private void OnDestroy()
    {
        ShutdownOpenVR();
    }

    ～省略～
}
```

`OpenVR.Overlay.DestroyOverlay()` は `OpenVR.Shutdown()` よりも先に実行する必要があるため、`OnDestory()` より先に実行される `OnApplicationQuit()` に書いています。
https://docs.unity3d.com/Manual/ExecutionOrder.html

### オーバーレイの確認

オーバーレイが作成できたか確認してみましょう。
SteamVR に同梱されている **Overlay Viewer** を使うと、作成されたオーバーレイを確認できます。

SteamVR のウィンドウのメニューから **Developer > Overlay Viewer** を選択します。
![](/images/steamvr-menu.png)
![](/images/overlay-viewer-menu.png)

作成されているオーバーレイの一覧が表示されます。
既に SteamVR のシステムによって作成されたオーバーレイがたくさん表示されていますね。
![](/images/overlay-list.png)

この状態で Unity からプログラムを実行してください。

オーバーレイの一覧に、先ほど指定したキー `WatchOverlayKey` が追加されます。
クリックするとオーバーレイの詳細が表示されます。
![](/images/overlay-viewer-created.png)

右側の灰色の領域にオーバーレイのプレビューが表示されますが、まだ何も描画していないので空の状態です。

オーバーレイが作成できていることを確認できたので、プログラムを終了して Overlay Viewer を閉じます。

:::message
Overlay Viewer は SteamVR のインストールディレクトリに入っています。
`C:\Program Files (x86)\Steam\steamapps\common\SteamVR\bin\win64\overlay_viewer.exe`

開発中は何度も起動することになるので、ショートカットを作成しておくと便利です。
:::

## コード整理

### オーバーレイの作成

オーバーレイの作成は `CreateOverlay()` という関数を作って分けておきます。
`key` と `name` を受け取って、オーバーレイハンドルを返します。
関数に分ける際に、変数名を変えている部分があるので注意してください。

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
+       // 少しコードが変わっています
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

同様に `DestroyOverlay()` という関数に分けておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void OnApplicationQuit()
    {
-       if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
-       {
-           var error = OpenVR.Overlay.DestroyOverlay(overlayHandle);
-           if (error != EVROverlayError.None)
-           {
-               throw new Exception("オーバーレイの破棄に失敗しました: " + error);
-           }
-       }
+       DestroyOverlay(overlayHandle);
    }

    ～省略～

+   // overlayHandle -> handle に変数名を変えています
+   private void DestroyOverlay(ulong handle)
+   {
+       if (handle != OpenVR.k_ulOverlayHandleInvalid)
+       {
+           var error = OpenVR.Overlay.DestroyOverlay(handle);
+           if (error != EVROverlayError.None)
+           {
+               throw new Exception("オーバーレイの破棄に失敗しました: " + error);
+           }
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

    private void OnApplicationQuit()
    {
        DestroyOverlay(overlayHandle);
    }

    private void OnDestroy()
    {
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

    private ulong CreateOverlay(string key, string name)
    {
        var handle = OpenVR.k_ulOverlayHandleInvalid;
        var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
        return handle;
    }

    private void DestroyOverlay(ulong handle)
    {
        if (handle != OpenVR.k_ulOverlayHandleInvalid)
        {
            var error = OpenVR.Overlay.DestroyOverlay(handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの破棄に失敗しました: " + error);
            }
        }
    }
}
```

オーバーレイの作成ができたので、次のページではオーバーレイに画像を表示してみます。
