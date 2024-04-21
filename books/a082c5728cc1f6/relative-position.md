---
title: "デバイスへの追従"
free: false
---

## HMD に追従させる
![](/images/relative-transform-hmd.gif)
*HMD に追従するオーバーレイ*

### 位置指定のコードを削除
まず、前のページで作成した、空間内の固定表示のコードは使わないので削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        SetOverlayFromFile(overlayHandle, filePath);
        
        SetOverlaySize(overlayHandle, 0.5f);

-       var position = new Vector3(0, 2, 3);
-       var rotation = Quaternion.Euler(0, 0, 45);
-       SetOverlayTransformAbsolute(overlayHandle, position, rotation);
        
        ShowOverlay(overlayHandle);

```

### Device Index
どのデバイスに追従させるかは、接続されているデバイスに割り振られる識別番号 **Device Index** で指定します。
HMD の場合は [OpenVR.k_unTrackedDeviceIndex_Hmd](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_k_unTrackedDeviceIndex_Hmd) として定義されていて 0 で固定です。

### 位置と回転の準備
今回は HMD の正面 2m 先にオーバーレイを表示してみます。
前回と同じように `position` と `rotation` を作成します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+   var position = new Vector3(0, 0, 2);
+   var rotation = Quaternion.Euler(0, 0, 0);

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);
    
    SetOverlaySize(overlayHandle, 0.5f);
    ShowOverlay(overlayHandle);
}
```

### 変換行列の作成
ユーティリティを使って変換行列を作成します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);
    
    SetOverlaySize(overlayHandle, 0.5f);
    ShowOverlay(overlayHandle);
}
```

### HMD を基準にした相対位置を指定
デバイスに追従させるためには [SetOverlayTransformTrackedDeviceRelative()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformTrackedDeviceRelative_System_UInt64_System_UInt32_Valve_VR_HmdMatrix34_t__) を使用します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformTrackedDeviceRelative) を参照）
指定したデバイスを基準とした相対的な位置に表示されるようになります。


```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
    var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
    var matrix = rigidTransform.ToHmdMatrix34();
+   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);
    ShowOverlay(overlayHandle);
} 
```

引数には基準となるデバイスの Device Index（ここでは `OpenVR.k_unTrackedDeviceIndex_Hmd`）と変換行列を渡します。
プログラムを実行して、HMD の正面 2m 先にオーバーレイが表示されれば OK です。
![](/images/relative-transform-hmd.gif)

### コード整理
相対位置の指定処理を `SetOverlayTransformRelative()` として関数に分けておきます。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
-   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-   var matrix = rigidTransform.ToHmdMatrix34();
-   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-   }
+   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);
    ShowOverlay(overlayHandle);
} 

～省略～

+ // deviceIndex を引数として指定する
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

## コントローラに追従させる
![](/images/controller-tracked-overlay.gif)
*コントローラに追従するオーバーレイ*

HMD の代わりにコントローラの Devce Index を指定すれば、コントローラに追従するオーバーレイが作れます。

### コントローラの Device Index の取得
コントローラの Device Index は `OpenVR.System` の [GetTrackedDeviceIndexForControllerRole()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRSystem.html#Valve_VR_CVRSystem_GetTrackedDeviceIndexForControllerRole_Valve_VR_ETrackedControllerRole_) で取得できます。
左手のコントローラの Device Index を取得するコードを追加します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
+   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
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

### コントローラに追従させる
左手のコントローラの取得に成功したら、コントローラにオーバーレイを追従させてみます。
先ほど作成した `SetOverlayTransformRelative()` を使い、HMD の代わりに、コントローラの Device Index を指定します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, 0.5f);

    ShowOverlay(overlayHandle);
}
```

SteamVR のウィンドウで**左手のコントローラが認識されていることを確認してから**、プログラムを実行してください。

![](/images/left-controller-connected.png)

HMD の代わりに、コントローラの 2m 先にオーバーレイが表示されるようになっているはずです。

![](/images/controller-tracked-overlay.gif)
*コントローラに追従するオーバーレイ*

## 位置の調整

時計アプリを作るため、左手首にオーバーレイが来るように調整します。
プログラムを実行しながら Unity 上でパラメータを調整できるようにしてみます。

