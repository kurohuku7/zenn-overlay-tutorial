---
title: "オーバーレイの作成・画像の表示"
free: false
---

それでは実際にオーバーレイを表示してみましょう。
画像ファイルをオーバーレイとして表示するシンプルな関数があるので、まずはこれを使ってみましょう。

### スクリプトを作成
Scripts フォルダ内に C# スクリプト FileOverlay.cs を新規作成します。

```cs:FileOverlay.cs
using UnityEngine;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

Hierarchy パネルを右クリック > Create Empty で空のオブジェクトをシーンに作成します。
オブジェクトの名前を FileOverlay に変更して、先程の FileOverlay.cs をコンポーネントとして追加してください。
![](https://storage.googleapis.com/zenn-user-upload/6621088b89bc-20240301.png)


OpenVR の API が定義されている名前空間 Valve.VR を追加しておきます。

```diff cs:FileOverlay.cs
using UnityEngine;
+ using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

### OpenVR の初期化とクリーンアップ
まず OpenVR API の初期化と、終了時の解放を追加します。
[Wiki](https://github.com/ValveSoftware/openvr/wiki/API-Documentation#initialization-and-cleanup)に書かれているように [Init()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Init_Valve_VR_EVRInitError__Valve_VR_EVRApplicationType_System_String_) と [Shutdown()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Shutdown) を呼び出します。

Init() の第 1 引数には、初期化時に発生したエラーを格納するための enum の EVRInitError の変数の参照を渡します。発生したエラーが変数に書き込まれまれます。エラーが発生しなかった場合は、EVRInitError.None が書き込まれます。

Init() の第 2 引数には、アプリケーションの種類を enum の EVRApplicationType で指定します。EVRApplicationType.VRApplication_Overlay を渡すとオーバーレイアプリケーションとして初期化することができます。一方、ゲームなどの通常のアプリケーションは VRApplication_Scene となります。

OpenVR API が初期化されているかどうかは、OpenVR.System が null かどうかで判定できます。

```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
+        if (OpenVR.System != null)
+        {
+            Debug.Log("OpenVR は既に初期化されています");
+        }
+        var initError = EVRInitError.None;
+        OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
+        if (initError != EVRInitError.None)
+        {
+            throw new Exception("OpenVR の初期化に失敗しました: " + initError);
+        }
+    }

+    private void OnDestroy()
+    {
+        if (OpenVR.System != null)
+        {
+            OpenVR.Shutdown();
+        }
+    }
}
```

OpenVR API の初期化が成功すると、各種 API にアクセスできるようになります。

- OpenVR.System
- OpenVR.Chaperone
- OpenVR.Compositor
- OpenVR.Overlay
- OpenVR.RenderModels
- OpenVR.Screenshots
- OpenVR.Input

試しに OpenVR.Overlay が使えるか確認してみましょう。

```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
+       Debug.Log(OpenVR.Overlay);

        if (OpenVR.System == null)
        {
            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
                return;
            }
        }

+       Debug.Log(OpenVR.Overlay);
    }

    private void OnDestroy()
    {
        if (OpenVR.System != null)
        {
            OpenVR.Shutdown();
        }
    }
}
```
![](https://storage.googleapis.com/zenn-user-upload/f7f7fe7912d6-20240306.png)

Init() によって OpenVR.Overlay に [CVROverlay](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html) クラスのインスタンスがセットされていることが確認できました。このクラスを使ってオーバーレイを表示します。

また SteamVR を起動していない場合は、実行時に SteamVR が自動的に起動するようになります。

![](https://storage.googleapis.com/zenn-user-upload/30d14f878b47-20240306.png)

:::message
SteamVR Unity Plugin では、Project Setting から下記の設定をすることで、プラグイン側で OpenVR API の初期化を自動的に行うことも可能です。このチュートリアルではこれらの設定は使用せず、自分で初期化する方法で進めていきます。

- XR Plug-in Management で Initialize XR on Startup にチェックを入れる
- OpenVR で Application Type を Overlay にする

![](https://storage.googleapis.com/zenn-user-upload/626c86347ef3-20240306.png)
![](https://storage.googleapis.com/zenn-user-upload/67317b23e1eb-20240306.png)
:::

### オーバーレイの作成
一旦、先程の初期化処理を関数 InitOpenVR() として分けておきます。

```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
+       InitOpenVR();

-       Debug.Log(OpenVR.Overlay);
-
-        if (OpenVR.System == null)
-        {
-            var initError = EVRInitError.None;
-            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
-            if (initError != EVRInitError.None)
-            {
-                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
-            }
-        }
-
-       Debug.Log(OpenVR.Overlay);
    }

    private void OnDestroy()
    {
        if (OpenVR.System != null)
        {
            OpenVR.Shutdown();
        }
    }

+    private void InitOpenVR()
+    {
+        if (OpenVR.System == null)
+        {
+            var initError = EVRInitError.None;
+            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
+            if (initError != EVRInitError.None)
+            {
+                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
+            }
+        }
+    }
}
```

同様にクリーンアップ処理も ShutdownOpenVR() として関数に分けておきます。
```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private void Start()
    {
       InitOpenVR();
    }

    private void OnDestroy()
    {
-        if (OpenVR.System != null)
-        {
-            OpenVR.Shutdown();
-        }
+        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System == null)
        {
            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
            }
        }
    }

+    private void ShutdownOpenVR();
+    {
+        if (OpenVR.System != null)
+        {
+            OpenVR.Shutdown();
+        }
+    }
}
```


オーバーレイは [CreateOverlay()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_CreateOverlay_System_String_System_String_System_UInt64__) で作成します。[Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::CreateOverlay) に詳しい説明が乗っています。
OpenVR の初期化に続けて、下記のコードを追加します。

```diff cs:FileOverlay.cs
void Start()
{
    InitOpenVR();

+    var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
+    var key = "FileOverlayKey";
+    var name = "FileOverlay";
+    var error = OpenVR.Overlay.CreateOverlay(key, name, ref overlayHandle);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("オーバーレイの作成に失敗しました: " + error);
+    }
}
```

CreateOverlay() を見てみます。

```cs
public EVROverlayError CreateOverlay(string pchOverlayKey, string pchOverlayName, ref ulong pOverlayHandle)
```

pchOverlayKey は他のオーバーレイと被らないユニークな文字列である必要があります。オーバーレイの検索時に使われたりします。

pchOverlayName は任意の名前です。ユーザに表示されることがあります。

pOverlayHandle は作成したオーバーレイのハンドルを保存する ulong 型の変数の参照を渡します。作成したオーバーレイを操作するときに使います。
作成に失敗した場合は、定数 [OpenVR.k_ulOverlayHandleInvalid](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_k_ulOverlayHandleInvalid) がセットされます。

戻り値は enum の [EVROverlayError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVROverlayError.html) の値です。エラーが発生しなかった場合は EVROverlayError.None が返されるので、None なら成功です。

### オーバーレイのクリーンアップ
作成したオーバーレイはアプリケーション終了時に破棄していなければゴミが残ります。
DestroyOverlay() を OnDestroy() に追加して、終了時にオーバーレイを破棄します。
CreateOverlay() 時に取得したハンドルを保存して、DestroyOverlay() に渡します。
アプリケーションの終了時までハンドルを保存しておくため、変数 overlayHandle は private フィールドとして保存しておくようにします。

```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
+    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
       InitOpenVR();

-       var overlayHandle = OpenVR.k_ulOverlayHandleInvalid;
        var error = OpenVR.Overlay.CreateOverlay("FileOverlayKey", "FileOverlay", ref overlayHandle);
        if (error != EVROverlayError.None)
        {
            throw new Exception("オーバーレイの作成に失敗しました: " + error);
        }
    }

    private void OnDestroy()
    {
+        if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
+        {
+            OpenVR.Overlay.DestroyOverlay(overlayHandle);
+        }

        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System == null)
        {
            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
            }
        }
    }

    private void ShutdownOpenVR();
    {
        if (OpenVR.System != null)
        {
            OpenVR.Shutdown();
        }
    }
}
```

### オーバーレイが作成されたか確認
SteamVR に同梱されている Overlay Viewer を使って、実際にオーバーレイが作られているか確認してみましょう。

SteamVR のウィンドウの左上のメニューから Developer > Overlay Viewer を選択します。
![](https://storage.googleapis.com/zenn-user-upload/ca13bcee2bec-20240306.png)

Overlay Viewer では、現在 SteamVR 上に作成されたすべてのオーバーレイを確認できます。SteamVR のシステムメニューなどもオーバーレイで表示されていることがわかります。
![](https://storage.googleapis.com/zenn-user-upload/af20c02e2a1b-20240306.png)

この状態でプログラムを実行してみましょう。
Overlay Viewer の左上の一覧に、先ほど指定したキー FileOverlayKey が追加されます。
クリックすると、左下にオーバーレイの詳細が表示されますね。
![](https://storage.googleapis.com/zenn-user-upload/b4d02e627428-20240306.png)

右側の灰色はオーバーレイの描画内容が表示されますが、今は何も描画していないので灰色です。

:::message
Overlay Viewer の実行ファイルは、Steam のインストールディレクトリに入っています。
デフォルトの場合は "C:\Program Files (x86)\Steam\steamapps\common\SteamVR\bin\win32\overlay_viewer.exe" です。
何度も起動することになるので、ショートカットを作成しておくのがおすすめです。
:::

次へ進む前にオーバーレイの作成と破棄を関数に分けて整理しておきます。

```diff cs:FileOverlay.cs
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
       InitOpenVR();

-       var error = OpenVR.Overlay.CreateOverlay("FileOverlayKey", "FileOverlay", ref overlayHandle);
-       if (error != EVROverlayError.None)
-       {
-           throw new Exception("オーバーレイの作成に失敗しました: " + error);
-       }
+       overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
    }

    private void OnDestroy()
    {
-        if (overlayHandle != OpenVR.k_ulOverlayHandleInvalid)
-        {
-            OpenVR.Overlay.DestroyOverlay(overlayHandle);
-        }
+        DestroyOverlay(overlayHandle);
        ShutdownOpenVR();
    }

    private void InitOpenVR()
    {
        if (OpenVR.System == null)
        {
            var initError = EVRInitError.None;
            OpenVR.Init(ref initError, EVRApplicationType.VRApplication_Overlay);
            if (initError != EVRInitError.None)
            {
                throw new Exception("OpenVRの初期化に失敗しました: " + initError);
            }
        }
    }

    private void ShutdownOpenVR();
    {
        if (OpenVR.System != null)
        {
            OpenVR.Shutdown();
        }
    }

+    private ulong CreateOverlay(string key, string name) {
+        var handle = OpenVR.k_ulOverlayHandleInvalid;
+        var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
+        if (error != EVROverlayError.None)
+        {
+            throw new Exception("オーバーレイの作成に失敗しました: " + error);
+        }
+        return handle;
+    }

+    private void DestroyOverlay(ulong handle) {
+        if (handle != OpenVR.k_ulOverlayHandleInvalid)
+        {
+            OpenVR.Overlay.DestroyOverlay(handle);
+        }
+    }
}
```

### オーバーレイの表示状態の変更
オーバーレイはデフォルトで非表示状態になっているため、表示状態に切り替える必要があります。
表示状態の切り替えは ShowOverlay() と HideOverlay() で行います。

```cs
ShowOverlay(ulong ulOverlayHandle)
HideOverlay(ulong ulOverlayHandle)
```

Wiki
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::ShowOverlay

SteamVR Unity Plugin
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_ShowOverlay_System_UInt64_
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_HideOverlay_System_UInt64_

Start() にオーバーレイの表示を追加します。

```diff cs:FileOverlay.cs
private void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

+    var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("オーバーレイの表示に失敗しました: " + error);
+    }
}
```


### 画像ファイルの準備
オーバーレイに画像を描画してみましょう。
画像はなんでもいいですが、ここでは自分のアイコンを使ってみます。
![](https://storage.googleapis.com/zenn-user-upload/9ba58573d5a9-20240302.jpg =100x)

Unity で Assets の下に StreamingAssets フォルダを作ります。
用意した画像を StreamingAssets フォルダに追加します。
ここでは sns-icon.jpg という名前で追加しました。

### 画像の描画
オーバーレイに画像ファイルを描画するには [SetOverlayFromFile()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayFromFile_System_UInt64_System_String_) を使用します。関数の詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayFromFile) に載っています。

SetOverlayFromFile() を見てみます。

```cs
public EVROverlayError SetOverlayFromFile(ulong ulOverlayHandle, string pchFilePath)
```

ulOverlayHandle は CreateOerlay() で取得したハンドル、pchFilePath は画像ファイルパスです。
戻り値は CreateOverlay() と同様に EVROverlayError 型で、エラーがなければ EVROverlayError.None となります。

早速使ってみましょう。

```diff cs:FileOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
    if (error != EVROverlayError.None)
    {
        throw new Exception("オーバーレイの表示に失敗しました: " + error);
    }

+    var file = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
+    error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("画像ファイルの描画に失敗しました: " + error);
+    }
}
```

画像を [StreamingAssets](https://docs.unity3d.com/Manual/StreamingAssets.html) フォルダに入れたのは、[Application.streamingAssetsPath](https://docs.unity3d.com/ScriptReference/Application-streamingAssetsPath.html) で画像ファイルパスを取得するためです。

プログラムを実行して、Overlay Viewer で FileOverlayKey のオーバーレイを確認しましょう。
正常に動作していれば、プレビュー部分に画像が表示されているはずです。
![](https://storage.googleapis.com/zenn-user-upload/c7cc3e4edf39-20240306.png)


そのまま HMD を装着して、足元を確認してください。
プレイエリアの中心にオーバーレイが表示されているはずです。オーバーレイは裏側から見ると透明になるので、見えない場合は反対側に回り込んでみてください。

![](/images/file-overlay-in-vr.jpg)


![](/images/overlay-in-game.jpg)
*ゲームの起動中でも動きます*

VR 空間内へのオーバーレイの表示ができました。次は、オーバーレイの表示位置や大きさを変更してみます。
その前に、一旦コードを整理しておきましょう。

オーバーレイの表示や画像ファイルの描画を関数にして分けておきます。


```diff cs:FileOverlay.cs
void Start()
{
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
+    ShowOverlay(overlayHandle);
-    var error = OpenVR.Overlay.ShowOverlay(overlayHandle);
-    if (error != EVROverlayError.None)
-    {
-        throw new Exception("オーバーレイの表示に失敗しました: " + error);
-    }

+    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg")
+    SetOverlayFromFile(overlayHandle, filePath);
-    var file = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
-    var error = OpenVR.Overlay.SetOverlayFromFile(overlayHandle, filePath);
-    if (error != EVROverlayError.None)
-    {
-        throw new Exception("画像ファイルの描画に失敗しました: " + error);
-    }
}

...

+ private void ShowOverlay(ulong handle)
+ {
+     var error = OpenVR.Overlay.ShowOverlay(handle);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("オーバーレイの表示に失敗しました: " + error);
+     }
+ }

+ private void SetOverlayFromFile(ulong handle, string path)
+ {
+     var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
+     if (error != EVROverlayError.None)
+     {
+         throw new Exception("画像ファイルの描画に失敗しました: " + error);
+     }
+ }
```

最終的なスクリプトは下記の通りです。
```cs:FileOverlay.cs
using System;
using UnityEngine;
using Valve.VR;

public class FileOverlay : MonoBehaviour
{
    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {        
        InitOpenVR();
        overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");
        ShowOverlay(overlayHandle);

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        SetOverlayFromFile(overlayHandle, filePath);
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
}
```