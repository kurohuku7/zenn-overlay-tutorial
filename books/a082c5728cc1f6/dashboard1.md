---
title: "ダッシュボードオーバーレイの作成"
free: false
---

Steam のダッシュボードに表示できる設定画面を作ります。
設定画面で左右のコントローラどちらに時計を表示するか選べるようにします。
![](/images/switch-hand.gif)

## ダッシュボードオーバーレイの作成
SteamVR のダッシュボードに表示できるオーバーレイが、**ダッシュボードオーバーレイ**です。
新しいスクリプトを作成して、ダッシュボードの処理はそちらに書いていきます。

### スクリプトの作成
`Scripts` フォルダに `DashboardOverlay.cs` を新規作成して、下のコードをコピーしてください。
![](/images/dashboard-overlay-file.png)
```cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class DashboardOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

### スクリプトをシーンに設置
Hierarchy ビューで**右クリック > Create Empty** で空のゲームオブジェクトを作成します。
オブジェクトの名前を `DashboardOverlay` にして、今作成した`DashboardOverlay.cs` をドラッグして追加します。
![](/images/create-dashboard-object.png)


### オーバーレイハンドルの準備
ダッシュボードオーバーレイは、メインのオーバーレイと、サムネイルのオーバーレイの 2 つから構成されます。
サムネイルはダッシュボードの下に表示される小さなオーバーレイで、表示するダッシュボードオーバーレイの切り替えに使われます。
それぞれが別のオーバーレイとなっており、ハンドルも個別に持ちます。
![](/images/thumbnail-texture.png)
*赤線で囲われているのがサムネイルオーバーレイ
Right Hand というボタンが設置されている大きいものがメインのオーバーレイ*

2 つのオーバーレイのハンドルを入れる変数を準備します。
```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class DashboardOverlay : MonoBehaviour
{
+   private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
+   private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;
}
```

### ダッシュボードオーバーレイを作成
ダッシュボードオーバーレイの作成は [CreateDashboardOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateDashboardOverlay_System_String_System_String_System_UInt64__System_UInt64__) で行います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateDashboardOverlay) を参照）
オーバーレイの `key` と `name` は **"WatchDashboardKey"** と **"Watch Setting"** にします。

```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class DashboardOverlay : MonoBehaviour
{
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

+   private void Start()
+   {
+       var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
+       }
+   }
}
```

`CreateDashboardOverlay()` を実行すると、ダッシュボードとサムネイルの 2 つのオーバーレイがペアとして作成され、それぞれのハンドルが引数で渡した変数の参照にセットされます。

## ユーティリティクラスの作成
このまま実行したいところですが、OpenVR の API を使用するためには、OpenVR が初期化されている必要があります。
これまで作成していた `WatchOveraly.cs` で OpenVR の初期化を行っており、それより先に `DashboardOverlay.cs` から OpenVR API を呼び出しをしてしまうとエラーとなります。

色々な対処法がありますが、今回は `WatchOverlay.cs` に書かれている OpenVR の初期化処理を、共有のユーティリティとして抜き出して、他のクラスからも呼び出せるようにします。

### ファイルの作成
`Scripts` フォルダに `OpenVRUtil.cs` を新規作成して、下のコードをコピーします。
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

### OpenVR の初期化を移動
`WatchOverlay.cs` から `InitOpenVR()` を `OpenVRUtil.cs` に移動します。
この時、他のクラスから使いやすくするため `static` メソッドとして追加しておきます。

```diff cs:WatchOverlay.cs
～省略～

private void OnDestroy()
{
    DestroyOverlay(overlayHandle);
    ShutdownOpenVR();
}

