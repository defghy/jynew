# Unity + Lua 项目上手说明

本文面向有 Web 前端背景、刚接触 Unity + Lua 的开发者。内容基于当前合并后的项目结构整理，重点帮助你快速跑起来、知道从哪里读代码、如何调试一个按钮，以及初期适合做哪些小改动。

## 1. 项目概览

本仓库主体是一个 Unity 武侠 RPG 项目，根目录下的 `jyx2/` 才是真正的 Unity 工程目录。

项目大体分为三层：

- Unity/C# 框架层：负责启动流程、资源加载、UI 框架、场景控制、存档、战斗系统、输入、平台能力等。
- Lua 逻辑层：负责剧情指令、事件脚本、部分战斗逻辑、配置读取、MOD 热更新逻辑等。
- MOD/配置/资源层：每个可游玩的内容都按 MOD 组织，配置主要由 Excel 编写，运行时转成 Lua 表。

当前 Unity 版本：

```text
2022.3.62f3c1
```

对应文件：

```text
jyx2/ProjectSettings/ProjectVersion.txt
```

请优先使用这个 Unity 版本打开项目。使用旧版本打开可能触发资源升级、Package 解析失败或序列化差异。

## 2. 重要目录地图

```text
jyx2/
  Assets/
    0_Init.unity
    0_GameStart.unity
    0_MODLoaderScene.unity
    0_MainMenu.unity
    Scripts/
      GameCore/
      LuaCore/
      UI/
      GameMaps/
      GameBattle/
      GameSave/
      ResourceManagement/
      MOD/
      Utils/
    LuaScripts/
    BuildSource/
      Lua/
    Mods/
      JYX2/
      SAMPLE/
      xiastart_roguelike/
    Prefabs/
      Jyx2UI/
    Resources/
    XLua/
  Packages/
  ProjectSettings/
```

重点关注：

- `Assets/Scripts/GameCore/`：游戏启动、运行环境、全局常量。
- `Assets/Scripts/LuaCore/`：XLua 初始化、C# 与 Lua 的桥接。
- `Assets/Scripts/UI/`：UGUI 面板逻辑。
- `Assets/Scripts/UI/UIData/`：由工具生成的 UI 子节点引用代码，类似前端里的 refs。
- `Assets/Prefabs/Jyx2UI/`：UI prefab，相当于 UI 模板或组件树。
- `Assets/BuildSource/Lua/main.lua`：Lua 全局剧情 API 入口。
- `Assets/LuaScripts/`：Lua 模块，比如配置管理、战斗逻辑。
- `Assets/Mods/*/Configs/`：MOD 的 Excel 配置表。
- `Assets/Mods/*/Lua/`：MOD 剧情或事件 Lua 脚本。
- `Assets/Mods/*/Skills/`：合并后已有的技能表现资源，很多招式配置在这里。

## 3. 启动链路

Build Settings 当前启用的场景顺序是：

```text
Assets/0_Init.unity
Assets/0_GameStart.unity
Assets/0_MODLoaderScene.unity
Assets/0_MainMenu.unity
```

对应配置：

```text
jyx2/ProjectSettings/EditorBuildSettings.asset
```

主流程如下：

1. `0_Init.unity`
   - 挂载 `InitScene`。
   - 普通编辑器运行时直接加载 `0_GameStart`。
   - TapTap 编译宏开启时会走登录和防沉迷逻辑。

2. `0_GameStart.unity`
   - 挂载 `GameStart`。
   - 先播放 intro 淡入淡出。
   - 然后调用 `ModPanelNew.SwitchSceneTo()` 进入 MOD 选择。

### `GameStart.cs` 做了什么

`GameStart.cs` 是 `0_GameStart.unity` 场景里的启动过渡脚本。它不负责真正初始化游戏运行环境，也不直接打开主菜单；它更像一个“启动中转页”。

文件位置：

```text
jyx2/Assets/Scripts/GameCore/GameStart.cs
```

它的执行流程：

1. Unity 调用 `Start()`。
2. `Start()` 先调用 `FixSaves()`。
3. `FixSaves()` 做一次旧存档迁移：
   - 检查 `PlayerPrefs` 里是否已有 `save_fixed_20221204_1` 标记。
   - 如果没有，就检查旧目录 `wuxia_launch` 是否存在。
   - 如果存在，把旧目录内容复制到当前 `Application.persistentDataPath`。
   - 完成后写入 `save_fixed_20221204_1`，避免下次重复迁移。
