---
title: "画像ファイルの表示"
free: false
---

### オーバーレイの表示状態の変更
オーバーレイはデフォルトで非表示状態になっているため、表示状態に切り替える必要があります。
表示状態の切り替えは ShowOverlay() と HideOverlay() で行います。

```cs
ShowOverlay(ulong ulOverlayHandle)
HideOverlay(ulong ulOverlayHandle)
```

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::ShowOverlay

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_ShowOverlay_System_UInt64_
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_HideOverlay_System_UInt64_

Start() にオーバーレイの表示を追加します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの表示に失敗しました: " + error);
+   }
}
```


### 画像ファイルの準備
オーバーレイに画像を描画してみましょう。
画像はなんでもいいですが、ここでは自分のアイコンを使ってみます。
![](https://storage.googleapis.com/zenn-user-upload/9ba58573d5a9-20240302.jpg =100x)

Unity で Assets の下に StreamingAssets フォルダを作ります。
用意した画像を StreamingAssets フォルダに追加します。
ここでは sns-icon.jpg という名前で追加しました。

### 画像の描画
オーバーレイに画像ファイルを描画するには [SetOverlayFromFile()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayFromFile_System_UInt64_System_String_) を使用します。関数の詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayFromFile) に載っています。

SetOverlayFromFile() を見てみます。

```cs
public EVROverlayError SetOverlayFromFile(ulong ulOverlayHandle, string pchFilePath)
```

ulOverlayHandle は CreateOerlay() で取得したハンドル、pchFilePath は画像ファイルパスです。
戻り値は CreateOverlay() と同様に EVROverlayError 型で、エラーがなければ EVROverlayError.None となります。

早速使ってみましょう。

```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var file = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
+   error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("画像ファイルの描画に失敗しました: " + error);
+   }

    var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイの表示に失敗しました: " + error);
    }
}
```

画像を [StreamingAssets](https://docs.unity3d.com/Manual/StreamingAssets.html) フォルダに入れたのは、[Application.streamingAssetsPath](https://docs.unity3d.com/ScriptReference/Application-streamingAssetsPath.html) で画像ファイルパスを取得するためです。

プログラムを実行して、Overlay Viewer で WatchOverlayKey のオーバーレイを確認しましょう。
正常に動作していれば、プレビュー部分に画像が表示されているはずです。
![](https://storage.googleapis.com/zenn-user-upload/c7cc3e4edf39-20240306.png)


そのまま HMD を装着して、足元を確認してください。
プレイエリアの中心にオーバーレイが表示されているはずです。オーバーレイは裏側から見ると透明になるので、見えない場合は反対側に回り込んでみてください。

![](/images/file-overlay-in-vr.jpg)


![](/images/overlay-in-game.jpg)
*ゲームの起動中でも動作する。画像は [Legendary Tales](https://store.steampowered.com/app/1465070/Legendary_Tales/)。*

VR 空間内へのオーバーレイの表示ができました。次は、オーバーレイの表示位置や大きさを変更してみます。
その前に、一旦コードを整理しておきましょう。

オーバーレイの表示や画像ファイルの描画を関数にして分けておきます。


```diff cs:WatchOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg")
+   SetOverlayFromFile(overlayHandle, filePath);
-   var file = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
-   var error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("画像ファイルの描画に失敗しました: " + error);
-   }

+   ShowOverlay(overlayHandle);
-   var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("オーバーレイの表示に失敗しました: " + error);
-   }
}

...

+ private void ShowOverlay(ulong handle)
+ {
+     var error = OpenVR.Overlay.ShowOverlay(handle);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの表示に失敗しました: " + error);
+     }
+ }

+ private void SetOverlayFromFile(ulong handle, string path)
+ {
+     var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("画像ファイルの描画に失敗しました: " + error);
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
        if (OpenVR.System != null)
        {
            return;
        }

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