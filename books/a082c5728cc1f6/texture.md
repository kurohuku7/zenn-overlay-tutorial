---
title: "カメラ映像の表示"
free: false
---

## カメラ映像の準備

時刻表示を作る前に、まずは簡単な 3D シーンのカメラ映像をオーバーレイに表示してみます。

### シーンの準備
Unity の Hierarchy を右クリックして

- Camera
- 3D Object > Cube
- Light > Directional Light

を新規作成します。

![](/images/3d-scene-hierarchy.png)

カメラに Cube が映るように、それぞれのオブジェクトを配置してください。細かい部分は適当で大丈夫です。
![](/images/3d-scene-layout.png)

### Cube を回転させる
リアルタイムに動いている映像を作りたいので、Cube が回転するスクリプトを作ります。
`Scripts` フォルダに `Rotate.cs` を新規作成して、下のコードをコピーしてください。

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
![](/images/add-rotate.png)

プログラムを実行して Cube が回転することを確認します。
![](/images/rotate-cube.gif)

これでシーンの準備は完了です。

### カメラの参照を追加

`WatchOverlay.cs` にカメラのメンバ変数を追加します。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
+   public Camera camera;
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;
    [Range(-0.2f, 0.2f)] public float x;
    [Range(-0.2f, 0.2f)] public float y;
    [Range(-0.2f, 0.2f)] public float z;
    [Range(0, 360)] public int rotationX;
    [Range(0, 360)] public int rotationY;
    [Range(0, 360)] public int rotationZ;

    ～省略～
```

Unity のインスペクタで、`WatchOverlay` の `Camera` に、先ほど作成したカメラを設定します。
![](/images/attach-camera.png)

### 画像ファイルのコードを削除
カメラの映像を表示するため、画像ファイルの表示は削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    
-   var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
-   SetOverlayFromFile(overlayHandle, filePath);

    SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}
```

### RenderTexture の作成
Unity の Project ウィンドウで `Assets` の下に `RenderTextures` フォルダを作成します。
作成した `RenderTextures` フォルダを **右クリック > Create > RenderTexture** で Render Texture アセットを作成します。
![](/images/rendertexture-menu.png)

アセット名を `WatchRenderTexture` に変更します。
![](/images/watch-render-texture-asset.png)

### カメラ映像を RenderTexture に書き出す
Hierarchy でシーン上の `Camera` をクリックして、インスペクタの **Target Texture** に、今作った `WatchRenderTexture` アセットをドラッグします。
![](/images/attach-rendertexture-to-camera.png)
これでカメラの映像が `WatchRenderTexture` に書き込まれるようになります。

### RenderTexture の設定
Project ウィンドウで `WatchRenderTexture` をクリックしてインスペクタを表示します。
**Size** を **512 x 512** に変更します。
![](/images/change-size.png)

:::details スクリプトで RenderTexture を生成したい場合
事前に RenderTexture のアセットを作成せずに、コード内で RenderTexture を生成したい場合は、例えば下記のように作成できます。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
+   private RenderTexture renderTexture;
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～

    private void Start()
    {        
        InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

+       // カメラの targetTexture に renderTexture を設定すると、カメラの映像が renderTexture へ書き込まれるようになります。
+       renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
+       camera.targetTexture = renderTexture;
        
        SetOverlaySize(overlayHandle, size);
        ShowOverlay(overlayHandle);
    }

    ～省略～
```
:::

### Render Texture の変数を作成
RenderTexture を入れるメンバ変数を作成します。
```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
+   public RenderTexture renderTexture;
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

`WatchOverlay` のインスペクタを開き、`renderTexture` に `WatchRenderTexture` アセットを設定します。
![](/images/attach-render-texture-to-overlay.png)

### テクスチャの作成を待つ
レンダーテクスチャの作成が完了していない場合は、描画を行わないようにします。
```diff cs:WatchOerlay.cs
private void Update()
{
    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
    if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        var position = new Vector3(x, y, z);
        var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
        SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
    }

+   if (!renderTexture.IsCreated())
+   {
+       return;
+   }
+
+   // ここに描画処理を追加
}

```

