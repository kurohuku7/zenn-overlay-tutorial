---
title: "オーバーレイの大きさと位置"
free: false
---

オーバーレイの大きさと位置を変更してみます。

## オーバーレイの大きさ
オーバーレイの大きさは [SetOverlayWidthinMeters()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayWidthInMeters_System_UInt64_System_Single_) で設定できます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayWidthInMeters)）
引数はオーバーレイハンドルと、横幅（m 単位）です。高さは画像のアスペクト比に合わせて自動的に計算されます。
オーバーレイの横幅を 0.5 m に変更してみます。

```diff cs:WatchOverlay.cs
private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

+   var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);
}
```

実行すると、先程よりも小さくオーバーレイが表示されます。
![](/images/small-overlay.jpg)

## オーバーレイの表示位置
オーバーレイを VR 空間内の指定した位置に固定表示してみます。
SteamVR のプレイエリアの原点（床の中心）を基準とする [SetOverlayTransform()](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute) を使います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute)）


```cs
EVROverlayError SetOverlayTransformAbsolute(ulong ulOverlayHandle, ETrackingUniverseOrigin eTrackingOrigin, ref HmdMatrix34_t pmatTrackingOriginToOverlayTransform)
```

### ulOverlayHandle
オーバーレイのハンドルです。

### etrackingOrigin
[ETrackingUniverseOrigin](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.ETrackingUniverseOrigin.html) etrackingOrigin はトラッキングの基準になる原点です。
`ETrackingUniverseOrigin.TrackingStanding` だと SteamVR のプレイエリアの床の中心が原点となります。
`ETrackingUniverseOrigin.TrackingUniverseSeated` だとユーザが座った状態でリセットした HMD の位置が原点になります。

### pmatTrackingOriginOverlayTransform
`HmdMatrix34_t pmatTrackingOriginToOverlayTransform` が原点からの変形を表す変換行列です。

SteamVR Unity Plugin に position (Vector3) と rotation (Quarternion) から HmdMatrix34_t の変換行列を作るユーティリティが入っているので、今回はこちらを使います。

:::details 変換行列とは？
https://qiita.com/suzuryo3893/items/9e543cdf8bc64dc7002a

:::


### 固定位置に表示
プレイエリアの中心を基準として、正面方向に 3 m、上方向に 2 m にして、オーバーレイを Z 軸周りに 45 度回転させてみます。

※ 図を追加

:::details 左手系と右手系
Unity は左手系、OpenVR は右手系で座標系が異なります。
OpenVR では +y が上、+x が右、-z が前方となります。
Unity SteamVR Plugin のユーティリティが内部で座標系の変換を行うため、サンプルでは意識する必要がないですが、自前で変換行列を作成する場合は Z 軸の向きや回転の方向が逆になる点に注意が必要です。
:::


```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

+   // Y 軸方向に 2 m, Z 軸方向 3 m
+   var position = new Vector3(0, 2, 3);
+
+   // Z 軸周りに 45 度
+   var rotation = Quaternion.Euler(0, 0, 45);
+
+   // ユーティリティを使って変換行列を作成
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();
+
+   // プレイスペースの原点を基準にして変換行列を渡す
+   error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);
}
```

![](/images/far-overlay-position.jpg)
*床から高さ 2m、正面方向に 3m、Z 軸周りに 45 度回転*

## コード整理

### 大きさの変更
`SetOverlaySize()` として関数を分けておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

+       SetOverlaySize(overlayHandle, 0.5f);
-       var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
-       }
   
        // Y 軸方向に 2 m, Z 軸方向 3 m
        var position = new Vector3(0, 2, 3);

        // Z 軸周りに 45 度
        var rotation = Quaternion.Euler(0, 0, 45);

        // ユーティリティを使って変換行列を作成
        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
        var matrix = rigidTransform.ToHmdMatrix34();

        // プレイスペースの原点を基準にして変換行列を渡す
        error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
        }
    
        ShowOverlay(overlayHandle);
    }

    ～省略～

+   private void SetOverlaySize(ulong handle, float size)
+   {
+       var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+       }
+   }
}
```

### 位置の指定
`SetOverlayTransformAbsolute()` として関数に分けておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, 0.5f);

+       var position = new Vector3(0, 2, 3);
+       var rotation = Quaternion.Euler(0, 0, 45);
+       SetOverlayTransformAbsolute(overlayHandle, position, rotation);

-       // Y 軸方向に 2 m, Z 軸方向 3 m
-       var position = new Vector3(0, 2, 3);
-
-       // Z 軸周りに 45 度
-       var rotation = Quaternion.Euler(0, 0, 45);
-
-       // ユーティリティを使って変換行列を作成
-       var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-       var matrix = rigidTransform.ToHmdMatrix34();
-
-       // プレイスペースの原点を基準にして変換行列を渡す
-       error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-       }
        ShowOverlay(overlayHandle);
    }

    ～省略～

+   private void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
+   {
+       var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+       var matrix = rigidTransform.ToHmdMatrix34();
+       var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+       }
+   }
}
```

## 最終的なコード
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

        SetOverlaySize(overlayHandle, 0.5f);

        var position = new Vector3(0, 2, 3);
        var rotation = Quaternion.Euler(0, 0, 45);
        SetOverlayTransformAbsolute(overlayHandle, position, rotation);

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

    private void SetOverlaySize(ulong handle, float size)
    {
        var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
        }
    }

    private void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
    {
        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
        var matrix = rigidTransform.ToHmdMatrix34();
        var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
        }
    }
}
```
