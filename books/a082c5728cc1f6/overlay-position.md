---
title: "オーバーレイの表示位置と大きさ"
free: false
---

オーバーレイは VR 空間のどこにどのくらいの大きさで表示するかを指定できます。


## オーバーレイの大きさ
まずはオーバーレイの大きさを変更してみます。
大きさは SetOverlayWidthinMeters() で設定できます。
オーバーレイの幅を m 単位で指定します。高さは表示する画像のアスペクト比に合わせて自動的に計算されます。
今回は正方形の画像をサンプルで使用しているため、縦横同サイズとなります。
とりあえず幅 1m に設定してみましょう。

Wiki はこちら
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayWidthInMeters

SteamVR Plugin はこちら
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayWidthInMeters_System_UInt64_System_Single_


オーバーレイの横幅を 0.5 m にしてみます。

```diff cs:FileOverlay.cs
private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

+   var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 0.5f);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);
}
```

先程よりも小さくオーバーレイが表示されます。（デフォルトでは 1m で表示されます。）
![](/images/small-overlay.jpg)


## オーバーレイの表示位置
次にオーバーレイの表示位置を設定します。
表示位置は SteamVR のプレイエリアの原点（床の中心）を基準とする SetOverlayTransform() と、HMD やコントローラの位置を基準とする SetOverlayTransformTrackedDeviceRelative() があります。
TODO: SetOverlayTransformTrackedDeviceComponent もある

空間の特定の位置に固定するか、コントローラや HMD に追従させるかによって使い分けます。
最初は絶対座標で、空間の特定の場所に固定で表示してみましょう。

### 絶対位置で指定する

位置の指定は 3DCG では同じもの変換行列によって行われます。
引数には座標を変換するための行列 HmdMatrix を与えます。

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute

SteamVR Plugin
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformAbsolute

```cs
EVROverlayError SetOverlayTransformAbsolute(ulong ulOverlayHandle, ETrackingUniverseOrigin eTrackingOrigin, ref HmdMatrix34_t pmatTrackingOriginToOverlayTransform)
```

`ulOverlayHandle` は CreateOverlay() で作成したオーバーレイのハンドルです。

`ETrackingUniverseOrigin etrackingOrigin` はトラッキングの基準になる原点です。
`ETrackingUniverseOrigin.TrackingStanding` が SteamVR のプレイエリアの床の中心が原点となります。今回はこちらを使います。
`ETrackingUniverseOrigin.TrackingUniverseSeated` はユーザが座った状態でポジションをリセットした HMD の位置が原点になります。

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.ETrackingUniverseOrigin.html


`HmdMatrix34_t pmatTrackingOriginToOverlayTransform` が変換行列です。
直接行列の各要素を指定して作成することもできますが、SteamVR Unity Plugin に position (Vector3) と rotation (Quarternion) から HmdMatrix34_t の変換行列を作るユーティリティが入っているので、今回はこちらを使います。

オーバーレイの表示位置を、プレイエリアの中心から正面方向に 3 m、上方向に 2 m にして、オーバーレイを Z 軸周りに 45 度回転させてみます。

```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
    }

+   var position = new Vector3(0, 2, 3);
+   var rotation = Quaternion.Euler(0, 0, 45);
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();
+   error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    ShowOverlay(overlayHandle);
}
```

![](/images/far-overlay-position.jpg)
*床から高さ 2m、正面方向に 3m、Z 軸周りに 45 度回転*

一旦、サイズと位置の処理を関数に分けて整理しておきます。

```diff cs:FileOverlay.cs
public class FileOverlay : MonoBehaviour
{
    // ...中略...

    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
+       SetOverlaySize(overlayHandle, 0.5f);
    
+       var position = new Vector3(0, 2, 3);
+       var rotation = Quaternion.Euler(0, 0, 45);
+       SetOverlayTransformAbsolute(overlayHandle, position, rotation);
    
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);
        
-       var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
-       }
-   
-       var position = new Vector3(0, 2, 3);
-       var rotation = Quaternion.Euler(0, 0, 45);
-       var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-       var matrix = rigidTransform.ToHmdMatrix34();
-       error = OpenVR.Overlay.SetOverlayTransformAbsolute(overlayHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-       }
    
        ShowOverlay(overlayHandle);
    }

    // ...中略...

+   private void SetOverlaySize(ulong handle, float size)
+   {
+       var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+       }
+   }
+  
+   private void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
+   {
+       var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+       var matrix = rigidTransform.ToHmdMatrix34();
+       var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
+       if (error != EVROverlayError.None)
+       {
+           throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+       }
+   }
}
```


### デバイスからの相対位置で指定する

#### HMD を基準にする
HMD やコントローラなどのデバイスを基準とした相対的な位置指定ができます。
常にユーザの正面に表示したり、コントローラにくっつけることができます。

