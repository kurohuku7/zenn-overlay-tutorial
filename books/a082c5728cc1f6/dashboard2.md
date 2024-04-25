---
title: "イベントの処理"
free: false
---

## 右手のコントローラに時刻を表示

右手にも時計を表示できるようにします。

### 変数の作成
時計を表示する `WatchOverlay.cs` に左右のどちらの手に表示するかを決める変数を作成します。
OpenVR の [ETrackedControllerRole](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.ETrackedControllerRole.html) 型に左手と右手を識別する定数が定義されているので、これを使うことにします。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
+   public ETrackedControllerRole targetHand = EtrackedControllerRole.LeftHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    ～省略～
```

左手の値が`EtrackedControllerRole.LeftHand`
右手の値が `ETrackedControllerRole.RightHand` です。

### 選択中のコントローラの Device Index を取得
左手固定の Device Index を取得するようになっているので、選択中のコントローラを取得するように修正します。

```diff cs:WatchOverlay.cs
private void Update()
{
    var position = new Vector3(x, y, z);
    var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);

-   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
-   {
-       Overlay.SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
-   }
+   var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
+   if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
+   }

    Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
}
```

`WatchOverlay` のインスペクタで、`Target Hand` を `Right Hand` に切り替えて、右手に時刻が表示されることを確認します。
![](/images/change-hand-inspector.png)

ただし、左手のコントローラを基準にして位置を設定しているため、右手に表示した場合は、しっくりこない位置に時刻が表示されます。
![](/images/wrong-right-hand-position.jpg)

右手のちょうどいい位置にオーバーレイが表示されるように調整していきます

### 既存の設定値のメモ

まず `WatchOverlay` のインスペクタを開き、`X, Y, Z, Rotation X, Rotation Y, Rotation Z` の値をメモしておきます。
```
左手用のメモ
x = -0.044
y = 0.015
z = -0.131
rotationX = 154
rotationY = 262
rotationZ = 0
```

### 左手用と右手用の変数を作成
オーバーレイの位置と回転を設定しているメンバ変数 `x, y, z, rotationX, rotationY, rotationZ` を削除します。
ただし `size` は左右で共通にするので、そのまま残します。

代わりに左手用の `leftX, leftY, leftZ, leftRotationX, leftRotationY, leftRotationZ` と、右手用の `rightX, rightY, rightZ, rightRotationX, rightRotationY, rightRotationZ` を作ります。

下記のコードを参考に変数を作ってください。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    // size は残す
    [Range(0, 0.5f)] public float size;

-   // 既存の変数は削除
-   [Range(-0.2f, 0.2f)] public float x;
-   [Range(-0.2f, 0.2f)] public float y;
-   [Range(-0.2f, 0.2f)] public float z;
-   [Range(0, 360)] public int rotationX;
-   [Range(0, 360)] public int rotationY;
-   [Range(0, 360)] public int rotationZ;

+   // 左手用の変数を追加
+   [Range(-0.2f, 0.2f)] public float leftX;
+   [Range(-0.2f, 0.2f)] public float leftY;
+   [Range(-0.2f, 0.2f)] public float leftZ;
+   [Range(0, 360)] public int leftRotationX;
+   [Range(0, 360)] public int leftRotationY;
+   [Range(0, 360)] public int leftRotationZ;

+   // 右手用の変数を追加
+   [Range(-0.2f, 0.2f)] public float rightX;
+   [Range(-0.2f, 0.2f)] public float rightY;
+   [Range(-0.2f, 0.2f)] public float rightZ;
+   [Range(0, 360)] public int rightRotationX;
+   [Range(0, 360)] public int rightRotationY;
+   [Range(0, 360)] public int rightRotationZ;

```

### 対応する位置と角度をセット
`targetHand` に合わせた位置に表示するように `Update()` を修正します。
```diff cs:WatchOverlay.cs
private void Update()
{
-   var position = new Vector3(x, y, z);
-   var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
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
   
    var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
    if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
    {
        Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
    }

    Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
}
```

### 右手用の表示位置の調整
プログラムを実行して `WatchOveraly` のインスペクタを開きます。
**Target Hand** が **Right Hand** になっているのを確認します。

右手用の変数のスライダーを使って、ちょうどいい座標と角度を決定します。
![](/images/right-hand-params.png)

以下は設定値の例です。
```
右手用の設定値
x = 0.04
y = 0.003
z = -0.107
rotationX = 24
rotationY = 258
rotationZ = 179
```

