---
title: "デバイスに追従させる"
free: false
---

HMD やコントローラにオーバーレイをくっつけてみます。
まずは HMD に追従させてみます。

## HMD に追従させる
![](/images/relative-transform-hmd.gif)
*HMD に追従するオーバーレイ*

相対位置の指定では関数 [SetOverlayTransformTrackedDeviceRelative()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformTrackedDeviceRelative_System_UInt64_System_UInt32_Valve_VR_HmdMatrix34_t__) を使用します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformTrackedDeviceRelative)）

### Device Index
どのデバイスを基準にするかは、接続されているデバイスを識別するための番号 (Device Index) で指定します。
HMD の場合は [OpenVR.k_unTrackedDeviceIndex_Hmd](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_k_unTrackedDeviceIndex_Hmd) の定数で定義されていて 0 で固定です。

:::details OpenVR 内のコードを確認
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L251-L256
:::

### 固定位置表示を削除
前のページで作成した、VR 空間内の固定位置への表示は削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

-   var position = new Vector3(0, 2, 3);
-   var rotation = Quaternion.Euler(0, 0, 45);
-   SetOverlayTransformAbsolute(overlayHandle, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
} 
```

### HMD を基準とした相対位置を指定
HMD の正面 2m 先にオーバーレイを表示してみます。
基準となるデバイスの番号と、デバイスの位置を基準とした変換行列を渡します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

+   var position = new Vector3(0, 0, 2);
+   var rotation = Quaternion.Euler(0, 0, 0);
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();
+   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
} 
```

プログラムを実行して、HMD の正面に常にオーバーレイが表示されることを確認します。

### コード整理
`SetOverlayTransformRelative()` として相対位置指定を関数に分けておきます。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
+   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);
-   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-   var matrix = rigidTransform.ToHmdMatrix34();
-   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
} 

～省略～

+ private void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
+ {
+     var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+     var matrix = rigidTransform.ToHmdMatrix34();
+     var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(handle, deviceIndex, ref matrix);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+     }
+ }
```

## コントローラを基準にする
![](/images/controller-tracked-overlay.gif)
*コントローラに追従するオーバーレイ*

`SetOverlayTransformTrackedDeviceRelative()` に、コントローラの Devce Index を与えれば、コントローラを基準にした位置指定ができます。

### HMD の位置指定を削除
HMD を基準とした位置指定は削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

-   var position = new Vector3(0, 0, 2);
-   var rotation = Quaternion.Euler(0, 0, 0);
-   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
}
```

### Device Index の取得
コントローラの Device Index は [GetTrackedDeviceIndexForControllerRole()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRSystem.html#Valve_VR_CVRSystem_GetTrackedDeviceIndexForControllerRole_Valve_VR_ETrackedControllerRole_) で取得できます。
左手のコントローラを取得するなら `ETrackedControllerRole.LeftHand`、右手なら `EtrackedControllerRole.RightHand` を渡します。
コントローラが接続されていない場合など、取得に失敗すると `k_unTrackedDeviceIndexInvalid` が返ってきます。

:::details OpenVR のヘッダファイルを確認
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L2344-L2345
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L272-L282
:::

左手のコントローラの番号を取得するコードを追加します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

+   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
}
```


### コントローラに追従させる
コントローラの Z 軸方向 2 m 先にオーバーレイを表示してみます。
HMD の代わりに、コントローラのデバイス番号を渡します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       var position = new Vector3(0, 0, 2);
+       var rotation = Quaternion.Euler(0, 0, 0);
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
}
```

SteamVR 上で左手のコントローラが接続されていることを確認してから、プログラムを実行してください。
HMD の代わりに、コントローラに追従してオーバーレイが表示されるようになっているはずです。


## 位置の調整

時計アプリを作るため、左手首の位置にオーバーレイが来るように調整します。
プログラムを実行しながら Unity 上でパラメータを調整できるようにしてみます。

### メンバの作成
大きさ、位置、角度をクラスのメンバに追加します。
Range() 属性を使って、各メンバをインスペクタのスライダーで調整できるようにしておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

+   [Range(0, 0.5f)] public float size = 0.5f;
+   [Range(-0.5f, 0.5f)] public float x;
+   [Range(-0.5f, 0.5f)] public float y;
+   [Range(-0.5f, 0.5f)] public float z;
+   [Range(0, 360)] public int rotationX;
+   [Range(0, 360)] public int rotationY;
+   [Range(0, 360)] public int rotationZ;

    ～省略～
