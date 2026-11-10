# 团结引擎（Tuanjie）全身动捕 + VRM 虚拟人完整全教程

> 
> 目标：普通 USB 摄像头 → MediaPipe 全身姿态识别 → 驱动 VRM 虚拟人，摄像头画面做背景，VRM 模型叠在真人画面前方（虚拟人遮挡真人画面）
> 环境：Windows、团结引擎 1.11.0、内置渲染管线、普通电脑摄像头
> 前置：电脑摄像头可用，网络正常

## 目录

1. 安装团结 Cowork + 团结引擎
2. 创建团结引擎项目（关键：选内置管线）
3. 导入 UniVRM（加载 VRM 模型）
4. 导入 MediaPipeUnityPlugin（摄像头全身姿态识别）
5. 场景搭建（层级、UI、相机、灯光）
6. 全部 C# 脚本粘贴与挂载设置
7. VRM 模型导入 + Humanoid 骨骼配置
8. 引用绑定、参数调试
9. 运行测试 + 常见报错排查
10. 进阶优化（IK 脚部锁定、动作平滑、坐标翻转）

---

# 1. 安装团结 Cowork + 团结引擎

你现在就在官网页面

1. 点击【下载团结 Cowork】，下载并安装团结 Cowork（等同于 Unity Hub，用来管理引擎版本、新建项目）
2. 打开团结 Cowork → 引擎管理 → 安装引擎版本：**团结引擎 1.11.0**
   - 安装组件勾选：**Windows Build Support + Visual Studio 编辑器支持**，其他可选
3. 等待引擎下载安装完成

# 2. 创建团结引擎项目

1. Cowork 首页 → 创建项目
2. 模板选择：**3D Core（内置渲染管线！！不要 URP！！）**

> 
> ❗重要：URP 管线会导致 RawImage 背景渲染出问题，新手直接用内置管线，避免大量渲染 bug

3. 项目名称：`VRMFullBodyMocap`，选择项目存放路径（路径**不要中文、不要空格**）
4. 点击创建，等待项目打开

# 3. 导入 UniVRM（VRM 模型加载库）

UniVRM 用来解析`.vrm`虚拟人模型，支持 VRM0/VRM1.0

1. 顶部菜单：【窗口】→【包管理器】
2. 包管理器左上角 `+` → 从 Git URL 添加包
粘贴：

```
https://github.com/vrm-c/UniVRM.git?path=/Assets/UniVRM10
```

> 
> 如果你的模型是旧版 VRM0，再添加一条：

```
https://github.com/vrm-c/UniVRM.git?path=/Assets/UniVRM0
```

3. 等待包下载、编译，完成后关闭包管理器

> 
> 如果 Git 导入失败（国内网络经常超时）：
> 去 UniVRM 的 Release 页面，下载离线`.unitypackage`，在项目窗口右键【导入包→自定义包】

# 4. 导入 MediaPipeUnityPlugin（摄像头全身姿态识别）

> 
> 推荐离线 unitypackage 导入，Git 在团结引擎容易失败