4. `Start()` 再异步执行 `StartAsync().Forget()`。
5. `StartAsync()` 显示 `introPanel`。
6. 使用 DOTween 把 `introPanel` 从透明渐变到可见，等待 1 秒，再渐变隐藏。
7. 淡出完成后销毁 `introPanel`。
8. 注册 `Application.logMessageReceived += OnErrorMsg`。
9. 调用 `ModPanelNew.SwitchSceneTo()`，切换到 MOD 选择流程。

可以把它理解为前端里的启动 splash 页面：

```text
旧数据迁移 -> 播放开场过渡动画 -> 注册全局错误监听 -> 跳转到 MOD 选择页
```

其中 `introPanel` 是 Unity Inspector 上绑定的 `CanvasGroup`，动画用的是 DOTween：

```csharp
await introPanel.DOFade(1, 1f).SetEase(Ease.Linear);
await UniTask.Delay(TimeSpan.FromSeconds(1f));
await introPanel.DOFade(0, 1f).SetEase(Ease.Linear)
```

`OnErrorMsg()` 目前只在收到 `Exception` 或 `Error` 日志时，额外打印应用版本和触发时间，方便后续定位线上错误：

```text
Exception版本:{Application.version}, 触发时间:{DateTime.Now}
Error版本:{Application.version}, 触发时间:{DateTime.Now}
```

排查启动问题时，可以这样判断：

- 卡在 intro 前：看 `FixSaves()` 是否迁移旧存档时报错。
- intro 播完没进 MOD 选择：看 `ModPanelNew.SwitchSceneTo()` 或 MOD 场景加载。
- Console 重复出现版本和时间日志：说明有 Error/Exception 被全局日志钩子捕获，继续看它前面的真实异常。

3. `0_MODLoaderScene.unity`
   - 展示 MOD 选择面板。
   - 选择后设置当前 MOD。

4. `0_MainMenu.unity`
   - 挂载 `BootMainMenu`。
   - 调用 `Jyx2_UIManager.Instance.GameStart()`。
   - 初始化运行环境，加载主菜单 UI。

核心入口文件：

```text
jyx2/Assets/Scripts/GameCore/InitScene.cs
jyx2/Assets/Scripts/GameCore/GameStart.cs
jyx2/Assets/Scripts/GameCore/BootMainMenu.cs
jyx2/Assets/Scripts/UI/Framework/Jyx2_UIManager.cs
jyx2/Assets/Scripts/GameCore/RuntimeEnvSetup.cs
```

## 4. 运行环境初始化

`RuntimeEnvSetup.Setup()` 是理解项目运行时的关键入口。它大致做这些事：

1. 读取全局资源配置 `GlobalAssetConfig`。
2. 在 Editor 下尝试根据当前场景推断正在调试的 MOD。
3. 初始化 `ResLoader`。
4. 启动当前 MOD 的资源环境。
5. 加载当前 MOD 的 `ModSetting.asset`。
6. 初始化游戏设置。
7. 初始化模型、技能、Lua、配置表。
8. 执行 Lua MOD 初始化。

对应文件：

```text
jyx2/Assets/Scripts/GameCore/RuntimeEnvSetup.cs
jyx2/Assets/Scripts/ResourceManagement/Jyx2ResourceHelper.cs
```

你遇到启动失败、配置表没加载、MOD 不对、资源路径不对时，优先从这里看 Console 日志。

## 5. Lua 初始化与脚本体系

Lua 使用 XLua 接入。核心文件：

```text
jyx2/Assets/Scripts/LuaCore/LuaManager.cs
jyx2/Assets/Scripts/LuaCore/LuaExecutor.cs
jyx2/Assets/Scripts/LuaCore/Jyx2LuaBridge.cs
jyx2/Assets/Scripts/LuaCore/Jyx2LuaToCsBridge.cs
jyx2/Assets/BuildSource/Lua/main.lua
jyx2/Assets/LuaScripts/InitLuaScripts.lua
```

`LuaManager.Init()` 会创建 `LuaEnv`，并注册 Lua loader。当前 loader 会从这些位置找 Lua：

```text
Assets/LuaScripts/{filename}.lua
Assets/BuildSource/Lua/{filename}.lua
```

Editor 下执行 MOD Lua 时，`LuaManager.LoadLua()` 会直接读：

```text
Assets/Mods/{CurrentModId}/Lua/{path}.lua
```

这个热加载行为对调试很友好。修改 Lua 后通常不用重新打包，只要重新触发对应事件即可。

