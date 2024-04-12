---
title: "ダッシュボードオーバーレイ"
free: false
---

Steam のダッシュボードに設定画面を作ります。
設定画面で左右のコントローラどちらに時計を表示するか選べるようにします。

![](/images/dashboard-preview.jpg)

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
- `flipOverlayVertical()`
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
+       OpenVRUtil.System.InitOpenVR();
-       overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
+       overlayHandle = Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay");

-       flipOverlayVertical(overlayHandle);
+       Overlay.flipOverlayVertical(overlayHandle);
-       SetOverlaySize(overlayHandle, size);
+       Overlay.SetOverlaySize(overlayHandle, size);
-       ShowOverlay(overlayHandle);
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
また、`flipOverlayVertical()` でテクスチャの上下を反転させておきます。
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
+       Overlay.flipOverlayVertical(dashboardHandle);
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
        Overlay.flipOverlayVertical(dashboardHandle);
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

## ボタンのイベント作成
ボタンのクリックイベントで、どちらのコントローラに表示するかを切り替えられるようにします。
Scripts フォルダに `WatchSettingController.cs` を新規作成します。
Hierarchy を右クリックして Create Empty でからのオブジェクトを作成し、`SettingController` という名前に変更します。
作成した `SettingController` オブジェクトに `WatchSettingController.cs` を追加します。
![](/images/add-watch-controller.png)


`WatchSettingController` に `WatchOverlay` の変数を作成します。
```cs:WatchSettingController.cs
using UnityEngine;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
}
```

インスペクタから `WatchOverlay` コンポーネントの Watch Overlay にドラッグします。
![](/images/attach-watch-overlay.png)

`WatchSettingController.cs` に、ボタンが押されたときのイベントを定義します。
```diff cs:WatchSettingController.cs
using UnityEngine;
+ using Valve.VR;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
    
+   public void OnLeftHandButtonClick()
+   {
+       watchOverlay.targetHand = ETrackedControllerRole.LeftHand;
+   }
+
+   public void OnRightHandButtonClick()
+   {
+       watchOverlay.targetHand = ETrackedControllerRole.RightHand;
+   }
}
```

## ボタンにイベントを割り当て
Hierarchy の Dashboard > Canvas > LeftHandButton を選択します。
インスペクタで `Button` コンポーネントの `OnClick()` の + をクリックします。
`SettingController` を OnClick へドラッグして、`WatchSettingController.OnLeftHandButtonClick()` を設定します。
![](/images/attach-left-button-event.png)

同様に `RightHandButton` の `OnClick()` に `WatchSettingController.OnRightHandButtonClick()` を設定します。
![](/images/attach-right-hand-event.png)


## ダッシュボードのイベント取得
ダッシュボードでボタンがクリックされたイベントを取得します。
オーバーレイのイベントは `PollNextOverlayEvent()` を使ったポーリングで検出します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::PollNextOverlayEvent) を参照）

`Update()` 内で `PollNextOverlayEvent()` を使ってダッシュボードオーバーレイで発生したイベントを監視します。。
指定したオーバーレイでイベントが発生していれば、`PollNextOverlayEvent()` の戻り値が true になり、発生したイベントを一つ取り出すことができます。
全てのイベントを取り出すと、false が返ってきます。

```diff cs:DashboardOverlay.cs
void Update()
{
    Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);

+   var vrEvent = new VREvent_t();
+   var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
+
+   // 未処理のイベントが残っていれば true
+   while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
+   {
+       // vrEvent に取り出したイベントがセットされる
+   }
+
+   // 全てのイベントを取り出し終わったらループを抜ける
}
```

uncbVREvent は [VREvent_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VREvent_t.html) 構造体のサイズ（バイト数）です。


## クリックイベントの取得
レーザーポインタでダッシュボードがクリックされると `EVREventType.VREvent_MouseButtonDown` と `EVREventType.VREvent_MouseButtonUP` が発生します。
その他のイベントは [EVREventType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) に定義されています。

試しに `EVREventType.VREvent_MouseButtonDown` と `EVREventType.VREvent_MouseButtonUp` を取得してみます。

```diff cs
private void Update()
{
    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));

    while (OpenVR.Overlay.PollNextOverlayEvent(overlayHandle, ref vrEvent, uncbVREvent))
    {
+       switch (vrEvent.eventType)
+       {
+           case (uint)EVREventType.VREvent_MouseButtonDown:
+               Debug.Log("MouseDown");
+               break;
+
+           case (uint)EVREventType.VREvent_MouseButtonUp:
+               Debug.Log("MouseUp");
+               break;
+       }
    }
}
```

