---
title: "ダッシュボードオーバーレイ（描画）"
free: false
---

Steam のダッシュボードに設定画面を作ります。
設定画面で左右のコントローラどちらに時計を表示するか選べるようにします。
![](/images/switch-hand.gif)

## DashboardOverlay の作成
`Scripts/DashboardOverlay.cs` を新規作成します。
![](/images/dashboard-overlay-file.png)

## ダッシュボードオーバーレイの作成
ダッシュボードオーバーレイの作成は [CreateDashboardOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateDashboardOverlay_System_String_System_String_System_UInt64__System_UInt64__) で行います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateDashboardOverlay) を参照）
下のコードを `DashboardOverlay.cs` へコピーしてください。

```cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class DashboardOverlay : MonoBehaviour
{
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
        }
    }
}
```

作成するとダッシュボードとサムネイルの 2 つが作成され、それぞれのオーバーレイハンドルが取得されます。
`dashboardHandle` が設定画面、`thumbnailHandle` がダッシュボードの下に表示されるアイコン用のオーバーレイです。

## シーンに設置
Hierarchy ビューで右クリック > Create Empty で空のゲームオブジェクトを作成します。
オブジェクトの名前を `DashboardOverlay` にして、`Scripts/DashboardOverlay.cs` をドラッグして追加します。

![](/images/create-dashboard-object.png)

## OpenVR の初期化、クリーンアップ
OpenVR の API を使用するためには、OpenVR が初期化されている必要があります。
今回は、`WatchOverlay.cs` で作成した OpenVR の初期化処理を、共有のユーティリティとして抜き出して、他のクラスからも呼び出せるようにします。

### ユーティリティクラスの作成
`Scripts/OpenVRUtil.cs` を作成します。
```cs:OpenVRUtil.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OpenVRUtil
{
    public static class System
    {
    }
}
```

ここでの namespace は名前衝突の回避と、わかりやすさのためにつけています。

### OpenVR の初期化処理を移動
`WatchOverlay.cs` から `InitOpenVR()` を `OpenVRUtil.cs` に移動します。
この時、他のクラスから使いやすくするため static メソッドとして追加しておきます。

```diff cs:WatchOverlay.cs
～省略～

private void OnDestroy()
{
    DestroyOverlay(overlayHandle);
    ShutdownOpenVR();
}

- private void InitOpenVR()
- {
-     if (OpenVR.System != null) return;
- 
-     var initError = EVRInitError.None;
-     OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
-     if (initError != EVRInitError.None)
-     {
-         throw new Exception("OpenVRの初期化に失敗しました: " + initError);
-     }
- }

private void ShutdownOpenVR()
{
    if (OpenVR.System != null)
    {
        OpenVR.Shutdown();
    }
}

～省略～
```

```diff cs:OpenVRUtil.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OpenVRUtil
{
    public static class System
    {
+       public static void InitOpenVR()
+       {
+           if (OpenVR.System != null) return;
+
+           var initError = EVRInitError.None;
+           OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
+           if (initError != EVRInitError.None)
+           {
+               throw new Exception("OpenVRの初期化に失敗しました: " + initError);
+           }
+       }
    }
}
```

### OpenVR のクリーンアップ処理を移動
同様に `ShutdownOpenVR()` も `static` メソッドとして移動します。


```diff cs:WatchOverlay.cs
～省略～

private void OnDestroy()
{
    DestroyOverlay(overlayHandle);
    ShutdownOpenVR();
}

- private void ShutdownOpenVR()
- {
-     if (OpenVR.System != null)
-     {
-         OpenVR.Shutdown();
-     }
- }

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

～省略～
```

```diff cs:OpenVRUtil.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OpenVRUtil
{
    public static class System
    {
        public static void InitOpenVR()
        {
            if (OpenVR.System != null) return;

            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
            }
        }
        
+       public static void ShutdownOpenVR()
+       {
+           if (OpenVR.System != null)
+           {
+               OpenVR.Shutdown();
+           }
+       }
    }
}

```

### オーバーレイ関連処理の移動
オーバーレイ関連の処理も `WatchOverlay.cs` から `OpenVRUtil.cs` へ移動します。
`CreateOverlay()` から `SetOverlayRenderTexture()` までの処理を移動します。

- `CreateOverlay()`
- `DestroyOverlay()`
- `ShowOverlay()`
- `SetOverlayFromFile()`
- `SetOverlaySize()`
- `SetOverlayTransformRelative()`
- `FlipOverlayVertical()`
- `SetOverlayRenderTexture()`

