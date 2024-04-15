---
title: "OpenVR の初期化"
free: false
---

## スクリプトを作成
`Assets/Script` フォルダを作成します。
C# スクリプト `WatchOverlay.cs` を `Scripts` フォルダ内に新規作成します。
このスクリプトにオーバーレイの処理を書いていきます。

![](/images/create-watch-overlay.png)

`WatchOverlay.cs` に下記のコードをコピーします。

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
Hierarchy ウィンドウ（以下 Hierarchy）を右クリック > Create Empty で、空のオブジェクトを作成します。
オブジェクトの名前を `WatchOverlay` に変更し、先程の `WatchOverlay.cs` をドラッグしてコンポーネントとして追加します。
![](/images/add-watch-overlay-component.png)

## OpenVR API の読み込み
OpenVR の API が定義されている名前空間 `Valve.VR` を読み込みます。これは SteamVR Plugin に含まれているものです。

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

初期化時のエラーは [EVRInitError](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRInitError.html) 型として `enum` で定義されています。
`Init()` の第 1 引数に `ref` で `EVRInitError` 型の変数 `error` の参照を渡すと、`error` にエラーの内容が書き込まれます。

第 2 引数には初期化するアプリケーションの種類を渡します。`EVRApplicationType.VRApplication_Overlay` を指定すると、オーバーレイアプリとして初期化します。その他のアプリケーションの種類は [EVRApplicationType](https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.EVRApplicationType.html) で定義されています。
オーバーレイアプリとして初期化すると、他の VR アプリと同時に起動できるようになります。

## 初期化失敗時の処理
初期化に失敗した場合のエラー処理を追加します。
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

初期化が成功した場合は「エラーが発生しなかった」という意味の `EVRInitError.None` が `error` にセットされます。
逆に `error` が `EVRInitError.None` 以外なら、エラーが発生しています。
※ `using System;` は `Exception` を使用するために追加しています。 


## 既に初期化されている場合の処理
既に OpenVR が他の場所で初期化されている場合には、初期化を実行しないようにします。
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

OpenVR が初期化されているかどうかは `OpenVR.System` が `null` かどうかで判定できます。
OpenVR API は下のようにいくつかのグループに分かれていて、初期化前に参照すると `null` になります。

- OpenVR.System
- OpenVR.Chaperone
- OpenVR.Compositor
- OpenVR.Overlay
- OpenVR.RenderModels
- OpenVR.Screenshots
- OpenVR.Input

## クリーンアップ
初期化した OpenVR を破棄する後処理を追加します。
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
試しに OpenVR の Overlay API にアクセスできるか確認してみます。
簡単に `Debug.Log()` で確認してみます。

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

初期化前は null だった API が、初期化後にアクセスできる状態になっています。
また SteamVR を起動していない場合は、プログラム実行時に `Init()` によって SteamVR が起動されるようになります。
![](/images/steamvr.png)

:::details SteamVR Plugin のプロジェクト設定による初期化
このチュートリアルでは使用しませんが、SteamVR Plugin では、Project Setting で下記の設定を行うことで、プラグイン側で OpenVR を自動的に初期化することも可能です。

- XR Plug-in Management で Initialize XR on Startup にチェックを入れる

![](/images/initialize-xr-on-startup.png)

- OpenVR で Application Type を Overlay にする

![](/images/application-type-overlay.png)
:::

確認できたら `Debug.Log()` は削除しておきます。
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

## コードの整理
OpenVR の初期化とクリーンアップができたので、一旦コードを整理しておきます。
メソッドが巨大化すると説明が難しくなるため、このチュートリアルでは、ある程度コードを追加したタイミングで、処理を関数に分けてコードを整理するステップを挟みます。

### 初期化
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
チュートリアルでは、コードの見やすさのため、エラー発生時は基本的に`throw` で例外を発生させています。`return` でエラーコードや `bool` 値を返したい場合は、下記のように読み替えてください。

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

OpenVR の初期化とクリーンアップはこれで完了です。
次のページでは、初期化によって使えるようになった OpenVR Overlay API を使って、オーバーレイを作成していきます。