![](/images/correct-right-hand.jpg)

ちょうどいい位置に表示できたら、前回と同じように `WatchOverlay` コンポーネントを**右クリック > Copy Component** で値をコピーします。
![](/images/copy-component-righthand.png)

プログラムを終了して、再度**右クリック > Paste Component Values** で値をペーストします。
左手用には、先ほどメモしておいた値を入力します。
**Target Hand** の初期値を **Left Hand** に戻しておきます。

![](/images/controller-params.png)

これで、右手用と左手用の位置の調整が完了です。

## ボタンが押されたときの処理を作成
設定画面のボタンを押して、コントローラを切り替えられるようにします。
`Scripts` フォルダに `WatchSettingController.cs` を新規作成します。
設定画面のボタンが押されたときの処理を、このスクリプトに書いていきます。

Hierarchy を**右クリック > Create Empty** で空のオブジェクトを作成し、`SettingController` という名前に変更します。
作成した `SettingController` オブジェクトに `WatchSettingController.cs` を追加します。
![](/images/add-watch-controller.png)


`WatchSettingController.cs` を開き、下のコードをコピーします。
```cs:WatchSettingController.cs
using UnityEngine;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
}
```

`SettingController` のインスペクタを開き、`WatchOverlay` を `Watch Overlay` にドラッグします。
![](/images/attach-watch-overlay.png)

`WatchSettingController.cs` に、ボタンが押されたときのイベントを追加します。
```diff cs:WatchSettingController.cs
using UnityEngine;
+ using Valve.VR;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
    
+   public void OnLeftHandButtonClick()
+   {
+       // Left Hand ボタンが押されたら左手に変更
+       watchOverlay.targetHand = ETrackedControllerRole.LeftHand;
+   }
+
+   public void OnRightHandButtonClick()
+   {
+       // Right Hand ボタンが押されたら右手に変更
+       watchOverlay.targetHand = ETrackedControllerRole.RightHand;
+   }
}
```

## ボタンにイベントを割り当て
Hierarchy の `Dashboard > Canvas > LeftHandButton` オブジェクトを選択します。
インスペクタで `Button` コンポーネントの `OnClick()` の `+` をクリックします。
`SettingController` を `OnClick()` の `None (Object)` へドラッグして `WatchSettingController.OnLeftHandButtonClick()` を設定します。
![](/images/attach-left-button-event.png)

これで uGUI のボタンが押されたときに `SettingController.OnLeftHandButtonClick()` が呼ばれます。

同様に `RightHandButton` の `OnClick()` に `WatchSettingController.OnRightHandButtonClick()` を設定します。
![](/images/attach-right-hand-event.png)

## ダッシュボードのイベント取得
このままではダッシュボード上でボタンをクリックしても何も起こりません。
今設定したのはあくまで Unity 側のイベントになります。

OpenVR 側のイベントと Unity 側のイベントは別になっているので、ダッシュボードオーバーレイがクリックされたときに、OpenVR から Unity へイベントを繋ぎこむ必要があります。

まず、OpenVR 側でダッシュボードのボタンがクリックされたイベントを取得します。
オーバーレイのイベントは `PollNextOverlayEvent()` を使ったポーリングで検出します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::PollNextOverlayEvent) を参照）

`Update()` 内で `PollNextOverlayEvent()` を使ってダッシュボードオーバーレイで発生したイベントを監視します。
指定したオーバーレイでイベントが発生していれば、`PollNextOverlayEvent()` の戻り値が `true` になり、発生したイベントを一つ取り出されます。
全てのイベントを取り出すと、戻り値が `false` になります。

`DashboardOverlay.cs` の `Update()` 内にイベントの検出処理を追加します。
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

イベントは [VREvent_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VREvent_t.html) 型で取得されます。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/VREvent_t) を参照）
`uncbVREvent` は [VREvent_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VREvent_t.html) 型の構造体のサイズ（バイト数）です。


## クリックイベントの取得
レーザーポインタでダッシュボードオーバーレイがクリックされると `EVREventType.VREvent_MouseButtonDown` と `EVREventType.VREvent_MouseButtonUp` が発生します。
その他のイベントは [EVREventType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVREventType.html) に定義されています。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/VREvent_t) を参照）

試しに `EVREventType.VREvent_MouseButtonDown` と `EVREventType.VREvent_MouseButtonUp` を取得してみます。

