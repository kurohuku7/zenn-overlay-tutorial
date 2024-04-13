---
title: "ダッシュボードオーバーレイ 2"
free: false
---

## 最終的なコード
最終的なコードはこちら。

```cs:WatchOverlay.cs
using System;
using UnityEngine;
using Valve.VR;
using OpenVRUtil;

public class WatchOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public ETrackedControllerRole targetHand = ETrackedControllerRole.RightHand;

    private ulong overlayHandle = OpenVR.k_ulOverlayHandleInvalid;

    [Range(0, 0.5f)] public float size;

    [Range(-0.5f, 0.5f)] public float leftX;
    [Range(-0.5f, 0.5f)] public float leftY;
    [Range(-0.5f, 0.5f)] public float leftZ;
    [Range(0, 360)] public int leftRotationX;
    [Range(0, 360)] public int leftRotationY;
    [Range(0, 360)] public int leftRotationZ;

    [Range(-0.5f, 0.5f)] public float rightX;
    [Range(-0.5f, 0.5f)] public float rightY;
    [Range(-0.5f, 0.5f)] public float rightZ;
    [Range(0, 360)] public int rightRotationX;
    [Range(0, 360)] public int rightRotationY;
    [Range(0, 360)] public int rightRotationZ;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        overlayHandle = Overlay.CreateOverlay("WatchOverlayKey", "WatchOverlay");
        
        Overlay.FlipOverlayVertical(overlayHandle);
        Overlay.SetOverlaySize(overlayHandle, size);
        Overlay.ShowOverlay(overlayHandle);
    }

    private void Update()
    {
        Vector3 position;
        Quaternion rotation;

        if (targetHand == ETrackedControllerRole.LeftHand)
        {
            position = new Vector3(leftX, leftY, leftZ);
            rotation = Quaternion.Euler(leftRotationX, leftRotationY, leftRotationZ);
        }
        else
        {
            position = new Vector3(rightX, rightY, rightZ);
            rotation = Quaternion.Euler(rightRotationX, rightRotationY, rightRotationZ);
        }
        
        var controllerIndex = OpenVR.System.GetTrackedDeviceIndexForControllerRole(targetHand);
        if (controllerIndex != OpenVR.k_unTrackedDeviceIndexInvalid)
        {
            Overlay.SetOverlayTransformRelative(overlayHandle, controllerIndex, position, rotation);
        }

        Overlay.SetOverlayRenderTexture(overlayHandle, renderTexture);
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(overlayHandle);
    }
    
    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }
}
```

```cs:DashboardOverlay.cs
using UnityEngine;
using Valve.VR;
using System;
using OpenVRUtil;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using System.Collections.Generic;

public class DashboardOverlay : MonoBehaviour
{
    public Camera camera;
    public RenderTexture renderTexture;
    public GraphicRaycaster graphicRaycaster;
    public EventSystem eventSystem;
    
    private ulong dashboardHandle = OpenVR.k_ulOverlayHandleInvalid;
    private ulong thumbnailHandle = OpenVR.k_ulOverlayHandleInvalid;

    private void Start()
    {
        OpenVRUtil.System.InitOpenVR();
        (dashboardHandle, thumbnailHandle) = Overlay.CreateDashboardOverlay("WatchDashboardKey", "Watch Setting");

        var filePath = System.IO.Path.Combine(Application.streamingAssetsPath, "sns-icon.jpg");
        Overlay.SetOverlayFromFile(thumbnailHandle, filePath);

        Overlay.SetOverlaySize(dashboardHandle, 2.5f);
        Overlay.FlipOverlayVertical(dashboardHandle);
        Overlay.SetOverlayMouseScale(dashboardHandle, renderTexture.width, renderTexture.height);
    }

    void Update()
    {
        Overlay.SetOverlayRenderTexture(dashboardHandle, renderTexture);
        ProcessOverlayEvents();
    }

    private void OnApplicationQuit()
    {
        Overlay.DestroyOverlay(dashboardHandle);
        Overlay.DestroyOverlay(thumbnailHandle);
    }

    private void OnDestroy()
    {
        OpenVRUtil.System.ShutdownOpenVR();
    }

    private void ProcessOverlayEvents()
    {
        var vrEvent = new VREvent_t();
        var uncbVREvent = (uint)System.Runtime.InteropServices.Marshal.SizeOf(typeof(VREvent_t));
        while (OpenVR.Overlay.PollNextOverlayEvent(dashboardHandle, ref vrEvent, uncbVREvent))
        {
            switch (vrEvent.eventType)
            {
                case (uint)EVREventType.VREvent_MouseButtonUp:
                    vrEvent.data.mouse.y = renderTexture.height - vrEvent.data.mouse.y;
                    var button = GetButtonByPosition(vrEvent.data.mouse.x, vrEvent.data.mouse.y);
                    if (button != null)
                    {
                        button.onClick.Invoke();
                    }
                    break;
            }
        }
    }
    
    private Button GetButtonByPosition(float x, float y)
    {
        var raycastResultList = new List<RaycastResult>();
        var pointerEventData = new PointerEventData(eventSystem);
        pointerEventData.position = new Vector2(x, y);
        
        graphicRaycaster.Raycast(pointerEventData, raycastResultList);
        var raycastResult = raycastResultList.Find(element => element.gameObject.GetComponent<Button>());
        if (raycastResult.gameObject == null)
        {
            return null;
        }
        return raycastResult.gameObject.GetComponent<Button>();
    }
}
```

