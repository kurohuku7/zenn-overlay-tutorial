---
title: "OpenVR の初期化"
free: false
---

実際にオーバーレイを表示してみます。
画像ファイルをオーバーレイとして表示するシンプルな関数があるので、まずはこれを使ってみましょう。

## スクリプトを作成
Scripts フォルダ内に C# スクリプト WatchOverlay.cs を新規作成します。

```cs:WatchOverlay.cs
using UnityEngine;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

Hierarchy パネルを右クリック > Create Empty で空のオブジェクトをシーンに作成します。
オブジェクトの名前を WatchOverlay に変更して、先程の WatchOverlay.cs をコンポーネントとして追加してください。
![](/images/add-watch-overlay-component.png)


OpenVR の API が定義されている名前空間 Valve.VR を追加しておきます。

```diff cs:WatchOverlay.cs
using UnityEngine;
+ using Valve.VR;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

## 初期化
[Wiki](https://github.com/ValveSoftware/openvr/wiki/API-Documentation#initialization-and-cleanup) に書かれているように [Init()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Init_Valve_VR_EVRInitError__Valve_VR_EVRApplicationType_System_String_) を呼び出します。
```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
+       var error = EVRInitError.None;
+       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
   }
}
```

`Init()` にはエラーを保存する変数（の参照）と、アプリケーションの種類を渡します。
ここで渡した `error`（の参照）に、初期化時のエラー内容が書き込まれます。
オーバーレイアプリの場合は `EVRApplicationType.VRApplication_Overlay` を指定します。
その他のアプリケーションの種類は [EVRApplicationType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRApplicationType.html) で定義されています。


## 失敗時の処理
初期化に失敗した場合の処理を追加します。
`using System` は `Exception` を使用するために追加しています。 
```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
+ using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
       var error = EVRInitError.None;
       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
+      if (error != EVRInitError.None)
+      {
+          throw new Exception("OpenVR の初期化に失敗しました: " + error);
+      }
   }
}
```

初期化が成功した場合は「**エラーが発生しなかった**」という意味の `EVRInitError.None` が `error` に書き込まれます。
つまり `EVRInitError.None` 以外だった場合は、エラーが発生したということになります。
エラーの種類は [EVRInitiError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRInitError.html) に定義されています。

## 既に初期化されている場合の対処
他の場所で既に初期化されている場合には、初期化を実行しないようにします。
```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
+      if (OpenVR.System != null) return;

       var error = EVRInitError.None;
       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
       if (error != EVRInitError.None)
       {
           throw new Exception("OpenVR の初期化に失敗しました: " + error);
       }
   }
}
```

OpenVR が初期化される前は OpenVR.System が null になっています。
OpenVR API は下記のようにいくつかに分かれていて、初期化が成功すると、それぞれにアクセスできるようになります。

- OpenVR.System
- OpenVR.Chaperone
- OpenVR.Compositor
- OpenVR.Overlay
- OpenVR.RenderModels
- OpenVR.Screenshots
- OpenVR.Input

## クリーンアップ
[Wiki](https://github.com/ValveSoftware/openvr/wiki/API-Documentation#initialization-and-cleanup) を参考にして、終了時に [Shutdown()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Shutdown) を呼び出します。

```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
       if (OpenVR.System != null) return;
       
       var error = EVRInitError.None;
       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
       if (error != EVRInitError.None)
       {
           throw new Exception("OpenVR の初期化に失敗しました: " + error);
       }
   }

+   private void OnDestroy()
+   {
+       if (OpenVR.System != null)
+       {
+           OpenVR.Shutdown();
+       }
+   }
}
```

## 初期化できているか確認
試しに Overlay API にアクセスできるか確認してみましょう。
下記のログ出力を追加して、プログラムを実行してください。

```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
        if (OpenVR.System != null) return;

+       Debug.Log(OpenVR.Overlay);
        var error = EVRInitError.None;
        OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
        if (error != EVRInitError.None)
        {
            throw new Exception("OpenVRの初期化に失敗しました: " + error);
            return;
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

初期化後に [CVROverlay](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html) が使えるようになっています。
このクラスを使ってオーバーレイを作成します。

また SteamVR を起動していない場合は、プログラムの実行時に SteamVR が起動されるようになります。

![](https://storage.googleapis.com/zenn-user-upload/30d14f878b47-20240306.png)

:::details SteamVR Unity Plugin による初期化の設定
このチュートリアルでは使用しませんが、SteamVR Unity Plugin では、Project Setting で下記の設定を行うことで、プラグイン側で OpenVR を初期化することも可能です。

- XR Plug-in Management で Initialize XR on Startup にチェックを入れる
- OpenVR で Application Type を Overlay にする

![](https://storage.googleapis.com/zenn-user-upload/626c86347ef3-20240306.png)
![](https://storage.googleapis.com/zenn-user-upload/67317b23e1eb-20240306.png)
:::

確認できたら、先程追加したデバッグログは削除しておきます。
```diff cs:WatchOverlay.cs
private void Start()
{
    if (OpenVR.System != null) return;

-   Debug.Log(OpenVR.Overlay);
    var error = EVRInitError.None;
    OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
    if (error != EVRInitError.None)
    {
        throw new Exception("OpenVRの初期化に失敗しました: " + error);
        return;
    }
-   Debug.Log(OpenVR.Overlay);
}

```

## 初期化処理の整理
一旦、初期化処理を関数 `InitOpenVR()` として分けておきます。

```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
+       InitOpenVR();
-       if (OpenVR.System != null) return;
-
-       var error = EVRInitError.None;
-       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
-       if (error != EVRInitError.None)
-       {
-           throw new Exception("OpenVRの初期化に失敗しました: " + error);
-       }
    }

    private void OnDestroy()
    {
        if (OpenVR.System != null)
        {
            OpenVR.Shutdown();
        }
    }

+   private void InitOpenVR()
+   {
+       if (OpenVR.System != null) return;
+
+       var error = EVRInitError.None;
+       OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
+       if (error != EVRInitError.None)
+       {
+           throw new Exception("OpenVRの初期化に失敗しました: " + error);
+       }
+   }
}
```

## クリーンアップ処理の整理
同様にクリーンアップ処理も `ShutdownOpenVR()` として関数に分けておきます。
```diff cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
       InitOpenVR();
    }

    private void OnDestroy()
    {
-       if (OpenVR.System != null)
-       {
-           OpenVR.Shutdown();
-       }
+       ShutdownOpenVR();
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

+   private void ShutdownOpenVR();
+   {
+       if (OpenVR.System != null)
+       {
+           OpenVR.Shutdown();
+       }
+   }
}
```

## 最終的なコード
```cs:WatchOverlay.cs
using UnityEngine;
using Valve.VR;
using System;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
       InitOpenVR();
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

   private void ShutdownOpenVR();
   {
       if (OpenVR.System != null)
       {
           OpenVR.Shutdown();
       }
   }
}
```