相対位置の指定では関数 SetOverlayTransformTrackedDeviceRelative() を使用します。

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayTransformTrackedDeviceRelative

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayTransformTrackedDeviceRelative_System_UInt64_System_UInt32_Valve_VR_HmdMatrix34_t__

どのデバイスを基準にするかは、デバイスを識別するための番号で指定します。
HMD の場合は `OpenVR.k_unTrackedDeviceIndex_Hmd` の定数で定義されていて 0 で固定です。

https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h#L251-L256

https://github.com/ValveSoftware/openvr/blob/master/headers/openvr_api.cs#L7706-L7709


HMD の正面 2m 先にオーバーレイを表示してみます。

```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

-   var position = new Vector3(0, 2, 3);
-   var rotation = Quaternion.Euler(0, 0, 45);
-   SetOverlayTransformAbsolute(overlayHandle, position, rotation);

+   var position = new Vector3(0, 0, 2);
+   var rotation = Quaternion.Euler(0, 0, 0);
+   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+   var matrix = rigidTransform.ToHmdMatrix34();
+   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
+   if (error != EVROverlayError.None)
+   {
+       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
} 
```

オーバーレイの相対位置指定を関数に分けて整理しておきます。
```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

    var position = new Vector3(0, 0, 2);
    var rotation = Quaternion.Euler(0, 0, 0);
+   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);
-   var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
-   var matrix = rigidTransform.ToHmdMatrix34();
-   var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, ref matrix);
-   if (error != EVROverlayError.None)
-   {
-       throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
-   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
} 

...

+ private void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
+ {
+     var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
+     var matrix = rigidTransform.ToHmdMatrix34();
+     var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(handle, deviceIndex, ref matrix);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
+     }
+ }
```

![](/images/relative-transform-hmd.gif)
*HMD に追従するオーバーレイ*

:::message
Unity は左手系、OpenVR は右手系で座標系が異なります。
OpenVR では +y が上、+x が右、-z が前方です。

SteamVR Unity Plugin のユーティリティが座標系を変換してくれているため +z を前方として指定していますが、自前で変換行列を作る場合には z 軸が逆向きになる点に注意してください。
:::

#### コントローラを基準にする

同様に SetOverlayTransformTrackedDeviceRelative() にコントローラの DevceIndex を与えれば、コントローラの位置を基準にした位置指定ができます。
コントローラの DeviceIndex は GetTrackedDeviceIndexForControllerRole() で取得できます。
コントローラが接続されていない場合など、取得に失敗すると k_unTrackedDeviceIndexInvalid が返ってきます。

https://github.com/ValveSoftware/openvr/blob/v2.2.3/headers/openvr.h#L2336-L2337

https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVRSystem.html#Valve_VR_CVRSystem_GetTrackedDeviceIndexForControllerRole_Valve_VR_ETrackedControllerRole_

左手の場合は ETrackedControllerRole.LeftHand, 右手の場合は EtrackedControllerRole.RightHand を渡します。
ここでは左手に表示してみます。

https://github.com/ValveSoftware/openvr/blob/v2.2.3/headers/openvr.h#L272-L282

https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.ETrackedControllerRole.html

先程の HMD の代わりに、左手のコントローラの番号を取得して、コントローラに追従させてみます。

```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
    SetOverlaySize(overlayHandle, 0.5f);

-   var position = new Vector3(0, 0, 2);
-   var rotation = Quaternion.Euler(0, 0, 0);
-   SetOverlayTransformRelative(overlayHandle, OpenVR.k_unTrackedDeviceIndex_Hmd, position, rotation);
+   var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
+       var position = new Vector3(0, 0, 2);
+       var rotation = Quaternion.Euler(0, 0, 0);
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);
    
    ShowOverlay(overlayHandle);
}
```

SteamVR 上で左手のコントローラが接続されていることを確認してからプログラムを実行してください。HMD の代わりにコントローラに追従してオーバーレイが表示されます。
![](/images/left-controller-connected.png)


![](/images/controller-tracked-overlay.gif)
*コントローラに追従するオーバーレイ*


#### 位置の調整

時計アプリを作るため、手首の丁度いい位置にオーバーレイが来るように調整します。
プログラムの実行中に Unity のインスペクタを使って実際の位置や大きさを確認しながら調整できるようにしてみます。

大きさ、位置、角度をインスペクタから変更できるようにします。
Rande() 属性でスライダーを使えるようにします。

```diff cs:FileOverlay.cs
public class FileOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

+   [Range(0, 0.5f)] public float size = 0.5f;
+   [Range(-0.5f, 0.5f)] public float x;
+   [Range(-0.5f, 0.5f)] public float y;
+   [Range(-0.5f, 0.5f)] public float z;
+   [Range(0, 360)] public int rotationX;
+   [Range(0, 360)] public int rotationY;
+   [Range(0, 360)] public int rotationZ;

...
```
![](/images/inspector-overlay-position.png)