```
![](/images/inspector-overlay-position.png)


### 大きさと位置を差し替え
作成したメンバを使うように変更します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
-   SetOverlaySize(overlayHandle, 0.5f);
+   SetOverlaySize(overlayHandle, size);

    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
-       var position = new Vector3(0, 0, 2);
-       var rotation = Quaternion.Euler(0, 0, 0);
+       var position = new Vector3(x, y, z);
+       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

    SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    ShowOverlay(overlayHandle);
}
```

### Update() に処理を追加
現在は Start() で処理しているため、起動時に一度だけ大きさと位置が設定されます。
Update() でも処理するようにして、実行中に変更した値が反映されるようにします。
※ これはオーバーレイの表示位置を調整するためのもので、調整後に削除します。
```diff cs:WatchOverlay.cs
～省略～

+ private void Update()
+ {
+     SetOverlaySize(overlayHandle, size);
+ 
+     var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+     if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+     {
+         var position = new Vector3(x, y, z);
+         var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
+         SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+     }
+ }

～省略～
```

プログラムを実行した状態で、インスペクタのスライダーを動かすと、オーバーレイに反映されることを確認してください。

### 位置の調整
左手首の位置にオーバーレイが来るように、各メンバの値を調整してください。
VR 内で SteamVR のダッシュボードを開き、デスクトップ画面を表示して、VR 内から直接 Unity のスライダーを操作すると調整しやすいかと思います。
![](/images/steamvr-dashboard-unity-window.jpg)
![](/images/adjust-in-vr.gif)
*SteamVR ダッシュボードから Unity を操作*

他には SteamVR のメニューから Display VR View で HMD の映像を表示することもできます。
![](/images/display-overlay-view-menu.png)
![](/images/adjust-with-vrview.gif)
*HMD を被りたくないときはこれで*

調整できたら、**プログラムを終了させずに**インスペクタの各パラメータの数値をメモしておいてください。
```
パラメータの例
size = 0.08
x = -0.044
y = 0.015
z = -0.131
rotationX = 154
rotationY = 262
rotationZ = 0
```

プログラムを終了させたら、メモした値をインスペクタに入力します。
![](/images/set-overlay-position-inspector.png)

プログラムを実行して、左手首にオーバーレイが表示されていれば OK です。
![](/images/overlay-fixed-position.jpg)

### Update() の削除
パラメータを調整するために追加した `Update()` 内の処理を削除します。
```diff cs:WatchOverlay.cs
private void Update()
{
-     SetOverlaySize(overlayHandle, size);
- 
-     var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-     var position = new Vector3(x, y, z);
-     var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
-     SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
}
```

## コントローラが接続されていない場合の対処

現在は Start() でコントローラの Device Index を取得しているため、起動時にコントローラが接続されている必要があります。
実行中にコントローラが接続・切断される場合に備えて、Device Index 取得と位置指定の処理を Update() の中に移動しておきます。

```diff cs:WatchOverlay.cs
    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        SetOverlaySize(overlayHandle, size);

-       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-       var position = new Vector3(x, y, z);
-       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

-       ShowOverlay(overlayHandle);
    }

+   private void Update()
+   {
+       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+       var position = new Vector3(x, y, z);
+       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+       ShowOverlay(overlayHandle);
+   }
```

:::details 毎フレーム取得するのはどうなの？
今回はシンプルに Update() 内で毎回取得していますが、[VREvent_TrackedDeviceRoleChanged](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) イベントを使ってデバイスが接続されたタイミングを検出することも可能です。
:::

## 最終的なコード
```cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;

public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;
    [Range(-0.5f, 0.5f)] public float x;
    [Range(-0.5f, 0.5f)] public float y;
    [Range(-0.5f, 0.5f)] public float z;
    [Range(0, 360)] public int rotationX;
    [Range(0, 360)] public int rotationY;
    [Range(0, 360)] public int rotationZ;

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
        SetOverlaySize(overlayHandle, size);
        
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

        ShowOverlay(overlayHandle);
    }

    private void Update()
    {
        var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
        if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
        {
            var position = new Vector3(x, y, z);
            var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
            SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
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

    private void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
    {
        var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
        var matrix = rigidTransform.ToHmdMatrix34();
        var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(handle, deviceIndex, ref matrix);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
        }
    }
}
```
