---
title: "オーバーレイの作成"
free: false
---

## key と name
オーバーレイは `key` と `name` という文字列を持ちます。
`key` は他のオーバーレイと重複しない一意な文字列です。
`name` は表示用の任意の文字列です。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

+   var key = "WatchOverlayKey";
+   var name = "WatchOverlay";
}
```

※ 実際の `key` には `Application.companyName` や `Application.productName` を組み合わせるのが良いかと思います。

### オーバーレイハンドル
ファイルをファイルハンドルで操作するように、オーバーレイはオーバーレイハンドルで操作します。
先にオーバーレイハンドルを保存するための変数を作っておきます。
オーバーレイが作成できていない場合のハンドルの値が `OpenVR.k_ulOverlayHandleInvalid` として定義されているので、これで初期化しておきます。
```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();

    var key = "WatchOverlayKey";
    var name = "WatchOverlay";
+   var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
}
```

### オーバーレイを作成
[Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateOverlay) を参考にして [CreateOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateOverlay_System_String_System_String_System_UInt64__) を使います。
引数には `key`, `name`, `overlayHandle` の参照を渡します。

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

作成に成功すると `overlayHandle` に、作成されたオーバーレイのハンドルが保存されます。
エラーは `CreateOverlay()` の戻り値として取得できます。

### エラー処理
オーバーレイが作成できなかった場合のエラー処理を追加します。
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
成功したら「エラーが発生しなかった」という意味の EVROverlayError.None が返ってきます。
その他のエラーも [EVROverlayError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVROverlayError.html) に定義されています。

## オーバーレイのクリーンアップ
作成したーバーレイを、アプリケーション終了時に破棄します。
[Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::DestroyOverlay) によると、オーバーレイの破棄は `DestroyOverlay()` です。
`Start()` でオーバーレイを作成して、`OnDestroy()` で破棄することにします。

### overlayHandle を移動
Start() で作成した overlayHanlde を OnDestroy() から使うため、クラスのメンバに移動します。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
+   private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();

-       var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
        var error = OpenVR.Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay", ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
    }

    ～省略～
}
```

### オーバーレイの破棄
`DestroyOverlay()` でオーバーレイを破棄します。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        InitOpenVR();

        var error = OpenVR.Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay", ref overlayHandle);
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

SteamVR のウィンドウの左上のメニューから Developer > Overlay Viewer を選択します。
![](https://storage.googleapis.com/zenn-user-upload/ca13bcee2bec-20240306.png)

Overlay Viewer では、SteamVR 上に作成されているすべてのオーバーレイを確認できます。SteamVR のシステムメニューなどもオーバーレイで表示されていることがわかります。
![](https://storage.googleapis.com/zenn-user-upload/af20c02e2a1b-20240306.png)

この状態でプログラムを実行してみましょう。
Overlay Viewer の左上の一覧に、先ほど指定したキー WatchOverlayKey が追加されます。
クリックすると、左下にオーバーレイの詳細が表示されますね。
![](/images/overlay-viewer-created.png)

右側の灰色はオーバーレイの描画内容が表示されますが、今は何も描画していない状態です。

:::message
Overlay Viewer の実行ファイルは、Steam のインストールディレクトリに入っています。
デフォルトの場合は "C:\Program Files (x86)\Steam\steamapps\common\SteamVR\bin\win32\overlay_viewer.exe" です。
何度も起動することになるので、ショートカットを作成しておくと便利です。
:::

## コード整理
ここまでのコードを関数に分けつつ整理しておきます。

### オーバーレイの作成処理の整理
オーバーレイの作成は `CreateOverlay()` という関数に分けておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
       InitOpenVR();

-      var error = OpenVR.Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay", ref overlayHandle);
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

        var error = OpenVR.Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay", ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
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

## まとめ
- CreateOverlay() でオーバーレイを作成しました
- DestroyOverlay() でオーバーレイを破棄しました
- 作成したオーバーレイを Overlay Viewer で確認しました
- 作成したオーバーレイは olveryHandle で操作できます

次のページでは、オーバーレイに画像ファイルを表示してみます。