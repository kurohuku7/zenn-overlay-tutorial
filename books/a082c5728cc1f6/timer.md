---
title: "オーバーレイに時刻を表示する"
free: false
---

オーバーレイに時刻をリアルタイムに表示してみます。
これまでは画像ファイルをオーバーレイに表示していましたが、カメラの映像をリアルタイムに表示できるようにしてみます。

## カメラ映像の準備

時刻表示を作る前に、適当な 3D シーンのカメラ映像をオーバーレイに表示してみます。

### シーンの準備
Unity のシーンに

- Camera
- 3D Object → Cube
- Light → Directional Light

を新規作成します。
カメラに Cube が映るように、それぞれを配置してください。
![](/images/cube-scene.png)

### Cube を回転させる
リアルタイムに動いているカメラ映像を作りたいので、Cube が回転するアニメーションを作ります。
新規に `Scripts/Rotate.cs` を作成します。

```cs:Rotate.cs
using UnityEngine;

public class Rotate : MonoBehaviour
{
    void Update()
    {
        transform.Rotate(0, 0.5f, 0);
    }
}
```

`Rotate.cs` を Cube オブジェクトに追加します。
![](/images/cube-rotate-component.png)

プログラムを実行して Cube が回転することを確認します。
![](/images/rotate-cube.gif)

### カメラの参照を追加

`WatchOverlay.cs` にカメラの変数を作成します。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
+   public Camera camera;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;
    [Range(-0.5f, 0.5f)] public float x;
    [Range(-0.5f, 0.5f)] public float y;
    [Range(-0.5f, 0.5f)] public float z;
    [Range(0, 360)] public int rotationX;
    [Range(0, 360)] public int rotationY;
    [Range(0, 360)] public int rotationZ;
...
```

追加した変数に、シーン上のカメラを設定します。
![](/images/attach-camera.png)

### 画像ファイルのコードを削除
画像ファイルは、もう使わないので処理を削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    
-   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
-   SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, size);

    ShowOverlay(overlayHandle);
}
```


### RenderTexture の作成
[RenderTexture](https://docs.unity3d.com/ja/2018.4/Manual/class-RenderTexture.html) を作成して、カメラの映像が書き込まれるようにします。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
+   private RenderTexture renderTexture;
    
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
    [Range(0, 0.5f)] public float size = 0.5f;
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

+       renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
+       camera.targetTexture = renderTexture;
        
        ShowOverlay(overlayHandle);
    }

    ～省略～
```

カメラの targetTexture に renderTexture を設定することで、カメラの映像が renderTexture へ書き込まれるようになります。

:::details Gamma Color Space に対応させる場合
チュートリアルの環境ではデフォルトで Linear Color Space になっていますが、
Gamma Color Space を使用する場合は、RenderTexture のカラー形式を RenderTextureFormat.ARGB32 にすると描画されます。
プロジェクトの Color Space に合わせて変更する場合は、下記のようなコードのになります。
```cs
var colorFormat = QualitySettings.activeColorSpace == ColorSpace.Linear ? RenderTextureFormat.ARGBFloat : RenderTextureFormat.ARGB32;
renderTexture = new RenderTexture(512, 512, 16, colorFormat);
```
:::

### ネイティブテクスチャの取得
OpenVR に渡すテクスチャは Unity の下のレイヤーで使われている DirectX や OpenGL のテクスチャになります。
Unity のテクスチャ型は、そのままでは使えないので、GetNativeTexturePtr() を使って DirectX や OpenGL のネイティブテクスチャのポインタを取得します。

Update() 内で [GetNativeTexturePtr()](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html) を呼び出します。

```diff cs:WatchOverlay.cs
～省略～

private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
    camera.targetTexture = renderTexture;
    
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

+   var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
}

～省略～
    
