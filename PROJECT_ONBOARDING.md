# 项目上手指南（Unity + Lua）

本文面向有 Web 前端经验、刚接触 Unity、C# 和 Lua 的开发者。目标不是一次读懂整个项目，而是在较短时间内完成下面几件事：

1. 用正确的 Unity 版本打开工程并跑到主菜单。
2. 知道启动、UI、Lua、MOD、配置表分别从哪里进入。
3. 能独立追踪一次按钮点击，并完成一个低风险改动。
4. 遇到常见启动问题时，知道先看哪里。

本文结论以当前仓库文件为依据。发生冲突时，以下文件比旧 README、截图或外部教程更可信：

- Unity 版本：[ProjectVersion.txt](jyx2/ProjectSettings/ProjectVersion.txt)
- 启动场景：[EditorBuildSettings.asset](jyx2/ProjectSettings/EditorBuildSettings.asset)
- Package 依赖：[manifest.json](jyx2/Packages/manifest.json)
- 实际源码与 Unity Console 日志

> **当前工程版本是 `2022.3.62f3c1`。** 根目录 README 上的 Unity 版本徽章仍是旧版本，不要以它作为安装依据。

## 1. 第一次运行：先完成这条最短路径

### 1.1 准备环境

建议准备：

- Unity Hub。
- Unity Editor `2022.3.62f3c1`。
- Rider、Visual Studio 或 VS Code，用于阅读和调试 C#。
- Excel、WPS 或其他可以编辑 `.xlsx` 的工具。

本项目不是通过 `npm install`、`dotnet run` 或命令行启动的。Unity 会负责 Package 解析、资源导入和 C# 编译。

### 1.2 打开正确目录

在 Unity Hub 中添加并打开：

```text
<仓库根目录>/jyx2
```

不要打开仓库根目录。Unity 项目必须直接包含 `Assets/`、`Packages/` 和 `ProjectSettings/`。

第一次打开时可能需要较长时间导入资源。先等待 Unity 右下角的导入、编译状态结束，再进行后续操作。

### 1.3 检查 Console

打开 Unity 的 `Window > General > Console`：

- 红色是 Error，优先处理。
- 黄色是 Warning，不一定阻塞运行，但要阅读内容。
- 双击日志可以跳到对应 C# 文件和行。

Package 配置中有多个 Gitee Git 依赖。若首次打开一直卡在 Package 解析，先检查网络和 Package Manager 日志，不要急着修改业务代码。

### 1.4 首次生成 XLua 代码

`jyx2/Assets/XLua/Gen/` 被 `.gitignore` 忽略，因此全新克隆后很可能没有生成代码。

在 Unity 第一次完成 C# 编译后，执行：

```text
XLua > Generate Code
```

然后再次等待 Unity 编译完成，并确认 Console 没有新的红色错误。

如果 `Assets/XLua/Gen/` 已经有大量 `.cs` 文件，且 MOD 选择界面没有提示“没有手动生成XLua代码”，通常不用重复生成。不要为了解决无关问题频繁清空或重建 XLua 代码，否则会增加排查噪音。

### 1.5 从标准启动场景运行

在 Project 窗口打开：

```text
Assets/0_Init.unity
```

点击顶部 Play 按钮。在 MOD 列表中选中 `SAMPLE` 后，点击启动按钮或双击该条目。正常流程是：

```text
0_Init
  -> 0_GameStart
  -> 0_MODLoaderScene
  -> 选择 SAMPLE
  -> 0_MainMenu
  -> 显示游戏主菜单
```

建议第一次选择 `SAMPLE`，它的目录较小，更适合熟悉 MOD、Lua 和配置结构。

到达主菜单，且 Console 没有阻塞运行的 Error，就可以认为基础环境已经跑通。

> **Unity 新手提醒：** Play Mode 中对 Scene、Prefab 实例或 Inspector 数据的很多修改，在退出 Play 后会被还原。练习编辑资源前先停止运行；修改 C# 和 Lua 时也要明确自己是否仍在 Play Mode。

## 2. 用前端思维理解 Unity

下面的类比不完全等价，但足够帮助你建立第一层心智模型：