```diff cs:WatchOverlay.cs
private void OnDestroy()
{
    DestroyOverlay(overlayHandle);
    OpenVRUtil.System.ShutdownOpenVR();
}

- private ulong CreateOverlay(string key, string name)
- {
-     var handle = OpenVR.k_ulOverlayHandleInvalid;
-     var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
-     if (error != EVROverlayError.None)
-     {
-         throw new Exception("オーバーレイの作成に失敗しました: " + error);
-     }
- 
-     return handle;
- }
- 
- ～省略～
- 
- private void SetOverlayRenderTexture(RenderTexture renderTexture)
- {
-     var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
-     var texture = new Texture_t
-     {
-         eColorSpace = EColorSpace.Auto,
-         eType = ETextureType.DirectX,
-         handle = nativeTexturePtr
-     };
-     var error = OpenVR.Overlay.SetOverlayTexture(overlayHandle, ref texture);
-     if (error != EVROverlayError.None)
-     {
-         throw new Exception("テクスチャの描画に失敗しました: " + error);
-     }
- }
```

`static class Overlay` を作成して、全てのメソッドを `public static` メソッドとして追加します。

```diff cs:OpenVRUtil.cs
namespace OpenVRUtil
{
    public static class System
    {
        public static void InitOpenVR()
        {
            if (OpenVR.System != null) return;

            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
            }
        }
        
        public static void ShutdownOpenVR()
        {
            if (OpenVR.System != null)
            {
                OpenVR.Shutdown();
            }
        }
    }
    
+   public static class Overlay
+   {
+       public static ulong CreateOverlay(string key, string name)
+       {
+           var handle = OpenVR.k_ulOverlayHandleInvalid;
+           var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
+           if (error != EVROverlayError.None)
+           {
+               throw new Exception("オーバーレイの作成に失敗しました: " + error);
+           }
+
+           return handle;
+       }
+
+       ～省略～
+
+       public static void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
+       {
+           var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
+           var texture = new Texture_t
+           {
+               eColorSpace = EColorSpace.Auto,
+               eType = ETextureType.DirectX,
+               handle = nativeTexturePtr
+           };
+           var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
+           if (error != EVROverlayError.None)
+           {
+               throw new Exception("テクスチャの描画に失敗しました: " + error);
+           }
+       }
+   }
}
```

### 処理の呼び出し部分の変更
移動した処理を `OpenVRUtil` から呼び出すように `WatchOverlay.cs` を修正します。

```diff cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;
+ using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
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
-       InitOpenVR();
-       overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
+       OpenVRUtil.System.InitOpenVR();
+       overlayHandle = Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay");

-       FlipOverlayVertical(overlayHandle);
-       SetOverlaySize(overlayHandle, size);
-       ShowOverlay(overlayHandle);
+       Overlay.FlipOverlayVertical(overlayHandle);
+       Overlay.SetOverlaySize(overlayHandle, size);
+       Overlay.ShowOverlay(overlayHandle);
    }
    
    private void Update()
    {
        var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
        if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
        {
            var position = new Vector3(x, y, z);
            var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-           SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+           Overlay.SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
        }

-       SetOverlayRenderTexture(overlayHandle, renderTexture);
+       Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
    }

+   private void OnApplicationQuit()
+   {
+       Overlay.DestroyOverlay(overlayHandle);
+   }

    private void OnDestroy()
    {
-       DestroyOverlay(overlayHandle);
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

複数個所から `ShutdownOpenVR()` が呼び出されても良いように、`DestroyOverlay()` は、`OnDestory()` より先に実行される `OnApplicationQuit()` に移動します。
https://docs.unity3d.com/Manual/ExecutionOrder.html

## DashboardOverlay.cs へ初期化とクリーンアップを追加
`OpenVR.Overlay.CreateDashboardOverlay()` に OpenVR の初期化とクリーンアップ処理を追加します。


```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
+ using OpenVRUtil;

public class DashboardOverlay : MonoBehaviour
{
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
+       OpenVRUtil.System.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }
    }

+   private void OnDestroy()
+   {
+       OpenVRUtil.System.ShutdownOpenVR();
+   }
}
```

## オーバーレイのクリーンアップを追加する
`DashboardOverlay.cs` にダッシュボードとサムネイルのオーバーレイの破棄を追加します。

```diff cs:DashboardOverlay.cs
public class DashboardOverlay : MonoBehaviour
{
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }
    }

+   private void OnApplicationQuit()
+   {
+       Overlay.DestroyOverlay(dashboardHandle);
+       Overlay.DestroyOverlay(thumbnailHandle);
+   }

    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## サムネイルの表示
サムネイルオーバーレイに画像を表示します。
以前画像の表示に使用した `SetOverlayFromFile()` と SNS のアイコンがあるので、それを使用します。

```diff cs:DashboardOverlay.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();
    
    var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
    }

+   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
+   Overlay.SetOverlayFromFile(thumbnailHandle, filePath);
}
```

プログラムを実行して、ダッシュボードの下にサムネイルが表示されていることを確認します。