- private void InitOpenVR()
- {
-   if (OpenVR.System != null) return;
-
-   var error = EVRInitError.None;
-   OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
-   if (error != EVRInitError.None)
-   {
-       throw new Exception("OpenVRの初期化に失敗しました: " + error);
-   }
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
+       // public static にします
+       public static void InitOpenVR()
+       {
+           if (OpenVR.System != null) return;
+
+           var error = EVRInitError.None;
+           OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
+           if (error != EVRInitError.None)
+           {
+               throw new Exception("OpenVRの初期化に失敗しました: " + error);
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
オーバーレイ関連の処理も、他のファイルから使うことになるので `WatchOverlay.cs` から `OpenVRUtil.cs` へ移動します。

`WatchOverlay.cs` の `CreateOverlay()` から `SetOverlayRenderTexture()` までの下記の処理をまとめて移動します。

- `CreateOverlay()`
- `DestroyOverlay()`
- `SetOverlayFromFile()`
- `ShowOverlay()`
- `SetOverlaySize()`
- `SetOverlayTransformAbsolute()`
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

`OpenVRUtil.cs` に `static class Overlay` を作成して、全てのメソッドを `private` → `public static` に変更した状態で追加します。

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

### 既存のコードの呼び出し部分を変更
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

    private void OnApplicationQuit()
    {
-       DestroyOverlay(overlayHandle);
+       Overlay.DestroyOverlay(overlayHandle);
    }

    private void OnDestroy()
    {
-       ShutdownOpenVR();
+       OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## OpenVR の初期化とクリーンアップを追加
ユーティリティクラスの作成と `WatchOverlay.cs` の修正が終わったので `DashboardOverlay.cs` に戻ります。

OpenVR の初期化とクリーンアップを呼び出します。
`WatchOverlay.cs` でも初期化とクリーンアップを実行していますが、既に初期化（または破棄）されている場合は、処理を実行しないように作っているため、複数のスクリプトから重複して呼び出しても問題ありません。
これにより、どちらのスクリプトが先に実行されても、OpenVR が初期化されている状態になります。今回はシンプルなツールなので、これで十分でしょう。

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
            throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
        }
    }

+   private void OnDestroy()
+   {
+       OpenVRUtil.System.ShutdownOpenVR();
+   }
}
```

## ダッシュボードオーバーレイの破棄
作成したダッシュボードオーバーレイを終了時に破棄します。

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
            throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
        }
    }

+   private void OnApplicationQuit()
+   {
+       Overlay.DestroyOverlay(dashboardHandle);
+   }

    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

注意点として、破棄に使用するのはダッシュボードのメインオーバーレイのハンドルのみです。
サムネイルのハンドルを渡すと `ThumbnailCantBeDestroyed` エラーが発生します。

## サムネイルの表示
サムネイルオーバーレイに画像を表示します。
[画像ファイルの表示](./image-file)で使用した `SetOverlayFromFile()` と画像ファイルがあるので、それを使用します。

```diff cs:DashboardOverlay.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();
    
    var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
    }

