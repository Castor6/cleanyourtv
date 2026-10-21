# 小米 L43M7-EA：HDMI 开机自动选源探索记录

记录日期：2026-10-21（Asia/Shanghai）。本记录只适用于下面列出的实测固件，不能直接套用于其他小米电视。

**当前状态：按用户要求暂停探索；本次 HDMI 实测改动已于 2026-10-21 23:11:48（Asia/Shanghai）回滚并读回核验。自动选源目标未完成。**

## 设备与最终需求

| 项目 | 实测信息 |
| --- | --- |
| 零售型号 | 小米 L43M7-EA |
| 系统型号 / 产品 | `MiTV-ANTL0` / `starwars` |
| 系统 | Android 5.1.1，API 22，固件 2225 |
| 原厂主页 | `com.mitv.tvhome/.MainActivityUserMode` |
| HDMI 播放器 | `com.xiaomi.mitv.tvplayer/.ExternalSourceActivity` |
| 接线用途 | HDMI1：机顶盒；HDMI2：Nintendo Switch |

最终需求是在 HDMI1、HDMI2 **只有一个连接**时，电视开机或接入设备后直接显示该输入，不再选择信号源并确认。两口同时连接不在本次需求范围；较早提出的 HDMI2 优先规则已取消。用户后来强调完整关机再开机是主要场景。

用户平时会断开不使用的设备连接，不能把“未使用”一律解释为待机。返回主页后应能正常使用主页，不能持续被自动切回 HDMI。

## 证据保存与隐私边界

本次基线、日志、固件分析、临时应用源码及 APK 位于本机 [`backup/xiaomi-hdmi-autostart-20261021-212024/`](../../backup/xiaomi-hdmi-autostart-20261021-212024/)。`backup/` 被 Git 忽略，下面的证据链接只在保留这些文件的本地工作区可用，不会随本文提交。

原始日志可能包含设备及账户标识，不应整包提交或公开。本文不记录电视地址、Wi-Fi、MAC、ADB 密钥或账户数据。

## 实际改动与回滚基线

三项设置均位于 `Settings.System`。`null` 表示本次备份时键不存在；回滚应删除该键，不能用写入 `0` 代替。

| 设置 | 实测前 | 探索中的变化 | 请求回滚前 | 回滚后读回 |
| --- | --- | --- | --- | --- |
| `boot_auto_play` | `null` | 原厂开机播放测试时设为 `1`；辅助程序接管方案改为 `0` | `0` | `null`（已删除） |
| `dk_desktop_pip` | `2`（HDMI1） | 曾试 `9`（最近输入），随后恢复 `2` | `2` | `2` |
| `tvapp_one_touch_play` | `null` | 热插直接播放测试设为 `1`；无信号对照时改为 `0` | `0` | `null`（已删除） |

安装过诊断应用 `org.cleanyourtv.hdmiprobe`，已在诊断结束后卸载。随后安装辅助程序 `org.cleanyourtv.hdmiauto`；请求回滚前仍安装，但本次重启后处于 `stopped=true` 且没有运行服务。它只有开机广播权限，没有声明 HOME、无障碍、联网或修改系统设置权限。

没有更换默认主页、停用原厂软件、修改固件白名单，也没有写入后来发现的 `tv_start_up_*` 设置。关于自定义桌面、指定应用入口的后续讨论只进行了本地只读分析，没有连接电视或部署新方案。此前另一次开机广告配置不属于本次 HDMI 回滚范围。

依据：[原始设置与变更记录](../../backup/xiaomi-hdmi-autostart-20261021-212024/change.json)、[验证状态](../../backup/xiaomi-hdmi-autostart-20261021-212024/verification-status.json)、[回滚程序](../../backup/xiaomi-hdmi-autostart-20261021-212024/restore-hdmi-settings.py)。

回滚已完成，核验结果如下：

- 辅助应用卸载返回 `Success`；两个实验应用的 `pm path` 均无输出，辅助服务列表为空。
- 上表三项设置与实测前完全一致；两个 `tv_start_up_*` 键仍为 `null`。
- 本次临时 logcat 进程已停止，电视上的 `codex-hdmi-*` 临时文件已清理。本地源码、APK 和证据备份保留。
- 核验时前台仍为原厂 HDMI2 播放器；回滚没有重启电视。核验后已关闭电脑上的持久 ADB 连接。
- 用户手动开启的 ADB 调试状态未由回滚更改；此前独立配置的开机广告设置也未更改。

