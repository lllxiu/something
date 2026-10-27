# 换一次音频设备，CS2 就死给你看：一次从现象到根因的排查

>  Windows 11 笔记本，CS2 运行中切换音频输出设备后，退出时必卡死。

## 

CS2 的音频走 SDL3。系统级环境变量 `SDL_AUDIODRIVER` 被设成了 `directsound`，导致音频绕过 WASAPI 走遗留的 DirectSound 后端。切换默认输出设备时，DirectSound 设备对象需要销毁重建，在「保存配置 → 拆除设备」的中间态上与关闭流程互相等待，整个进程死锁。

把 `SDL_AUDIODRIVER` 改回 `wasapi` 即可。这一行设置会影响**所有基于 SDL2/SDL3 的程序**，不只是 CS2。

---

## 一、症状

事情开始得很简单：

1. CS2 正在运行
2. 用任务栏音量弹窗切换输出设备（音箱 ↔ USB 耳机 ↔ 蓝牙）
3. 切换瞬间游戏还能动，看不出异常
4. 点**退出**或者**返回主菜单**
5. 画面冻结，进程再无响应，只能强杀

Steam 侧留下的记录是 `exit code -805306369`（`0xCFFFFFFF`）——这是 Windows「进程已挂起、被强制结束」的标准退出码，不是崩溃。

---

## 二、环境

```
系统    Windows 11 家庭中文版 26H2 (10.0.26300.9711)
机器    ASUS Vivobook M6501RM
音频    Realtek ALC294 (驱动 6.0.9381.1 / 2022-07-12)
        + USB 耳机 
启动    完美世界竞技平台 → Steam → cs2.ex
```

---

## 三、一次误判：`snd_mixahead`

翻查 CS2 配置时发现一个可疑项：

```cfg
"snd_mixahead"    "0.001"
```

CS2 默认是 `0.025`（25 ms 混音提前量）。`0.001` 只有默认值的 **1/25**，音频线程几乎没有余量，任何一次设备初始化抖动都会变成 underrun。

缓冲太小 → 重建时立刻见底 → `Initialize` 卡住。

-  `cs2_machine_convars.vcfg` 全部改成 `0.025`
- 写进 `autoexec.cfg` 固化（CS2 不会自动执行 `autoexec.cfg`，还得加 `+exec autoexec` 启动参数）
- 顺带把启动项里的 `-high -threads 8` 也去掉

改完重启，**照样卡死**。

---

## 四、转折：抓到一个进程

靠猜是猜不出来的。得看进程内部在干什么。

写了个轮询脚本，每秒检查 `cs2.exe` 的 `Responding` 属性，一旦检测到未响应，立刻抓线程快照。

抓到的那一瞬间是这样的：

```
pid=13588  responding=False  title='Counter-Strike 2'
CPU=185.5s  WS=4,160MB  Threads=135

TID 35324  Wait  Unknown       CPU 50187ms   ΔCPU 0   ← 主线程
TID 26040  Wait  UserRequest   CPU  2750ms   ΔCPU 0
... 其余 132 个线程             ΔCPU 全部为 0
```

**135 个线程全部 `Wait`，1.5 秒采样窗口内 CPU 增量全部为零。**

这个特征非常有辨识度：

| 特征 | 含义 |
|---|---|
| 全线程 Wait + ΔCPU=0 | **死锁**（互相等一把永远不会释放的锁） |
| 单线程高 CPU + 其他 Wait | 死循环 / 忙等 |
| 有 TDR 事件（4101） | GPU 挂起 |
| 有 Application Error + dump | 崩溃 |

同时排除了 GPU 挂起（System 日志无 4101）和崩溃（无 `Application Error`、无 dump、`HangType=Unknown`）。

问题被锁死在一个词上：**死锁**。

---

## 五、打开黑盒：minidump 与栈回溯

线程表告诉我们都卡住了，不能告诉我们卡在哪。需要栈。

更新脚本，在检测到未响应的**第一秒**调用 `MiniDumpWriteDump` 落盘（`MiniDumpNormal | WithThreadInfo`，不到 1 MB），然后写了个几百行的 Python 解析器，直接从 dump 里读出线程列表、模块列表和栈内存，扫描栈上的返回地址并映射回模块。

结果一出来，答案几乎是自明的：

```
TID 8432   rip = ntdll!NtWaitForSingleObject
    KERNELBASE < AudioSes.dll < AudioSes.dll < AudioSes.dll < AudioSes.dll
              < dsound.dll < KERNELBASE < dsound.dll < dsound.dll < dsound.dll

TID 9644 / 31332 / 30976   (优先级 15 = DirectSound 混音线程)
    powrprof / dsound.dll ×十余层 ...

TID 29800
    ... dsound.dll ×12 < SDL3.dll ×4      ← SDL3 创建、走 dsound 的音频线程
```

以及模块表里同时出现的三个名字：

```
DSOUND.DLL
ResampleDmo.DLL     ← DirectSound 专用重采样器
msdmo.dll
```

**2026 年的游戏，为什么在跑 DirectSound？**v社也太懒了

CS2 的音频链路是 `SDL3.dll → MMDevAPI.dll`，也就是标准的 WASAPI。正常情况下 `dsound.dll` 根本不该被加载。

