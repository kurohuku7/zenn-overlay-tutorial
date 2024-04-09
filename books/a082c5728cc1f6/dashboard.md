---
title: "ダッシュボードオーバーレイ"
free: false
---

Steam のダッシュボードに設定画面を作ります。
設定画面で左右のコントローラどちらに時計を表示するか選べるようにします。

TODO: ダッシュボードの画像

## DashboardOverlay の作成
`Scripts/DashboardOverlay.cs` を新規作成します。
![](/images/dashboard-overlay-file.png)

## ダッシュボードオーバーレイの作成
ダッシュボードオーバーレイの作成は [CreateDashboardOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateDashboardOverlay_System_String_System_String_System_UInt64__System_UInt64__) で行います。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateDashboardOverlay) を参照）
下のコードを DashboardOverlay.cs へコピーしてください。

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
dashboardHandle が設定画面、thumbnailHandle がダッシュボードの下に表示されるアイコン用のオーバーレイです。

## OpenVR の初期化、クリーンアップ
OpenVR.Overlay API を使用するためには、OpenVR が初期化されている必要があります。
今回は、WatchOverlay.cs で作成した OpenVR の初期化処理を、共有のユーティリティとして抜き出して、他のクラスからも呼び出せるようにします。

### ユーティリティクラスの作成
`Scripts/Overlay.cs` を作成します。
```cs:Overlay.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OverlayUtil
{
    public class Overlay : MonoBehaviour
    {
    }
}
```

ここでの namespace は名前衝突の回避と、わかりやすさのためにつけているだけです。

### OpenVR の初期化処理を移動
`WatchOverlay.cs` から `InitOpenVR()` を `Overlay.cs` に移動します。
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

```diff cs:Overlay.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OverlayUtil
{
    public class Overlay : MonoBehaviour
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

### 初期化の呼び出しを変更（WatchOverlay.cs）
OvelrayUtil に移動した初期化処理を呼び出すように変更します。

```diff cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;
+ using OverlayUtil;

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
+       Overlay.InitOpenVR();
-       InitOpenVR();
        overlayHandle = CreateOverlay("WatchOverlayKey", "WatchOverlay");

        renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGBFloat);
        camera.targetTexture = renderTexture;

        flipOverlayVertical(overlayHandle);
        SetOverlaySize(overlayHandle, size);
        ShowOverlay(overlayHandle);
    }
    
    ～省略～
```

### 初期化処理を呼び出し（DashboardOverlay.cs）
`OpenVR.Overlay.CreateDashboardOverlay()` No前に OpenVR の初期化処理を呼び出します。
`InitOpenVR()` ではすでに初期化されている場合は、何もしないように作ったので、複数の場所から呼び出しても問題ありません。

```diff cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
+ using OverlayUtil;

public class DashboardOverlay : MonoBehaviour
{
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
+       Overlay.InitOpenVR();
        
        var error = OpenVR.Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting", ref dashboardHandle, ref thumbnailHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
        }
    }
}
```

### クリーンアップ処理をユーティリティに移動
初期化と同様にクリーンアップ処理も移動させて、ユーティリティとして呼び出すように変更します。

```diff cs:Overlay.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OverlayUtil
{
    public class Overlay : MonoBehaviour
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

```diff cs:WatchOverlay.cs
～省略～

private void OnDestroy()
{
    DestroyOverlay(overlayHandle);
+   Overlay.ShutdownOpenVR();
-   ShutdownOpenVR();
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

```diff cs:DashboardOverlay.cs

```

## アイコンの表示


### Canvas と Camera を設置
Canvas とカメラを作成

### UI の作成
uGUI を使ってボタンを作成します。
片方のボタンを「左手に表示」
もう片方のボタンを「右手に表示」
します。

### 左右のどちらのコントローラに表示するかを保存する変数
WatchOverlay.cs に左右のどちらの手に表示するかを決めるメンバを作成します。
```cs
private ETrackedControllerRole targetHand = EtrackedControllerRole.LeftHand;
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

### サムネイルオーバーレイの表示
ダッシュボードの下に表示されるサムネイルも表示しておきます。
何でも良いので画像を準備します。
ここでは、以前使った SNS のアイコンをサムネイルとして `SetOverlayFromFile()` で表示してみます。

```cs
private void Start()
{
  // ここで SetOverlayFromFile()
}
```

プログラムを実行して、ダッシュボードの下にサムネイルが表示されていることを確認します。

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