```diff cs
private void Update()
{
    var vrEvent = new VREvent_t();
    var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));

    while (OpenVR.Overlay.PollNextOverlayEvent(overlayHandle, ref vrEvent, uncbVREvent))
    {
+       switch ((EVREventType)vrEvent.eventType)
+       {
+           case EVREventType.VREvent_MouseButtonDown:
+               Debug.Log("MouseDown");
+               break;
+
+           case EVREventType.VREvent_MouseButtonUp:
+               Debug.Log("MouseUp");
+               break;
+       }
    }
}
```

`vrEvent.eventType` は `uint` 型でイベントコードが入っているので、`EVREventType` にキャストして比較しています。
プログラムを実行して、ダッシュボードのボタンをクリックすると、"MouseDown" と "MouseUp" がコンソールに出力されます。

![](/images/mouse-up-console.png)
![](/images/click-dashboard.jpg)
*マウスポインタをオーバーレイに合わせてクリック*

:::details EVREventType.VREvent_MouseClick はない？
ありません。

OpenVR Overlay のマウスイベントはシンプルで `MouseButtonDown`, `MouseButtonUp`, `MousMove` の 3 つです。
OnClick を作るなら MouseDown と MouseUp を組み合わせて「MouseUp 発生時に MouseDown 発生時と同じ要素にマウスが乗っていたらクリックしたとする」というような処理を作ります。

OnMouseEnter、OnMouseLeave、OnDrag なども、上の 3 つを組み合わせて作ります。
スライダーなどを作ろうとすると Unity UI のみで作るよりも、少し手間がかかります。

:::details そこを何とかならないでしょうか？
ドラッグ＆ドロップで構築できる SteamVR ダッシュボード用 UI アセットを Unity Asset Store で販売中です。
https://assetstore.unity.com/packages/tools/gui/ovrle-ui-dashboard-ui-kit-for-steamvr-270636
https://youtu.be/JHFiPjIXEsE
:::


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

`vrEvent` には [VREvent_Mouse_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VREvent_Mouse_t.html?q=VREvent_Mouse_t) 型でマウスの座標がセットされています。
マウス座標は UV 座標でセットされ、x, y ともに 0 ~ 1 の範囲の値となります。

プログラムを実行して、ダッシュボード開き、レーザポインタでオーバーレイをクリックしてください。マウスの座標がコンソールに表示されます。
![](/images/click-uv-position.png)

## Mouse Scaling Factor の適用
受け取ったマウス座標は 0 ~ 1 の UV 座標になっていますが、**Mouse Scaling Factor** を設定すると、実際の UI の座標（ピクセル数）で受け取ることができます。
Mouse Scaling Factor は [SetOverlayMouseScale()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayMouseScale_System_UInt64_Valve_VR_HmdVector2_t__) で設定します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayMouseScale) を参照）

`DashboardOveraly.cs` の `Start()` に Mouse Scaling Factor の設定を追加します。
```diff cs:DashboardOverlay.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();

    (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

    var filePath = Application.streamingAssetsPath + "/sns-icon.jpg";
    Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

    Overlay.FlipOverlayVertical(dashboardHandle);
    Overlay.SetOverlaySize(dashboardHandle, 2.5f);

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
OpenVR でマウスイベントが発生した際に、マウスの UV 座標にこの値が掛け算されて、実際の UI の座標に合わせた座標で受け取れます。

プログラムを実行して、ダッシュボードをクリックしてください。
クリックされた座標が (0, 0) ～ (1024, 768) にスケーリングされた座標で取得できるようになります。
![](/images/scaled-mouse-position.png)


## Overlay Viewer でのイベント確認
ちなみに Overlay Viewer でもマウスイベントを発行できます。

プログラムの実行後、Overlay Viewer を起動して、一覧から `WatchDashboardKey` を選択します。
右下の **Mouse Capture** にチェックを入れた状態で、プレビューをクリックすると、マウスイベントが発生して Unity のコンソールにログが表示されます。
![](/images/overlay-viewer-event.png)


HMD を被らずにイベントの動作確認をするときには Overlay Viewer を使うと便利です。


## クリックされたボタンを判定する
マウスイベントの座標から、Unity 側のどのボタンがクリックされたのかを判定します。
Canvas の [Graphic Raycaster](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.UI.GraphicRaycaster.html) を使って、クリックされた座標にある要素を取得します。

### Graphic Raycaster を取得
`DashboardOverlay.cs` に `graphicRaycaster` 変数を追加します。
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

Hierarchy で `Dashboard > DashboardOverlay` オブジェクトをクリックしてインスペクタを開きます。
`GraphicRaycaster` 変数に `Dashboard > Canvas` を設定します。
![](/images/drag-graphic-raycaster.png)

### 検出用のメソッドを作成
`DashboardOverlay.cs` に、指定された座標にあるボタンを返すメソッドを追加します。

```diff cs:DashboardOverlay.cs
private void OnDestroy()
{
    OpenVRUtil.System.ShutdownOpenVR();
}