执行与读回见 [回滚核验原始输出](../../backup/xiaomi-hdmi-autostart-20261021-212024/rollback-verification.txt) 和 [回滚结果记录](../../backup/xiaomi-hdmi-autostart-20261021-212024/rollback-result.json)。本文不是可直接照搬的安装说明；后续恢复研究前必须重新读取设备状态。

## 已验证的基础能力

普通应用 UID 可调用厂商 `mitv.tv.TvContext` / `SourceManager.isSourceConnected(int)`。本机厂商源编号为 HDMI1=`23`、HDMI2=`24`。

仅连接 Switch 时，诊断应用读到 `HDMI1=false`、`HDMI2=true`，与实际接线一致。普通应用也能显式启动原厂播放器，并传入整数 extra `input=24`；没有遇到播放入口权限拒绝。

辅助程序手动检测时连续两次读到唯一 HDMI2，发出一次播放请求后停止该轮选择；原厂播放器进入前台。该测试证明读取和启动能力，不等于证明冷启动、热插及画面长期稳定均成功。

相关证据：[探测及手动检测状态](../../backup/xiaomi-hdmi-autostart-20261021-212024/verification-status.json)、[辅助程序事件](../../backup/xiaomi-hdmi-autostart-20261021-212024/helper-events-before-restart.jsonl)、[源码](../../backup/xiaomi-hdmi-autostart-20261021-212024/android-hdmi-auto/)。

## 实测结果与失败定位

### 原厂热插与无信号退出

开启 `tvapp_one_touch_play=1` 后，原厂能在热插时收到 source24 并启动播放器。早期用户回复“直接显示了 Switch”，只能支持当时自动进入的观察，不能作为持续稳定播放的验收结果。

后来用户明确复现：Switch 掌机可进入游戏，放回底座后绿灯常亮，但电视一直搜索信号，随后退回首页。实时记录确认：

| 时间 | 日志 |
| --- | --- |
| 22:44:07.603 | `onSourceSignalStable 24, false` |
| 22:44:07.613 | `startSearchSignal` |
| 22:44:32.104 | `am_finish_activity ... app-request` |

原厂播放器代码的对应流程是：搜索约 15 秒仍无视频 → 显示“没有检测到视频信号” → 确认按钮约 9 秒倒计时后自动确认 → `finish()`。日志间隔 24.491 秒与之吻合，解释了这次自动退首页。

这轮辅助程序没有启动，原厂热插请求来自系统 UID；不能归因于辅助程序反复抢屏。识别连接、CEC 显示 NintendoSwitch、底座亮灯，也不等于视频信号已稳定。

关闭 `tvapp_one_touch_play`、保持辅助程序停止，再于 22:46:50 单次手动打开 HDMI2 后，22:46:52.332 底层记录稳定 60 Hz，22:46:53.492 播放器报告 `true`，用户确认看到游戏。之后超过 10 分钟仍保持该播放器前台，没有同类退出记录。

**尚未隔离自动播放失败的具体根因。** 上述对照同时改变了设置、启动时机及 Intent 内容；不能认定仅关闭某一开关就永久修复，也不能认定线材或底座损坏。某些失败有重复 START / Session 重建，但另一次单 START 也失败，重复启动不是所有失败的必要条件。

证据：[实时失败记录](../../backup/xiaomi-hdmi-autostart-20261021-212024/live-hdmi-observe-failure.txt)、[手动成功对照](../../backup/xiaomi-hdmi-autostart-20261021-212024/manual-play-success-log.txt)、[最终抓取](../../backup/xiaomi-hdmi-autostart-20261021-212024/final-hdmi-capture.txt)。

### 辅助程序冷启动失败

完整重启后，应用已授予 `RECEIVE_BOOT_COMPLETED` 且以前启动过，但包状态为 `stopped=true / notLaunched=false`，服务为空，私有事件没有本次 BOOT 记录。冷启动自动进入测试失败，不能继续把它描述为单纯“尚未验收”。

本机 framework 中找到直接对应路径：`Settings.readLPw()` → `handleMiPackageRestrictionsLPr(userId)` → 对非系统、非更新系统且未获 `isBootAutoStartAllowed()` 许可的包设置 stopped。该路径不要求应用先启动，也不一定留下 ActivityManager force-stop 日志。

有效许可使用 `boot_autostart` 白名单 bit1，来源为受保护的 `/data/system/mitv_security_center.xml`。shell 读取被拒；没有找到已验证的普通用户逐包配置入口。未更改全局控制开关、名单或系统组件。

另一个旧 Provider 的 `AutoStartAppController` API 写 `/user_app_control_list`，权限为 `signature|system`，输出 `app_control.xml`。它与本机 framework 使用的安全配置来源不同，不能直接当成本机有效开机授权方案。

