---
title: "画像ファイルの表示"
free: false
---

オーバーレイに画像を描画してみましょう。

## 画像ファイルの準備
下記の条件の画像ファイルを準備してください。

- 大きさが 1920 x 1080 以内
- PNG, JPG, TGA 形式（24 or 32 ビットカラー）

ここではサンプルとして zenn のプロフィール画像を使ってみます。
![](/images/sns-icon.jpg =100x)

## 画像ファイルの設置
Unity で `Assets` フォルダの下に `StreamingAssets` フォルダを作ります。
用意した画像を `StreamingAssets` フォルダに追加します。
サンプルでは `sns-icon.jpg` という名前にしています。

![](/images/add-image-file.png)


## 画像の描画
画像ファイルの描画には [SetOverlayFromFile()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayFromFile_System_UInt64_System_String_) を使用します。詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayFromFile) を参照。

引数はオーバーレイハンドルと画像ファイルパスです。
エラーがなければ `EVROverlayError.None` が返ります。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
+   error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("画像ファイルの描画に失敗しました: " + error);
+   }
}
```

[StreamingAssets](https://docs.unity3d.com/Manual/StreamingAssets.html) を使用したのは、[Application.streamingAssetsPath](https://docs.unity3d.com/ScriptReference/Application-streamingAssetsPath.html) で画像ファイルパスを取得するためです。

プログラムを実行後、Overlay Viewer オーバーレイを確認します。
正常に動作していれば、で WatchOverlayKey に画像が描画されているはずです。
![](https://storage.googleapis.com/zenn-user-upload/c7cc3e4edf39-20240306.png)


## 表示状態の切り替え
Overlay Viewer には表示されていますが、VR 内には表示されていません。
オーバーレイはデフォルトで非表示になっているため、表示状態を切り替える必要があります。
表示状態の切り替えは [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::ShowOverlay) の通り [ShowOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_ShowOverlay_System_UInt64_) と [HideOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_HideOverlay_System_UInt64_) で行います。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
   var error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
   if (error != EVROverlayError.None)
   {
       throw new Exception("画像ファイルの描画に失敗しました: " + error);
   }

+   error = OpenVR.Overlay.ShowOverlay(overlayHandle);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの表示に失敗しました: " + error);
+   }
}
```

プログラムを実行したら HMD を装着して、足元を確認してください。
プレイエリアの中心にオーバーレイが表示されているはずです。
オーバーレイは裏側から見ると表示されないため、見つからない場合は反対側に回り込んでみてください。

![](/images/file-overlay-in-vr.jpg)


![](/images/overlay-in-game.jpg)
*ゲームの起動中でも動作する。画像は [Legendary Tales](https://store.steampowered.com/app/1465070/Legendary_Tales/)。*

## コードの整理
### 画像ファイルの描画

`SetOverlayFromFile()` として関数に分けておきます。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg")
+    SetOverlayFromFile(overlayHandle, filePath);
-    var error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
-    if (error != EVROverlayError.None)
-    {
-        throw new Exception("画像ファイルの描画に失敗しました: " + error);
-    }

    error = OpenVR.Overlay.ShowOverlay(overlayHandle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイの表示に失敗しました: " + error);
    }
}

～省略～

+ private void SetOverlayFromFile(ulong handle, string path)
+ {
+     var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("画像ファイルの描画に失敗しました: " + error);
+     }
+ }
```

### オーバーレイの表示
`ShowOverlay()` として関数を分けておきます。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg")
    SetOverlayFromFile(overlayHandle, filePath);

+   ShowOverlay(overlayHandle);
-   error = OpenVR.Overlay.ShowOverlay(overlayHandle);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("オーバーレイの表示に失敗しました: " + error);
-   }
}

～省略～

+ private void ShowOverlay(ulong handle)
+ {
+     var error = OpenVR.Overlay.ShowOverlay(handle);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの表示に失敗しました: " + error);
+     }
+ }

```

### 最終的なコード
```cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;

public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {        
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

        ShowOverlay(overlayHandle);
    }
    
    private void OnDestroy()
    {
        DestroyOverlay(overlayHandle);
        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System != null) return;

        var initError = EVRInitError.None;
        OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
        if (initError != EVRInitError.None)
        {
            throw new Exception("OpenVRの初期化に失敗しました: " + initError);
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
            OpenVR.Overlay.DestroyOverlay(handle);
        }
    }

    private void ShowOverlay(ulong handle)
    {
        var error = OpenVR.Overlay.ShowOverlay(handle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの表示に失敗しました: " + error);
        }
    }

    private void SetOverlayFromFile(ulong handle, string path)
    {
        var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
        if (error != EVROverlayError.None)
        {
            throw new Exception("画像ファイルの描画に失敗しました: " + error);
        }
    }
}
```