`main.lua` 把 C# 的 `Jyx2LuaBridge` 包成全局剧情 API，例如：

```lua
Talk(...)
TryBattle(...)
AddItem(...)
ShowToast(...)
SetFlag(...)
GetFlag(...)
scene_api.BindEvent(...)
```

所以你看 MOD 的 Lua 剧情脚本时，会看到很多像脚本指令一样的函数调用。

## 6. UI 框架如何理解

这个项目使用 UGUI，不是 UI Toolkit。对前端来说，可以这样类比：

| Web 前端 | 本项目 |
| --- | --- |
| JSX/模板 | Prefab |
| DOM 节点 | GameObject/RectTransform |
| ref | `*_UIData.cs` 中的字段 |
| 组件逻辑 | `GameMainMenu.cs`、`BagUIPanel.cs` 等面板脚本 |
| onClick | `BindListener(button, handler)` |
| 路由/弹窗打开 | `Jyx2_UIManager.ShowUIAsync(...)` |
| 组件挂载 | `OnCreate()` |
| 组件显示 | `OnShowPanel(...)` |
| 组件隐藏 | `OnHidePanel()` |

UI 加载规则在 `GameConst.UI_PREFAB_PATH`：

```csharp
public const string UI_PREFAB_PATH = "Assets/Prefabs/Jyx2UI/{0}.prefab";
```

因此：

```csharp
await Jyx2_UIManager.Instance.ShowUIAsync(nameof(GameMainMenu));
```

会加载：

```text
Assets/Prefabs/Jyx2UI/GameMainMenu.prefab
```

再自动给这个 prefab 挂上同名脚本 `GameMainMenu`。

### 主菜单按钮例子

先看：

```text
jyx2/Assets/Scripts/UI/GameMainMenu.cs
```

在 `RegisterEvent()` 里可以看到按钮绑定：

```csharp
BindListener(this.NewGameButton_Button, OnNewGameClicked);
BindListener(this.LoadGameButton_Button, OnLoadGameClicked);
BindListener(this.GameSettingsButton_Button, OpenSettingsPanel);
BindListener(this.QuitGameButton_Button, OnQuitGameClicked);
```

而这些按钮字段来自：

```text
jyx2/Assets/Scripts/UI/UIData/GameMainMenu_UIData.cs
```

例如：

```csharp
NewGameButton_Button = transform.Find("mainPanel/homeBtnAndTxtPanel/NewGameButton").GetComponent<Button>();
```

这就像在模板里找到一个按钮 ref，然后在组件初始化时绑定点击事件。

## 7. MOD 与配置表

项目里所有可游玩内容都被视为 MOD。每个 MOD 通常有：

```text
Assets/Mods/{ModId}/
  ModSetting.asset
  Configs/
  Configs/Lua/
  Lua/
  Skills/
  BuildSource/
  Maps/
```

`ModSetting.asset` 对应类型：

```text
jyx2/Assets/Scripts/MOD/MODRootConfig.cs
```

这里配置 MOD ID、名称、作者、Lua 文件名规则、存档版本、是否只允许大地图存档、是否自动战斗、战斗倍速、技能名显示等。

配置表使用 Excel 编写，例如：

```text
Assets/Mods/SAMPLE/Configs/人物.xlsx
Assets/Mods/SAMPLE/Configs/物品.xlsx
Assets/Mods/SAMPLE/Configs/场景.xlsx
Assets/Mods/SAMPLE/Configs/战斗.xlsx
Assets/Mods/SAMPLE/Configs/武功.xlsx
Assets/Mods/SAMPLE/Configs/游戏设置.xlsx
```

运行 Editor 时，项目会自动把 Excel 导出成 Lua 配置表：

```text
Assets/Mods/{ModId}/Configs/Lua/
```

也可以在 Unity 里选中某个 MOD 的 `ModSetting.asset`，在 Inspector 点击“生成配置表”手动生成。

注意：不要手动改 `Configs/Lua/` 里的生成结果。下次启动或生成配置表时会被覆盖。要改数据，改 Excel。

## 8. 推荐上手步骤

### 第 1 步：打开项目

用 Unity Hub 打开：

```text
jyx2/
```

不要打开仓库根目录。

Unity 版本使用：

```text
2022.3.62f3c1
```

第一次导入可能较慢。等 Package、脚本编译、资源导入完成后再操作。Package 依赖里有 Gitee Git URL，网络不稳定时 Package Manager 可能会卡住；这种情况先看 Unity Console 和 Package Manager 报错，而不是急着改代码。

