---
title: "オーバーレイの表示位置と大きさ"
free: false
---

Overlay Viewer 上で画像が表示されていることを確認できましたが、肝心の HMD には何も表示されていません。
オーバーレイは VR 空間のどこにどのくらいの大きさで表示するかを指定する必要があります。
早速オーバーレイの表示場所を指定して、VR 内で確認できるようにしてみましょう。

## オーバーレイの大きさ
まずはオーバーレイの大きさを設定します。
大きさは SetOverlayWidthinMeters() で設定できます。
オーバーレイの幅を m 単位で指定します。高さは表示する画像のアスペクト比に合わせて自動的に計算されます。
今回は正方形の画像をサンプルで使用しているため、縦横同サイズとなります。
とりあえず幅 1m に設定してみましょう。

```cs:FileOverlay.cs
private void Start()
{        
    InitOpenVR();
    overlayHandle = CreateOverlay("FileOverlayKey", "FileOverlay");

    var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
    SetOverlayFromFile(overlayHandle, filePath);

+    var error = OpenVR.Overlay.SetOverlayWidthInMeters(overlayHandle, 1);
+    if (error != EVROverlayError.None)
+    {
+        throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
+    }
}
```

Wiki はこちら
https://github.com/ValveSoftware/openvr/wiki/IVROverlay::SetOverlayWidthInMeters

SteamVR Plugin はこちら
https://valvesoftware.github.io/steamvr_unity_plugin/api/Valve.VR.CVROverlay.html#Valve_VR_CVROverlay_SetOverlayWidthInMeters_System_UInt64_System_Single_


## オーバーレイの表示位置
次にオーバーレイの表示位置を設定します。
表示位置は SteamVR のプレイエリアの原点（床の中心）を基準とする SetOverlayTransform() と、HMD やコントローラの位置を基準とする SetOverlayTransformTrackedDeviceRelative() があります。
空間の特定の位置に固定するか、コントローラや HMD に追従させるかによって使い分けます。
最初は絶対座標で、空間の特定の場所に固定で表示してみましょう。

位置の指定は 3DCG では同じもの変換行列によって行われます。
引数には座標を変換するための行列 HmdMatrix を与えます。



### 空間に固定で出してみる
まずは絶対座標で空間に固定で表示してみましょう。

### HMD からの相対座標で出してみる
次は、HMD の相対座標で出してみましょう。
顔の前に常に固定しておいて、オーバーレイをプレイヤーの視界に出し続ける事ができます。
酔い対策ツールのヴィネット表示はこれを使って、目の前にヴィネットを表示しています。

### コントローラにくっつけてみる
同じようにコントローラにくっつけてみましょう。
今回はサンプルとして腕時計のアプリを作って見るので、手首に固定される感じで作ってみましょう。

表示場所についてはこれでできましたね。次はオーバーレイに時刻を表示してみましょう。