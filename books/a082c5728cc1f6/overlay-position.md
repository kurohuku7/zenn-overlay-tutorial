---
title: "オーバーレイの表示位置と大きさ"
free: false
---

オーバーレイは VR 空間のどこにどのくらいの大きさで表示するかを指定できます。


## オーバーレイの大きさ
まずはオーバーレイの大きさを変更してみます。
大きさは SetOverlayWidthinMeters() で設定できます。
オーバーレイの幅を m 単位で指定します。高さは表示する画像のアスペクト比に合わせて自動的に計算されます。
今回は正方形の画像をサンプルで使用しているため、縦横同サイズとなります。
とりあえず幅 1m に設定してみましょう。

Wiki はこちら
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayWidthInMeters

SteamVR Plugin はこちら
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayWidthInMeters_System_UInt64_System_Single_


オーバーレイの横幅を 0.5 m にしてみます。

```diff cs:FileOverlay.cs
private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

+    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+    }

    ShowOverlay(overlayHandle);
}
```

先程よりも小さくオーバーレイが表示されます。（デフォルトでは 1m で表示されます。）
![](/images/small-overlay.jpg)


## オーバーレイの表示位置
次にオーバーレイの表示位置を設定します。
表示位置は SteamVR のプレイエリアの原点（床の中心）を基準とする SetOverlayTransform() と、HMD やコントローラの位置を基準とする SetOverlayTransformTrackedDeviceRelative() があります。
TODO: SetOverlayTransformTrackedDeviceComponent もある

空間の特定の位置に固定するか、コントローラや HMD に追従させるかによって使い分けます。
最初は絶対座標で、空間の特定の場所に固定で表示してみましょう。

### 絶対位置で指定する

位置の指定は 3DCG では同じもの変換行列によって行われます。
引数には座標を変換するための行列 HmdMatrix を与えます。

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute

SteamVR Plugin
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute

```cs
EVROverlayError SetOverlayTransformAbsolute(ulong ulOverlayHandle, ETrackingUniverseOrigin eTrackingOrigin, ref HmdMatrix34_t pmatTrackingOriginToOverlayTransform)
```

`ulOverlayHandle` は CreateOverlay() で作成したオーバーレイのハンドルです。

`ETrackingUniverseOrigin etrackingOrigin` はトラッキングの基準になる原点です。
`ETrackingUniverseOrigin.TrackingStanding` が SteamVR のプレイエリアの床の中心が原点となります。今回はこちらを使います。
`ETrackingUniverseOrigin.TrackingUniverseSeated` はユーザが座った状態でポジションをリセットした HMD の位置が原点になります。

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.ETrackingUniverseOrigin.html


`HmdMatrix34_t pmatTrackingOriginToOverlayTransform` が変換行列です。
直接行列の各要素を指定して作成することもできますが、SteamVR Unity Plugin に position (Vector3) と rotation (Quarternion) から HmdMatrix34_t の変換行列を作るユーティリティが入っているので、今回はこちらを使います。

オーバーレイの表示位置を、プレイエリアの中心から正面方向に 3 m、上方向に 2 m にして、オーバーレイを Z 軸周りに 45 度回転させてみます。

```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

+    var position = new Vector3(0, 2, 3);
+    var rotation = Quaternion.Euler(0, 0, 45);
+    var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+    var matrix = rigidTransform.ToHmdMatrix34();
+    error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+    }

    ShowOverlay(overlayHandle);
}
```

![](/images/far-overlay-position.jpg)
*床から高さ 2m、正面方向に 3m、Z 軸周りに 45 度回転*

位置指定処理を関数に分けて整理しておきます。

```diff cs:FileOverlay.cs
public class FileOverlay : MonoBehaviour
{
    // ...中略...

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
+        SetOverlaySize(overlayHandle, 0.5f);
    
+        var position = new Vector3(0, 2, 3);
+        var rotation = Quaternion.Euler(0, 0, 45);
+        SetOverlayTransformAbsolute(overlayHandle, position, rotation);
    
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);
        
-        var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
-        if (error != EVROverlayError.None)
-        {
-            throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
-        }
-    
-        var position = new Vector3(0, 2, 3);
-        var rotation = Quaternion.Euler(0, 0, 45);
-        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-        var matrix = rigidTransform.ToHmdMatrix34();
-        error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
-        if (error != EVROverlayError.None)
-        {
-            throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-        }
    
        ShowOverlay(overlayHandle);
    }

    // ...中略...

+   private void SetOverlaySize(ulong handle, float size)
+   {
+       var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+       }
+   }
+   
+    private void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
+    {
+        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+        var matrix = rigidTransform.ToHmdMatrix34();
+        var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+        if (error != EVROverlayError.None)
+        {
+            throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+        }
+    }
}
```


### デバイスからの相対位置で指定する

#### HMD を基準にする
HMD やコントローラなどのデバイスを基準とした相対的な位置指定ができます。
常にユーザの正面に表示したり、コントローラにくっつける場合はこちらを使います。

相対位置の指定では関数 SetOverlayTransformTrackedDeviceRelative() を使用します。

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformTrackedDeviceRelative

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformTrackedDeviceRelative_System_UInt64_System_UInt32_Valve_VR_HmdMatrix34_t__

どのデバイスを基準にするかは、デバイスを識別するための番号で指定します。
HMD の場合は `OpenVR.k_unTrackedDeviceIndex_Hmd` の定数で定義されていて 0 で固定です。



leftIndex = system.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
rightIndex = system.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.RightHand);


#### コントローラを基準にする