`vrEvent.eventType` は uint 型でイベントコードが入っているので、EVREventType を uint にキャストして比較しています。
プログラムを実行して、ダッシュボードのボタンをクリックすると、"MouseDown" と "MouseUp" がコンソールに出力されます。

![](/images/mouse-up-console.png)
![](/images/click-dashboard.jpg)
*オーバーレイのどこをクリックしてもイベントが発生する*

## クリック座標の取り出し
クリックされた座標をイベントから取得します。

```diff cs:DashboardOverlay.cs
private void Update()
{
    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));

    // 未処理のイベントが残っていれば true
    while (OpenVR.Overlay.PollNextOverlayEvent(overlayHandle, ref vrEvent, uncbVREvent))
    {
        // イベントが発生していれば、一つ取り出して vrEvent にセットする
        switch (vrEvent.eventType)
        {
            case (uint)EVREventType.VREvent_MouseButtonDown:
-               Debug.Log("MouseDown");            
+               Debug.Log($"MouseDown: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
                break;

            case (uint)EVREventType.VREvent_MouseButtonUp:
-               Debug.Log("MouseUp");
+               Debug.Log($"MouseUp: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
                break;
        }
    }

    // 全てのイベントを処理し終わった
}
```

vrEvent には VREvent_Mouse_T 型でクリックされたマウスの座標が渡されます。
座標はオーバーレイの UV 座標で返され、左上が (0, 0)、右下が (1, 1) となります。

プログラムを実行して、ダッシュボード開き、レーザポインタでオーバーレイをクリックしてください。
クリックされた座標がコンソールに表示されるはずです。
![](/images/click-uv-position.png)

## Mouse Scaling Factor の適用
受け取ったマウス座標はデフォルトで UV 座標になっていますが、Mouse Scaling Factor を設定すると、実際の UI の座標（ピクセル数）で受け取ることができます。
Mouse Scaling Factor は [SetOverlayMouseScale()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayMouseScale_System_UInt64_Valve_VR_HmdVector2_t__) で設定します。（詳細は Wiki を参照）

```diff cs:DashboardOverlay.cs
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

    Overlay.SetOverlaySize(dashboardHandle, 2.5f);
    Overlay.flipOverlayVertical(dashboardHandle);

+   var mouseScalingFactor = new HmdVector2_t()
+   {
+       v0 = renderTexture.width,
+       v1 = renderTexture.height
+   };
+   error = OpenVR.Overlay.SetOverlayMouseScale(dashboardHandle, ref mouseScalingFactor);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("Mouse Scaling Factor の設定に失敗しました: " + error);
+   }
}
```

`mouseScalingFacor` の `v0` が幅、`v1` が高さです。

プログラムを実行して、ダッシュボードをクリックしてください。
クリックされた座標が (0, 0) ～ (1024, 768) にスケーリングされた座標で取得できるようになります。
![](/images/scaled-mouse-position.png)


## Overlay Viewer でのイベント確認
ちなみに Overlay Viewer でもマウスイベントを発行できます。

プログラムの実行後、Overlay Viewer を起動して、左側の一覧から `WatchDashboardKey` を選択します。
右下の "Mouse Capture" にチェックを入れた状態で、右側のオーバーレイをクリックすると、イベントが発生して Unity のコンソールにログが表示されます。
![](/images/overlay-viewer-event.png)


HMD を被らずにイベントの動作確認をするときには Overlay Viewer が便利です。


## クリックされた要素を取得する
OpenVR のクリック座標が上下逆になるので反転させます。
vrEvent.data.mouse.y = dashboardTexture.height - vrEvent.data.mouse.y;

クリックされた座標がわかれば、あとはよく使われる Canvas の [Graphic Raycaster](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.UI.GraphicRaycaster.html) を使ってクリックされた UI 要素を取得できます。

GraphicRaycaster の変数を DashboardOverlay.cs に追加します

インスペクタから GraphicRaycaster に Canvas をドラッグして設定します。

EventSystem の変数を DashboardOverlay.cs につしかします。

インスペクタから EventSystem を設定します。


今回は Button のみを使った UI なので、Button クラスを探します。
```cs
// GraphicRaycaster で要素を取得する処理
```

## ボタンをクリックする処理
もしダッシュボードのクリック座標に Button があれば、その Button の OnClick() イベントを実行します。
左手と右手の切り替えボタンの OnClick() は既に実装済みなので、OnClick() を呼び出せば使用するコントローラの切り替えが動きます。

```cs
// 対応するボタンの OnClick() を呼び出す
```

プログラムを実行してダッシュボードを開き、ボタンをクリックすると、左右どちらの手に時計を表示するか設定できるように鳴っていることを確認します。


## コード整理
コードを綺麗にします