### ネイティブテクスチャポインタの取得
OpenVR に渡すテクスチャのデータ形式は、Unity の API の下のレイヤーで使われている DirectX や OpenGL といったグラフィックス API のデータ形式になります。
Unity の [GetNativeTexturePtr()](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html) を使って、DirectX や OpenGL のテクスチャデータへアクセスするための参照（ネイティブテクスチャポインタ）を取得できます。

`Update()` 内で、レンダーテクスチャの `GetNativeTexturePtr()` を呼び出して、OpenVR に渡すためのテクスチャの参照を取得します。

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

    if (!renderTexture.IsCreated())
    {
        return;
    }
+   var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
}
```

:::details レンダリングスレッドの同期について
[GetNativeTexturePtr() のドキュメント](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html)では、`GetNativeTexturePtr()` は初期化時に一度だけ呼び出すことが推奨されています。
しかしオーバーレイへの描画処理は、レンダリングスレッドが同期したタイミングで実行しなければクラッシュすることがあるため、敢えて `Update()` 内で `GetNativeTexturePtr()` を呼び出しています。
:::

### OpenVR のテクスチャを作成
OpenVR 側のテクスチャのデータ型 `Texture_t` の変数を作成します。

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

    if (!renderTexture.IsCreated())
    {
        return;
    }
    var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
+   var texture = new Texture_t
+   {
+       eColorSpace = EColorSpace.Auto,
+       eType = ETextureType.DirectX,
+       handle = nativeTexturePtr
+   };
}
```

`eType` にグラフィックス API の種類を指定します。このチュートリアルの環境では、
デフォルトで DirectX が使われるので、今後は DirectX を前提としてコードを作成します。

`handle` に先ほど取得したネイティブテクスチャのポインタ `nativeTexturePtr` をセットします。
このテクスチャにカメラの映像が書き込まれています。

:::details DirectX 以外に対応させる場合
Unity が実行時に使用しているグラフィックス API の種類は `SystemInfo.graphicsDeviceType` で判定できます。
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

**Project Settings > Player > Other Settings > Auto Graphics API for Windows** のチェックを外して、**Graphics APIs for Windows** のリストの一番上に Direct3D11 以外の API を追加すると、その API を使った動作確認が可能です。
![](/images/graphics-api.png)
https://docs.unity3d.com/ja/2019.4/Manual/GraphicsAPIs.html
:::

### オーバーレイにテクスチャを書き込む
OpenVR の [SetOverlayTexture()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTexture_System_UInt64_Valve_VR_Texture_t__) で、オーバーレイにテクスチャを表示できます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTexture) を参照）
先ほど作成した `Texture_t` 型のテクスチャを渡します。

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


    if (!renderTexture.IsCreated())
    {
        return;
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
+       throw new Exception($"テクスチャの描画に失敗しました: " + error);
+   }
}
```

プログラムを実行して、カメラの映像がリアルタイムにオーバーレイに表示されることを確認してください。

![](/images/realtime-rendering.gif)

### 上下を反転させる
カメラ映像が上下逆さまに表示されているので、正しい向きに直します。

テクスチャの UV 座標系が、Unity は左下が原点で V 軸が上向き、DirectX は左上が原点で V 軸が下向きのため、最終的に OpenVR が描画した結果が上下逆さまになっています。
https://docs.unity3d.com/ja/current/Manual/SL-PlatformDifferences.html

正しい向きに直す方法は色々ありますが、今回は OpenVR の [SetOverlayTextureBounds()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTextureBounds_System_UInt64_Valve_VR_VRTextureBounds_t__) でテクスチャの V 軸を逆にして割り当てるすることで上下を反転させます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTextureBounds) を参照）

`SetOverlayTextureBounds()` はテクスチャを表示する範囲を UV 座標で指定する関数ですが、上下を反転させるためにも使えます。デフォルトでは **vMin = 0, vMax = 1** ですが、縦方向を逆にするため **vMin = 1, vMax = 0** に設定します。

```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

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

