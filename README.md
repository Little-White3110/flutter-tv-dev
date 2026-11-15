# flutter-tv-dev

> 一个 **Agent Skill**（不是 Flutter 库，也不是应用模板）。用于让编码 Agent 在开发或审查 Flutter Android TV / Google TV / Fire TV 应用时，按统一的 TV 规范产出代码与方案。

## 这个 Skill 解决什么

Flutter 官方并未提供 TV 支持。一个 Flutter 应用要成为「TV 应用」，依赖三件事同时成立：

| 要素 | 作用 | 缺失时的表现 |
|---|---|---|
| TV Manifest | 让 TV Launcher 识别应用（`LEANBACK_LAUNCHER`、`android.software.leanback`、`touchscreen required=false`、320x180 banner） | 安装后 TV 桌面看不到图标；Play 提示设备不兼容 |
| D-pad 焦点模型 | 无触控，焦点即「光标」（`dpad` 的区域记忆 + beam 遍历） | 焦点跳栏、跳行、离开后回到第 0 项、弹窗关闭后无处可去 |
| 10-foot 布局 | 三米外阅读（1920x1080 画布、约 5% overscan 边距、正文 >= 18sp、焦点目标 >= 48dp） | 文字不可读、焦点特效被裁切、边缘内容被 overscan 吞掉 |

任一环节缺失，症状都像「Flutter 的 bug」，实则是 TV 规范问题。

## 触发场景

- 目标平台为 Android TV / TV box / Google TV / Fire TV
- D-pad / 遥控器导航、焦点遍历、焦点记忆
- TV overscan 安全区、横屏布局、10-foot 可读性
- TV AVD / 真机 ADB 调试、遥控器按键模拟
- Google Play TV 上架清单与 banner / 截图要求

## 目录结构

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 入口：加载条件（frontmatter）+ 7 节主规范 + 发布前验证清单 |
| `references/dpad-focus.md` | `fluttercandies/dpad` v3 完整 API：`Dpad.wrap`、`DpadFocusable`、`DpadRegion`、enter / edge 行为、2.x -> 3.0 迁移 |
| `references/tv-build-debug-release.md` | Manifest / Gradle、TV AVD、真机 ADB、遥控器 keyevent、媒体播放注意事项、Play TV 上架清单 |

加载规则：先读 `SKILL.md`；写焦点 / 导航代码时读 `dpad-focus.md`；处理 Manifest、AVD、ADB、包名入口或发布时读 `tv-build-debug-release.md`。按需加载对应 reference，避免一次性把两份都塞进上下文。

## 安装与使用

本仓库为私有仓库，clone 后放到 Agent 的 skills 目录即可：

```bash
# 用户级（跨项目可用）
git clone https://github.com/Little-White3110/flutter-tv-dev.git \
  ~/.workbuddy/skills/flutter-tv-dev

# 或项目级（团队共享）
git clone https://github.com/Little-White3110/flutter-tv-dev.git \
  <project>/.workbuddy/skills/flutter-tv-dev
```

Agent 通过 Skill 工具按 `name: flutter-tv-dev` 加载，`SKILL.md` 的 `description` 决定何时被触发。本仓库不含可执行代码，不发布到 pub.dev。

## 版本与依赖基线

| 项 | 值 |
|---|---|
| Skill 版本 | 1.0.0 |
| Flutter | >= 3.24 |
| dpad | ^3.0.0（fluttercandies，MIT） |
| 目标平台 | Android TV / Google TV / Fire TV；Desktop / Web 可用于焦点调试内循环 |

## 关键约束速览

1. 每屏仅一个 `autofocus: true`，落在主操作（Play / 首张卡片）上。
2. 每个视觉分区一个 `DpadRegion`；分区切换器 / Tab 内必须带 `memoryKey`。
3. 侧边栏用 `stop`，轮播用 `wrap`；padding 放在滚动视图内部，避免焦点特效被裁切。
4. 动作绑在 `DpadFocusable.onSelect`，不要把可聚焦的子按钮再包一层（`excludeChildFocus` 默认 true）。
5. `TextField` 保持裸用：方向键先移动光标，到边界再导航离开。
6. 手机与 TV 用 flavors + 双入口（`lib/main.dart` / `lib/main_tv.dart`）拆分，不要给同一个 Activity 同时声明 `LAUNCHER` 与 `LEANBACK_LAUNCHER`。
7. 开发期开启 `debugOverlay: true`；发布前在真机或 TV AVD 上跑完 `SKILL.md` 第 7 节验证清单。

## 许可与资料来源

`SKILL.md` frontmatter 标注 `license: MIT`；本仓库当前为**私有**仓库，访问范围以 GitHub 权限为准。内容整理自：

- Android TV 开发者文档（developer.android.com/training/tv）
- fluttercandies/dpad v3 README（MIT）
- Google Play TV 应用质量指南
