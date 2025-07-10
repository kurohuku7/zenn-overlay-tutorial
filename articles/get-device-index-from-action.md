---
title: "GetTrackedDeviceIndexForControllerRole() を使わずに Device Index を取得する" # 記事のタイトル
emoji: "🙉" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["steamvr", "openvr", "vr"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# 概要

- GetTrackedDeviceIndexForControllerRole() と GetControllerRoleForTrackedDeviceIndex() は deprecated
- 他の方法でコントローラの Device Index を取りたい人向け
- SteamVR Input (IVRInput) の Pose アクションからコントローラの Device Index を取得する

# deprecated になっている関数

[以前作成したチュートリアル](https://zenn.dev/kurohuku/books/a082c5728cc1f6/viewer/relative-position)で

```cs
var leftControllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(ETrackedControllerRole.LeftHand);
```

のように GetTrackedDeviceIndexForControllerRole() でコントローラの Device Index を取得しているのですが、このメソッドが header ファイル上のコメントで deprecated とされているため、IVRInput(SteamVR Input) を使って、Device Index を取得する方法です。
※ deprecated とはなっていますが、実際には OpenVR のコードの内でも使われているので、個人的にはこのままでも問題ないとは思っています。

参考
https://github.com/ValveSoftware/openvr/blob/master/headers/openvr.h

# Unity Project 作成

新規プロジェクトを作成します。下記の環境で作成しています。

- Unity 6.1
- URP テンプレート

シーンのオブジェクトは削除して空っぽにしておきます。

# SteamVR Plugin インストール

Package Manager で SteamVR Plugin をインストールします。

https://assetstore.unity.com/packages/tools/integration/steamvr-plugin-32647

# コントローラのバインディング

下記の手順で、コントローラの Pose をアクションにバインドします。

- Window -> SteamVR Input
- Copy Examples は NO を選択
- ActionSet 名 "DeviceIndex" (何でも OK)、モードは per hand を選択
- Actions は LeftHandPose と RightHandPose の 2 つを作成
- どちらも Type を pose にする

![](/images/get-device-index-from-action/action-setting.png)

- Save and generate ボタンを押す
- Open binding UI ボタンを押す
- Create New Binding

![](/images/get-device-index-from-action/create-new-binding.png)

- Poses ボタンを押す

![](/images/get-device-index-from-action/pose-button.png)

- Left Hand Raw と Right Hand Raw にそれぞれ LeftHandPose と RightHandPose アクションを割り当て

![](/images/get-device-index-from-action/bind-pose-actions.png)

- Replace Default Binding を押してデフォルトとして保存

## Poses ボタンがないとき

上の Poses ボタンですが、時間の都合で詳しく検証できていないのですが、表示されないことがあります。
actions.json に pose のアクションが定義されていれば表示される仕様だと思うのですが、何回か試してみて表示されないことがありました。

表示されていない場合は、SteamVR を再起動してみたり、直接 actions.json ファイルを作ったりしてみたり、他のアプリから actions.json を持ってくるなど、色々試してみて下さい。

# スクリプト作成

`GetControllerDeviceIndex.cs` を作成します。
ここでは左手のコントローラの Device Index を取ってみます。

アクションの取得方法など、コードの詳細は[こちらのチュートリアル](https://zenn.dev/kurohuku/books/a082c5728cc1f6/viewer/controller-input)を見て下さい。
重要なのは `Update()` の下の方で `controllerOriginInfo` を取得している箇所以降です。

```cs
using System;
using Valve.VR;
using UnityEngine;

class GetControllerDeviceIndex : MonoBehaviour
{
    ulong actionSetHandle = OpenVR.k_ulInvalidActionSetHandle;
    ulong actionHandle = OpenVR.k_ulInvalidActionHandle;

    void Start()
    {
        var error = OpenVR.Input.SetActionManifestPath(Application.streamingAssetsPath + "/SteamVR/actions.json");
        if (error != EVRInputError.None)
        {
            Debug.LogError("Failed to set action manifest path: " + error);
            return;
        }

        error = OpenVR.Input.GetActionSetHandle("/actions/DeviceIndex", ref actionSetHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("Failed to get action set /actions/DeviceIndex: " + error);
        }

        error = OpenVR.Input.GetActionHandle("/actions/DeviceIndex/in/LeftHandPose", ref actionHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("Failed to get action /actions/DeviceIndex/in/LeftHandPose: " + error);
        }
    }

    void Update()
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
            throw new Exception("Failed to update action state: " + error);
        }

        var result = new InputPoseActionData_t();
        var poseActionSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(InputPoseActionData_t));
        error = OpenVR.Input.GetPoseActionDataForNextFrame(actionHandle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref result, poseActionSize, OpenVR.k_ulInvalidInputValueHandle);
        if (error != EVRInputError.None)
        {
            throw new Exception("Failed to get pose action data: " + error);
        }

        // GetOriginTrackedDeviceInfo() にアクションの値の activeOrigin を渡すと、アクションの発生源のデバイスを取得できる。
        var controllerOriginInfo = new InputOriginInfo_t();
        var inputOriginInfoSize = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(InputOriginInfo_t));
        error = OpenVR.Input.GetOriginTrackedDeviceInfo(result.activeOrigin, ref controllerOriginInfo, inputOriginInfoSize);
        if (error != EVRInputError.None)
        {
            Debug.LogError("Failed to get left controller origin info: " + error);
            return;
        }

        // 取得したデバイスの Device Index を取得
        var deviceIndex = controllerOriginInfo.trackedDeviceIndex;
        Debug.Log($"Left Controller Device Index: {deviceIndex}");
    }
}
```

# 実行

適当な Game Object をシーンに作成して、上のコンポーネントを追加して、実行します。
コンソールに左手のコントローラの Device Index が表示されます。

![](/images/get-device-index-from-action/device-index.png)

このような感じでアクションから Device Index を取得した後、`SetOverlayTransformRelative()` の引数に渡すなどして使うことができます。
