# EaringAid（聆声助听）

实验性 Android 实时助听器应用：**极低延迟音频链路 + AI 降噪 + 专业助听 DSP + 个性化验配**。

> ⚠️ **免责声明**：本项目为教学/实验用途，**不是医疗器械**，不能替代专业验配与诊断。
> 处方公式为简化近似实现，真实听力验配须由持证听力师完成。使用中如感不适请立即停止。

---

## 功能总览

| 模块 | 说明 |
|---|---|
| 低延迟引擎 | Oboe 全双工，48kHz/Float/双声道，Exclusive→Shared 三级降级 |
| AI 降噪 | DTLN 双模型（ONNX Runtime），16k 重采样 + 无锁环形缓冲 |
| 助听 DSP | 120Hz 高通 → LR4 分频(400/2500Hz) → 3 段 WDRC → 11 频点 EQ → 限幅(-4dBFS) |
| 个性化验配 | 听力图(11 频点×双耳) → 处方(NAL-NL2/DSL v5/半增益简化) → 曲线热注入 |
| 安全兜底 | 啸叫（声反馈）检测抑制、±1.0 硬钳位、PTA>55/70dB 分级提示 |
| 校准工具 | 1kHz 测试音 / 100Hz–8kHz 对数扫频 / ±12dB 左右平衡 |
| 输入路由 | 输入设备选择（USB 麦克风/声卡等）；插拔监听，拔出自动停播并回退内置麦 |

## 音频链路

```
麦克风 → [Oboe 输入流] → DTLN AI 降噪（可开关）
       → 120Hz 高通 → LR4@400Hz 分频 → LR4@2500Hz 分频
       → 3 段独立 WDRC（软拐点前馈压缩）
       → 段合成 → 11 频点 peaking EQ（125Hz–8kHz 补偿曲线）
       → 安全限幅（-4dBFS + ±1.0 钳位）
       → 啸叫抑制（FFT 谱分析 + 增益回退）
       → + 测试音（点频/扫频）
       → [Oboe 输出流] → 耳机
```

**实时约束**：音频回调内零分配、零加锁；所有参数经原子量 + 块边界 `try_lock` 生效；
全部切换走 10ms 交叉淡化（无爆音）。

## 技术栈与环境

- Kotlin 2.0.21 + C++17（NDK 29.0.14206865）+ CMake
- Gradle 8.11.1 / AGP 8.7.3 / compileSdk 35 / minSdk & targetSdk 34 / arm64-v8a
- Oboe 1.9.3、ONNX Runtime 1.17.3（头文件 + 预编译 `jniLibs` 内联，无 Prefab/AAR）
- 模型：`assets/dtln/model_1.onnx` / `model_2.onnx`

## 构建

```bash
./gradlew assembleDebug
```

- 需本机 `local.properties` 指向 SDK/CMake；
- 产物：`app/build/outputs/apk/debug/app-debug.apk`
- 说明：必须在常规文件系统（ext4）构建——部分环境（FUSE/sdcard）会破坏 Gradle 的 `.l2s` 临时文件机制。

## 使用流程

1. 安装 APK → 授予录音权限 → 启动引擎（戴好耳机）
2. 主界面：开关 AI 降噪 / 助听处理；「测试音」验证耳机与左右平衡
3. 「验配设置」：输入纯音听阈 → 选处方公式 → 预览曲线 → 保存并实时生效
4. 观察统计行：延迟/xruns/降噪/压缩/限幅/啸叫抑制实时数据
5. USB 输入：插入 USB 麦克风/声卡 → 开启「USB 音频输入」开关 →「启动助听引擎」
   （切换设备会停止播放、不自动恢复；USB 拔出后开关置灰但保留显示）

## 开发阶段历史

| 版本 | 阶段 | 内容 |
|---|---|---|
| v0.1.0 | 1 | 项目骨架、权限、构建链路 |
| v0.2.0 | 2 | Oboe 低延迟双工引擎 |
| v0.3.0 | 3 | DTLN AI 降噪（真机 -37dB 抑制验证） |
| v0.4.0 | 4 | 多频段 WDRC + 11 频点 EQ + 限幅 |
| v0.5.0 | 5 | 个性化验配（听力图 → 处方 → 热注入） |
| v0.6.0 | 6 | 啸叫抑制 + 测试音 + 左右平衡校准 |
| v0.7.0 | 7 | USB 音频输入支持（设备选择/插拔监听/拔出自动回退） |

## 目录结构

```
app/src/main/
├── kotlin/com/DearZhou/earingaid/
│   ├── MainActivity.kt / FittingActivity.kt
│   ├── audio/AudioEngine.kt          # JNI 声明（14 项）
│   └── fitting/Fitting.kt            # 听力图/处方/持久化
└── cpp/
    ├── native-lib.cpp                # JNI 入口
    ├── audio_engine.{h,cpp}          # 双工引擎 + 链路编排
    ├── denoise/                      # DTLN + 重采样 + 无锁管线
    └── dsp/
        ├── biquad.h                  # RBJ Cookbook 双精度滤波器
        ├── hearing_processor.{h,cpp} # WDRC + EQ + 限幅
        ├── howl_suppressor.{h,cpp}   # 啸叫检测抑制（FFT）
        └── test_tone.h               # 测试音发生器
```

## 离线验证基础设施

`tools/dev/` 内含真机离线验证程序（NDK 交叉编译后于设备上运行）：
- `hearing_test.cpp` — DSP 链 7 组测试（WDRC/EQ/限幅/注入）
- `howl_test.cpp` — 啸叫抑制 9 组测试（含防误报与 41 频点扫描）

---
*EaringAid · 实验性助听器研究项目*
