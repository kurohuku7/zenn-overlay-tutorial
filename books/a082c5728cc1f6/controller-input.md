---
title: "コントローラの入力"
free: false
---

時計は基本的に非表示にしておき、特定のボタンが押されたときに数秒間だけ表示するように変更してみます。

![](/images/3sec-display.gif)

Unity で VR のコントローラの入力を取る方法は色々あるため、お好みの方法で実装して頂ければ大丈夫ですが、今回は折角なので [OpenVR の Input API (SteamVR Input)](https://github.com/ValveSoftware/openvr/wiki/SteamVR-Input) を使った入力方法で作ってみます。

## 事前設定

SteamVR の **Setting** を開きます。
![](/images/steamvr-setting.png)

**Advanced Settings** を **Show** に切り替えます。
![](/images/advanced-settings.png)

**Developer** の **Enable debugging options in the input binding user interface** を **On** にします。
![](/images/enable-debugging-options.png)

## Action Manifest の作成

SteamVR Input では、直接ボタン等の入力は使用せず、アプリケーション側で定義したアクションが発生したかどうかで、入力を処理します。
SteamVR Input の設定画面でコントローラの各種ボタンやジェスチャーと、アクションの対応付けを行います。
プラットフォームレベルで用意された便利なキーコンフィグ機能のようなイメージです。
![](/images/y-button-plus.png)

まず、アプリケーションで使用するアクションを JSON 形式でリトアップした [Action Manifest](https://github.com/ValveSoftware/openvr/wiki/Action-manifest) を作成します。
SteamVR Plugin には Action Manifest を GUI 上で作れる機能があるので、これを使います。

### Action Manifest の生成

Unity のメニューから **Window > SteamVR Input** を選択します。
![](/images/menu-steamvr-input.png)

初回起動時に Action Manifest のサンプルファイルを使用するか聞かれます。
今回は一から作成するので **No** を選択します。
![](/images/create-default-actionmanifest.png)

**Action Sets** に **Watch** と入力し、その下のドロップダウンを **per hand** に変更します。
アクションセットとは複数のアクションをまとめたもので、アプリケーションごとに複数登録できます。
![](/images/watch-action.png)

**per hand** は、左手と右手のコントローラにそれぞれ異なるアクションを設定できます。
**mirrored** は、片手のみにアクションを設定し、もう片方はそれを鏡写しにしたようにマッピングされます。

下の方にある **Actions** の **In** というボックス内にあるアクション **NewAction** の名前をクリックします。
右側にアクションの詳細が表示されるので、**Name** を **WakeUp** に変更します。
これを時計を表示状態に切り替えるためのアクションとして使います。
変更したら左下の **Save and generate** ボタンを押して Action Manifest を生成します。
![](/images/change-action-name.png)

`StreamingAssets/SteamVR/actions.json` として Action Manifest が書き出されます。
![](/images/generated-action-manifest.png)

```json:actions.json
{
  "actions": [
    {
      "name": "/actions/Watch/in/WakeUp",
      "type": "boolean"
    }
  ],
  "action_sets": [
    {
      "name": "/actions/Watch",
      "usage": "leftright"
    }
  ],
  "default_bindings": [],
  "localization": []
}
```

## デフォルトバインディングの作成

右下の **Open binding UI** ボタンを押します。
![](/images/open-binding-ui.png)

もし VR のコントローラが表示されない場合は、HMD が SteamVR に接続されているか確認してください。

**Create New Binding** をクリックします。
![](/images/create-new-binding.png)

Y ボタンを押したら時計が表示されるように設定します。
（ボタンは各自で使用しているコントローラに合わせて設定してください）

**Y Button** の **+** をクリック
![](/images/y-button-plus.png)

**BUTTON** をクリック
![](/images/button.png)

**Click** の右の **None** を選択して、**wakeup** を割り当てる。
![](/images/y-button-click.png)
![](/images/wakeup.png)

左下のチェックマークをクリックして確定
![](/images/check-mark.png)

右下の **Replace Default Binding** をクリック
![](/images/replace-default-binding.png)

**Save** をクリック
![](/images/save-binding.png)

この時点で Action Manifest のデフォルトバインディングが自動的に追加されます。

```diff json:actions.json
{
   "action_sets" : [
      {
         "name" : "/actions/Watch",
         "usage" : "single"
      }
   ],
   "actions" : [
      {
         "name" : "/actions/Watch/in/WakeUp",
         "type" : "boolean"
      }
   ],
   "default_bindings" : [
      {
+        "binding_url" : "application_generated_unity_retryoverlay_exe_binding_oculus_touch.json",
+        "controller_type" : "oculus_touch"
      }
   ],
   "localization" : []
}

```

また、Action Manifest と同じディレクトリにデフォルトバインディングの設定ファイルが追加されます。プロジェクトをビルドしたときに、これらの設定ファイルが同梱されます。
![](/images/default-binding-file.png)

左上の **← BACK** をクリックして前の画面に戻ると、新しい設定が有効化されていることが確認できます。
![](/images/binding-added.png)

デフォルトバインディング設定が作成できたので、ウィンドウを閉じます。

Unity の SteamVR Input のウィンドウも閉じます。
このとき変更を保存するかのダイアログが表示されるのですが、**保存せずに Close を選択**してください。
![](/images/close-binding-dialog.png)

このチュートリアルを作成している環境だと、このタイミングで Save を押すと Action Manifest の `default_bindings` が空になっているデータで上書きされてしまうためです。
（次回以降バインディング画面を開くときに出てくるダイアログで保存する際には問題ありませんでした。）

:::details 他のコントローラのデフォルトバインディングは？
バインディングの設定画面で、右側のコントローラ名をクリックすると、他のコントローラのバインディング設定も作れます。
![](/images/click-controller.png)

他のコントローラのデフォルトバインディングを作ると、Action Manifest の `default_bindings` に追加されます。

```diff json
{
    "action_sets" : [
       {
          "name" : "/actions/NewSet",
          "usage" : "leftright"
       }
    ],
    "actions" : [
       {
          "name" : "/actions/NewSet/in/NewAction",
          "type" : "boolean"
       }
    ],
    "default_bindings" : [
      {
         "binding_url" : "application_generated_unity_steamvr_inputbindingtest_exe_binding_oculus_touch.json",
         "controller_type" : "oculus_touch"
      },
+     {
+        "binding_url" : "application_generated_unity_steamvr_inputbindingtest_exe_binding_vive_controller.json",
+        "controller_type" : "vive_controller"
+     }
   ],
   "localization" : []
}
```

現在の SteamVR では、1 つデフォルトバインディングを作っておくと、他のコントローラは近いボタンに自動でリマップされる機能があるため、作成が必須というわけではないです。
![](/images/remapped-binding.png)
:::

### スクリプトの作成

`Scripts` フォルダの下に `InputController.cs` を新規作成します。
このスクリプトにコントローラの入力関連の処理を書いていきます。

Hierarchy を**右クリック > CreateEmpty** で空のオブジェクトを作成し、オブジェクト名を `InputController` に変更します。
Project ウィンドウから `InputController.cs` を `InputController` へドラッグしてスクリプトを追加します。
![](/images/add-input-controller.png)

### OpenVR の初期化、クリーンアップを追加

`InputController.cs` へ以下のコードをコピーします。

```cs:InputController.cs
using System;
using UnityEngine;
using Valve.VR;

public class InputController : MonoBehaviour
{
    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
    }

    private void Destroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## アクションマニフェストパスの指定

アプリケーションの開始時に Action Manifest ファイルのパスを [SetActionManifestPath()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html#Valve_VR_CVRInput_SetActionManifestPath_System_String_) で指定します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/SteamVR-Input#api-documentation) を参照）
Action Manifest は **StreamingAssets/SteamVR/actions.json** に生成されているので、そのパスを指定します。

```diff cs:InputController.cs
using System;
using UnityEngine;
using Valve.VR;

public class InputController : MonoBehaviour
{
    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();

+       var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
+       if (error != EVRInputError.None)
+       {
+           throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
+       }
    }

    private void Destroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## アクションセットハンドルの取得

アプリケーションには複数のアクションをまとめたアクションセットをいくつか登録でき、どのアクションセットを使うかをハンドルで指定します。
アクションセットのハンドルは [GetActionSetHandle()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html#Valve_VR_CVRInput_GetActionSetHandle_System_String_System_UInt64__) で取得します。

アクションセットは `/actions/[アクションセット名]` という文字列で指定します。
今回の場合は `/actions/Watch` です。

```diff cs:InputController.cs
public class InputController : MonoBehaviour
{
+   ulong actionSetHandle = 0;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();

        var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
        if (error != EVRInputError.None)
        {
            throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
        }

+       error = OpenVR.Input.GetActionSetHandle("/actions/Watch", ref actionSetHandle);
+       if (error != EVRInputError.None)
+       {
+           throw new Exception("アクションセット /actions/Watch の取得に失敗しました: " + error);
+       }
    }

    private void Destroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## アクションハンドルの取得

次はアクションのハンドルです。[GetActionHandle()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html#Valve_VR_CVRInput_GetActionHandle_System_String_System_UInt64__) で取得します。
アクションは `/actions/[アクションセット名]/in/[アクション名]` で指定します。
今回の場合は `/actions/Watch/in/WakeUp` です。

```diff cs:InputController.cs
public class InputController : MonoBehaviour
{
    ulong actionSetHandle = 0;
+   ulong actionHandle = 0;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();

        var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
        if (error != EVRInputError.None)
        {
            throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
        }

        error = OpenVR.Input.GetActionSetHandle("/actions/Watch", ref actionSetHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("アクションセット /actions/Watch の取得に失敗しました: " + error);
        }

+       error = OpenVR.Input.GetActionHandle($"/actions/Watch/in/WakeUp", ref actionHandle);
+       if (error != EVRInputError.None)
+       {
+           throw new Exception("アクション /actions/Watch/in/WakeUp の取得に失敗しました: " + error);
+       }
    }

    private void Destroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

## アクションの状態を更新

各フレームの最初で、各アクションの入力状態を最新に更新します。

### 更新するアクションセットの準備

アクションの更新は、アクションセットの単位で指定します。
複数のアクションセットを [VRActiveActionSet_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VRActiveActionSet_t.html) 型の配列として渡す形式になっています。
今回はアクションセットが `/actions/Watch` だけなので、長さが 1 の `VRActionActiveSet_t[]` を作ります。

```diff cs:InputController.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();

    var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
    if (error != EVRInputError.None)
    {
        throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
    }

    error = OpenVR.Input.GetActionSetHandle("/actions/Watch", ref actionSetHandle);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクションセット /actions/Watch の取得に失敗しました: " + error);
    }

    error = OpenVR.Input.GetActionHandle($"/actions/Watch/in/WakeUp", ref actionHandle);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクション /actions/Watch/in/WakeUp の取得に失敗しました: " + error);
    }
}

+ private void Update()
+ {
+     // 更新したいアクションセットのリスト
+     var actionSetList = new VRActiveActionSet_t[]
+     {
+         // 今回は Watch アクションセットのみ
+         new VRActiveActionSet_t()
+         {
+             // Watch アクションセットのハンドルを渡す
+             ulActionSet =  actionSetHandle,
+             ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
+         }
+     };
+ }

 ～省略～
```

`ulRestrictedToDevice` は特定のデバイスからのみ入力できるようにするもので、左右のコントローラに異なるアクションセットをバインドするときなどに使われます。基本的には使わず `k_ulInvalidInputValueHandle` をセットします。

### 状態の更新

アクションの更新に使うのは [UpdateActionState()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html) です。
先ほど作ったアクションセットの配列を渡します。

```diff cs:InputController.cs
private void Update()
{
    var actionSetList = new VRActiveActionSet_t[]
    {
        new VRActiveActionSet_t()
        {
            ulActionSet =  actionSetHandle,
            ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
        }
    };

+   var activeActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VRActiveActionSet_t));
+   var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
+   if (error != EVRInputError.None)
+   {
+       throw new Exception("アクションの状態の更新に失敗しました: " + error);
+   }
}
```

第 2 引数の `activeActionSize` は `VRActiveActionSet_t` の構造体のサイズ（バイト数）です。

## アクションの値を取得

アクションの状態を更新したら、各アクションの値を取得します。
アクションの種類によって使用する関数が変わります。

**GetDigitalActionData()**
ボタンを押しているかどうかなどの On/Off の値を取得する関数

**GetAnalogActionData()**
スティックの方向やトリガーの引き具合などの連続的な値を取得する関数

**GetPoseActionDataRelativeToNow()**
**GetPoseActionDataForNextFrame()**
コントローラの位置や角度などの姿勢情報を取得する関数

今回はボタンの On/Off なので `GetDigitalActionData()` で値を取得します。
既に取得してある `WakeUp` アクションのハンドルを使います。

```diff cs:InputController.cs
private void Update()
{
    var actionSetList = new VRActiveActionSet_t[]
    {
        new VRActiveActionSet_t()
        {
            ulActionSet =  actionSetHandle,
            ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
        }
    };

    var activeActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VRActiveActionSet_t));
    var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクションの状態の更新に失敗しました: " + error);
    }

+   var result = new InputDigitalActionData_t();
+   var digitalActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(InputDigitalActionData_t));
+   error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
+   if (error != EVRInputError.None)
+   {
+       throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
+   }
}

```

第 3 引数の `digitalActionSize` は `InputDigitalActionData_t` 型の構造体のサイズ（バイト数）です。
第 4 引数は `ulRestrictToDevice` で、基本的に使わないので `OpenVR.k_ulInvalidInputValueHandle` をセットします。

取得結果は [InputDigitalActionData_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.InputDigitalActionData_t.html) 型です。

## アクションが実行されたか判定

`GetDigitalActionData()` で取得される `InputDigitalActionData_t` 型を見てみます。

```cpp
struct InputDigitalActionData_t
{
	bool bActive;
	VRInputValueHandle_t activeOrigin;
	bool bState;
	bool bChanged;
	float fUpdateTime;
};
```

アクションの On/Off 状態が `bState` に入ります。
On/Off が切り替わったフレームのみ `bChanged` が `true` になります。
これらを使ってアクションが実行されたことを判定します。

```diff cs:InputController.cs
private void Update()
{
    var actionSetList = new VRActiveActionSet_t[]
    {
        new VRActiveActionSet_t()
        {
            ulActionSet =  actionSetHandle,
            ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
        }
    };

    var activeActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VRActiveActionSet_t));
    var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクションの状態の更新に失敗しました: " + error);
    }

    var result = new InputDigitalActionData_t();
    var digitalActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(InputDigitalActionData_t));
    error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
    if (error != EVRInputError.None)
    {
        throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
    }

+   if (result.bState && result.bChanged)
+   {
+       Debug.Log("WakeUp が実行されました");
+   }
}
```

プログラムを実行して Y ボタンを押すと、ログが表示されることを確認します。
![](/images/controller-button-log.png)

## イベントの作成

### Unity Event への通知

`InputController.cs` に `WakeUp` アクションが実行されたことを通知するための `UnityEvent` を作成し、`Invoke()` で呼び出します。

```diff cs:InputController.cs
using System;
using UnityEngine;
using Valve.VR;
+ using using UnityEngine.Events;

public class InputController : MonoBehaviour
{
+   public UnityEvent OnWakeUp;

    private ulong actionSetHandle = 0;
    private ulong actionHandle = 0;

    ～省略～

    private void Update()
    {
        var actionSetList = new VRActiveActionSet_t[]
        {
            new VRActiveActionSet_t()
            {
                ulActionSet =  actionSetHandle,
                ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
            }
        };

        var activeActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VRActiveActionSet_t));
        var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
        if (error != EVRInputError.None)
        {
            throw new Exception("アクションの状態の更新に失敗しました: " + error);
        }

        var result = new InputDigitalActionData_t();
        var digitalActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(InputDigitalActionData_t));
        error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
        }

        if (result.bState && result.bChanged)
        {
-           Debug.Log("WakeUp が実行されました");
+           OnWakeUp.Invoke();
        }
    }
```

### イベントの割り当て

時計のオーバーレイ `WatchOverlay.cs` で `WakeUp` アクションが実行されたときに呼ばれるメソッドを作ります。

```diff cs:WatchOverlay.cs
public class WatchOverlay : MonoBehaviour
{

    ～省略～

+   public void OnWakeUp()
+   {
+       // 時計を表示する処理
+   }
}
```

Hierarchy で `InputController` のインスペクタを開き `OnWakeUp()` に `WatchOverlay` の `OnWakeUp()` を設定します。
![](/images/attach-onwakeup-event.png)

## 時計の表示・非表示処理を作成

`WakeUp` アクションが発生したときだけ表示されるように `WatchOverlay.cs` を編集します。

### 初期状態を非表示にする

`Start()` から `ShowOverlay()` を削除して、オーバーレイを非表示にします。

```diff cs:WatchOverlay.cs
private void Start()
{
    OpenVRUtil.System.InitOpenVR();
    overlayHandle = Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay");

    Overlay.FlipOverlayVertical(overlayHandle);
    Overlay.SetOverlaySize(overlayHandle, size);
-   Overlay.ShowOverlay(overlayHandle);
}
```

### WakeUp が実行されたら表示する

WakeUp アクションが実行されたらオーバーレイを表示します。

```diff cs:WatchOverlay.cs
public void OnWakeUp()
{
+   Overlay.ShowOverlay(overlayHandle);
}
```

### 非表示にする関数を追加

`OpenVRUtil.cs` に `HideOverlay()` を作成します。

```diff cs:OpenVRUtil.cs
public static void ShowOverlay(ulong handle)
{
    var error = OpenVR.Overlay.ShowOverlay(handle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイの表示に失敗しました: " + error);
    }
}

+ public static void HideOverlay(ulong handle)
+ {
+     var error = OpenVR.Overlay.HideOverlay(handle);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの非表示に失敗しました: " + error);
+     }
+ }

public static void SetOverlayFromFile(ulong handle, string path)
{
    var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
    if (error != EVROverlayError.None)
    {
        throw new Exception("画像ファイルの描画に失敗しました: " + error);
    }
}
```

### 一定時間後に非表示にする

`WakeUp` で表示したあと、3 秒後に非表示にする処理を作ります。
今回は Unity の [Croutine](https://docs.unity3d.com/Manual/Coroutines.html) を使って 3 秒待つ処理を作ります。

```diff cs:WatchOverlay.cs
using System;
+ using System.Collections;
using UnityEngine;
using Valve.VR;
using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
+   private Coroutine sleepCoroutine;

    ～省略～

    public void OnWakeUp()
    {
        Overlay.ShowOverlay(overlayHandle);
+       if (sleepCoroutine != null)
+       {
+           StopCoroutine(sleepCoroutine);
+       }
+       sleepCoroutine = StartCoroutine(Sleep());
    }

+   private IEnumerator Sleep()
+   {
+       yield return new WaitForSeconds(3);
+       Overlay.HideOverlay(overlayHandle);
+   }
}
```

プログラムを実行して、Y ボタンを押したときに時計が表示されることを確認します。
![](/images/3sec-display.gif)

## 完成 🎉

これで時計アプリは完成です！
チュートリアルはここまでなので、最後のコード整理はお任せします。

ここまでの内容を振り返ってみると

- OpenVR の初期化
- オーバーレイの作成
- 画像ファイルの表示
- 大きさと位置の変更
- デバイス追従
- カメラ映像の表示
- ダッシュボードの作成
- イベントの処理
- コントローラの操作

と、オーバーレイアプリケーション開発の入門的なところは押さえられたかなと思います。
OpenVR には他にも色々な API があるので、何ができそうか調べつつ、オリジナルツールの開発に挑戦してみてはいかがでしょうか。

次のページでは、更に詳しく調べる際にどこを見ればよいかなど、参考情報をまとめてあります。