| Web 前端 | Unity / 本项目 |
| --- | --- |
| 页面 | Scene |
| 组件模板 | Prefab |
| DOM 节点 | GameObject |
| 布局节点 | RectTransform |
| 组件逻辑 | MonoBehaviour / `Jyx2_UIBase` 子类 |
| props 配置 | Inspector 中序列化的字段 |
| ref | `*_UIData.cs` 中缓存的组件字段 |
| onClick | `Button.onClick` / `BindListener(...)` |
| 路由或弹窗管理 | `Jyx2_UIManager.ShowUIAsync(...)` |
| JSON/业务配置 | ScriptableObject、Excel、Lua 表 |
| 异步 Promise | `UniTask` |

阅读 C# 时先记住几个项目中常见的语法：

- `Awake()`、`Start()`：Unity 生命周期入口，由 Unity 自动调用。
- `[SerializeField] private ...`：私有字段，但可以在 Inspector 中绑定。
- `partial class`：同一个类拆在多个文件中。本项目的 UI 行为和 UIData 就使用这种方式组合。
- `async UniTask`：项目常用的异步方法，作用类似返回 Promise。
- `.Forget()`：启动异步任务但不等待结果。排错时要继续关注 Console 中的异常。
- `nameof(GameMainMenu)`：得到字符串 `"GameMainMenu"`，可避免手写类名字符串。

## 3. 项目目录地图

仓库根目录下的 `jyx2/` 才是真正的 Unity 工程：