+   var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
+   Overlay.SetOverlayFromFile(thumbnailHandle, filePath);
}
```

プログラムを実行して、ダッシュボードの下にサムネイルが表示されていれば OK です。

![](/images/dashboard-thumbnail.jpg)

ダッシュボードオーバーレイには、まだ何も描画していないので、サムネイルを選択しても表示されません。
サムネイルにレーザポインタを当てると、`CreateDashboardOverlay()` に渡した `name`（ここでは "Watch Setting"）が表示されます。

## 設定画面の作成
ダッシュボードの画面を作っていきます。
まず、時計のオーバーレイ用のオブジェクトと、ダッシュボード用のオブジェクトを分けます。

### 時計用のオブジェクトをグループ化

Hierarchy で**右クリック > Create Empty** で空のゲームオブジェクトを作成します。
オブジェクト名を `Watch` に変更します。
既存の `WatchOverlay`, `Camera`, `Canvas` オブジェクトを `Watch` の中に移動させます。
![](/images/watch-group.png)

### ダッシュボード用のオブジェクトのグループ化
同様に Hierarchy で**右クリック > Create Empty** で空のゲームオブジェクトを作成します。
オブジェクト名を `Dashboard` に変更します。
`DashboardOverlay` オブジェクトを `Dashboard` の中に移動させます。
![](/images/dashboard-overlay-group.png)

### カメラと Canvas の作成
`Dashboard` オブジェクトを右クリックして、下記を新規作成します。

- Camera
- UI > Canvas

![](/images/dashboard-canvas-camera.png)

### カメラの設定
Dashboard の下の Camera を選択して、**Clear Flags** を **Solid Color** に設定します。
そのまま **Background** の色をクリックして、**不透明な灰色 (RGBA = 64, 64, 64, 255)** に設定します。
![](/images/gray-bg-color.png)

そのまま **AudioListener** コンポーネントを削除します。
![](/images/remove-dashboard-audio-listener.png)

### Render Texture の作成
ダッシュボード用の Render Texture を作成します。

Project ウィンドウで `Assets/RenderTexture` フォルダを**右クリック > Render Texture** でアセットを新規作成します。
アセット名を `DashboardRenderTexture` に変更します。
![](/images/create-dashboard-rendertexture.png)

インペクタでサイズを **1024 x 768** に設定します。
![](/images/set-dashboard-rendertexture-size.png)

Hierarchy で `Dashboard` の下の `Camera` を選択し `Target Texture` に `DashboardRenderTexture` アセットをドラッグします。
![](/images/attach-dashboard-render-texture-to-camera.png)


これでダッシュボード用のカメラ映像がレンダーテクスチャに書き込まれます。

### Canvas の設定
`Dashboard` の下の `Canvas` オブジェクトを選択します。
**Render Mode** を **Screen Space - Camera** に設定します。
**Render Camera** に、`Dashboard` の下の `Camera` オブジェクトをドラッグします。
**Plane Distance** を **10** に設定します。
![](/images/dashboard-canvas-setting.png)

### グループの移動
`Watch` と `Dashboard` が重なっているので、`Dashboard` の **Position** の **X を 20** に変更します。
![](/images/shift-dashboard-group.png)

### ボタンの作成
`Dashboard` の下の `Canvas` を**右クリック > UI > Button - TextMeshPro** でボタンを追加します。
オブジェクト名を `LeftHandButton` に変更します。
![](/images/add-left-hand-button.png)

`LeftHandButton` の下の階層にある `Text (TMP)` をクリックして、ボタンのテキストを **Left Hand** に変更します。
![](/images/left-hand-button.png)

`LeftHandButton` の **Width を 700**、**Height を 200** に設定します。
![](/images/change-button-size.png)

`LeftHandButton` の下の `Text (TMP)` の **Font Size** を **100** に設定します。
![](/images/change-button-font-size.png)

`LeftHandButton` を**右クリック > Duplicate** で複製します。
オブジェクト名を `RightHandButton` に変更します。
先ほどと同様にボタンのテキストを **Right Hand** に変更します。
![](/images/right-hand-button-text.png)

`LeftHandButton` の **Pos Y** を **150** に、
`RightHandButton` の **Pos Y** を **-150** にして、上下に並べます。
![](/images/buttons-layout.png)

これで設定画面の見た目は完成です。

## ダッシュボードに描画
作成した設定画面をダッシュボードオーバーレイとして表示します。

### カメラの追加
`DashboardOverlay.cs` に `Camera` のメンバ変数を作成します。
```diff cs:Dashboardoverlay.cs
public class DashboardOverlay : MonoBehaviour
{
+   public Camera camera;
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

Hierarchy で `DashboardOverlay` をクリックして、`Camera` 変数に `Dashboard` の下の `Camera` オブジェクトをドラッグします。
![](/images/add-dashboard-camera.png)

### レンダーテクスチャの関連付け
`DashboardOverlay.cs` でレンダーテクスチャの変数を作成します。
```diff cs:DashboardOverlay.cs
public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
+   public RenderTexture renderTexture; 
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

Hierarchy で `DashboardOverlay` を選択し、`Assets/RenderTextures/DashboardRenderTexture` を **Render Texture** へドラッグします。
![](/images/attach-dashboard-rendertexture-to-overlay.png)


### 反転と大きさの設定
事前に `FlipOverlayVertical()` でテクスチャの上下を反転させておきます。
また `SetOverlaySize()` でオーバーレイのサイズを **2.5 m** に設定します。

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
            throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
        }

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

+       Overlay.FlipOverlayVertical(dashboardHandle);
+       Overlay.SetOverlaySize(dashboardHandle, 2.5f);        
    }

    ～省略～
```


### レンダーテクスチャをダッシュボードオーバーレイに描画
`Update()` を作成して、ダッシュボードのカメラ映像が書き込まれているレンダーテクスチャを、オーバーレイに描画します。
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
            throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
        }

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);
        
        renderTexture = new RenderTexture(1024, 768, 16, RenderTextureFormat.ARGBFloat);
        camera.targetTexture = renderTexture;

        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
        Overlay.FlipOverlayVertical(dashboardHandle);
    }

+   private void Update()
+   {
+       Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
+   }

    ～省略～
