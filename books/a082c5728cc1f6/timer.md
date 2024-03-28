---
title: "オーバーレイに時刻を表示する"
free: false
---

オーバーレイに時刻をリアルタイムに表示してみます。
現在は画像ファイルを表示していますが、Unity のカメラで作成した画像をオーバーレイとして表示してみます。

## オーバーレイへのテクスチャの書き込み
SetOverlayTexture() メソッドで、任意のテクスチャをオーバーレイに書き込むことができます。

https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTexture

DirectX や OpenGL のテクスチャデータのポインタを渡します。
Unity の Texture クラスの GetNativeTexturePtr() メソッドで、DirectX や OpenGL のテクスチャデータのポインタを取得できるので、これを SetOverlayTexture() に渡します。
https://docs.unity3d.com/ScriptReference/Texture.GetNativeTexturePtr.html

## テクスチャの準備

### シーンの準備
Unity のシーンに

- Camera
- Cube (3D Object)
- Directional Light

を追加します。

カメラに Cube が映るように配置してください。
![](/images/cube-scene.png)

### Cube を回転させる
新規に C# スクリプト Rotate.cs を作成します。
リアルタイムにカメラ映像が動いていることがわかるように、常に回転させます。

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

作成したスクリプトを Cube オブジェクトのコンポーネントとして追加します。
![](/images/cube-rotate-component.png)

Cube が回転することを確認します。
![](/images/rotate-cube.gif)

### 流れ

下記の流れでカメラの映像をオーバーレイに書き込みます。

- スクリプトで RenderTexture を作成
- スクリプトからカメラの描画先を RenderTexture に設定
- カメラが RenderTexture に書き込むようになる
- スクリプトで RenderTexture から Native Texture ポインタを取得
- スクリプトで NativeTexture ポインタを SetOverlayTexturePtr() にわたす

### カメラの参照を追加

スクリプトにカメラの参照を追加します。

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



### RenderTexture への書き込み

画像ファイルはもう使わないので削除します。

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




GetNativePointer() でメインスレッドとレンダリングスレッドを同期させる必要があります。

Unity のドキュメントによると、この関数は毎フレーム呼び出さないように注意書きがあるのですが、呼ばないとアプリがクラッシュしてしまうので呼びます。

## 時刻を表示する Canvas を作る
uGUI を使って時計の UI（HUD）を作っていきます。
詳しくは uGUI のドキュメントなどを見てください。

## レンダーテクスチャの描画
カメラでレンダーテクスチャを取ってみます。
できましたね。

## レンダーテクスチャをオーバーレイに表示する
OpenVR の API を使って、カメラの映像をオーバーレイに表示してみましょう。