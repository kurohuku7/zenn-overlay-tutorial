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

    private void OnDestroy()
    {
-       DestroyOverlay(overlayHandle);
+       Overlay.DestroyOverlay(overlayHandle);
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

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

    private void OnDestroy()
    {
+       Overlay.DestroyOverlay(dashboardHandle);
+       Overlay.DestroyOverlay(thumbnailHandle);
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

## 左右コントローラの切り替え

### 変数の作成
`WatchOverlay.cs` に左右のどちらの手に表示するかを決める変数を作成します。
`EtrackedControllerRole.LeftHand`, `ETrackedControllerRole.RightHand` が定義されているので、今回は `ETrackedControllerRole` 型を使います。
```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;
+   private ETrackedControllerRole targetHand = EtrackedControllerRole.LeftHand;

    ～省略～
```

### 左右の切り替え
GetTrackedDeviceIndexForControllerRole() で、設定されている値に応じて、左右のコントローラのどちらを取得するか変更する。
```cs
var targetHand = targetHand; // 保存された変数に応じて、左右どちらのコントローラに表示するかを切り替える
   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targethand);
   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
   {
       var position = new Vector3(x, y, z);
       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
   }
```

targetHand の初期値を `ETrackedControllerRole.RightHand` に変更してみて、右手にも表示できることを確認します。
```diff cs
-private ETrackedControllerRole targetHand = EtrackedControllerRole.LeftHand;
+private ETrackedControllerRole targetHand = EtrackedControllerRole.RightHand;
```

確認できたら、初期値を左手に戻しておきます。

### ボタンとのつなぎ込み
「左手に表示」ボタンを押したら左手に、「右手に表示」ボタンを押したら右手に表示されるようにしています。

Scripts フォルダに WatchSettingController.cs を新規作成します。

WatchSettingController に WatchOverlay の参照をもたせます。
インスペクタから WatchOverlay コンポーネントをドラッグして参照をセットします。
```cs:WatchSettingController.cs
[SerializeField] private WatchOverlay watchOverlay;
```

WatchSettingController に、ボタンが押されたときのイベントを定義します。
```cs:WatchSettingController.cs
public OnLeftHandButtonClick()
{
  watchOverlay.targetHand = EtrackedControllerRole.LeftHand;
}

public OnRightHandButtonClick()
{
  watchOverlay.targetHand = EtrackedControllerRole.RightHand;
}
```

### UnityEditor 上で動作確認
Unity Editor の Play Mode で、マウスでボタンをクリックすると、左右のコントローラのどちらに時計を表示するか切り替えられるようになっていることを確認します。’


### Dashboard Overlay の作成
Dashboard Overlay の表示には `CreateDashboardOverlay()` を使用します。
`CreateDashboardOverlay()` でダッシュボードオーバーレイを作成すると、`CreateOverlay()` とは異なり 2 枚のオーバーレイのハンドルがセットされます。

片方はダッシュボードのコンテンツを表示するオーバーレイ、もう片方がダッシュボードの下に表示されるサムネイルのオーバーレイです。
基本的に使い方は通常の Overlay と同様ですが、表示位置はダッシュボードに固定になるため、位置の指定は行いません。

DashboardOverlay.cs の Start() で、CreateDashboardOverlay() を実行して、ダッシュボードとサムネイルのハンドルを取得します。

### Dashboard Overlay の描画

先ほど作成した設定画面の Render Texture を Dashboard Overlay に描画します。
前のページの時計の表示と同様に、RenderTexture から DirectX のテクスチャのポインタを取得して、`SetOverlayTexturePtr()` でオーバーレイに書き込みます。

プログラムを実行後、SteamVR のダッシュボードを VR 内で開き、作成したダッシュボードが表示されていれば OK です。

## ダッシュボードのイベント取得

SteamVR のダッシュボードには、コントローラから伸びているレーザポインタで UI を操作する機能があります。
設定画面のボタンをレーザーポインタでクリックして、操作できるようにしてみます。

ダッシュボードのイベント取得には `PollNextOverlayEvent()` を使用します。（[Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::PollNextOverlayEvent)）
Update() 内に PollNextOverlayEvent() を追加します。

OpenVR のイベントが発生していれば、PollNextOverlayEvent() を実行した時に true が返ってきて、発生したイベントを一つ取り出すことができます。
全てのイベントを取り出すと、false が返ってくるようになるので、false が返ってくるまでイベントを取り出して、順次処理していきます。

```cs
private void Update()
{
    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));

    // 未処理のイベントが残っていれば true
    while (OpenVR.Overlay.PollNextOverlayEvent(overlayHandle, ref vrEvent, uncbVREvent))
    {
      // イベントが発生していれば、一つ取り出して vrEvent にセットする
    }

    // 全てのイベントを処理し終わった
}
```

## クリックイベントの取得

レーザーポインタでダッシュボードがクリックされたときは `EVREventType.VREvent_MouseButtonDown` と `EVREventType.VREvent_MouseButtonUP` が発生します。
その他のイベントも [EVREventType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) に定義されています。

```diff cs
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
                inputConverter.OnMouseDown(vrEvent.data.mouse);
                break;

            case (uint)EVREventType.VREvent_MouseButtonUp:
                var mouse = vrEvent.data.mouse;
                inputConverter.OnMouseUp(vrEvent.data.mouse);
                break;
        }
    }

    // 全てのイベントを処理し終わった
}
```

## クリック座標の取り出し
vrEvent には VREvent_Mouse_T 型でクリックされたマウスの座標が渡されます。
```diff cs
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
+               Debug.Log($"ButtonDown: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})")
                break;

            case (uint)EVREventType.VREvent_MouseButtonUp:
+               Debug.Log($"ButtonUp: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})")
                break;
        }
    }

    // 全てのイベントを処理し終わった
}
```

## クリックされた要素を取得する
クリックされた座標を下に、uGUI のどの要素がクリックされたのかを特定します。
Graphic Raycaster を使って、カメラから Canvas にレイを飛ばし、レイがぶつかった UI 要素を取得します。

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