![](/images/dashboard-thumbnail.jpg)

ダッシュボードオーバーレイには、まだ何も描画していないので、選択しても表示されません。
サムネイルにレーザポインタを当てたときに表示されるテキスト（ここでは "Watch Setting"）は、オーバーレイの name に設定した文字列です。

## 設定画面の作成

### 既存のオブジェクトを分ける
Hierarchy で右クリック > Create Empty で空のゲームオブジェクトを作成します。
オブジェクト名を `Watch` に変更します。
既存の `WatchOverlay`, `Camera`, `Canvas` を `Watch` の中に移動させます。
![](/images/watch-group.png)

### ダッシュボード関連オブジェクトのグループ化
同様に Hierarchy の最上位に Create Empty で空のゲームオブジェクトを作成します。
オブジェクト名を `Dashboard` に変更します。
`Dashboardoverlay` を `Dashboard` の中に移動させます。
![](/images/dashboard-overlay-group.png)

### カメラの作成
`Dashboard` の下に Camera を新規作成します。

### Canvas の作成
`Dashboard` の下に UI > Canvas を新規作成します。
![](/images/dashboard-canvas-camera.png)

### カメラの設定
ダッシュボード用のカメラについて、
Camera の Clear Flags を Solid Color に設定します。
Camera の Background の色をクリックして、不透明な灰色 (RGBA = 64, 64, 64, 255) に設定します。
![](/images/gray-bg-color.png)

Camera の AudioListener コンポーネントを削除します。
![](/images/remove-dashboard-audio-listener.png)

### Render Texture の作成
ダッシュボード用の Render Texture を作成します。
`Assets/RenderTexture` フォルダ内に Render Texture アセットを新規作成します。
アセット名を `DashboardRenderTexture` に変更します。
![](/images/create-dashboard-rendertexture.png)

インペクタでサイズを 1024 x 768 に設定します。
![](/images/set-dashboard-rendertexture-size.png)

ダッシュボード用のカメラの Render Texture に、`DashboardRenderTexture` を設定します。
![](/images/attach-dashboard-rendertexture.png)

### Canvas の設定
ダッシュボード用の Canvas について
Canvas の Render Mode を Screen Space - Camera に設定します。
Render Camera に、先ほど作成した Camera をドラッグします。
Plane Distance を 10 に設定します。
![](/images/dashboard-canvas-setting.png)

### グループの移動
`Watch` と `Dashboard` が重なっているので、`Dashboard` の Position の X を 20 に変更します。
![](/images/shift-dashboard-group.png)

### ボタンの作成
Hierarchy で `Dashboard/Canvas` を右クリック > Button - TextMeshPro を追加します。
オブジェクト名を `LeftHandButton` に変更します。
Hierarchy でボタンの下の階層にある `Text (TMP)` をクリックして、インスペクタからボタンのテキストを「Left Hand」に変更します。
![](/images/left-hand-button.png)

`LeftHandButton` の Width = 700、Height = 200 に設定します。
![](/images/change-button-size.png)

ボタンのテキストの Font Size を 100 に設定します。
![](/images/change-button-font-size.png)

Hierarchy で `LeftHandButton` を右クリック > Duplicate でボタンを複製します。
オブジェクト名を `RightHandButton` に変更します。
先ほどと同様にテキストを「Right Hand」に変更します。
![](/images/right-hand-button-text.png)

`LeftHandButton` の Pos Y を 150 に、
`RightHandButton` の Pos Y を -150 にして、上下に並べます。
![](/images/buttons-layout.png)

これでボタンの作成は完了です。

## ダッシュボードに描画

### カメラの追加
`DashboardOverlay.cs` に Camera 変数を追加します。
```diff cs:Dashboardoverlay.cs
public class DashboardOverlay : MonoBehaviour
{
+   public Camera camera;
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);
    }

    ～省略～
```

Hierarchy で `DashboardOverlay` をクリックして、`Camera` 変数に `Dashboard/Camera` をドラッグします。
![](/images/add-dashboard-camera.png)

### レンダーテクスチャの関連付け
`DashboardOverlay.cs` でレンダーテクスチャの変数を作成して、カメラの映像が書き込まれるようにします。
また、`FlipOverlayVertical()` でテクスチャの上下を反転させておきます。
`SetOverlaySize()` でオーバーレイのサイズを 2.5 m に設定します。

```diff cs:DashboardOverlay.cs
public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
+   public RenderTexture renderTexture; 
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

+       Overlay.SetOverlaySize(dashboardHandle, 2.5f);        
+       Overlay.FlipOverlayVertical(dashboardHandle);
    }

    ～省略～
```