```

:::details レンダリングスレッドの同期について
[GetNativeTexturePtr()](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html) のドキュメントでは、GetNativeTexturePtr() は初期化時に一度だけ呼び出すことが推奨されています。
しかしオーバーレイへの描画処理は、レンダリングスレッドが同期した状態で実行しなければクラッシュすることがあるため、スレッドの同期のために敢えて Update() 内で GetNativeTexturePtr() を呼び出しています。
GetNativeTexturePtr() を呼び出した後に、続けてオーバーレイへの描画処理を書きます。
:::

### OpenVR のテクスチャを作成
OpenVR の Texture_t 型の変数を作成します。

```diff cs:WatchOverlay.cs
private void Update()
{
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

    var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
+   var texture = new Texture_t
+   {
+       eColorSpace = EColorSpace.Auto,
+       eType = ETextureType.DirectX
+       handle = nativeTexturePtr
+   };
}
```

オーバーレイに描画するテクスチャは、この Texture_t 型で渡します。
handle に、先ほど取得したネイティブテクスチャのポインタ nativeTexturePtr をセットします。

:::details DirectX 以外に対応させる場合
チュートリアルの環境では、デフォルトで DirectX が使用されています。
Unity が実行時に使用している API の種類は `SystemInfo.graphicsDeviceType` で判定できるので、DirectX 以外の API が使われている場合に対応させる場合は、下記のように書けます。
```cs
switch (SystemInfo.graphicsDeviceType)
{
    case GraphicsDeviceType.Direct3D11:
        texture.eType = ETextureType.DirectX;
        break;
    case GraphicsDeviceType.Direct3D12:
        texture.eType = ETextureType.DirectX12;
        break;
    case GraphicsDeviceType.OpenGLES2:
    case GraphicsDeviceType.OpenGLES3:
    case GraphicsDeviceType.OpenGLCore:
        texture.eType = ETextureType.OpenGL;
        break;
    case GraphicsDeviceType.Vulkan:
        texture.eType = ETextureType.Vulkan;
        break;
}
```

Project Settings > Player > Other Settings > Auto Graphics API for Windows のチェックを外して、Graphics APIs for Windows のリストの一番上に Direct3D11 以外の API を追加すると、その API を使った動作確認が可能です。
![](/images/graphics-api.png)
https://docs.unity3d.com/ja/2019.4/Manual/GraphicsAPIs.html
:::

### オーバーレイにテキスチャを書き込む
[SetOverlayTexture()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTexture_System_UInt64_Valve_VR_Texture_t__) で、オーバーレイにテクスチャを表示します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTexture) を参照）
先ほど作成した Texture_t 型のテクスチャを使います。

```diff cs:WatchOverlay.cs
private void Update()
{
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

    var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
    var texture = new Texture_t
    {
        eColorSpace = EColorSpace.Auto,
        eType = ETextureType.DirectX,
        handle = nativeTexturePtr
    };
+   var error = OpenVR.Overlay.SetOverlayTexture(overlayHandle, ref texture);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception($"テクスチャの描画に失敗しました({error})");
+   }
}
```

プログラムを実行して、カメラの映像がリアルタイムにオーバーレイに表示されることを確認してください。

![](/images/realtime-rendering.gif)

### 上下を反転させる
オーバーレイの下側に Unity の空が表示されていますね。
テクスチャの上下が反転しているので、正しい向きに直します。
逆さまになっているのは DirectX と OpenGL の UV 座標系の違いによるもので、DirectX が使用されていると反転します。

https://docs.unity3d.com/ja/current/Manual/SL-PlatformDifferences.html

[SetOverlayTextureBounds()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTextureBounds_System_UInt64_Valve_VR_VRTextureBounds_t__) で UV 座標系の V 軸（縦方向）を逆にすることで上下を反転できます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTextureBounds) を参照）
これはテクスチャのどの範囲を描画するか指定する関数ですが、向きを変えるためにも使用できます。
デフォルトでは vMin = 0, vMax = 1 ですが、縦方向を逆にするため vMin = 1, vMax = 0 を渡します。


```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

    renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
    camera.targetTexture = renderTexture;
    
+   var bounds = new VRTextureBounds_t
+   {
+       uMin = 0,
+       uMax = 1,
+       vMin = 1,
+       vMax = 0
+   };
+   var error = OpenVR.Overlay.SetOverlayTextureBounds(overlayHandle, ref bounds);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("テクスチャの反転に失敗しました: " + error);
+   }
    
    SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}