### 第 2 步：从启动场景运行

打开：

```text
Assets/0_Init.unity
```

点击 Play。正常路径应该是：

```text
Init -> GameStart -> MOD 选择 -> MainMenu
```

建议先选 `SAMPLE`，它相对适合学习 MOD 结构。

如果 MOD 选择或当前 MOD 状态异常，可以试 Unity 菜单：

```text
Game Tools/修复启动MOD
```

对应代码：

```text
Assets/Editor/FixModTool.cs
```

### 第 3 步：看主菜单 UI

先读：

```text
Assets/Scripts/UI/GameMainMenu.cs
Assets/Scripts/UI/UIData/GameMainMenu_UIData.cs
Assets/Prefabs/Jyx2UI/GameMainMenu.prefab
```

目标是搞懂三件事：

- prefab 里的按钮路径是什么。
- UIData 如何把按钮变成 C# 字段。
- GameMainMenu 如何绑定点击事件。
- `OnShowPanel()` 什么时候触发，以及它如何调用 `RuntimeEnvSetup.Setup()` 初始化当前 MOD。

### 第 4 步：追一次新游戏点击

从 `OnNewGameClicked()` 往下追：

```text
OnNewGameClicked
OnNewGame
OnCreateBtnClicked
OnCreateRoleYesClick
```

你会看到角色创建、随机属性、读取起始地图、进入地图加载等逻辑。

### 第 5 步：看一条 Lua 剧情

打开：

```text
Assets/Mods/SAMPLE/Lua/
```

找一个短 Lua 文件，看里面的：

```lua
Talk(...)
AddItem(...)
TryBattle(...)
SetFlag(...)
```

再回到：

```text
Assets/BuildSource/Lua/main.lua
Assets/Scripts/LuaCore/Jyx2LuaBridge.cs
```

理解这些 Lua 指令最终如何调用 C#。

### 第 6 步：改一个配置表

在 `Assets/Mods/SAMPLE/Configs/` 里改一个简单字段，比如物品名称、游戏设置、角色初始数据。

然后运行游戏或手动点击 `ModSetting.asset` 的“生成配置表”。

观察：

```text
Assets/Mods/SAMPLE/Configs/Lua/
```

里生成的 Lua 配置变化。

## 9. 适合初期练手的小功能

### 任务 A：调试主菜单“新游戏”按钮

目标：确认 UI 点击链路。

修改：

```text
Assets/Scripts/UI/GameMainMenu.cs
```

在 `OnNewGameClicked()` 加日志或断点：

```csharp
Debug.Log("Click New Game");
```

运行后点击“新游戏”，在 Console 确认日志出现。若想用 IDE 断点，建议安装 Unity 对应 IDE 插件，并从 IDE 附加到 Unity Editor 进程。

进一步可以在 `OnNewGame()`、`OnCreateBtnClicked()`、`OnCreateRoleYesClick()` 各加一个日志，串起完整链路。

### 任务 B：给空名字增加提示

当前 `SetPlayerName(string newName)` 遇到空名字会直接 return。可以改成弹提示：

```csharp
if (string.IsNullOrWhiteSpace(newName))
{
    StoryEngine.DisplayPopInfo("请输入名字");
    return;
}
```

这是一个很适合前端同学入门的改动：表单校验、用户反馈、UI 调试，一次都碰到。

### 任务 C：调试一个弹窗按钮

看：

```text
Assets/Scripts/UI/SystemUIPanel.cs
Assets/Scripts/UI/SavePanel.cs
Assets/Scripts/UI/ChatUIPanel.cs
```

从系统菜单里打开读档/存档/确认弹窗，找到按钮绑定点，加日志或断点。

### 任务 D：改一个 Lua 剧情提示

在 `Assets/Mods/SAMPLE/Lua/` 找到一条 `Talk(...)` 或 `ShowToast(...)`，改文案，重新触发对应事件。

这个任务可以帮助你理解“不改 C#，只改 Lua/MOD 内容”的工作方式。

### 任务 E：改一个配置数据

在 `Assets/Mods/SAMPLE/Configs/物品.xlsx` 或 `游戏设置.xlsx` 里改一个低风险字段。生成配置表后运行验证。

这个任务可以帮助你理解 Excel -> Lua 配置 -> C#/Lua 读取链路。

## 10. 调试建议