Hierarchy で `DashboardOverlay` を選択し、`Assets/RenderTextures/DashboardRenderTexture` を Render Texture へドラッグします。
![](/images/attach-dashboard-rendertexture-to-overlay.png)

### レンダーテクスチャをダッシュボードオーバーレイに描画
`Update()` を作成して、Render Texture をオーバーレイに描画します。
```diff cs:DashboardOverlay.cs
public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture; 
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);
        
        renderTexture = new RenderTexture(1024, 768, 16, RenderTextureFormat.ARGBFloat);
        camera.targetTexture = renderTexture;

        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
        Overlay.FlipOverlayVertical(dashboardHandle);
    }

+   void Update()
+   {
+       Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
+   }

    ～省略～
```

プログラムを実行して、ダッシュボードを開き、サムネイルをクリックしてください。
設定画面が描画されていれば OK です。
![](/images/dashboard-preview.jpg)

## 右手のコントローラに時刻を表示

### 変数の作成
`WatchOverlay.cs` に左右のどちらの手に表示するかを決める変数を作成します。
`EtrackedControllerRole.LeftHand`, `ETrackedControllerRole.RightHand` が定義されているので、今回は `ETrackedControllerRole` 型を使います。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
+   public ETrackedControllerRole targetHand = EtrackedControllerRole.LeftHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

### コントローラの取得を修正
`targetHand` のコントローラの Device Index を取得するように修正します。

```diff cs:WatchOverlay.cs
private void Update()
{
-   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);

-   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-       Overlay.SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+       Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
    }

    Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
}
```

`WatchOverlay` のインスペクタで、`Target Hand` を `Right Hand` に切り替えて、右手に時刻が表示されることを確認します。
![](/images/change-hand-inspector.png)

ただし、左手のコントローラを基準にして位置を設定しているため、右手に表示した場合は、手のひら側に表示されます。
![](/images/wrong-right-hand-position.jpg)

### 右手用の座標を設定
右手に切り替えたときに、ちょうどいい位置にオーバーレイが表示されるように位置を調整します。
既存の設定値をコピーして、右手用の位置を指定するための変数を作ります。

左手用は変数名に left を追加します。
左手と右手で size は同じとします。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;
-   [Range(-0.5f, 0.5f)] public float x;
-   [Range(-0.5f, 0.5f)] public float y;
-   [Range(-0.5f, 0.5f)] public float z;
-   [Range(0, 360)] public int rotationX;
-   [Range(0, 360)] public int rotationY;
-   [Range(0, 360)] public int rotationZ;

+   [Range(-0.5f, 0.5f)] public float leftX;
+   [Range(-0.5f, 0.5f)] public float leftY;
+   [Range(-0.5f, 0.5f)] public float leftZ;
+   [Range(0, 360)] public int leftRotationX;
+   [Range(0, 360)] public int leftRotationY;
+   [Range(0, 360)] public int leftRotationZ;

+   [Range(-0.5f, 0.5f)] public float rightX;
+   [Range(-0.5f, 0.5f)] public float rightY;
+   [Range(-0.5f, 0.5f)] public float rightZ;
+   [Range(0, 360)] public int rightRotationX;
+   [Range(0, 360)] public int rightRotationY;
+   [Range(0, 360)] public int rightRotationZ;

```

targetHand に合わせて、それぞれの位置をセットするように変更します。
```diff cs:WatchOverlay.cs
private void Update()
{
+   Vector3 position;
+   Quaternion rotation;
+       
+   if (targetHand == ETrackedControllerRole.LeftHand)
+   {
+       position = new Vector3(leftX, leftY, leftZ);
+       rotation = Quaternion.Euler(leftRotationX, leftRotationY, leftRotationZ);
+   }
+   else
+   {
+       position = new Vector3(rightX, rightY, rightZ);
+       rotation = Quaternion.Euler(rightRotationX, rightRotationY, rightRotationZ);
+   }
+       
    var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
    if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
-       var position = new Vector3(x, y, z);
-       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
    }

    Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
}
```

インスペクタで各座標と角度を入力します。
左手用はメモした値を入力し、右手用は前回と同じようにプログラムを実行しながら、インスペクタでちょうどいい値に設定してください。
```diff cs:WatchOverlay.cs
private void Update()
{
    var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
    if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
-       var position = new Vector3(x, y, z);
-       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
+       var position = new Vector3(rightX, rightY, rightZ);
+       var rotation = Quaternion.Euler(rightRotationX, rightRotationY, rightRotationZ);

        Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
    }

    Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
}

```

以下は右手に設定した値の例です。
```
右手用のパラメータ例
x = 0.04
y = 0.003
z = -0.107
rotationX = 24
rotationY = 258
rotationZ = 179
```

右手のちょうどいい位置に表示されていれば OK です。
![](/images/correct-right-hand.jpg)

次のページでは、引き続きイベントの処理を作成していきます。