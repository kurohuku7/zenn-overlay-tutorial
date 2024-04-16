---
title: "イベントの処理"
free: false
---

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

`Update()` 内で `PollNextOverlayEvent()` を使ってダッシュボードオーバーレイで発生したイベントを監視します。
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
その他のイベントは [EVREventType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) に定義されています。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/VREvent_t) を参照）

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

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

    Overlay.SetOverlaySize(dashboardHandle, 2.5f);
    Overlay.FlipOverlayVertical(dashboardHandle);

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

マウスイベントの座標から、どのボタンがクリックされたのかを判定します。
Canvas の [Graphic Raycaster](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.UI.GraphicRaycaster.html) を使って、クリックされた座標の要素を取得します。

### Graphic Raycaster を取得
GraphicRaycaster の変数を追加します。
```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;
+ using UnityEngine.UI;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
+   public GraphicRaycaster graphicRaycaster;
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

Hierarchy で `Dashboard/Canvas` を `DashboardOverlay` の `GraphicRaycaster` へドラッグします。
インスペクタから GraphicRaycaster に Canvas をドラッグして設定します。
![](/images/drag-graphic-raycaster.png)

### 検出用のメソッドを作成
```diff cs:DashboardOverlay.cs
private void OnDestroy()
{
    OpenVRUtil.System.ShutdownOpenVR();
}

+ private Button GetButtonByPosition(float x, float y)
+ {
+    //  から Button を探して返す
+ }

～省略～
```

引数はマウスがクリックされた座標で、`vrEvent.data.mouse.x` と `vrEvent.data.mouse.y` をを渡します。
戻り値は、今回は Button のみを使った UI なので Button にします。

### EventSystem と PointerEventData の準備
Graphic Raycaster に渡すための EventSystem と PointerEventData を準備します。
```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;
using UnityEngine.UI;
+ using UnityEngine.EventSystems;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public GraphicRaycaster graphicRaycaster;
+   public EventSystem eventSystem;
    
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～

    private Button GetButtonByPosition(float x, float y)
    {
+       var pointerEventData = new PointerEventData(eventSystem);
+       pointerEventData.position = new Vector2(x, y);
    }
```

Hierarchy で `EventSystem` を `DashboardOverlay` の `EventSystem` にドラッグします。
![](/images/drag-event-system.png)


### GraphicRaycaster で Button を取得する
[GraphicRaycaster.Raycast()](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.UI.GraphicRaycaster.html#UnityEngine_UI_GraphicRaycaster_Raycast_UnityEngine_EventSystems_PointerEventData_System_Collections_Generic_List_UnityEngine_EventSystems_RaycastResult__) で、クリック座標の要素を取得し、Button があれば返すようにします。


```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;
using UnityEngine.UI;
using UnityEngine.EventSystems;
+ using System.Collections.Generic;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public GraphicRaycaster graphicRaycaster;
    public EventSystem eventSystem;

    private Button GetButtonByPosition(float x, float y)
    {
        var pointerEventData = new PointerEventData(eventSystem);
        pointerEventData.position = new Vector2(x, y);

+       // マウス座標にある要素を取得
+       var resultList = new List<RaycastResult>();
+       graphicRaycaster.Raycast(pointerEventData, raycastResultList);
+
+       // 取得した要素から Button コンポーネントを持つゲームオブジェクトを返す
+       var raycastResult = raycastResultList.Find(element => element.gameObject.GetComponent<Button>());
+       if (raycastResult.gameObject == null)
+       {
+           // Button がなければ null を返す
+           return null;
+       }
+
+       // 見つかった Button を返す
+       return raycastResult.gameObject.GetComponent<Button>();
    }