```

プログラムを実行して、ダッシュボードを開き、サムネイルをクリックしてください。
設定画面が表示されていれば OK です。
![](/images/dashboard-preview.jpg)


## コード整理
ダッシュボードオーバーレイの作成を `CreateDashboardOverlay()` として関数に分けます。
戻り値は `dashboardHandle` と `thumbnailHandle` の 2 つ必要なので、タプルでまとめて返すようにします。

ユーティリティの `Overlay` クラス内へメソッドを追加します。
```diff cs:OpenVRUtil.cs
～省略～

public static ulong CreateOverlay(string key, string name)
{
    var handle = OpenVR.k_ulOverlayHandleInvalid;
    var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイの作成に失敗しました: " + error);
    }

    return handle;
}

+ public static (ulong, ulong) CreateDashboardOverlay(string key, string name)
+ {
+     ulong dashboardHandle = 0;
+     ulong thumbnailHandle = 0;
+     var error = OpenVR.Overlay.CreateDashboardOverlay(key, name, ref dashboardHandle, ref thumbnailHandle);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
+     }
+ 
+     return (dashboardHandle, thumbnailHandle);
+ }

～省略～
```

呼び出し側も変更します。
```diff cs:DashboardOverlay
private void Start()
{
    OpenVRUtil.System.InitOpenVR();

-   var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
-   }
+   (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

    Overlay.SetOverlaySize(dashboardHandle, 2.5f);
    Overlay.FlipOverlayVertical(dashboardHandle);
}
```

## 最終的なコード
```cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using OpenVRUtil;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();

        (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

        Overlay.FlipOverlayVertical(dashboardHandle);
        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
    }

    private void Update()
    {
        Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(dashboardHandle);
    }

    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

```cs:OpenVRUtil.cs
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

            var error = EVRInitError.None;
            OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
            if (error != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + error);
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

    public static class Overlay
    {
        public static ulong CreateOverlay(string key, string name)
        {
            var handle = OpenVR.k_ulOverlayHandleInvalid;
            var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの作成に失敗しました: " + error);
            }

            return handle;
        }

        public static (ulong, ulong) CreateDashboardOverlay(string key, string name)
        {
            ulong dashboardHandle = 0;
            ulong thumbnailHandle = 0;
            var error = OpenVR.Overlay.CreateDashboardOverlay(key, name, ref dashboardHandle, ref thumbnailHandle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("ダッシュボードオーバーレイの作成に失敗しました: " + error);
            }
 
            return (dashboardHandle, thumbnailHandle);
        }
        
        public static void DestroyOverlay(ulong handle)
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

        public static void SetOverlayFromFile(ulong handle, string path)
        {
            var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
            if (error != EVROverlayError.None)
            {
                throw new Exception("画像ファイルの描画に失敗しました: " + error);
            }
        }

        public static void ShowOverlay(ulong handle)
        {
            var error = OpenVR.Overlay.ShowOverlay(handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの表示に失敗しました: " + error);
            }
        }

        public static void SetOverlaySize(ulong handle, float size)
        {
            var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
            }
        }

        public static void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
        {
            var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
            var matrix = rigidTransform.ToHmdMatrix34();
            var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
            }
        }

        public static void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
        {
            var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
            var matrix = rigidTransform.ToHmdMatrix34();
            var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(handle, deviceIndex, ref matrix);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
            }
        }

        public static void FlipOverlayVertical(ulong handle)
        {
            var bounds = new VRTextureBounds_t
            {
                uMin = 0,
                uMax = 1,
                vMin = 1,
                vMax = 0
            };

            var error = OpenVR.Overlay.SetOverlayTextureBounds(handle, ref bounds);
            if (error != EVROverlayError.None)
            {
                throw new Exception("テクスチャの反転に失敗しました: " + error);
            }
        }

        public static void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
        {
            if (!renderTexture.IsCreated()) return;

            var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
            var texture = new Texture_t
            {
                eColorSpace = EColorSpace.Auto,
                eType = ETextureType.DirectX,
                handle = nativeTexturePtr
            };
            var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
            if (error != EVROverlayError.None)
            {
                throw new Exception("テクスチャの描画に失敗しました: " + error);
            }
        }
    }
}
```

これでダッシュボードオーバーレイを使って、設定画面を表示することができました。
次のページでは、引き続きダッシュボードの設定画面でボタンを押したときのイベントを作成していきます。