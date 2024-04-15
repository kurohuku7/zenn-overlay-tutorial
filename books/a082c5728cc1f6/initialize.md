---
title: "OpenVR の初期化"
free: false
---

## スクリプトを作成
Assets/Script フォルダを作成して、C# スクリプト WatchOverlay.cs を新規作成します。

![](/images/create-watch-overlay.png)

```cs:WatchOverlay.cs
using UnityEngine;

public class WatchOverlay : MonoBehaviour
{
    private void Start()
    {
    }
}
```

## スクリプトをシーンに設置
Hierarchy パネルを右クリック > Create Empty で、空のオブジェクトを作成します。
オブジェクトの名前を WatchOverlay に変更し、先程の WatchOverlay.cs をコンポーネントとして追加してください。
![](/images/add-watch-overlay-component.png)

## OpenVR API の読み込み
OpenVR の API が定義されている名前空間 Valve.VR を読み込みます。

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

## OpenVR の初期化
OpenVR の API を使用するために [Init()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Init_Valve_VR_EVRInitError__Valve_VR_EVRApplicationType_System_String_) で OpenVR を初期化します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/API-Documentation#initialization-and-cleanup) を参照）
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

初期化時に発生したエラーが error にセットされます。
EVRApplicationType.VRApplication_Overlay を指定すると、オーバーレイアプリとして初期化します。その他のアプリケーションの種類は [EVRApplicationType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRApplicationType.html) で定義されています。
オーバーレイアプリとして初期化すると、他の VR アプリの動作中に起動できるようになります。

## 初期化失敗
初期化に失敗した場合の処理を追加します。
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

初期化が成功した場合は「**エラーが発生しなかった**」という意味の `EVRInitError.None` が `error` にセットされます。
`EVRInitError.None` 以外ならエラーが発生しています。
初期化時のエラーは [EVRInitiError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRInitError.html) に定義されています。
`using System` は `Exception` を使用するために追加しています。 


## 既に初期化されている場合
OpenVR が他の場所で既に初期化されている場合には、初期化を実行しないようにします。
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

OpenVR が初期化されているかどうかは OpenVR.System が null かどうかで判定できます。
OpenVR API は下記のようにいくつかに分かれていて、初期化が成功するとアクセスできるようになります。

- OpenVR.System
- OpenVR.Chaperone
- OpenVR.Compositor
- OpenVR.Overlay
- OpenVR.RenderModels
- OpenVR.Screenshots
- OpenVR.Input

## クリーンアップ
プログラムの終了時に [Shutdown()](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.OpenVR.html#Valve_VR_OpenVR_Shutdown) を呼び出します。（詳細は [Wiki](https://github.com/ValveSoftware/openvr/wiki/API-Documentation#initialization-and-cleanup) を参照）

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

## 動作確認
試しに Overlay API にアクセスできるか確認してみます。
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
![](/images/api-debug.png)

[OpenVR.CVROverlay](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html) が、初期化前は Null, 初期化後にインスタンスになっています。
また SteamVR を起動していない場合は、Init() によって SteamVR が起動されるようになります。
![](/images/steamvr.png)

:::details SteamVR Plugin による初期化の設定
このチュートリアルでは使用しませんが、SteamVR Plugin では、Project Setting で下記の設定を行うことで、プラグイン側で OpenVR を初期化することも可能です。

- XR Plug-in Management で Initialize XR on Startup にチェックを入れる
- OpenVR で Application Type を Overlay にする

![](/images/initialize-xr-on-startup.png)
![](/images/application-type-overlay.png)
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
初期化処理を `InitOpenVR()` として関数に分けておきます。

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

:::details throw ではなく return でエラー処理をしたい
このチュートリアルでは、コードの見やすさを優先して、基本的にエラー時は `throw` で例外を発生させて処理を中断させています。
`return` でエラーコードや `bool` 値を返したい場合は、下記のように読み替えてください。

```cs:エラーコードを返す例
private bool InitOpenVR()
{
    if (OpenVR.System != null) return;

    var error = EVRInitError.None;
    OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
    if (error != EVRInitError.None)
    {
        Debug.LogError("OpenVR の初期化に失敗しました: " + error);
    }
    return error;
}
```
```cs:bool 値を返す例
// 初期化に成功したら true, 失敗したら false を返す
private bool InitOpenVR()
{
    if (OpenVR.System != null) return;

    var error = EVRInitError.None;
    OpenVR.Init(ref error, EVRApplicationType.VRApplication_Overlay);
    if (error != EVRInitError.None)
    {
        Debug.LogError("OpenVR の初期化に失敗しました: " + error);
        return false;
    }
    return true;
}
```
:::

## クリーンアップ処理の整理
同様にクリーンアップ処理を `ShutdownOpenVR()` として関数に分けておきます。
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

+   private void ShutdownOpenVR()
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

   private void ShutdownOpenVR()
   {
       if (OpenVR.System != null)
       {
           OpenVR.Shutdown();
       }
   }
}
```