+ private Button GetButtonByPosition(Vector2 position)
+ {
+    // 座標 position.x, position.y にあるボタンを返す
+    // ボタンがなければ null を返す
+    return null;
+ }

～省略～
```

今回は `Button` のみを使った UI なので `Button` に限定して処理を作成します。

## EventSystem の追加
`GraphicRaycaster` に渡す座標は Unity 側の UI イベントを仕切っている [EventSystem](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/manual/EventSystem.html) の [PointerEventData](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.EventSystems.PointerEventData.html) クラスで渡します。
OpenVR から受け取ったマウス座標で `PointerEventData` を作るために、まずは `DashboardOverlay.cs` に `EventSystem` を保存する変数を追加します。

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

```

Hierarchy で `Dashboard > DashboardOverlay` のインスペクタを開きます。
`EventSystem` オブジェクトを `Event System` 変数にドラッグします。
![](/images/drag-event-system.png)


### PointerEventData の作成
追加した `EventSystem` を使って、`PointerEventData` を作成します。
`GetButtonByPosition()` 内で `position` から `PointerEventData` を作成します。

```diff cs:DashboardOverlay.cs
private Button GetButtonByPosition(Vector2 position)
{
+   var pointerEventData = new PointerEventData(eventSystem);
+   pointerEventData.position = position;
    return null;
}
```

`PointerEventData` クラスのコンストラクタに `EventSystem` を渡します。


