---
title: "大きさと位置の変更"
free: false
---


## オーバーレイのサイズ変更
オーバーレイの大きさは [SetOverlayWidthInMeters()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayWidthInMeters_System_UInt64_System_Single_) で設定します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayWidthInMeters) を参照）
m 単位で幅を指定します。デフォルトは 1 m です。高さは画像のアスペクト比に合わせて自動的に計算されます。
試しに 0.5 m に変更してみます。

```diff cs:WatchOverlay.cs
private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

+   var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);
}
```

プログラムを実行すると、先程の半分の大きさでオーバーレイが表示されます。
![](/images/small-overlay.jpg)

## オーバーレイの固定表示
オーバーレイを VR 空間内の指定した位置に表示してみます。
オーバーレイを空間内の特定の位置に固定表示する場合は [SetOverlayTransformAbsolute()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformAbsolute_System_UInt64_Valve_VR_ETrackingUniverseOrigin_Valve_VR_HmdMatrix34_t__) を使います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute) を参照）


### 位置と角度の準備
プレイエリアの原点（床の中央）を基準にして、Y 軸方向（上）に 2 m、Z 軸方向（正面）に 3m の位置に表示してみます。
また、オーバーレイを 45 度回転させてみます。

まず位置と角度を準備します。回転は `Quaternion` で作成します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

+   var position = new Vector3(0, 2, 3);
+   var rotation = Quaternion.Euler(0, 0, 45);
    
    ShowOverlay(overlayHandle);
}
```

### 変換行列の作成
オーバーレイの位置は、変換行列として指定します。プレイエリアの原点など、変形の基準となる座標と、指定した変換行列の積によって表示位置が決まります。変換行列は [HmdMatrix34_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.HmdMatrix34_t.html) 型を使用します。

SteamVR Plugin に `Vector3` の座標と `Quaternion` の回転から、変換行列を作成するユーティリティ `SteamVR_Utils.RigidTransform.ToHmdMatrix34()` が入っているので、今回はこちらを使用します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

    var position = new Vector3(0, 2, 3);
    var rotation = Quaternion.Euler(0, 0, 45);
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();

    ShowOverlay(overlayHandle);
}

```

### 位置の変更
作成した変換行列を `SetOverlayTransformAbsolute()` に渡して、位置を変更します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

    var position = new Vector3(0, 2, 3);
    var rotation = Quaternion.Euler(0, 0, 45);
    var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
    var matrix = rigidTransform.ToHmdMatrix34();
+   error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);

```

#### SetOverlayTransformAbsolute() の引数
第 1 引数は**オーバーレイハンドル**です。

第 2 引数の`ETrackingUniverseOrigin.TrackingUniverseStanding` は、**プレイスペースの床の中心を原点として表示位置を指定する**ことを表しています。
他には `ETrackingUniverseOrigin.TrackingUniverseSeated` を指定すると、座ってプレイするユーザ向けに、ユーザがリセットした位置が基準となります。

第 3 引数の `ref matrix` は**変換行列**の参照です。

## 動作確認
プログラムを実行して、オーバーレイの位置が変わっていれば OK です。

![](/images/far-overlay-position.jpg)
*プレイエリアの原点から上方向に 2m、正面方向に 3m、Z 軸周りに 45 度回転した状態*

:::details 左手系と右手系
座標系は Unity が左手系、OpenVR が右手系です。
OpenVR では +y が上、+x が右、-z が正面（奥）となります。
`SteamVR_Utils.RigidTransform` が内部で座標系の変換を行っているため、ここでは意識する必要がないですが、自前で変換行列を作成する場合は Z 軸の向きや、回転方向が逆になる点に注意してください。
:::

## コード整理

### 大きさの変更
`SetOverlaySize()` として関数を分けます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        SetOverlayFromFile(overlayHandle, filePath);

-       var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
-       }
+       SetOverlaySize(overlayHandle, 0.5f);

        var position = new Vector3(0, 2, 3);
        var rotation = Quaternion.Euler(0, 0, 45);
        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
        var matrix = rigidTransform.ToHmdMatrix34();
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
`SetOverlayTransformAbsolute()` として関数に分けます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    ～省略～

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, 0.5f);

        var position = new Vector3(0, 2, 3);
        var rotation = Quaternion.Euler(0, 0, 45);
-       var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-       var matrix = rigidTransform.ToHmdMatrix34();
-       error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-       }
+       SetOverlayTransformAbsolute(overlayHandle, position, rotation);

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

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, 0.5f);

        var position = new Vector3(0, 2, 3);
        var rotation = Quaternion.Euler(0, 0, 45);
        SetOverlayTransformAbsolute(overlayHandle, position, rotation);

        ShowOverlay(overlayHandle);
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

    private void SetOverlayFromFile(ulong handle, string path)
    {
        var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
        if (error != EVROverlayError.None)
        {
            throw new Exception("画像ファイルの描画に失敗しました: " + error);
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

これでオーバーレイの大きさと位置を変更できるようになりました。
しかし、今回作りたいのは腕時計のオーバーレイアプリなので、コントローラにオーバーレイが張り付くような位置指定が必要になります。
次のページでは、オーバーレイを HMD やコントローラなどのデバイスを追従するようにします。