### メンバ変数の作成
大きさ、位置、角度をクラスのメンバに追加します。
`Range()` 属性を使って、各メンバをインスペクタのスライダーで調整できるようにしておきます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

+   [Range(0, 0.5f)] public float size = 0.5f;
+   [Range(-0.2f, 0.2f)] public float x;
+   [Range(-0.2f, 0.2f)] public float y;
+   [Range(-0.2f, 0.2f)] public float z;
+   [Range(0, 360)] public int rotationX;
+   [Range(0, 360)] public int rotationY;
+   [Range(0, 360)] public int rotationZ;

    ～省略～
```

インスペクタにスライダーが表示されます。

![](/images/inspector-overlay-position.png)


### 変数の差し替え
大きさと位置指定のパラメータを、今追加したメンバ変数に差し替えます。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

-   var position = new Vector3(0, 0, 2);
-   var rotation = Quaternion.Euler(0, 0, 0);
+   var position = new Vector3(x, y, z);
+   var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

    SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

-   SetOverlaySize(overlayHandle, 0.5f);
+   SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}
```

### Update() で大きさと位置を更新
`Update()` に大きさと位置の更新処理を追加して、実行しながら値の調整ができるようにします。
※ このコードは表示位置を調整するためのもので、後で削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    var position = new Vector3(x, y, z);
    var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}

+ private void Update()
+ {
+     SetOverlaySize(overlayHandle, size);
+ 
+     var position = new Vector3(x, y, z);
+     var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
+     var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+     if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+     {
+         SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+     }
+ }

～省略～
```

プログラムを実行した状態で、インスペクタのスライダーを動かすと、リアルタイムに変更が反映されることを確認します。
![](/images/adjusting-overlay.jpg)

### 位置の調整
左手首の位置にオーバーレイが来るように、各スライダーを調整してください。
VR 内で SteamVR のダッシュボードを開き、デスクトップ画面を表示して、VR 内から直接 Unity のスライダーを操作すると調整しやすいかと思います。
![](/images/steamvr-dashboard-unity-window.jpg)
![](/images/adjust-in-vr.gif)
*SteamVR ダッシュボードから Unity を操作*

他には SteamVR のメニューから **Display VR View** で HMD の映像を確認しながら、デスクトップで調整することもできます。
![](/images/display-overlay-view-menu.png)
![](/images/adjust-with-vrview.gif)
*HMD を被りたくないときはこれで*

こちらが調整したパラメータの例です。
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

調整できたら、インスペクタで `WatchOverlay` コンポーネント名を右クリックして、**Copy Component** を選択します。
![](/images/copy-component.png)

プログラムを終了させた後、もう 1 度インスペクタから `WatchOverlay` コンポーネントを右クリックして、**Paste Component Value** で、コピーした値をペーストします。
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
-   var position = new Vector3(x, y, z);
-   var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
-   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
-   {
-       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
-   }
}
```

## コントローラが接続されていない場合の対処

現在は `Start()` でコントローラの Device Index を取得しているため、プログラムの起動時にコントローラが接続されている必要があります。
途中でコントローラが接続・切断される場合に対応するため、Device Index の取得と位置指定を `Start()` から `Update()` の中に移動します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

-   var position = new Vector3(x, y, z);
-   var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
-   {
-       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
-   }

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, size);

    ShowOverlay(overlayHandle);
}

+ private void Update()
+ {
+   var position = new Vector3(x, y, z);
+   var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
+   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }
+ }
```

これで途中でコントローラが接続・切断されても動作するようになります。


## 最終的なコード
```cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;
    [Range(-0.2f, 0.2f)] public float x;
    [Range(-0.2f, 0.2f)] public float y;
    [Range(-0.2f, 0.2f)] public float z;
    [Range(0, 360)] public int rotationX;
    [Range(0, 360)] public int rotationY;
    [Range(0, 360)] public int rotationZ;
    
    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        SetOverlayFromFile(overlayHandle, filePath);

        SetOverlaySize(overlayHandle, size);
        ShowOverlay(overlayHandle);
    }

    private void Update()
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
        if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
        {
            SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
        }
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

これでオーバーレイを手首に表示することができるようになりました。
次は画像ファイルではなく Unity のカメラ映像をオーバーレイに描画することで、時刻を表示できるようにします。