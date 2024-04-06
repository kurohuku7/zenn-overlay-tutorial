---
title: "デバイスに追従させる"
free: false
---

## HMD に追従させる
![](/images/relative-transform-hmd.gif)
*HMD に追従するオーバーレイ*

### Device Index
どのデバイスに追従させるかは、接続されているデバイスを識別するための番号 (Device Index) で指定します。
HMD の場合は [OpenVR.k_unTrackedDeviceIndex_Hmd](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_k_unTrackedDeviceIndex_Hmd) の定数で定義されていて 0 で固定です。

:::details OpenVR 内のコードを確認
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L251-L256
:::

### 固定位置表示のコードを削除
前のページで作成した、空間内の固定位置への表示のコードは削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

-   var position = new Vector3(0, 2, 3);
-   var rotation = Quaternion.Euler(0, 0, 45);
-   SetOverlayTransformAbsolute(overlayHandle, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
} 
```

### HMD を基準とした相対位置を指定
HMD の正面 2m 先にオーバーレイを表示してみます。
相対位置の指定では [SetOverlayTransformTrackedDeviceRelative()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformTrackedDeviceRelative_System_UInt64_System_UInt32_Valve_VR_HmdMatrix34_t__) を使用します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformTrackedDeviceRelative) を参照）
基準となるデバイスの番号と、変換行列を渡します。


```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

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

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
} 
```

プログラムを実行して、HMD の正面に常にオーバーレイが表示されることを確認します。

### コード整理
`SetOverlayTransformRelative()` として関数に分けておきます。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

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

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
} 

～省略～

+ // overlayHandle -> handle に変数名を変更
+ // deviceIndex を引数で指定するように変更
+ private void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
+ {
+     var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+     var matrix = rigidTransform.ToHmdMatrix34();
+
+     // OpenVR.k_unTrackedDeviceIndex_Hmd -> deviceIndex に変更
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

HMD の代わりにコントローラの Devce Index を指定すれば、コントローラに追従するオーバーレイが作れます。

### HMD への追従を削除
HMD を基準とした位置指定を削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

-   var position = new Vector3(0, 0, 2);
-   var rotation = Quaternion.Euler(0, 0, 0);
-   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
}
```

### コントローラの Device Index の取得
コントローラの Device Index は [GetTrackedDeviceIndexForControllerRole()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRSystem.html#Valve_VR_CVRSystem_GetTrackedDeviceIndexForControllerRole_Valve_VR_ETrackedControllerRole_) で取得できます。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
}
```

引数は、
左手のコントローラなら `ETrackedControllerRole.LeftHand`
右手のコントローラなら `EtrackedControllerRole.RightHand`
です。

コントローラが接続されていない場合など、取得に失敗すると `k_unTrackedDeviceIndexInvalid` が返ってきます。

:::details OpenVR のヘッダファイルを確認
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L2344-L2345
https://github.com/ValveSoftware/openvr/blob/v2.5.1/headers/openvr.h#L272-L282
:::


### コントローラに追従させる
コントローラの Z 軸方向 2 m 先にオーバーレイを表示してみます。
先ほど作成した `SetOverlayTransformRelative()` を使います。
引数として HMD の代わりに、コントローラの Device Index を渡します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       var position = new Vector3(0, 0, 2);
+       var rotation = Quaternion.Euler(0, 0, 0);
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
}
```

**SteamVR 上で左手のコントローラが認識されていることを確認してから**、プログラムを実行してください。

![](/images/left-controller-connected.png)

HMD の代わりに、コントローラに追従してオーバーレイが表示されるようになっているはずです。

![](/images/controller-tracked-overlay.gif)
*コントローラに追従するオーバーレイ*

## 位置の調整

時計アプリを作るため、左手首にオーバーレイが来るように位置を調整します。
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

インスペクタにスライダーが表示されます。

![](/images/inspector-overlay-position.png)


### 大きさと位置を差し替え
大きさと位置指定のパラメータを、今追加した変数に差し替えます。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

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

-   SetOverlaySize(overlayHandle, 0.5f);
+   SetOverlaySize(overlayHandle, size);

    ShowOverlay(overlayHandle);
}
```

### Update() で大きさと位置を更新
Update() に大きさと位置の更新を追加して、実行しながら値の調整ができるようにします。
※ これは表示位置を調整するためのコードで、後で削除します。
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

プログラムを実行した状態で、インスペクタのスライダーを動かすと、リアルタイムに変更が反映されることを確認します。
![](/images/adjusting-overlay.jpg)

### 位置の調整
左手首の位置にオーバーレイが来るように、スライダーを調整してください。
VR 内で SteamVR のダッシュボードを開き、デスクトップ画面を表示して、VR 内から直接 Unity のスライダーを操作すると調整しやすいかと思います。
![](/images/steamvr-dashboard-unity-window.jpg)
![](/images/adjust-in-vr.gif)
*SteamVR ダッシュボードから Unity を操作*

他には SteamVR のメニューから "Display VR View" で HMD の映像を確認しながら、デスクトップで調整することもできます。
![](/images/display-overlay-view-menu.png)
![](/images/adjust-with-vrview.gif)
*HMD を被りたくないときはこれで*

調整できたら、**プログラムを終了させずに**各パラメータの数値をメモしてください。
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

プログラムを終了させた後、メモした値をインスペクタに再度入力します。
![](/images/set-overlay-position-inspector.png)

プログラムを実行して、調整した位置にオーバーレイが表示されていれば OK です。
![](/images/overlay-fixed-position.jpg)

### 調整用のコードを削除
パラメータを調整するために追加した `Update()` 内の処理を削除します。
```diff cs:WatchOverlay.cs
private void Update()
{
-   SetOverlaySize(overlayHandle, size);
-   
-   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
-   {
-       var position = new Vector3(x, y, z);
-       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
-       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
-   }
}
```

## コントローラが接続されていない場合の対処

現在はプログラムの起動時にコントローラが接続されている必要があります。
途中でコントローラが接続・切断される場合に対応するため、Device Index 取得、位置指定を Update() の中に移動します。

```diff cs:WatchOverlay.cs
    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

-       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-       if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
-       {
-           var position = new Vector3(x, y, z);
-           var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-           SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
-       }

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, size);

        ShowOverlay(overlayHandle);
    }

+   private void Update()
+   {
+       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+       if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+       {           
+           var position = new Vector3(x, y, z);
+           var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
+           SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+       }
+   }
```

これで途中でコントローラが接続・切断されても動作するようになります。

:::details デバイスが接続されたことを検出できる？
今回はシンプルに Update() 内で毎回取得していますが、[VREvent_TrackedDeviceRoleChanged](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) イベントを使ってデバイスが接続されたことを検出することも可能です。
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
        
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, size);

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