```text
jyx2/
  Assets/
    0_Init.unity
    0_GameStart.unity
    0_MODLoaderScene.unity
    0_MainMenu.unity
    Editor/
    Scripts/
      GameCore/
      LuaCore/
      UI/
        Framework/
        UIData/
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

第一周最值得关注的目录：

| 目录 | 作用 |
| --- | --- |
| `Assets/Scripts/GameCore/` | 启动流程、运行环境、全局设置 |
| `Assets/Scripts/UI/` | UI 面板行为 |
| `Assets/Scripts/UI/UIData/` | UI 子节点引用，通常视为生成代码 |
| `Assets/Prefabs/Jyx2UI/` | UI Prefab |
| `Assets/Scripts/LuaCore/` | XLua 初始化以及 C# / Lua 桥接 |
| `Assets/BuildSource/Lua/main.lua` | 剧情脚本可调用的全局 Lua API |
| `Assets/LuaScripts/` | Lua 功能模块 |
| `Assets/Mods/{ModId}/Lua/` | MOD 事件和剧情脚本 |
| `Assets/Mods/{ModId}/Configs/` | Excel 配置源文件 |
| `Assets/Mods/{ModId}/Configs/Lua/` | 自动生成的运行时 Lua 配置 |
| `Assets/Mods/{ModId}/ModSetting.asset` | MOD 根配置 |

### 源文件与生成文件

| 文件 | 是否直接修改 | 说明 |
| --- | --- | --- |
| `Configs/*.xlsx` | 是 | 配置数据的真实来源 |
| `Configs/Lua/*.lua` | 否 | Editor 自动生成，且被 Git 忽略 |
| `Scripts/UI/*.cs` | 是 | UI 行为逻辑 |
| `Scripts/UI/UIData/*_UIData.cs` | 通常否 | UI 引用生成结果，但当前生成工具有历史路径问题 |
| `Assets/XLua/Gen/*.cs` | 否 | 通过 `XLua > Generate Code` 生成，且被 Git 忽略 |
| Unity 资源对应的 `.meta` | 不单独手改 | Unity 用它保存 GUID，提交资源时通常要一起提交 |

移动或重命名 Scene、Prefab、材质、贴图等 Unity 资源时，优先在 Unity Project 窗口中操作，这样 Unity 能同时维护 `.meta` 和引用关系。

## 4. 启动链路

Build Settings 当前启用的场景顺序是：

```text
Assets/0_Init.unity
Assets/0_GameStart.unity
Assets/0_MODLoaderScene.unity
Assets/0_MainMenu.unity
```

### 4.1 各场景职责

| 场景 | 入口 | 主要职责 |
| --- | --- | --- |
| `0_Init` | [InitScene.cs](jyx2/Assets/Scripts/GameCore/InitScene.cs) | 普通 Editor 模式直接进入 `0_GameStart`；TapTap 编译宏下处理登录和防沉迷 |
| `0_GameStart` | [GameStart.cs](jyx2/Assets/Scripts/GameCore/GameStart.cs) | 迁移旧存档、播放 intro、注册错误日志钩子、进入 MOD 选择 |
| `0_MODLoaderScene` | [ModPanelNew.cs](jyx2/Assets/Scripts/MOD/ModV2/UI/ModPanelNew.cs) | 发现并展示可用 MOD，设置当前 MOD |
| `0_MainMenu` | [BootMainMenu.cs](jyx2/Assets/Scripts/GameCore/BootMainMenu.cs) | 初始化运行环境并显示主菜单 |

### 4.2 `GameStart.cs` 的定位

`GameStart` 更像启动 splash 页面，而不是业务系统总入口：

```text
旧存档迁移
  -> intro 淡入淡出
  -> 注册 Error / Exception 日志补充信息
  -> ModPanelNew.SwitchSceneTo()
```

动画使用 DOTween，异步流程使用 UniTask。排查启动问题时：

- intro 之前失败：先看旧存档迁移和场景组件引用。
- intro 播放完成但没有 MOD 界面：看 `ModPanelNew.SwitchSceneTo()` 和场景加载错误。
- 出现“Error版本/触发时间”：它只是额外日志，真正异常通常在它之前。

### 4.3 真正的运行环境初始化

[RuntimeEnvSetup.cs](jyx2/Assets/Scripts/GameCore/RuntimeEnvSetup.cs) 是理解运行时的关键入口。`RuntimeEnvSetup.Setup()` 主要执行：

1. 加载全局资源配置 `GlobalAssetConfig`。
2. 在 Editor 中根据当前场景尝试推断调试 MOD。
3. 初始化 `ResLoader` 并启动当前 MOD 资源环境。
4. 加载当前 MOD 的 `ModSetting.asset`。
5. 初始化游戏设置、模型和技能资源。
6. 初始化 XLua、Lua 模块和配置表。
7. 执行 MOD Lua 初始化与热更新入口。

[Jyx2_UIManager.cs](jyx2/Assets/Scripts/UI/Framework/Jyx2_UIManager.cs) 的 `GameStart()` 会先调用 `RuntimeEnvSetup.Setup()`，然后打开 `GameMainMenu`。

`GameMainMenu.OnShowPanel()` 内部又会触发一次 `Setup()`，但 `RuntimeEnvSetup` 有 `_isSetup` 防重复初始化。追调用链时看到两次调用，不代表正常情况下会完整初始化两遍。

## 5. UI 框架与按钮调试

本项目使用 UGUI，不是 UI Toolkit。

UI Prefab 路径规则定义在 [GameConst.cs](jyx2/Assets/Scripts/GameCore/GameConst.cs)：

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

加载后，`Jyx2_UIManager` 会按照 Prefab 名查找或添加同名组件 `GameMainMenu`，再调用：

```text
Init()
  -> OnCreate()
Show(...)
  -> OnShowPanel(...)
```

### 5.1 主菜单的三个对应文件

- 行为逻辑：[GameMainMenu.cs](jyx2/Assets/Scripts/UI/GameMainMenu.cs)
- 节点引用：[GameMainMenu_UIData.cs](jyx2/Assets/Scripts/UI/UIData/GameMainMenu_UIData.cs)
- UI 结构：`Assets/Prefabs/Jyx2UI/GameMainMenu.prefab`

`GameMainMenu` 和 `GameMainMenu_UIData` 都是 `partial class GameMainMenu`，编译后会合并成同一个 C# 类。

`OnCreate()` 中先初始化引用，再绑定事件：

```csharp
protected override void OnCreate()
{
    InitTrans();
    RegisterEvent();
}
```

`RegisterEvent()` 中可以看到典型按钮绑定：

```csharp
BindListener(NewGameButton_Button, OnNewGameClicked);
BindListener(LoadGameButton_Button, OnLoadGameClicked);
BindListener(GameSettingsButton_Button, OpenSettingsPanel);
BindListener(QuitGameButton_Button, OnQuitGameClicked);
```

### 5.2 UI 字段找不到时怎么查

按下面的顺序排查：

1. Prefab 中对应节点是否存在、名称是否一致。
2. 节点上是否真的挂了 `Button`、`InputField` 等目标组件。
3. `*_UIData.cs` 中的 `transform.Find("...")` 路径是否仍然匹配 Prefab。
4. `OnCreate()` 是否调用 `InitTrans()` 和 `RegisterEvent()`。
5. `BindListener(...)` 的参数和 handler 是否正确。
6. 是否有其他全屏 UI 或 CanvasGroup 挡住点击。

### 5.3 UIData 生成工具的当前注意事项

项目提供了菜单：

```text
GameObject > 选中物体操作 > 生成UI脚本文件
GameObject > 选中物体操作 > 生成UIData文件
```

但 [UIExportEditor.cs](jyx2/Assets/Scripts/Editor/UIExportEditor.cs) 当前仍把输出目录写成旧路径：

```text
Assets/Scripts/Jyx2UIScripts/
```

而现有 UI 文件实际位于：

```text
Assets/Scripts/UI/
Assets/Scripts/UI/UIData/
```

因此不要在不了解影响时直接重新生成。若任务确实涉及 UIData，先确认或修正生成器输出路径，再检查生成 diff，避免把文件写到不存在的历史目录。

## 6. Lua、MOD 与配置表

### 6.1 三类常见 Lua

| 位置 | 用途 | 修改后常见生效方式 |
| --- | --- | --- |
| `Assets/BuildSource/Lua/main.lua` | 包装 C# 桥接，提供全局剧情 API | 通常重新进入运行流程 |
| `Assets/LuaScripts/` | 框架级 Lua 模块 | 可能受 `require` 缓存影响，通常重启 Play 更稳妥 |
| `Assets/Mods/{ModId}/Lua/` | MOD 事件、剧情脚本 | Editor 下执行事件时直接读文件，重新触发事件通常即可 |

[LuaManager.cs](jyx2/Assets/Scripts/LuaCore/LuaManager.cs) 会为 `require` 注册两个加载位置：

```text
Assets/LuaScripts/{filename}.lua
Assets/BuildSource/Lua/{filename}.lua
```

Editor 下执行 MOD 事件 Lua 时，`LuaManager.LoadLua()` 会直接读取：

```text
Assets/Mods/{CurrentModId}/Lua/{path}.lua
```

所以“Lua 支持热加载”主要适用于这类 MOD 事件脚本，不应理解为所有已加载 Lua 模块都会自动刷新。

`main.lua` 中可以看到剧情脚本常用 API：

```lua
Talk(...)
TryBattle(...)
AddItem(...)
ShowToast(...)
SetFlag(...)
GetFlag(...)
scene_api.BindEvent(...)
```

前端开发者阅读 Lua 时先记住：

- `local` 类似块级局部变量声明。
- `nil` 类似“没有值”。
- 不等于使用 `~=`，逻辑运算使用 `and`、`or`、`not`。
- Lua 数组通常从 `1` 开始，但本项目很多配置表是按业务 `Id` 做 key，不能机械地按数组理解。

### 6.2 MOD 目录

每个可游玩内容都被视为一个 MOD：

```text
Assets/Mods/{ModId}/
  ModSetting.asset
  Configs/
    Lua/
  Lua/
  Skills/
  BuildSource/
  Maps/
```

[MODRootConfig.cs](jyx2/Assets/Scripts/MOD/MODRootConfig.cs) 定义 `ModSetting.asset` 的字段，包括：

- MOD ID、名称、作者和版本。
- Lua 文件命名规则 `LuaFilePatten`。
- 存档版本和存档限制。
- 自动战斗、战斗倍速、技能名显示等玩法设置。

Editor 中的 MOD 选择器会扫描 `Assets/Mods/` 下名为 `ModSetting` 的配置资产，因此新增 MOD 时，目录存在但缺少有效 `ModSetting.asset` 仍不会正常出现在列表中。

### 6.3 Excel 配置生成流程

配置的真实来源是：

```text
Assets/Mods/{ModId}/Configs/*.xlsx
```

运行时使用：

```text
Assets/Mods/{ModId}/Configs/Lua/*.lua
```

Editor 初始化当前 MOD 时，[Jyx2ResourceHelper.cs](jyx2/Assets/Scripts/ResourceManagement/Jyx2ResourceHelper.cs) 会自动执行 Excel 到 Lua 的转换。也可以：

1. 在 Project 窗口选中该 MOD 的 `ModSetting.asset`。
2. 在 Inspector 中点击“生成配置表”。
3. 查看 Console 是否出现转换错误。

不要手动修改 `Configs/Lua/`。这些文件会被重新生成，而且当前项目配置为不提交到 Git。字段格式、类型和读取方式可继续阅读：

- [SAMPLE 配置说明](jyx2/Assets/Mods/SAMPLE/Configs/README.md)
- [ExcelToLua.cs](jyx2/Assets/Scripts/Utils/Tools/ExcelToLua.cs)
- [Jyx2LuaToCsBridge.cs](jyx2/Assets/Scripts/LuaCore/Jyx2LuaToCsBridge.cs)

## 7. 推荐的上手练习

建议按风险从低到高完成，不要同时改 UI、Lua、配置和场景。

### 练习 1：追踪主菜单按钮

打开 [GameMainMenu.cs](jyx2/Assets/Scripts/UI/GameMainMenu.cs)，在 `OnNewGameClicked()` 中临时增加：

```csharp
Debug.Log("Click New Game");
```

运行 `0_Init`，选择 `SAMPLE`，点击“新游戏”，在 Console 中确认日志。

再依次追踪：

```text
OnNewGameClicked()
  -> OnNewGame()
  -> OnCreateBtnClicked()
  -> OnCreateRoleYesClick()
  -> LevelLoader.LoadGameMap(...)
```

这条链路会接触到按钮事件、角色创建、读取起始地图和场景加载。

### 练习 2：完善空名字校验

当前 `SetPlayerName()` 遇到空名字会直接返回，但 `OnCreateBtnClicked()` 仍会继续隐藏输入面板并显示角色属性面板。

因此只在 `SetPlayerName()` 中增加提示并不完整。更合理的练习目标是：

1. 让设置姓名的方法返回成功或失败。
2. 空名字时显示 `StoryEngine.DisplayPopInfo("请输入名字")`。
3. 校验失败时，让 `OnCreateBtnClicked()` 立即返回，不继续切换 UI。
4. 分别验证空字符串、只有空格和正常名字。

这个练习很接近前端表单校验，但会帮助你理解 C# 返回值和 Unity UI 状态切换。

### 练习 3：修改一条 MOD Lua 文案

在 `Assets/Mods/SAMPLE/Lua/` 找一个较短脚本，修改一条 `Talk(...)` 或 `ShowToast(...)` 文案。

重新触发对应事件，观察：

- 新文案是否出现。
- Console 是否有 Lua 语法错误。
- 当前运行的 MOD 是否确实是 `SAMPLE`。

### 练习 4：修改一个 Excel 配置

在 `Assets/Mods/SAMPLE/Configs/` 中修改一个低风险字段，例如物品名称：

1. 保存 Excel。
2. 选中 `SAMPLE/ModSetting.asset`。
3. 点击“生成配置表”。
4. 重新进入游戏验证。
5. 提交时只关注 Excel 源文件，不提交被忽略的 `Configs/Lua/`。

## 8. 常见问题排查

### Unity Hub 识别不到项目

判断依据：选中的目录下没有直接看到 `Assets/`、`Packages/`、`ProjectSettings/`。

处理：重新打开 `<仓库根目录>/jyx2`。

### 打开后大量 Package 或序列化错误

优先检查：

1. Unity 是否为 `2022.3.62f3c1`。
2. Package Manager 是否能访问 Gitee Git 依赖。
3. 是否在旧 Unity 版本中保存过场景或 Prefab。
4. Console 第一条 Error 是什么，而不是只看最后一条连锁报错。

### MOD 选择界面提示没有生成 XLua 代码

执行：

```text
XLua > Generate Code
```

等待 Unity 再次编译，并重新运行。

### MOD 列表为空或当前 MOD 异常

优先检查：

1. `Assets/Mods/*/ModSetting.asset` 是否能被 Unity 正常导入。
2. Console 是否有 `GameModEditorLoader` 或资源加载异常。
3. 是否仍在资源导入或 C# 编译。
4. 执行 Unity 菜单 `Game Tools > 修复启动MOD` 后重新进入。

“修复启动MOD”只会清理记录当前 MOD 的偏好项，不等于修复损坏资源。

### 启动时报“没有找到配置表”

处理顺序：

1. 确认当前 MOD 的 `Configs/` 中有有效 `.xlsx`。
2. 选中 `ModSetting.asset`，点击“生成配置表”。
3. 查看 Console 中 `xlsx to lua` 之后的第一条错误。
4. 确认 `ModRootDir` 指向正确的 MOD 根目录。

### UI 点击没有反应

检查：

1. Prefab 节点和组件。
2. `*_UIData.cs` 的查找路径。
3. `OnCreate()` 和 `RegisterEvent()`。
4. `BindListener(...)`。
5. Canvas 层级、Raycast Target、CanvasGroup 和遮挡。
6. 点击 handler 中增加的 `Debug.Log` 是否出现。

### Lua 文件没有执行

检查：

1. 当前 MOD 是否正确。
2. 事件配置是否指向目标 Lua。
3. 文件名是否符合 `ModSetting.asset` 的 `LuaFilePatten`。
4. 文件是否位于当前 MOD 的 `Lua/` 目录。
5. Console 是否有 Lua 语法或桥接错误。
6. 修改的是 MOD 事件脚本，还是已被 `require` 缓存的框架模块。

## 9. 调试与 Git 习惯

### C# 调试

- 日常定位先用 `Debug.Log(...)` 和 Unity Console。
- 需要查看变量和调用栈时，从 Rider、Visual Studio 或 VS Code 附加到 Unity Editor 进程。
- 修改 C# 后先等 Unity 编译完成，再继续点击游戏。
- 一条编译错误可能造成大量类型缺失报错，优先修复 Console 中最早出现的错误。

### Lua 调试

- 用 `print(...)` 确认脚本和分支是否执行。
- 用 `ShowToast(...)` 验证玩家可见结果。
- 每次先做一个小改动，方便判断是脚本没有执行还是逻辑错误。

### 提交前检查

执行 `git status`，重点区分：

- 你主动修改的 C#、Lua、Excel、Prefab、Scene。
- Unity 自动生成或更新的 `.meta`。
- Editor 布局、缓存、材质升级等无关变化。

不要直接删除看不懂的 `.meta`，也不要把 `Library/`、`Temp/`、`Logs/`、XLua Gen 或 `Configs/Lua/` 当成业务源码提交。

## 10. 建议阅读顺序

### 第一轮：只看完整链路

1. [InitScene.cs](jyx2/Assets/Scripts/GameCore/InitScene.cs)
2. [GameStart.cs](jyx2/Assets/Scripts/GameCore/GameStart.cs)
3. [ModPanelNew.cs](jyx2/Assets/Scripts/MOD/ModV2/UI/ModPanelNew.cs)
4. [BootMainMenu.cs](jyx2/Assets/Scripts/GameCore/BootMainMenu.cs)
5. [Jyx2_UIManager.cs](jyx2/Assets/Scripts/UI/Framework/Jyx2_UIManager.cs)
6. [RuntimeEnvSetup.cs](jyx2/Assets/Scripts/GameCore/RuntimeEnvSetup.cs)
7. [Jyx2ResourceHelper.cs](jyx2/Assets/Scripts/ResourceManagement/Jyx2ResourceHelper.cs)
8. [GameMainMenu.cs](jyx2/Assets/Scripts/UI/GameMainMenu.cs)
9. [main.lua](jyx2/Assets/BuildSource/Lua/main.lua)
10. [SAMPLE 配置说明](jyx2/Assets/Mods/SAMPLE/Configs/README.md)

第一轮只回答“谁调用谁、数据从哪里来”，不要展开每个类的所有细节。

### 第二轮：按任务选择方向

| 目标 | 推荐入口 |
| --- | --- |
| 修改 UI | `Jyx2_UIManager`、`Jyx2_UIBase`、具体 Panel、对应 Prefab 和 UIData |
| 修改剧情 | `main.lua`、`LuaExecutor`、`Jyx2LuaBridge`、MOD Lua |
| 修改数值 | 配置 README、Excel、`ExcelToLua`、`LuaToCsBridge` |
| 修改战斗 | `Assets/Scripts/GameBattle/`、`Assets/LuaScripts/Jyx2Battle/` |
| 制作 MOD | `MODRootConfig`、`ModEditorWindow`、`Assets/Mods/SAMPLE/` |
| 排查资源 | `RuntimeEnvSetup`、`ResLoader`、`Jyx2ResourceHelper` |

## 11. 第一周完成标准

完成下面清单，就不必再把自己当成“完全没上手”：

- [ ] 能用正确 Unity 版本打开 `jyx2/`。
- [ ] 能生成 XLua 代码并识别 Console 中的编译错误。
- [ ] 能从 `0_Init` 跑到 `SAMPLE` 主菜单。
- [ ] 能说明四个启动场景分别负责什么。
- [ ] 能从 Prefab 找到 UIData，再找到按钮 handler。
- [ ] 能通过日志追踪一次“新游戏”点击。
- [ ] 能修改并重新触发一条 MOD Lua。
- [ ] 能修改 Excel 并生成 Lua 配置。
- [ ] 知道哪些目录是生成物，不应该直接编辑或提交。
- [ ] 提交前会检查 `.meta` 和 Unity 自动产生的无关改动。

最短学习路径仍然是：

```text
先跑通
  -> 追一个按钮
  -> 改一条 Lua
  -> 改一个 Excel
  -> 再按实际需求深入系统
```
