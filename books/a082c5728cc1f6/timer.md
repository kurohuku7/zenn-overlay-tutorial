---
title: "オーバーレイに時刻を表示する"
free: false
---

オーバーレイに時刻をリアルタイムに表示してみます。
画像ファイルの代わりに、Unity のカメラ映像をリアルタイムでオーバーレイに表示してみます。

## カメラ映像の準備

### シーンの準備
Unity のシーンに

- Camera
- 3D Object → Cube
- Light → Directional Light

を追加します。
カメラに Cube が映るように配置してください。
![](/images/cube-scene.png)

### Cube を回転させる
新規に C# スクリプト `Rotate.cs` を作成します。
次のコードをコピーします。

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

作成したスクリプトを Cube オブジェクトに追加します。
![](/images/cube-rotate-component.png)

プログラムを実行して Cube が回転することを確認してください。
![](/images/rotate-cube.gif)

### カメラの参照を追加

`WatchOverlay.cs` にカメラの参照を追加します。

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

インスペクタで WatchOverlay の camera に、設置したカメラの参照を設定します。
![](/images/attach-camera.png)

### 画像ファイルのコードを削除
画像ファイルの表示は使わないので削除します。
```diff cs:WatchOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    SetOverlaySize(overlayHandle, size);
    
-   var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
-   SetOverlayFromFile(overlayHandle, filePath);

    ShowOverlay(overlayHandle);
}
```


### RenderTexture の作成
[RenderTexture](https://docs.unity3d.com/ja/2018.4/Manual/class-RenderTexture.html) を作成してカメラの描画先に設定します。
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

+       renderTexture = new RenderTexture(512, 512, 16, QualitySettings.activeColorSpace == ColorSpace.Linear ? RenderTextureFormat.ARGBFloat : RenderTextureFormat.ARGB32);
+       camera.targetTexture = renderTexture;
        
        ShowOverlay(overlayHandle);
    }

    ～省略～
```

これでカメラの映像が RenderTexture に書き込まれるようになります。

### ネイティブテクスチャの取得
OpenVR に渡すテクスチャは DirectX や OpenGL のテクスチャのポインタになります。
これらは Unity の下のレイヤーで使われているテクスチャ形式です。
Unity のテクスチャ型は、そのままでは使えないので、GetNativeTexturePtr() を使ってネイティブテクスチャのポインタを取得します。

Update() 内で [GetNativeTexturePtr()](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html) を呼び出します。

```diff cs:WatchOverlay.cs
～省略～

private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
    renderTexture = new RenderTexture(512, 512, 16, QualitySettings.activeColorSpace == ColorSpace.Linear ? RenderTextureFormat.ARGBFloat : RenderTextureFormat.ARGB32);
    camera.targetTexture = renderTexture;
    
    ShowOverlay(overlayHandle);
}
    
+ private void Update()
+ {
+     var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
+ }

～省略～
    
```

### OpenVR の Texture_t 型を作成
Texture_t 型の変数を作成してネイティブテクスチャのポインタを設定します。

```diff cs:WatchOverlay.cs
private void Update()
{
    var nativeTexturePtr = renderTexture.GetNativeTexturePtr();

+   var texture = new Texture_t
+   {
+       eColorSpace = EColorSpace.Auto,
+       eType = ETextureType.DirectX
+       handle = nativeTexturePtr
+   };
}
```

:::details DirectX 以外に対応させる場合
このチュートリアルを作成している環境では、デフォルトで DirectX が Unity のネイティブグラフィック API として使用されています。
Unity が実際に使用している API の種類は `SystemInfo.graphicsDeviceType` で判定できるので、DirectX 以外の API が使われている場合に対応させる場合は、例えば下記のように値を渡します。
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
:::

### サイズの調整
サイズが大きいので 25cm にします。
```diff cs:WatchOverlay.cs
rivate void Start()

   InitOpenVR();
   overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");
   renderTexture = new RenderTexture(512, 512, 16, QualitySettings.activeColorSpace == ColorSpace.Linear ? RenderTextureFormat.ARGBFloat : RenderTextureFormat.ARGB32);
   camera.targetTexture = renderTexture;