```

### 検出処理の呼び出し
今回はシンプルに MouseDown されたボタンをクリックするようにします。
MouseUp のイベント処理は不要なので削除します。
MouseDown のイベント処理にクリックされたボタンの取得を追加します。
```diff cs:DashboardOverlay.cs
void Update()
{
    Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);

    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
    while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
    {
        switch (vrEvent.eventType)
        {
-           case (uint)EVREventType.VREvent_MouseButtonDown:
-               Debug.Log($"MouseDown: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
-               break;

            case (uint)EVREventType.VREvent_MouseButtonUp:
-               Debug.Log($"MouseUp: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
+               var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
+               Debug.Log(button);
                break;
        }
    }
}
```

プログラムを実行して、ダッシュボードのボタンをクリックすると、ログにボタン名が表示されることを確認します。
（現状ではクリックしたボタンと違うボタン名が表示されますが正常です。）
なにもないところをクリックすると null が返ってきます。
![](/images/button-click-log.png)

## クリック座標の上下を反転させる
クリックしたボタンと逆のボタンが取得されているのは、マウスクリック時に通知される座標系の Y 軸の上下がオーバーレイと Unity の Canvas で逆になっているためです。

OpenVR Overlay のマウスイベントは、オーバーレイの左下を (0, 0) として返すようになっており、Unity と一致しています。
しかし、前のページで DirectX のテクスチャを上下反転して表示させるために使用した `SetOverlayTextureBounds()` の影響によって、通知される座標の上下も逆になっています。
（オーバーレイの左上をクリックすると (0, 0) が返ってくる状態になっています。）

このチュートリアルでは DirectX のテクスチャを使用し、必ず `SetOerlayTextureBounds()` を使ってテクスチャの上下を反転させているという前提の元、クリックされたマウス座標の Y 軸をイベント処理内で反転させます。
```diff cs:DashboardOverlay.cs
void Update()
{
    Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);

    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
    while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
    {
        switch (vrEvent.eventType)
        {
            case (uint)EVREventType.VREvent_MouseButtonUp:
+               vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
                var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
                Debug.Log(button);
                break;
        }
    }
}
```

プログラムを実行して、Left Hand と Right Hand のボタンをクリックして、正しいボタンが取れていることを確認します。
![](/images/correct-button-log.png)


※ DirectX 以外に対応させる場合や、`SetOverlayTextureBounds()` を使わない場合は、それぞれ適切に処理してください。


## ボタンのクリックで左右のコントローラ表示を入れ替え
これでクリックされたボタンを取得できたので、そのボタンに設定された OnClick() を実行します。

```diff cs:DashboardOverlay.cs
void Update()
{
    Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);

    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
    while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
    {
        switch (vrEvent.eventType)
        {
            case (uint)EVREventType.VREvent_MouseButtonUp:
                vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
                var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
-               Debug.Log(button);
+               if (button != null)
+               {
+                   button.onClick.Invoke();
+               }
                break;
        }
    }
}

```

プログラムを実行してダッシュボードを開き、ボタンをクリックすると、左右どちらの手に時計を表示するか設定できるようになっていることを確認します。
![](/images/switch-hand.gif)

これでダッシュボードオーバーレイの作成と、左右コントローラの切り替えは完了です。


## コード整理

### Mouse Scaling Factor の設定
Mouse Scaling Factor の設定を `SetOverlayMouseScale()` として関数に分けておきます。
これはマウスイベントの座標が 0 ~ 1 で返ってくるところを、実際の UI のピクセル数に合わせた座標で返すための処理です。

```diff cs:OpenVRUtil.cs
～省略～

public static void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
{
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
    var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
    if (error != EVROverlayError.None)
    {
        throw new Exception("テクスチャの描画に失敗しました: " + error);
    }
}

+ public static void SetOverlayMouseScale(ulong handle, int x, int y)
+ {
+     var pvecMouseScale = new HmdVector2_t()
+     {
+         v0 = x,
+         v1 = y
+     };
+     var error = OpenVR.Overlay.SetOverlayMouseScale(handle, ref pvecMouseScale);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("マウススケールの設定に失敗しました: " + error);
+     }
+ }

～省略～
```
```diff cs:DashboardOverlay.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();

    (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

    Overlay.SetOverlaySize(dashboardHandle, 2.5f);
    Overlay.FlipOverlayVertical(dashboardHandle);

-   var pvecMouseScale = new HmdVector2_t()
-   {
-       v0 = renderTexture.width,
-       v1 = renderTexture.height
-   };
-   var error = OpenVR.Overlay.SetOverlayMouseScale(dashboardHandle, ref pvecMouseScale);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("マウススケールの設定に失敗しました: " + error);
-   }
+   Overlay.SetoverlayMouseScale(dashboardHandle, renderTexture.width, renderTexture.height);
}
```

### イベントの処理
ダッシュボードオーバーレイのイベント処理を `ProcessOverlayEvents()` として関数に分けておきます。

```diff cs:DashboardOverlay.cs
void Update()
{
    Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
-
-   var vrEvent = new VREvent_t();
-   var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
-   while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
-   {
-       switch (vrEvent.eventType)
-       {
-           case (uint)EVREventType.VREvent_MouseButtonUp:
-               vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
-               var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
-               if (button != null)
-               {
-                   button.onClick.Invoke();
-               }
-               break;
-       }
-  }
+  ProcessOverlayEvents();
}

private void OnApplicationQuit()
{
    Overlay.DestroyOverlay(dashboardHandle);
    Overlay.DestroyOverlay(thumbnailHandle);
}

private void OnDestroy()
{
    OpenVRUtil.System.ShutdownOpenVR();
}

private void ProcessOverlayEvents()
{
    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
    while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
    {
        switch (vrEvent.eventType)
        {
            case (uint)EVREventType.VREvent_MouseButtonUp:
                vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
                var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
                if (button != null)
                {
                    button.onClick.Invoke();
                }
                break;
        }
    }
}