而 `AudioSes.dll` 是「DirectSound 架在 WASAPI 之上」的那层模拟。切换默认设备时，DSound 必须销毁并重建整个设备对象；这一层在设备切换 + 退出拆除同时发生时，极易互相等待。

到这里，问题从什么卡住了变成了谁让它走 DirectSound 的。

---

## 六、根因：一行系统环境变量

顺着「谁指定了音频后端」这条线，查环境变量：

```powershell
[Environment]::GetEnvironmentVariable('SDL_AUDIODRIVER','Machine')
# → directsound

[Environment]::GetEnvironmentVariable('SDL_AUDIODRIVER','User')
# → (空)
```

对应注册表：

```
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment
    SDL_AUDIODRIVER = directsound        ← 系统级，需要管理员权限才能写
```

SDL2 / SDL3 都会读这个变量来决定音频后端。被强制成 `directsound` 之后，CS2 就绕过了原生 WASAPI，改走遗留的 DirectSound 路径。

### 完整的死锁链条

```
运行中切换默认输出设备
 │
 ├─ Windows 广播 IMMNotificationClient::OnDefaultDeviceChanged
 │
 ├─ CS2 (SDL3 + DirectSound 后端) 需要重建 DSound 设备对象
 │     └─ DirectSoundCreate8 → dsound.dll → AudioSes.dll（模拟层）
 │
 ├─ 用户点「退出 / 返回主菜单」
 │     ├─ CS2 先写 boot.vcfg 与 cs2_machine_convars.vcfg（保存配置）
 │     └─ 再销毁混音器与音频设备
 │
 └─ 在「保存完成、设备尚未拆除」的中间态上，
    设备重建与销毁互相等待 → 全进程死锁
```

有一个细节佐证了这个时序：`boot.vcfg` 和 `cs2_machine_convars.vcfg` 的写入时间，与卡死时刻**精确到秒地重合**（15:49:32 / 15:59:23 / 16:08:0x）。进程正是死在「配置存完了、开始拆设备」这一步。

---

## 七、修复

### 修复

```powershell
# 管理员 PowerShell
[Environment]::SetEnvironmentVariable('SDL_AUDIODRIVER', $null, 'Machine')
```

不知道这个是什么来的变量，可能来自某个「音频优化 / FPS 优化」工具或者早年照抄的教程。

### 验证

```powershell
Get-Process cs2 | Select-Object -ExpandProperty Modules |
  Where-Object { $_.ModuleName -match 'dsound|ResampleDmo' } |
  Select-Object ModuleName
```

返回空，说明 CS2 已经走回 WASAPI，根因确认。

---

## 八、修复之后：那 1 秒卡顿涅

死锁消失后，切换设备时画面还是会顿约 1 秒。这是**另一个问题**，而且属于正常现象。

设备重建的完整成本：

```
切换默认设备
 ├─ CS2 (SDL3) 销毁旧流 → 枚举端点 → Activate(IAudioClient) → GetMixFormat
 │   → 协商共享模式格式 → Initialize → 切混音线程 → 重新灌 buffer
 ├─ audiodg 销毁旧端点音频引擎、新建新端点，并重载 APO（音频处理对象）
 └─ 其他客户端各自重建：Steam 覆盖层的 IAudioClient、ShadowPlay 的音频采集
```

全部在 CS2 的主线程/音频线程上**同步**完成，渲染线程被阻塞，于是掉帧。典型范围 0.2–1.5 秒，任何游戏在运行中切换默认输出设备都会这样，只是程度不同。

可以再压缩一点：

| 手段 | 效果 |
|---|---|
| `snd_autodetect_latency 0` | 省掉每次重建设备时的延迟探测 |
| 各设备统一 48 kHz / 24 bit | 省掉格式重协商 |
| 关闭「音频增强」 | audiodg 少加载一层 APO |
| 关闭 Steam 覆盖层 / ShadowPlay 采集 | 少几个客户端同时重建 |

---

## 附：自查清单

如果你也遇到「游戏切换音频设备后卡死/无声/卡顿」，按这个顺序查：

**1. 先看系统级 SDL 变量（30 秒）**

```powershell
[Environment]::GetEnvironmentVariable('SDL_AUDIODRIVER','Machine')
[Environment]::GetEnvironmentVariable('SDL_AUDIODRIVER','User')
```

有 `directsound` 就删掉或改成 `wasapi`，然后重启 Steam / 游戏。这条会影响所有 SDL2/SDL3 程序——包括不少独立游戏、模拟器、音游。

**2. 看音频端点是否在抖**

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Audio/Operational' -MaxEvents 200 |
  Where-Object { $_.Id -eq 65 }
```

大量 `NOTPRESENT` 说明驱动的插孔检测有问题，优先换驱动。

**3. 确认是死锁而不是崩溃**

```powershell
Get-Process <进程名> | Select-Object Responding, Threads
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1000,1001,1002}
```

有 `1000` 是崩溃，有 `1002` 是挂起。挂起再去看线程 CPU 增量是否全为零。

**4. 还是定不了位就抓 dump**

任务管理器 → 详细信息 → 右键进程 → 创建转储文件；或者用 `MiniDumpWriteDump`。没有符号也能靠「栈上的返回地址落在哪个 DLL」看出大致位置。

---

*本文记录的排查过程来自真实环境，命令与响应均为实机结果。*
