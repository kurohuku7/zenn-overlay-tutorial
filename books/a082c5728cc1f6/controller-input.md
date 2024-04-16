---
title: "コントローラの入力"
free: false
---

時計は基本的に非表示にしておき、特定のボタンが押されたときに数秒間だけ表示するように変更してみます。

![](/images/3sec-display.gif)


Unity で VR のコントローラの入力を取る方法は色々ありますが、ここでは [OpenVR の Input API (SteamVR Input)](https://github.com/ValveSoftware/openvr/wiki/SteamVR-Input) を使った入力方法で作ってみます。

## 事前設定
SteamVR の **Setting** を開きます。
![](/images/steamvr-setting.png)

**Advanced Settings** を **Show** に切り替えます。
![](/images/advanced-settings.png)

**Developer** の **Enable debugging options in the input binding user interface** を **On** にします。
![](/images/enable-debugging-options.png)

## Action Manifest の作成
SteamVR Input では、あらかじめアプリ側で使用するアクションを作成しておき、SteamVR Input の設定画面でボタンやトリガー等とアクションを関連付けることで、アプリを操作します。
プラットフォームレベルで用意されたキーコンフィグのようなイメージです。

まず、アプリケーションで使用するアクションを JSON 形式でリトアップした [Action Manifest](https://github.com/ValveSoftware/openvr/wiki/Action-manifest) を作成します。
SteamVR Plugin には Action Manifest を簡単に作れる機能があるので、これを使います。

### Action Manifest の生成
Unity のメニューから **Window > SteamVR Input** を選択します。
![](/images/menu-steamvr-input.png)

初回起動時に Action Manifest のサンプルファイルを使用するか聞かれます。
今回は一から作成するので **No** を選択します。
![](/images/create-default-actionmanifest.png)

アクションセット名を **Watch** に変更し、その下のドロップダウンを **per hand** に変更します。
![](/images/watch-action.png)

下の方にある **Actions** の **In** というボックス内にある **NewAction** の文字クリックします。
右側にアクションの詳細が表示されるので、**Name** を **WakeUp** に変更します。これが時計を表示状態に切り替えるためのアクションになります。
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

## デフォルトバインディングの設定
右下の **Open binding UI** ボタンを押します。
![](/images/open-binding-ui.png)

VR のコントローラが表示されない場合は、HMD を SteamVR に接続されているか確認してください。

**Create New Binding** をクリックします。
![](/images/create-new-binding.png)

Y ボタンを押したら時計が表示されるように設定します。
接続されているコントローラによってボタンが異なるので、使用しているコントローラに合わせてボタンは変えてください。

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

右上の × ボタンでバインディングの設定ウィンドウを閉じます。
Unity の SteamVR Input のウィンドウも閉じます。

### スクリプトの作成
`Scripts` フォルダの下に `InputController.cs` を新規作成します。
Hierarchy を**右クリック > CreateEmpty** して、オブジェクト名を `InputController` に変更します。
Project ウィンドウから `InputController.cs` を `InputController` へドラッグしてスクリプトを追加します。
このスクリプトにコントローラの入力関連の処理を書いていきます。
![](/images/add-input-controller.png)

### OpenVR の初期化、クリーンアップコードを追加
`InputController.cs` を開き以下のコードをコピーします。
OpenVR Input API を使うため、OpenVR の初期化をしておく必要があります。
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
先ほど生成した **actions.json** が **StreamingAssets** に入っているので、そのパスを指定します。

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
アクションセットのハンドルを [GetActionSetHandle()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html#Valve_VR_CVRInput_GetActionSetHandle_System_String_System_UInt64__) で取得します。
アプリケーションはいくつかのアクションをまとめたアクションセットを複数持つことができます。
今回は `/actions/Watch` という 1 つのアクションセットのみ作成したので、そのハンドルを読み込みます。
```diff cs:InputController.cs
void Start()
{
    OpenVRUtil.System.InitOpenVR();

    var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
    if (error != EVRInputError.None)
    {
        throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
    }

+   ulong actionSetHandle = 0;
+   error = OpenVR.Input.GetActionSetHandle("/actions/Watch", ref actionSetHandle);
+   if (error != EVRInputError.None)
+   {
+       throw new Exception("アクションセット /actions/Watch の取得に失敗しました: " + error);
+   }
}
```

アクションセット名は、先ほど生成した actions.json にも記載されています。
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

## アクションハンドルの取得
アクションセットに登録されている個々のアクションを [GetActionHandle()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRInput.html#Valve_VR_CVRInput_GetActionHandle_System_String_System_UInt64__) で取得します。
今回は `WakeUp` アクションのみなので、それを取得します。
```diff cs:InputController.cs
void Start()
{
    OpenVRUtil.System.InitOpenVR();

    var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
    if (error != EVRInputError.None)
    {
        throw new Exception("Action Manifest パスの指定に失敗しました: " + error);
    }

    ulong actionSetHandle = 0;
    error = OpenVR.Input.GetActionSetHandle("/actions/Watch", ref actionSetHandle);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクションセット /actions/Watch の取得に失敗しました: " + error);
    }

+   ulong actionHandle = 0;
+   error = OpenVR.Input.GetActionHandle($"/actions/Watch/in/WakeUp", ref actionHandle);
+   if (error != EVRInputError.None)
+   {
+       throw new Exception("アクション /actions/Watch/in/WakeUp の取得に失敗しました: " + error);
+   }
}
```

## アクションの状態を更新
### 更新するアクションセットの準備
どのアクションセットの状態を取得するかを [VRActiveActionSet_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.VRActiveActionSet_t.html) 型で準備します。
更新するアクションセットの配列を作成します。今回はアクションセットが `/actions/Watch` だけを入れた配列になります。
先ほど取得したアクションセットのハンドルを使用します。

```diff cs:InputController.cs
private void Update()
{
+   var actionSetList = new VRActiveActionSet_t[]
+   {
+       new VRActiveActionSet_t()
+       {
+           ulActionSet =  actionSetHandle,
+           ulRestrictedToDevice = OpenVR.k_ulInvalidInputValueHandle,
+       }
+   };
}
```

`ulRestrictedToDevice` は左右のコントローラに異なるアクションセットをバインドするときに使うもので、基本的に使わないので `k_ulInvalidInputValueHandle` をセットします。

### 状態の更新
作成したアクションセットのリストを `UpdateActionState()` に渡してアクションの入力状態を更新します。

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
+
+   var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
+   if (error != EVRInputError.None)
+   {
+       throw new Exception("アクションの状態の更新に失敗しました: " + error);
+   }
}
```

第 2 引数の `activeActionSize` には `VRActiveActionSet_t` の大きさ（バイト数）を渡します。

## 入力値の取得
アクションの状態を更新したら、現在の入力状態の値を取得します。
アクションの値の種類によって取得に使用する関数が代わります。

ボタンを押しているかどうかなどの On/Off の値を取るタイプのアクションは `GetDigitalActionData()`
スティックの方向やトリガーの引き具合などのデータは `GetAnalogActionData()`
コントローラの座標や角度などの姿勢情報をアクションとしている場合は `GetPoseActionData()`

を使います。
今回はボタンの On/Off のアクションなので `GetDigitalActionData()` で値を取得します。
先程取得した `WakeUp` アクションのハンドルを使って取得するアクションを指定します。

```diff cs:InputController.cs
public class InputController : MonoBehaviour
{
    private ulong actionSetHandle = 0;
    private ulong actionHandle = 0;

    private uint activeActionSize;
+   private uint digitalActionSize;

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
        
        var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
        if (error != EVRInputError.None)
        {
            throw new Exception("アクションの状態の更新に失敗しました: " + error);
        }
        
+       var result = new InputDigitalActionData_t();
+       error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
+       if (error != EVRInputError.None)
+       {
+           throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
+       }
    }

    ～省略～
```

`digitalActionSize` は `InputDigitalActionData_t` のサイズ（バイト数）です。
結果は [InputDigitalActionData_t](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.InputDigitalActionData_t.html) 型で取得されます。

## ボタンが入力されたことを検出
入力結果として取得できる `InputDigitalActionData_t` の中身を見てみます。

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

ボタンが押されているかどうかが `bState` に入ります。
ボタンの入力状態が変わったフレームのみ `bChanged` が `true` になります。
これを使って、ボタンが押されたことを検出してみます。

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
    
    var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
    if (error != EVRInputError.None)
    {
        throw new Exception("アクションの状態の更新に失敗しました: " + error);
    }
    
    var result = new InputDigitalActionData_t();
    error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
    if (error != EVRInputError.None)
    {
        throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
    }

+   if (result.bState && result.bChanged)
+   {
+       Debug.Log("ボタンが押されました");
+   }
}
```

プログラムを実行して Y ボタンを押すと、ログが表示されることを確認します。
![](/images/controller-button-log.png)

## イベントの作成

### Unity Event の作成
`InputController.cs` に WakeUp アクションが実行されたことを通知するイベントを作成します。

```diff cs:InputController.cs
public class InputController : MonoBehaviour
{
+   public UnityEvent OnWakeUpClick; 
    
    private ulong actionSetHandle = 0;
    private ulong actionHandle = 0;
    private uint activeActionSize;
    private uint digitalActionSize;

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
        
        var error = OpenVR.Input.UpdateActionState(actionSetList, activeActionSize);
        if (error != EVRInputError.None)
        {
            throw new Exception("アクションの状態の更新に失敗しました: " + error);
        }
        
        var result = new InputDigitalActionData_t();
        error = OpenVR.Input.GetDigitalActionData(actionHandle, ref result, digitalActionSize, OpenVR.k_ulInvalidInputValueHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("WakeUp アクションのデータ取得に失敗しました: " + error);
        }

        if (result.bState && result.bChanged)
        {
-           Debug.Log("ボタンが押されました");
+           OnWakeUpClick.Invoke();
        }
    }
```

### イベントの割り当て
`WatchOverlay.cs` で WakeUp アクションが実行されたときの処理を作成します。
```diff cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;
using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;

    ～省略～

+   public void OnWakeUpClick()
+   {
+       // 時計を表示する処理
+   }
}
```

Unity のインスペクタで `InputController` の `OnWakeUpClick()` に `WatchOverlay` の `OnWakeUpClick()` を設定します。
![](/images/attach-onwakeup-event.png)


## 時計を非表示に変更
### 初期状態を非表示にする
`ShowOverlay()` を削除して時計のオーバーレイを非表示にします。
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

### ボタンが押されたら表示する
WakeUp アクションが実行されたらオーバーレイを表示します。
```diff cs:WatchOverlay.cs
public void OnWakeUpClick()
{
+   Overlay.ShowOverlay(overlayHandle);
}
```

### 非表示にする関数を追加
`OpenVRUtil.cs` に `HideOverlay()` を追加します。
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
3 秒後に非表示にします。ここでは [Croutine](https://docs.unity3d.com/Manual/Coroutines.html) を使って非表示処理を作成します。
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
+   private Coroutine sleepAfterWaitCoroutine;    

    ～省略～

    public void OnWakeUpClick()
    {
        Overlay.ShowOverlay(overlayHandle);
        if (sleepAfterWaitCoroutine != null)
        {
            StopCoroutine(sleepAfterWaitCoroutine);
        }
        sleepAfterWaitCoroutine = StartCoroutine(SleepAfterWait());
    }

    private IEnumerator SleepAfterWait()
    {
        yield return new WaitForSeconds(3);
        Overlay.HideOverlay();
    }
}
```

プログラムを実行して、Y ボタンを押したときに時計が表示されることを確認します。
![](/images/3sec-display.gif)

## 完成
これでチュートリアルの時計アプリは完成です！

ここまで

- OpenVR の初期化
- オーバーレイの作成
- 画像ファイルの表示
- 大きさと位置の変更
- デバイス追従
- カメラ映像の表示
- ダッシュボードの作成
- コントローラの操作

と、オーバーレイアプリ開発に必要となりそうな、基本的なところは押さえられているかなと思います。

次のページでは、更に詳しく調べる際にどこを見ればよいかなど、参考情報をまとめてあります。次は VR ゲームに持ち込めるオリジナルの便利ツールを作ってみてください。