```

これで上下が反転します。オーバーレイの上側に Unity の空が表示されていますね。
![](/images/flip-y-axis.jpg)

:::details DirectX 以外に対応させる場合
このチュートリアルの動作環境では、デフォルトで DirectX が使われるため、上下を反転させています。
他の API が使われる環境に対応させる場合は、`graphicsDeviceType` を参照して、DirectX だったら反転させる処理をいれるといった方法に変えてください。
:::


## 時刻を表示する Canvas を作る
カメラの映像が表示できたので、次は時計を作成します。
Cube と Directional Light は使わないのでシーンから削除してください。

![](/images/only-overlay-camera.png)

シーンに下記のオブジェクトを追加します。
- UI > Canvas を作成
- Canvas の下に UI > Text - TextMeshPro を作成

![](/images/canvas-text-hierarchy.png)

TextMeshPro を作成するとダイアログが表示されるので、Import TMP Essentials をクリックします。完了したらダイアログを閉じます。
![](/images/import-tmp-essentials.png)

Canvas の Render mode を Screen Space - Camera にします。
Canvas の Render Camera にシーン上のカメラをドラッグしてください。
![](/images/set-canvas-camera.png)

Text (TMP) の Alignment でテキストの縦横の位置を中央に揃えます。
テキストに "00:00:00" を入力してください。
![](/images/text-alignment.png)

Camera の Clear Flags を Solid Color にします。
Camera の Background の色をクリックし、Alpha を 0 にして背景を透明にします。
これでカメラ映像の背景が透過されて、時刻だけが描画されるようになります。
![](/images/background-color.png)

Canvas の Plane Distance を 10 にします。大きさ調整のためです。
![](/images/plane-distance.png)

Text (TMP) を選択し、アンカーの設定（Rect Transform コンポーネント左上の四角形）をクリックして、右下の青い十字の矢印（上下左右ストレッチ）を選択します。
![](/images/change-anchor.png)

そのまま Left, Top, Right, Bottom の値を 0 にします。
これで Canvas 全体にテキストの領域が広がります。
![](/images/position-0.png)

Text (TMP) の Font Size を 70 にします。
![](/images/font-size.png)

プログラムを実行して、左手首にちょうどよく時刻が表示されることを確認してください。
必要に応じてオーバーレイの大きさや位置、フォントサイズなどを調整してください。
![](/images/show-time.jpg)

## 時計を動かす
`Scripts/Watch.cs` を新規作成します。
下記のコードをコピーしてください。

```cs:Watch.cs
using UnityEngine;
using System;
using TMPro;

public class Watch : MonoBehaviour
{
    private TextMeshProUGUI label;

    void Start()
    {
        label = GetComponent<TextMeshProUGUI>();
    }
    
    void Update()
    {
        var hour = DateTime.Now.Hour;
        var minute = DateTime.Now.Minute;
        var second = DateTime.Now.Second;
        label.text = $"{hour:00}:{minute:00}:{second:00}";
    }
}

```

Text (TMP) に Watch.cs を追加します。
![](/images/watch-to-text.png)


プログラムを実行して、現在の時刻が表示されていれば OK です。

![](/images/clock-check.jpg)

これで VR ゲームに持ち込める腕時計ができました。
時計の見た目は、通常の uGUI と同じように Canvas 上で編集できるので、デザインを変更してみてください。

## コードの整理

### オーバーレイの上下反転
`flipOverlayVertical()` として関数に分けておきます。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
    camera.targetTexture = renderTexture;
    
-   var bounds = new VRTextureBounds_t
-   {
-       uMin = 0,
-       uMax = 1,
-       vMin = 1,
-       vMax = 0
-   };
-   var error = OpenVR.Overlay.SetOverlayTextureBounds(overlayHandle, ref bounds);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("テクスチャの反転に失敗しました: " + error);
-   }
+   flipOverlayVertical(overlayHandle);    
    SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}

～省略～

+ private void flipOverlayVertical(ulong handle)
+ {
+    var bounds = new VRTextureBounds_t
+    {
+        uMin = 0,
+        uMax = 1,
+        vMin = 1,
+        vMax = 0
+    };
+
+    // overlayHandle -> handle に変数名を変更
+    var error = OpenVR.Overlay.SetOverlayTextureBounds(handle, ref bounds);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("テクスチャの反転に失敗しました: " + error);
+    }
+ }

```

### RenderTexture の描画
`SetOverlayRenderTexture()` として関数に分けておきます。
```diff cs:WatchOverlay.cs
private void Update()
{
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

-   var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
-   var texture = new Texture_t
-   {
-       eColorSpace = EColorSpace.Auto,
-       eType = ETextureType.DirectX,
-       handle = nativeTexturePtr
-   };
-   var error = OpenVR.Overlay.SetOverlayTexture(overlayHandle, ref texture);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("テクスチャの描画に失敗しました: " + error);
-   }
+   SetOverlayRenderTexture(overlayHandle, renderTexture);
}

～省略～

+ private void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
+ {
+     var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
+     var texture = new Texture_t
+     {
+         eColorSpace = EColorSpace.Auto,
+         eType = ETextureType.DirectX,
+         handle = nativeTexturePtr
+     };
+     var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("テクスチャの描画に失敗しました: " + error);
+     }
+ }

```

### 最終的なコード
```cs: WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    private RenderTexture renderTexture;

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

        renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
        camera.targetTexture = renderTexture;

        flipOverlayVertical(overlayHandle);
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

        SetOverlayRenderTexture(overlayHandle, renderTexture);
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

    private void flipOverlayVertical(ulong handle)
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
    
    private void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
    {
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
```
