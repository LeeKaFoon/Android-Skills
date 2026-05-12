---
name: classic-system-bars-android-view
description: >-
  Keeps traditional Android system bar behavior in View/XML projects targeting
  SDK 35: status and navigation bars remain independent opaque areas, while app
  content stays laid out below/above them instead of drawing fullscreen behind
  system bars. Use for traditional View, XML, Activity, AppCompatActivity, and
  WindowInsets work when the user wants non-edge-to-edge classic system bars.
disable-model-invocation: true
---

# 传统 View 项目保留传统系统栏模式

用户目标（原文）：传统view实现的安卓项目，targetSdk=35，将项目实现为保留传统模式的效果，传统模式：系统栏占据独立的不透明区域，应用内容默认被“挤”在系统栏下方（非全屏区域）

## 适用范围

- Android UI 以传统 **View / XML / `setContentView`** 为主，非 Jetpack Compose。
- 项目 `targetSdkVersion` 为 **35**，但产品目标不是沉浸式或 edge-to-edge，而是保留旧式视觉：状态栏、导航栏为独立不透明区域，应用内容默认不绘制到系统栏后方。
- 用户明确说“传统模式”“非全屏区域”“内容被系统栏挤下去/挤上去”“不要 edge-to-edge”时使用。

## 核心原则

1. **不要依赖系统自动恢复传统边界**：`targetSdkVersion 35` 下，`decorFitsSystemWindows(true)` 可能不足以阻止内容进入系统栏区域。
2. **用 insets 主动模拟传统占位**：允许窗口进入 edge-to-edge 布局，再对内容根应用 `systemBars + displayCutout + IME` padding，让应用内容保持在非全屏区域。
3. **系统栏应不透明**：显式设置 `statusBarColor` 和 `navigationBarColor`，避免透明栏导致内容看起来延伸到系统栏下方。
4. **内容不应重复加系统栏 padding**：根内容已经统一处理 `systemBars` 后，子 View 不要再次叠加同一组系统栏 padding。

## 实施步骤

### Step 1：盘点入口

1. 查找所有 `Activity`、全屏 `Dialog`、`DialogFragment`、悬浮窗或自定义 `Window`。
2. 检查是否有以下代码或等价逻辑：
   - `enableEdgeToEdge()`
   - `SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN`
   - `SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION`
   - `SYSTEM_UI_FLAG_LAYOUT_STABLE` 与 fullscreen/hide-navigation 组合
3. 若存在 `setDecorFitsSystemWindows(false)`，必须确认同一流程已经给内容根安装 `systemBars` padding；否则会变成真正的内容穿透系统栏。
4. 若目标是传统模式，移除或改写 fullscreen 配置。

### Step 2：恢复传统窗口边界

在每个需要传统模式的 `Activity` 中，建议在 `setContentView()` 之后配置窗口和内容根 insets：

```java
Window window = getWindow();
window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    window.setDecorFitsSystemWindows(false);
}

window.setStatusBarColor(statusBarColor);
window.setNavigationBarColor(navigationBarColor);
```

如果项目已接入 AndroidX Core，可用：

```java
WindowCompat.setDecorFitsSystemWindows(window, false);
```

然后在 `android.R.id.content` 或实际根布局上安装 `OnApplyWindowInsetsListener`，把系统栏高度转成 padding，模拟“内容被系统栏挤在内侧”的传统效果。

### Step 3：系统栏图标可读性

- 浅色状态栏背景：设置深色状态栏图标。
- 浅色导航栏背景：API 26+ 设置深色导航栏图标。
- 深色背景：不要设置 light flags，保持浅色系统图标。

传统 View 可使用：

```java
int flags = 0;
if (!isNightMode && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    flags |= View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR;
}
if (!isNightMode && Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    flags |= View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR;
}
window.getDecorView().setSystemUiVisibility(flags);
```

API 30+ 也可使用 `WindowInsetsController.APPEARANCE_LIGHT_STATUS_BARS` 与 `APPEARANCE_LIGHT_NAVIGATION_BARS`。

### Step 4：处理 targetSdk 35 的默认 edge-to-edge 影响

在 `targetSdkVersion 35` 下，如果系统或库默认让内容进入 edge-to-edge：

1. 不要只依赖 `setDecorFitsSystemWindows(true)`；在 Android 15/target 35 上应使用根内容 insets padding 主动占位。
2. 确认状态栏、导航栏不是透明色。
3. 确认根布局没有和工具类重复处理同一组系统栏 padding。
4. 若某些设备仍出现内容进入系统栏，检查 insets listener 是否安装在真实内容根上、是否在 `setContentView()` 之后调用、是否调用了 `requestApplyInsets()`。

## IME / 软键盘

- 有输入框的页面优先使用 `android:windowSoftInputMode="adjustResize"`，让内容区域随键盘调整。
- 不要同时使用“窗口 resize”与根布局手动 IME padding 造成双重底部留白。
- 如果历史页面使用 `adjustNothing`，只有在产品明确需要键盘覆盖内容时保留；否则改为 `adjustResize` 并验证输入框可见。

## 列表和底部固定按钮

- 传统模式下，根内容已经用 `systemBars` padding 后，`RecyclerView`、底部按钮栏默认应位于导航栏上方，不需要在子 View 再额外加同一份 bottom padding。
- 如果页面本身有底部工具栏或按钮栏，只处理应用内部间距，不要把系统导航栏高度再次加进去。
- 若发现底部内容被导航栏遮挡，先检查是否漏装根内容 insets padding、是否在 `setContentView()` 之后调用、是否被子 View 覆盖。

## 不应采用的 edge-to-edge 代码

传统模式下避免这些做法：

```java
window.setDecorFitsSystemWindows(false);
decorView.setSystemUiVisibility(
    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
        | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
window.setStatusBarColor(Color.TRANSPARENT);
window.setNavigationBarColor(Color.TRANSPARENT);
```

这些代码如果没有配套根内容 `systemBars` padding，会让内容绘制到系统栏后方，与“系统栏占据独立不透明区域”的目标相反。允许使用 `setDecorFitsSystemWindows(false)`，但必须同时安装内容根 insets padding 来模拟传统占位。

## 推荐封装

如果多个 `Activity` 都需要传统模式，创建类似 `SystemBarUtils.configureClassicSystemBars(Activity)` 的公共方法：

1. 设置 `decorFitsSystemWindows(false)`。
2. 在 `setContentView()` 之后给内容根安装 `systemBars + displayCutout + IME` padding。
3. 设置不透明状态栏、导航栏颜色。
4. 根据日夜间设置系统栏图标深浅。

## 与 edge-to-edge View 技能的区别

| 目标 | 使用技能 |
|------|----------|
| 内容绘制到系统栏后方，再用 Insets 避让关键控件 | `edge-to-edge-android-view` |
| 保留旧式窗口区域，系统栏独立不透明，内容不进入系统栏 | `classic-system-bars-android-view` |

## Checklist

- [ ] `targetSdkVersion` 是否为 35？
- [ ] 是否移除了 `enableEdgeToEdge()` 或 `decorFitsSystemWindows(false)`？
- [ ] API 30+ / target 35 是否通过根内容 insets padding 模拟传统占位？
- [ ] 状态栏和导航栏是否是不透明颜色？
- [ ] 浅色/深色主题下系统栏图标是否可读？
- [ ] 根布局和子 View 是否没有重复叠加 `systemBars` padding？
- [ ] 含输入框页面的 `windowSoftInputMode` 是否符合预期？
- [ ] 在 API 30+ 和 API 35 设备上是否验证内容没有被状态栏/导航栏遮挡？