证据：[重启后包状态](../../backup/xiaomi-hdmi-autostart-20261021-212024/post-reboot-package-broadcasts.txt)、[重启后事件](../../backup/xiaomi-hdmi-autostart-20261021-212024/post-reboot-helper-events.jsonl)、[固件分析](../../backup/xiaomi-hdmi-autostart-20261021-212024/native-startup-entry-analysis.md)。

### 息屏与 ADB 调试

用户使用长按电源菜单“息屏”后再唤醒，仍停在主页。该次 Android 电源状态保持 Awake，辅助程序没收到 SCREEN_OFF/ON；当前实现不支持这种仅关闭显示的路径，不能等同于普通待机测试通过。

ADB 曾被用户观察到从“允许”变回“拒绝”。后续“允许 → 退出设置 → 约 30 秒 → 再进入”的受控操作没有复现，连接在当次会话保持稳定。固件存在开机关闭 ADB 的路径，但不能据此解释用户先前没有重启时的变化。不能把原因归为用户没确认弹窗。

## 已分析但未完成实测的路线

### 原厂最近输入模式 9

设备配置的 `support_inputsource_latest_insert=NO` 使设置界面不提供该选项；写入 `dk_desktop_pip=9` 时界面显示 TV，随后已恢复 `2`。运行代码仍明确识别 `9`，所以界面隐藏本身不能证明它不可运行。

`getLastSource()` 只检查当前存在外部连接，再返回记忆的 `LastCurrentSource`（默认23）；不核实记忆端口仍连接，也不直接选唯一连接口。连接回调会更新记忆，但没有证据保证冷启动时回调早于一次性的开机选源。该组合没有完成冷启动验证，更不能作为已交付替代方案。

若未来验证，必须包含“记忆HDMI1而开机仅接HDMI2”及反向情况；只预先选好同一端口再重启，最多证明记忆播放。

### Android 5.1 自定义桌面

Android 5.1 可开发 HOME 应用；已有本机 API22 辅助应用证明基础编译、安装及厂商 API 可用。仓库现成 launcher 的最低 API 为26，不能直接安装，也不能仅降低 minSdk 就宣称完成兼容。

本机 DEX 定义 `isThirdpartyHomeAllowed`，但扫描未找到调用，也未发现另一台 Android9 小米报告中的同类 HOME 过滤。没有在本机切换默认 HOME；既不能确认被禁止，也不能保证可用。开机 stopped 规则不按默认 HOME 身份豁免，而显式 Activity 启动有清除 stopped 的路径，最终仍需实测。

### 原厂启动指定应用入口

本机 `ro.platform.has.tvuimode=true`。`ActivityManagerService.startHomeActivityLocked()` 在普通 HOME 前调用 `startTvActivityLocked()`；后者读取 System 的 `tv_start_up_enter_app`、`tv_start_up_app_name`，满足条件时显式启动指定组件。

两个设置键在原始快照中未出现；本次仅离线分析，**从未写入或测试**。该入口可能让最小辅助 Activity 获得启动机会，不必依赖 BOOT 接收器，也未必需要完整桌面。

它没有仅开机执行一次的保护，HOME 恢复可能重触发。配套 `com.droidlogic.tv.action.start.wallpaper` 接收器会创建 WallpaperManagerService 并启动 SystemUIService，不能当作无副作用通知反复发送。需要先验证系统 UI 初始化、HOME 重入和返回原主页是否循环。

完整定位见 [原厂入口分析](../../backup/xiaomi-hdmi-autostart-20261021-212024/native-startup-entry-analysis.md)。这只是一条值得验证的路线，不是已找到的修复。

## 暂停后的恢复研究顺序

当前不继续安装、改设置或要求用户重复开关机。只有用户明确恢复这项工作后，再考虑以下验证：

1. 读取最新设备状态并重新备份，不能沿用本次会话连接或假设设置未变。
2. 先验证最小应用的开机入口，只显示状态，不触碰 HDMI；检查冷启动、HOME 返回、系统 UI 及回滚。
3. 入口成立后再测试唯一 HDMI 检测和单次播放。分别记录连接状态、视频稳定状态与前台 Activity，避免把“检测到插入”当作“已有画面”。
4. 对自动与手动路径做尽可能单变量的时序对照；延迟启动是待验证假设，不是保证有效的修复。
5. 验收 HDMI1 单口、HDMI2 单口、无输入、开机前换口、热插、主动返回主页及真正待机；将息屏单列。双口同时在线仍不在范围内。

不能直接复用标准桌面安装脚本：它涉及默认 HOME 和原桌面状态，而当前需求与授权仅是在暂停前保留研究结论、撤销本次实测改动。