### GraphicRaycaster で Button を取得する
作成した `PointerEventData` を [GraphicRaycaster.Raycast()](https://docs.unity3d.com/Packages/com.unity.ugui@1.0/api/UnityEngine.UI.GraphicRaycaster.html#UnityEngine_UI_GraphicRaycaster_Raycast_UnityEngine_EventSystems_PointerEventData_System_Collections_Generic_List_UnityEngine_EventSystems_RaycastResult__) に渡して、クリック座標にある要素を探し、`Button` があれば返すようにします。


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

    ～省略～

    private Button GetButtonByPosition(Vector2 position)
    {
        var pointerEventData = new PointerEventData(eventSystem);
        pointerEventData.position = position;

+       // 見つかった要素を保存するリスト
+       var raycastResultList = new List<RaycastResult>();
+
+       // pointerEventData の座標にある UI 要素を探して raycastList に保存
+       graphicRaycaster.Raycast(pointerEventData, raycastResultList);
+
+       // リストからボタンだけを取り出す
+       var raycastResult = raycastResultList.Find(element => element.gameObject.GetComponent<Button>());
+
+       // ボタンが無かったら null を返す
+       if (raycastResult.gameObject == null)
+       {
+           return null;
+       }
+
+       // ボタンがあったら返す
+       return raycastResult.gameObject.GetComponent<Button>();
    }
```

これで座標からボタンを探す処理は完成です。

### 検出処理の呼び出し
OpenVR のクリックイベントを取得したら、マウス座標を取り出して、今作成した `GetButtonByPosition` に渡します。
OpenVR にクリックイベントは無いので、今回は `MouseDown` でボタンを押すようにします。

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
-           // MouseUp イベントは使わないので削除
-           case (uint)EVREventType.VREvent_MouseButtonDown:
-               Debug.Log($"MouseDown: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
-               break;

            case (uint)EVREventType.VREvent_MouseButtonUp:
-               Debug.Log($"MouseUp: ({vrEvent.data.mouse.x}, {vrEvent.data.mouse.y})");
+               var button = GetButtonByPosition(new Vector2(vrEvent.data.mouse.x, vrEvent.data.mouse.y));
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
クリックしたボタンと逆のボタンが取得されているのは、マウスクリック時に通知される UV 座標系の V 軸 (Y軸)の上下がオーバーレイと Unity の Canvas で逆になっているためです。

OpenVR Overlay のマウスイベントは、左下を (0, 0) として返すようになっており、本来 Unity と一致しています。
しかし、オーバーレイを表示するときに DirectX のテクスチャが上下反転する問題の対策として使用した `SetOverlayTextureBounds()` の影響によって、OpenVR から通知される UV 座標の上下も逆になっています。
（デフォルトでは左下を原点として返すところを、逆さまにしたため左上が原点として返ってくる状態。）

このチュートリアルでは必ず DirectX を使用され、必ず `SetOerlayTextureBounds()` を使ってテクスチャの上下を反転させるという前提の元、クリックされたマウス座標の V 軸 (Y軸) をイベント処理内で反転させます。
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
                var button = GetButtonByPosition(new Vector2(vrEvent.data.mouse.x, vrEvent.data.mouse.y));
                Debug.Log(button);
                break;
        }
    }
}
```

プログラムを実行して、Left Hand と Right Hand のボタンをクリックして、正しいボタンが取れていることを確認します。
![](/images/correct-button-log.png)


## ボタンクリックで左右のコントローラ表示を入れ替え
これでクリックされたボタンを取得できたので、あとは左右のコントローラの切り替えだけです。
既に Unity のインスペクタ上で各 `Button` の `onClick` に切り替え処理を割り当て済みなので、それを呼び出すだけです。

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
                var button = GetButtonByPosition(new Vector2(vrEvent.data.mouse.x, vrEvent.data.mouse.y));
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

プログラムを実行してダッシュボードを開き、ボタンをクリックしてください。
どちらの手に設定するか切り替えられるようになっていれば OK です。
![](/images/switch-hand.gif)

これでダッシュボードオーバーレイの作成と、左右コントローラの切り替えは完了です。


## コード整理

### Mouse Scaling Factor の設定
Mouse Scaling Factor の設定を `SetOverlayMouseScale()` として関数に分けておきます。

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
+   Overlay.SetOverlayMouseScale(dashboardHandle, renderTexture.width, renderTexture.height);
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

～省略～

+ private void ProcessOverlayEvents()
+ {
+     var vrEvent = new VREvent_t();
+     var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
+     while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
+     {
+         switch (vrEvent.eventType)
+         {
+             case (uint)EVREventType.VREvent_MouseButtonUp:
+                 vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
+                 var button = GetButtonByPosition(new Vector2(vrEvent.data.mouse.x, vrEvent.data.mouse.y));
+                 if (button != null)
+                 {
+                     button.onClick.Invoke();
+                 }
+                 break;
+         }
+     }
+ }

private Button GetButtonByPosition(float x, float y)
{
    ～省略～
```


## 最終的なコード

```cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.LeftHand;
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;

    [Range(-0.2f, 0.2f)] public float leftX;
    [Range(-0.2f, 0.2f)] public float leftY;
    [Range(-0.2f, 0.2f)] public float leftZ;
    [Range(0, 360)] public int leftRotationX;
    [Range(0, 360)] public int leftRotationY;
    [Range(0, 360)] public int leftRotationZ;

    [Range(-0.2f, 0.2f)] public float rightX;
    [Range(-0.2f, 0.2f)] public float rightY;
    [Range(-0.2f, 0.2f)] public float rightZ;
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

        Overlay.FlipOverlayVertical(dashboardHandle);
        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
        Overlay.SetOverlayMouseScale(dashboardHandle, renderTexture.width, renderTexture.height);
    }

    private void Update()
    {
        Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
        ProcessOverlayEvents();
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(dashboardHandle);
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
                    var button = GetButtonByPosition(new Vector2(vrEvent.data.mouse.x, vrEvent.data.mouse.y));
                    if (button != null)
                    {
                        button.onClick.Invoke();
                    }
                    break;
            }
        }
    }
    
    private Button GetButtonByPosition(Vector2 position)
    {
        var pointerEventData = new PointerEventData(eventSystem);
        pointerEventData.position = new Vector2(position.x, position.y);

        var raycastResultList = new List<RaycastResult>();
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

これで OpenVR 側のダッシュボードオーバーレイのイベントを処理して、Unity 側の UI を操作することができました。
次のページではコントローラのボタン操作を扱い、ボタンが押されたら一定時間だけ時刻が表示される処理を作ります。