+  SetOverlaySize(overlayHandle, 0.2f);
   ShowOverlay(overlayHandle);

```

### オーバーレイにテキスチャを書き込む
[SetOverlayTexture()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTexture_System_UInt64_Valve_VR_Texture_t__) で、ネイティブテクスチャをオーバーレイに書き込むことができます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTexture)）
先ほど作成した texture の参照を `SetOverlayTexture()` に渡します

```diff cs:WatchOverlay.cs
private void Update()
{
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

スクリプトを実行します。
カメラの映像がリアルタイムにオーバーレイに表示されていることを確認してください。

:::details レンダリングスレッドの同期について
[GetNativeTexturePtr()](https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html) のドキュメントでは、GetNativeTexturePtr() は初期化時に一度だけ呼び出すことが推奨されています。
オーバーレイへの描画は、メインスレッドとレンダリングスレッドが同期した状態でなければクラッシュすることがあるため、スレッドの同期のために Update() 内で GetNativeTexturePtr() を呼び出しています。
:::

![](/images/incorrect-direction.jpg)

### 上下を反転させる
テクスチャの上下が反転しているので、逆向きにします。
逆向きになっているのは OpenGL と DirectX の UV 座標系の違いによるものです。

https://docs.unity3d.com/ja/current/Manual/SL-PlatformDifferences.html

Overlay に描画するテクスチャの範囲を UV 座標系で指定できるので、縦方向(V軸)を逆にします。
`SetOverlayTextureBounds()` と `VRTextureBounds_t` 型でテクスチャの UV を指定できます。
デフォルトでは vMin = 0, vMax = 1 ですが、縦方向を逆にするため vMin = 1, vMax = 0 を渡します。

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
+       throw new Exception($"テクスチャの反転に失敗しました({error})");
+   }

-    var error = OpenVR.Overlay.SetOverlayTexture(overlayHandle, ref texture);
+    error = OpenVR.Overlay.SetOverlayTexture(overlayHandle, ref texture);
    if (error != EVROverlayError.None)
    {
        throw new Exception($"テクスチャの描画に失敗しました({error})");
    }
}
```

:::details DirectX 以外に対応させる場合
今回はデフォルトで DirectX が使われているため、必ず上下を反転させています。
Unity が使用しているグラフィックス API に合わせて処理する場合は、先程の `graphicsDeviceType` を使用できます。
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

var bounds = new VRTextureBounds_t();
switch (nativeTexture.eType)
{
    case ETextureType.OpenGL:
        bounds.uMin = 0;
        bounds.vMin = 0;
        bounds.uMax = 1;
        bounds.vMax = 1;
        break;
    case ETextureType.DirectX:
        bounds.uMin = 0;
        bounds.vMin = 1;
        bounds.uMax = 1;
        bounds.vMax = 0;
        break;
}
```
:::

これで上下が反転します。
![](/images/correct-direction.jpg)


## 時刻を表示する Canvas を作る
Cube と Directional Light は使わないので削除します。

シーンに UI > Canvas を作成します。
Canvas の下に UI > Text - TextMeshPro を作成します。

Scene
┗ Canvas
  ┗ Text(TMP)

Canvas の Render mode を Screen Space - Camera にします。
Render Camera にシーン上のカメラをドラッグしてください。

Text の Alignment でテキストの縦横の位置を中央に揃えます。
テキストに 18:30:25 のような時刻を入力してください。
各要素の大きさや位置を調整して、カメラの中央に時刻が表示されるように調整します。

Camera の Background で Alpha を 0 にして、カメラの背景を透明にします。
これで数字だけが描画されるようになります。

## 時計を動かす
スクリプト Watch.cs を新規作成します。
下記のコードをコピーしてください。
TextMeshPro にスクリプトを追加して、時刻が毎秒動くようにします。

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

プログラムを実行して、左手の時計が動くようになったら OK です。

TODO: 実行時の Gif を追加

## コードの整理

コードを整理します