private Button GetButtonByPosition(float x, float y)
{
    var raycastResultList = new List<RaycastResult>();
    var pointerEventData = new PointerEventData(eventSystem);
    pointerEventData.position = new Vector2(x, y);
    
    graphicRaycaster.Raycast(pointerEventData, raycastResultList);
    var raycastResult = raycastResultList.Find(element => element.gameObject.GetComponent<Button>());
    if (raycastResult.gameObject == null)
    {
        return null;
    }
    return raycastResult.gameObject.GetComponent<Button>();
}
```


### 最終的なコード

```cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;
using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;

    [Range(-0.5f, 0.5f)] public float leftX;
    [Range(-0.5f, 0.5f)] public float leftY;
    [Range(-0.5f, 0.5f)] public float leftZ;
    [Range(0, 360)] public int leftRotationX;
    [Range(0, 360)] public int leftRotationY;
    [Range(0, 360)] public int leftRotationZ;

    [Range(-0.5f, 0.5f)] public float rightX;
    [Range(-0.5f, 0.5f)] public float rightY;
    [Range(-0.5f, 0.5f)] public float rightZ;
    [Range(0, 360)] public int rightRotationX;
    [Range(0, 360)] public int rightRotationY;
    [Range(0, 360)] public int rightRotationZ;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        overlayHandle = Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        Overlay.FlipOverlayVertical(overlayHandle);
        Overlay.SetOverlaySize(overlayHandle, size);
        Overlay.ShowOverlay(overlayHandle);
    }

    private void Update()
    {
        Vector3 position;
        Quaternion rotation;

        if (targetHand == ETrackedControllerRole.LeftHand)
        {
            position = new Vector3(leftX, leftY, leftZ);
            rotation = Quaternion.Euler(leftRotationX, leftRotationY, leftRotationZ);
        }
        else
        {
            position = new Vector3(rightX, rightY, rightZ);
            rotation = Quaternion.Euler(rightRotationX, rightRotationY, rightRotationZ);
        }
        
        var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
        if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
        {
            Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
        }

        Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(overlayHandle);
    }
    
    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

```cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using System.Collections.Generic;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public GraphicRaycaster graphicRaycaster;
    public EventSystem eventSystem;
    
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

        var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
        Overlay.FlipOverlayVertical(dashboardHandle);
        Overlay.SetOverlayMouseScale(dashboardHandle, renderTexture.width, renderTexture.height);
    }

    void Update()
    {
        Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
        ProcessOverlayEvents();
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(dashboardHandle);
        Overlay.DestroyOverlay(thumbnailHandle);
    }

    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }

    private void ProcessOverlayEvents()
    {
        var vrEvent = new VREvent_t();
        var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
        while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
        {
            switch (vrEvent.eventType)
            {
                case (uint)EVREventType.VREvent_MouseButtonUp:
                    vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
                    var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
                    if (button != null)
                    {
                        button.onClick.Invoke();
                    }
                    break;
            }
        }
    }
    
    private Button GetButtonByPosition(float x, float y)
    {
        var raycastResultList = new List<RaycastResult>();
        var pointerEventData = new PointerEventData(eventSystem);
        pointerEventData.position = new Vector2(x, y);
        
        graphicRaycaster.Raycast(pointerEventData, raycastResultList);
        var raycastResult = raycastResultList.Find(element => element.gameObject.GetComponent<Button>());
        if (raycastResult.gameObject == null)
        {
            return null;
        }
        return raycastResult.gameObject.GetComponent<Button>();
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

        public static void DestroyOverlay(ulong handle)
        {
            if (handle != OpenVR.k_ulOverlayHandleInvalid)
            {
                OpenVR.Overlay.DestroyOverlay(handle);
            }
        }

        public static (ulong, ulong) CreateDashboardOverlay(string key, string name)
        {
            ulong dashboardHandle = 0;
            ulong thumbnailHandle = 0;
            var error = OpenVR.Overlay.CreateDashboardOverlay(key, name, ref dashboardHandle, ref thumbnailHandle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
            }

            return (dashboardHandle, thumbnailHandle);
        }

        public static void ShowOverlay(ulong handle)
        {
            var error = OpenVR.Overlay.ShowOverlay(handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの表示に失敗しました: " + error);
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
            var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
            if (error != EVROverlayError.None)
            {
                throw new Exception("テクスチャの描画に失敗しました: " + error);
            }
        }

        public static void SetOverlayMouseScale(ulong handle, int x, int y)
        {
            var pvecMouseScale = new HmdVector2_t()
            {
                v0 = x,
                v1 = y
            };
            var error = OpenVR.Overlay.SetOverlayMouseScale(handle, ref pvecMouseScale);
            if (error != EVROverlayError.None)
            {
                throw new Exception("マウススケールの設定に失敗しました: " + error);
            }
        }
    }
}
```

```cs:WatchSettingController.cs
using UnityEngine;
using Valve.VR;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
    
    public void OnLeftHandButtonClick()
    {
        watchOverlay.targetHand = ETrackedControllerRole.LeftHand;
    }

    public void OnRightHandButtonClick()
    {
        watchOverlay.targetHand = ETrackedControllerRole.RightHand;
    }
}
```