1. 打开仓库：[https://github.com/homuler/MediaPipeUnityPlugin](https://github.com/homuler/MediaPipeUnityPlugin)
2. 进入 Release，下载 `MediaPipeUnityPlugin.unitypackage`（Windows 版本）
3. 在团结引擎项目窗口空白处右键 →【导入包】→【自定义包】，选中下载好的 unitypackage，全部勾选导入
4. 导入完成后，等待插件编译
5. 【重要验证】打开示例场景：`MediaPipeUnityPlugin/Samples/Scenes/Pose`，点击 Play，摄像头启动，画面上出现人体 33 个姿态关键点，代表 MediaPipe 安装成功。

> 
> 如果摄像头打不开：关闭所有占用摄像头软件（微信、Python 脚本、OBS），给团结引擎开启摄像头权限

# 5. 场景搭建（Hierarchy 层级）

在层级窗口，删除默认的`Main Camera`和`Directional Light`，重新创建下面结构：

```
MainCamera (主相机)
└── BackgroundCanvas (Canvas)
    └── BackgroundRawImage (RawImage)
DirectionalLight (平行光，照亮VRM模型)
VRMAvatarRoot (空物体，VRM模型父物体)
```

### 分步创建

1. 右键层级 →【相机】命名`MainCamera`
2. 右键 MainCamera →【UI → Canvas】命名`BackgroundCanvas`
   - 选中 Canvas，设置：渲染模式 = **屏幕空间 - 相机**，指定 MainCamera 为目标相机
3. 右键 BackgroundCanvas →【UI → RawImage】命名`BackgroundRawImage`
   - 选中 RawImage，点击 RectTransform 左上角锚点，按住 Alt，选择**铺满整个画布**，RawImage 铺满屏幕
4. 右键层级 →【灯光→方向光】命名`DirectionalLight`，位置随便，强度默认 1
5. 右键层级 →【创建空物体】命名`VRMAvatarRoot`（用来放 VRM 模型）

# 6. 新建全部 C# 脚本

在 Project 窗口，右键 →【C# 脚本】，新建 3 个脚本，复制粘贴下面代码

## 脚本 1：MediaPipePoseProvider.cs

挂载到 `MainCamera`

```
using UnityEngine;
using Mediapipe;
using Mediapipe.Tasks.Vision.PoseLandmarker;

public class MediaPipePoseProvider : MonoBehaviour
{
    [Header("MediaPipe姿态识别")]
    public PoseLandmarkerRunner poseRunner;
    [ReadOnly] public bool isDetected;
    [ReadOnly] public PoseLandmarkResult currentResult;

    void Start()
    {
        // 使用Lite轻量模型，CPU可流畅运行
        var config = PoseLandmarkerConfig.Create(PoseLandmarkerModel.Lite);
        poseRunner = new PoseLandmarkerRunner(config);
        poseRunner.OnPoseLandmarkerOutput += OnPoseOutput;
        poseRunner.Start();
    }

    void Update()
    {
        poseRunner?.ProcessNextFrame();
    }

    void OnPoseOutput(PoseLandmarkResult result)
    {
        currentResult = result;
        isDetected = result.poseLandmarks != null && result.poseLandmarks.Count > 0;
    }

    void OnDestroy()
    {
        poseRunner?.Stop();
    }
}
```

## 脚本 2：PoseToVRMDriver.cs

挂载到 `VRMAvatarRoot`

```
using UnityEngine;
using HumanBodyBones = UnityEngine.HumanBodyBones;

public class PoseToVRMDriver : MonoBehaviour
{
    [Header("引用绑定")]
    public MediaPipePoseProvider poseProvider;
    public Animator avatarAnimator;

    [Header("动作平滑")]
    [Range(1, 30)] public float smoothSpeed = 12f;
    public bool invertX = true; // 摄像头画面左右翻转，默认开启

    private Transform hips, spine, head;
    private Transform leftShoulder, leftElbow, leftWrist;
    private Transform rightShoulder, rightElbow, rightWrist;
    private Transform leftHip, leftKnee, leftAnkle;
    private Transform rightHip, rightKnee, rightAnkle;

    void Start()
    {
        // 读取Humanoid骨骼
        hips = avatarAnimator.GetBoneTransform(HumanBodyBones.Hips);
        spine = avatarAnimator.GetBoneTransform(HumanBodyBones.Spine);
        head = avatarAnimator.GetBoneTransform(HumanBodyBones.Head);

        leftShoulder = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftUpperArm);
        leftElbow = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftLowerArm);
        leftWrist = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftHand);

        rightShoulder = avatarAnimator.GetBoneTransform(HumanBodyBones.RightUpperArm);
        rightElbow = avatarAnimator.GetBoneTransform(HumanBodyBones.RightLowerArm);
        rightWrist = avatarAnimator.GetBoneTransform(HumanBodyBones.RightHand);

        leftHip = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftUpperLeg);
        leftKnee = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftLowerLeg);
        leftAnkle = avatarAnimator.GetBoneTransform(HumanBodyBones.LeftFoot);

        rightHip = avatarAnimator.GetBoneTransform(HumanBodyBones.RightUpperLeg);
        rightKnee = avatarAnimator.GetBoneTransform(HumanBodyBones.RightLowerLeg);
        rightAnkle = avatarAnimator.GetBoneTransform(HumanBodyBones.RightFoot);
    }

    void Update()
    {
        if (!poseProvider.isDetected) return;

        var landmarks = poseProvider.currentResult.poseLandmarks[0];

        // 手臂IK映射
        ApplyArmRotation(landmarks[12], landmarks[14], landmarks[16], rightShoulder, rightElbow, rightWrist);
        ApplyArmRotation(landmarks[11], landmarks[13], landmarks[15], leftShoulder, leftElbow, leftWrist);
        // 腿部IK映射
        ApplyLegRotation(landmarks[24], landmarks[26], landmarks[28], rightHip, rightKnee, rightAnkle);
        ApplyLegRotation(landmarks[23], landmarks[25], landmarks[27], leftHip, leftKnee, leftAnkle);

        // 头部跟随
        Vector3 headDir = new Vector3(landmarks[0].x - landmarks[12].x, 0, landmarks[0].z - landmarks[12].z);
        if (invertX) headDir.x *= -1;
        Quaternion headTargetRot = Quaternion.LookRotation(headDir);
        head.rotation = Quaternion.Lerp(head.rotation, headTargetRot, Time.deltaTime * smoothSpeed);
    }

    void ApplyArmRotation(Landmark shoulderLm, Landmark elbowLm, Landmark wristLm, Transform upperArm, Transform lowerArm, Transform hand)
    {
        Vector3 sh = new Vector3(shoulderLm.x, shoulderLm.y, shoulderLm.z);
        Vector3 el = new Vector3(elbowLm.x, elbowLm.y, elbowLm.z);
        Vector3 wr = new Vector3(wristLm.x, wristLm.y, wristLm.z);
        if (invertX)
        {
            sh.x *= -1; el.x *= -1; wr.x *= -1;
        }

        Quaternion upperTarget = Quaternion.FromToRotation(Vector3.right, el - sh);
        upperArm.rotation = Quaternion.Lerp(upperArm.rotation, upperTarget, Time.deltaTime * smoothSpeed);

        Quaternion lowerTarget = Quaternion.FromToRotation(Vector3.right, wr - el);
        lowerArm.rotation = Quaternion.Lerp(lowerArm.rotation, lowerTarget, Time.deltaTime * smoothSpeed);
    }

    void ApplyLegRotation(Landmark hipLm, Landmark kneeLm, Landmark ankleLm, Transform upperLeg, Transform lowerLeg, Transform foot)
    {
        Vector3 hp = new Vector3(hipLm.x, hipLm.y, hipLm.z);
        Vector3 kn = new Vector3(kneeLm.x, kneeLm.y, kneeLm.z);
        Vector3 ak = new Vector3(ankleLm.x, ankleLm.y, ankleLm.z);
        if (invertX)
        {
            hp.x *= -1; kn.x *= -1; ak.x *= -1;
        }

        Quaternion upperTarget = Quaternion.FromToRotation(Vector3.right, kn - hp);
        upperLeg.rotation = Quaternion.Lerp(upperLeg.rotation, upperTarget, Time.deltaTime * smoothSpeed);

        Quaternion lowerTarget = Quaternion.FromToRotation(Vector3.right, ak - kn);
        lowerLeg.rotation = Quaternion.Lerp(lowerLeg.rotation, lowerTarget, Time.deltaTime * smoothSpeed);
    }
}
```

## 脚本 3：BackgroundCameraFeed.cs

挂载到 `MainCamera`

```
using UnityEngine;
using UnityEngine.UI;
using Mediapipe;

public class BackgroundCameraFeed : MonoBehaviour
{
    [Header("背景RawImage")]
    public RawImage rawImage;
    public MediaPipePoseProvider poseProvider;

    void Update()
    {
        if (poseProvider.poseRunner != null && rawImage != null)
        {
            // 将MediaPipe摄像头画面输出到UI背景
            rawImage.texture = poseProvider.poseRunner.texture;
        }
    }
}
```

> 
> 新建脚本后，团结引擎会自动编译代码，如果报红色错误，检查包是否全部导入完成。

# 7. 导入 VRM 模型 + Humanoid 骨骼设置

1. 把你的 `.vrm` 文件，直接拖进 Project 窗口，UniVRM 自动解析模型
2. 模型解析完成，把生成的 VRM 预制体，拖进层级窗口，作为`VRMAvatarRoot`的子物体
3. 选中 VRM 模型，找到 `Animator` 组件
   - Avatar 选项：**必须是 Humanoid 类型的 Avatar**（VRM 导入后 UniVRM 会自动生成 Humanoid Avatar）

> 
> 检查：选中模型 → Inspector Animator，Avatar 那里不是 None，并且标着 Humanoid。
> 如果不是 Humanoid：选中 VRM 模型的源文件，点击【配置】，骨骼类型选择 Humanoid，应用。

# 8. Inspector 面板引用绑定（最重要！漏了直接不工作）

1. 选中 MainCamera
   - `MediaPipePoseProvider` 脚本：无需额外赋值
   - `BackgroundCameraFeed`脚本：
     - rawImage 拖拽赋值场景里的`BackgroundRawImage`
     - poseProvider 拖拽赋值 MainCamera 上的`MediaPipePoseProvider`
2. 选中层级窗口 `VRMAvatarRoot`
   - `PoseToVRMDriver`脚本：
     - poseProvider：拖拽 MainCamera 物体
     - avatarAnimator：拖拽 VRM 模型身上的 Animator 组件

# 9. 运行测试

1. 点击顶部 Play 按钮进入运行模式
2. 摄像头自动启动，你的画面显示在背景 UI 上
3. 站在镜头前，**全身全部入镜（头、手脚全部在画面里）**
4. 识别成功后，VRM 虚拟人会模仿你的肢体动作，叠加在你的真人画面前面

# 10. 常见报错排查清单

表格

| 现象 | 解决方案 |
| --- | --- |
| 代码报找不到 Mediapipe 命名空间 | MediaPipeUnityPlugin 包导入失败，重新导入 unitypackage |
| 摄像头黑屏，没有画面 | 关闭占用摄像头软件；检查 MediaPipe 示例场景能否打开摄像头；重启引擎 |
| VRM 模型不动 | 检查 Animator 的 Avatar 是 Humanoid；检查脚本引用是否全部拖拽绑定；全身必须完整入镜 |
| 模型动作左右颠倒 | PoseToVRMDriver 脚本取消勾选`invertX` |
| 动作剧烈抖动 | 调高 smoothSpeed 数值，15~20 之间 |
| 模型动作完全反向 | invertX 开启，同时检查骨骼轴向 |
| 编译报错缺少 ReadOnly | 在脚本最顶部加入`using UnityEngine;`，或者把 [ReadOnly] 删掉 |

# 11. 进阶可选优化

## 优化 1：脚部 IK 锁定（防止虚拟人脚飘起来）

新建脚本 `AvatarFootIK.cs` 挂载 VRM 模型，实现脚贴地

```
using UnityEngine;
using HumanBodyBones = UnityEngine.HumanBodyBones;

public class AvatarFootIK : MonoBehaviour
{
    public Animator animator;
    public float footOffset = 0.05f;
    private Transform leftFoot, rightFoot;

    void Start()
    {
        leftFoot = animator.GetBoneTransform(HumanBodyBones.LeftFoot);
        rightFoot = animator.GetBoneTransform(HumanBodyBones.RightFoot);
    }

    void OnAnimatorIK(int layerIndex)
    {
        animator.SetIKPositionWeight(AvatarIKGoal.LeftFoot,1);
        animator.SetIKRotationWeight(AvatarIKGoal.LeftFoot,1);
        animator.SetIKPosition(AvatarIKGoal.LeftFoot,leftFoot.position + Vector3.up*footOffset);

        animator.SetIKPositionWeight(AvatarIKGoal.RightFoot,1);
        animator.SetIKRotationWeight(AvatarIKGoal.RightFoot,1);
        animator.SetIKPosition(AvatarIKGoal.RightFoot,rightFoot.position + Vector3.up*footOffset);
    }
}
```

> 
> 使用：给 VRM 的 Animator 开启 IK 通道（Animator -> Layer -> IK Pass 勾选）

## 优化 2：增加低通滤波，大幅度降低动作抖动

在 PoseToVRMDriver 内部增加平滑滤波，解决 2D 摄像头 3D 估算抖动问题

## 优化 3：手部识别升级

把 MediaPipe Pose 换成**Holistic 模型**，同时识别全身 + 手部 21 点，可以驱动 VRM 手指。