これで上下が反転します。プログラムを実行して正しい向きで表示されていることを確認してください。
![](/images/flip-y-axis.jpg)

:::details DirectX 以外に対応させる場合
このチュートリアルでは、DirectX を前提としているため、必ず上下を反転させています。
他の API に対応させる場合は、上の**DirectX 以外に対応させる場合**に書いた通り `graphicsDeviceType` を参照して「OpenVR なら反転させない」といった分岐を作成してください。
:::


## 時刻を表示する Canvas を作る
カメラの映像が表示できたので、次は時計を作成します。
Cube と Directional Light はもう使わないので、シーンから削除してください。

![](/images/only-overlay-camera.png)

Camera のインスペクタを開き、Transform コンポーネントから Reset をクリックして **(0, 0, 0) へリセット**します。
![](/images/reset-camera-position.png)

Hierarchy を右クリックして、シーンに下記のオブジェクトを追加します。
- **UI > Canvas**
- 作成した Canvas の下に **UI > Text - TextMeshPro**

![](/images/canvas-text-hierarchy.png)

初めて TextMeshPro を作成すると下のようなダイアログが表示されるので、**Import TMP Essentials** をクリックします。完了したらダイアログを閉じます。
![](/images/import-tmp-essentials.png)

Canvas のインスペクタを開き **Render mode** を **Screen Space - Camera** にします。
そのまま **Render Camera** にシーン上のカメラをドラッグしてください。
![](/images/set-canvas-camera.png)

Text (TMP) のインスペクタを開き **Alignment** でテキストの縦横の位置を中央に揃えます。
テキストに "00:00:00" と入力してください。
![](/images/text-alignment.png)

Camera を選択して **Clear Flags** を **Solid Color** にします。
また **Background** の色をクリックし、**A (Alpha) が 0** になっていることを確認します。
これでカメラ映像の背景が透過されて、時刻だけが描画されるようになります。
![](/images/background-color.png)

Canvas を選択して **Plane Distance を 10** にします。これは Editor で作業するときのためです。
![](/images/plane-distance.png)

Text (TMP) を選択して、**アンカーの設定（Rect Transform コンポーネント左上の四角形）をクリック**して、**右下の青い十字の矢印（上下左右ストレッチ）を選択**します。
![](/images/change-anchor.png)

そのまま **Left, Top, Right, Bottom の値を 0** にします。
![](/images/position-0.png)

インスペクタをスクロールして **TextMeshPro - Text (UI)** コンポーネントの **Font Size を 70** にします。
![](/images/font-size.png)

プログラムを実行して、左手首に時刻が表示されることを確認してください。
もし違和感があればオーバーレイの大きさや位置、フォントサイズなどを調整してください。
![](/images/show-time.jpg)

## 時計を動かす
`Scripts` フォルダの下に `Watch.cs` を新規作成します。
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

シーン上の **Text (TMP)** に `Watch.cs` を追加します。
![](/images/watch-to-text.png)


プログラムを実行して、現在の時刻が表示されていれば OK です。

![](/images/clock-check.jpg)


## コードの整理

### オーバーレイの上下反転
`FlipOverlayVertical()` として関数に分けておきます。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    
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
+   FlipOverlayVertical(overlayHandle);    
    SetOverlaySize(overlayHandle, size);
    ShowOverlay(overlayHandle);
}

～省略～

+ private void FlipOverlayVertical(ulong handle)
+ {
+    var bounds = new VRTextureBounds_t
+    {
+        uMin = 0,
+        uMax = 1,
+        vMin = 1,
+        vMax = 0
+    };
+
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
+     if (!renderTexture.IsCreated()) return;
+
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
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
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

        FlipOverlayVertical(overlayHandle);
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

        SetOverlayRenderTexture(overlayHandle, renderTexture);
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
    
    private void FlipOverlayVertical(ulong handle)
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
```