```cs:OpenVRUtil.cs
using UnityEngine;
using Valve.VR;
using System;

namespace OpenVRUtil
{
    public static class System
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

        public static void ShutdownOpenVR()
        {
            if (OpenVR.System != null)
            {
                OpenVR.Shutdown();
            }
        }
    }

    public static class Overlay
    {
        public static ulong CreateOverlay(string key, string name)
        {
            var handle = OpenVR.k_ulOverlayHandleInvalid;
            var error = OpenVR.Overlay.CreateOverlay(key, name, ref handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの作成に失敗しました: " + error);
            }

            return handle;
        }

        public static void DestroyOverlay(ulong handle)
        {
            if (handle != OpenVR.k_ulOverlayHandleInvalid)
            {
                OpenVR.Overlay.DestroyOverlay(handle);
            }
        }

        public static (ulong, ulong) CreateDashboardOverlay(string key, string name)
        {
            ulong dashboardHandle = 0;
            ulong thumbnailHandle = 0;
            var error = OpenVR.Overlay.CreateDashboardOverlay(key, name, ref dashboardHandle, ref thumbnailHandle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("ダッシュボード‐バーレイの作成に失敗しました: " + error);
            }

            return (dashboardHandle, thumbnailHandle);
        }

        public static void ShowOverlay(ulong handle)
        {
            var error = OpenVR.Overlay.ShowOverlay(handle);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの表示に失敗しました: " + error);
            }
        }

        public static void SetOverlayFromFile(ulong handle, string path)
        {
            var error = OpenVR.Overlay.SetOverlayFromFile(handle, path);
            if (error != EVROverlayError.None)
            {
                throw new Exception("画像ファイルの描画に失敗しました: " + error);
            }
        }

        public static void SetOverlaySize(ulong handle, float size)
        {
            var error = OpenVR.Overlay.SetOverlayWidthInMeters(handle, size);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイのサイズ設定に失敗しました: " + error);
            }
        }

        public static void SetOverlayTransformAbsolute(ulong handle, Vector3 position, Quaternion rotation)
        {
            var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
            var matrix = rigidTransform.ToHmdMatrix34();
            var error = OpenVR.Overlay.SetOverlayTransformAbsolute(handle, ETrackingUniverseOrigin.TrackingUniverseStanding, ref matrix);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
            }
        }

        public static void SetOverlayTransformRelative(ulong handle, uint deviceIndex, Vector3 position, Quaternion rotation)
        {
            var rigidTransform = new SteamVR_Utils.RigidTransform(position, rotation);
            var matrix = rigidTransform.ToHmdMatrix34();
            var error = OpenVR.Overlay.SetOverlayTransformTrackedDeviceRelative(handle, deviceIndex, ref matrix);
            if (error != EVROverlayError.None)
            {
                throw new Exception("オーバーレイの位置設定に失敗しました: " + error);
            }
        }

        public static void FlipOverlayVertical(ulong handle)
        {
            var bounds = new VRTextureBounds_t
            {
                uMin = 0,
                uMax = 1,
                vMin = 1,
                vMax = 0
            };
            var error = OpenVR.Overlay.SetOverlayTextureBounds(handle, ref bounds);
            if (error != EVROverlayError.None)
            {
                throw new Exception("テクスチャの反転に失敗しました: " + error);
            }
        }

        public static void SetOverlayRenderTexture(ulong handle, RenderTexture renderTexture)
        {
            if (!renderTexture.IsCreated())
            {
                return;
            }
            
            var nativeTexturePtr = renderTexture.GetNativeTexturePtr();
            var texture = new Texture_t
            {
                eColorSpace = EColorSpace.Auto,
                eType = ETextureType.DirectX,
                handle = nativeTexturePtr
            };
            var error = OpenVR.Overlay.SetOverlayTexture(handle, ref texture);
            if (error != EVROverlayError.None)
            {
                throw new Exception("テクスチャの描画に失敗しました: " + error);
            }
        }

        public static void SetOverlayMouseScale(ulong handle, int x, int y)
        {
            var pvecMouseScale = new HmdVector2_t()
            {
                v0 = x,
                v1 = y
            };
            var error = OpenVR.Overlay.SetOverlayMouseScale(handle, ref pvecMouseScale);
            if (error != EVROverlayError.None)
            {
                throw new Exception("マウススケールの設定に失敗しました: " + error);
            }
        }
    }
}
```

```cs:WatchSettingController.cs
using UnityEngine;
using Valve.VR;

public class WatchSettingController : MonoBehaviour
{
    [SerializeField] private WatchOverlay watchOverlay;
    
    public void OnLeftHandButtonClick()
    {
        watchOverlay.targetHand = ETrackedControllerRole.LeftHand;
    }

    public void OnRightHandButtonClick()
    {
        watchOverlay.targetHand = ETrackedControllerRole.RightHand;
    }
}
```