- 优先使用 Unity Console 看异常栈。很多初始化失败会在 `RuntimeEnvSetup.Setup()` 里打出错误。
- C# 逻辑建议用 Rider/VS Code 断点调试 Unity Editor。
- Lua 剧情建议先加 `print(...)` 或调用 `ShowToast(...)`，确认脚本是否被执行。
- Editor 下 Lua 支持热加载，修改 MOD Lua 后通常不需要重新打包。
- 如果 UI 字段找不到，先检查 prefab 层级路径是否和 `*_UIData.cs` 里的 `transform.Find(...)` 一致。
- 如果改了 prefab 子节点名，需要重新生成 UIData，入口大概率在 UI 相关 Editor 工具中。
- 如果配置没有生效，先确认改的是 Excel，不是生成后的 Lua 文件。
- 如果当前 MOD 不对，先回到 MOD 选择界面或使用“修复启动MOD”菜单。

## 11. 常见排查点

### Unity 版本不对

症状：Package 报错、资源大量重导、场景或 prefab 序列化异常。

处理：使用 `2022.3.62f3c1`，以 `jyx2/ProjectSettings/ProjectVersion.txt` 为准。

### 打开了错误目录

症状：Unity Hub 识别不到项目，或 Assets/ProjectSettings 不在预期位置。

处理：打开 `jyx2/`，不是仓库根目录。

### MOD 配置表为空

症状：启动时报“没有找到配置表”。

处理：确认当前 MOD 下有 `Configs/Lua/`，或选中 `ModSetting.asset` 点击“生成配置表”。

### UI 点击没反应

排查顺序：

1. prefab 上按钮对象是否存在并 active。
2. `*_UIData.cs` 的路径是否找得到该按钮。
3. 面板 `OnCreate()` 是否调用了 `InitTrans()` 和 `RegisterEvent()`。
4. `BindListener(...)` 是否绑定到正确 handler。
5. 是否有其他 UI 层挡住点击。

### Lua 文件没执行

排查顺序：

1. 当前 MOD 是否正确。
2. Lua 文件名是否符合 `ModSetting.asset` 里的 `LuaFilePatten`。
3. 事件配置表是否绑定到了对应 Lua。
4. Console 是否有 Lua 执行错误。
5. Editor 下实际读取路径是否是 `Assets/Mods/{CurrentModId}/Lua/`。

## 12. 建议阅读顺序

第一轮只看链路，不追细节：

1. `Assets/Scripts/GameCore/InitScene.cs`
2. `Assets/Scripts/GameCore/GameStart.cs`
3. `Assets/Scripts/MOD/ModV2/UI/ModPanelNew.cs`
4. `Assets/Scripts/GameCore/BootMainMenu.cs`
5. `Assets/Scripts/UI/Framework/Jyx2_UIManager.cs`
6. `Assets/Scripts/GameCore/RuntimeEnvSetup.cs`
7. `Assets/Scripts/ResourceManagement/Jyx2ResourceHelper.cs`
8. `Assets/Scripts/UI/GameMainMenu.cs`
9. `Assets/BuildSource/Lua/main.lua`
10. `Assets/Mods/SAMPLE/Configs/README.md`

第二轮选一个方向深入：

- 想改 UI：读 `Jyx2_UIManager`、`Jyx2_UIBase`、具体 Panel、对应 Prefab。
- 想改剧情：读 `main.lua`、`LuaExecutor`、`Jyx2LuaBridge`、MOD Lua。
- 想改数值：读 `Configs/README.md`、Excel 配置、`Jyx2ConfigMgr`。
- 想改战斗：读 `Assets/Scripts/GameBattle/` 和 `Assets/LuaScripts/Jyx2Battle/`。
- 想做 MOD：读 `MODRootConfig`、`ModEditorWindow`、`Assets/Mods/SAMPLE/`。

## 13. 给前端同学的一点心法

不要一开始试图理解所有 Unity 概念。先把它当作一个大型组件系统：

- 场景是页面。
- prefab 是组件模板。
- GameObject 是节点。
- MonoBehaviour 是组件逻辑。
- Inspector 是可视化 props 面板。
- ScriptableObject 是可序列化配置对象。
- Excel/Lua 是内容和玩法脚本。
- `Jyx2_UIManager.ShowUIAsync` 是打开页面或弹窗。

最短上手路径是：

```text
跑起来 -> 选 SAMPLE -> 点主菜单按钮 -> 改一条按钮日志 -> 改一条空名字提示 -> 改一个 Lua 文案 -> 改一个 Excel 配置
```

这条路径走完，你就已经穿过了 Unity、C#、UGUI、Lua、MOD、配置表这几个最重要的区域。