サイズと位置をインスペクタの値で更新します。
```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
-   SetOverlaySize(overlayHandle, 0.5f);
+   SetOverlaySize(overlayHandle, size);

    var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+   if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+   {
-       var position = new Vector3(0, 0, 2);
-       var rotation = Quaternion.Euler(0, 0, 0);
+       var position = new Vector3(x, y, z);
+       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+   }

    SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

    ShowOverlay(overlayHandle);
}
```

実行中に値を変更できるようにするため、Update() を追加して、サイズ指定、コントローラ取得、位置指定のコードをコピーします。
これはオーバーレイの表示位置を調整するためのもので、調整後に削除するコードです。
```diff cs:FileOverlay.cs
...

+ private void Update()
+ {
+     SetOverlaySize(overlayHandle, size);
+ 
+     var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+     if (leftControllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
+     {
+         var position = new Vector3(x, y, z);
+         var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
+         SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+     }
+ }

...
```

プログラムを実行した状態でスライダーを操作して、ちょうどいい位置にオーバーレイが来るように調整してください。

SteamVR のダッシュボードを開き、デスクトップ画面を表示して、VR 内から直接 Unity のスライダーを操作すると調整しやすいかと思います。
![](/images/steamvr-dashboard-unity-window.jpg)
![](/images/adjust-in-vr.gif)
*SteamVR ダッシュボードから Unity を操作*

他には SteamVR のメニューから Display VR View で HMD の映像を表示することもできます。
![](/images/display-overlay-view-menu.png)
![](/images/adjust-with-vrview.gif)
*HMD を被りたくないときはこれで*

調整できたら、**プログラムを終了させずに**インスペクタの各パラメータの数値をメモしておいてください。
```
パラメータの例
size 0.08
x -0.044
y 0.015
z -0.131
rotationX 154
rotationY 262
rotationZ 0
```

各パラメータをメモしたらプログラムを終了します。
インスペクタに各パラメータを入力してください。
![](/images/set-overlay-position-inspector.png)

プログラムを実行して、左手首にオーバーレイが表示されていることを確認します。
![](/images/overlay-fixed-position.jpg)

パラメータを調整するために追加した Update() を削除しておきます。
```diff cs:FileOverlay.cs
- private void Update()
- {
-     SetOverlaySize(overlayHandle, size);
- 
-     var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-     var position = new Vector3(x, y, z);
-     var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ); 
-     SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
- }
```

#### コントローラが接続されていない場合の対処

現在のコードではプログラムが実行された時にコントローラが接続されていなければいけません。
実行中にコントローラが接続・切断される場合があるため、コントローラの取得とコントローラからの相対位置指定を Update() の中に移動しておきます。

クラスのメンバに左手のコントローラの Device Index を保存するように変更します。
```diff cs:FileOverlay.cs
    private void Start()
    {
        InitOpenVR();
        overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
        SetOverlaySize(overlayHandle, size);

-       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
-       var position = new Vector3(x, y, z);
-       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
-       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

-       ShowOverlay(overlayHandle);
    }

+   private void Update()
+   {
+       var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
+       var position = new Vector3(x, y, z);
+       var rotation = Quaternion.Euler(rotationX, rotationY, rotationZ);
+       SetOverlayTransformRelative(overlayHandle, leftControllerIndex, position, rotation);
+       ShowOverlay(overlayHandle);
+   }

```

※ 今回はシンプルに Update() 内で処理しますが、VREvent_TrackedDeviceRoleChanged イベントを使って処理することも可能です。

:::message
GetTrackedDeviceIndexForControllerRole() でコントローラの Index は取れるけどメソッドは deprecated で IVRInput を使うと書いてある。
IVRInput でコントローラの Index は取得できる？
-> Index は使用せずに、SteamVR Input のコントローラの Poses に割り当てたアクションから姿勢を取得するっぽい。で、Tracked Relative じゃなくて Absolute で出す？
後で入力を扱う時に説明することにして、ここでは deprecated な API を使って取得することにする。OVRLE でもそうしているし。
:::

## 最終的なコード
```cs:FileOverlay.cs
using System;
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
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
        overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
        SetOverlaySize(overlayHandle, size);
        
        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);

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
    }

    private void OnDestroy()
    {
        DestroyOverlay(overlayHandle);
        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System != null)
        {
            Debug.Log("OpenVR は既に初期化されています");
            return;
        }

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
}
```

## 編集メモ
Wiki を貼る必要はない。（それが読める人はこれ読まないから。）
参考としてリンクは入れてもいいけど最後の方にまとめるとかして、本文のじゃまにならないようにする。
このチュートリアルは、これだけで独立するように。

https://github.com/ValveSoftware/openvr/issues/1551

OpenGL と DirectX の座標系
https://tech.drecom.co.jp/knowhow-about-unity-coordinate-system/

https://qiita.com/suzuryo3893/items/9e543cdf8bc64dc7002a

https://github.com/ValveSoftware/openvr/issues/1551