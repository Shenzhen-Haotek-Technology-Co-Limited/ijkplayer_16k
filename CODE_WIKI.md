# ijkplayer 项目Code Wiki文档

## 1. 项目概述

ijkplayer是Bilibili开源的基于FFmpeg的视频播放器项目，专为Android和iOS平台设计。该项目基于ffplay实现，提供轻量级、功能丰富的跨平台多媒体播放能力。项目经过深度定制，支持16K/4K page size、64位/32位ABI构建，并优化了与NDK r22的兼容性。

### 1.1 项目特点

- **跨平台支持**：Android（API 9+）和iOS（7.0+）
- **多ABI架构**：arm64-v8a（16K生产）、x86_64（16K）、armeabi-v7a（4K兼容）
- **硬件加速**：支持MediaCodec（H.264/HEVC/VP8/VP9等）硬件解码
- **多种输出后端**：NativeWindow、OpenGL ES 2.0、AudioTrack、OpenSL ES
- **轻量级配置**：提供module-lite、module-lite-hevc、module-default等多种配置

### 1.2 技术栈

| 层级 | 技术/框架 |
|------|----------|
| 核心解码 | FFmpeg 4.3 (ijk-n4.3-20260301-007) |
| 硬件解码 | Android MediaCodec / iOS VideoToolbox |
| 视频渲染 | OpenGL ES 2.0 / NativeWindow |
| 音频输出 | AudioTrack / OpenSL ES (Android) |
| 构建系统 | Gradle + NDK-Build |
| 加密支持 | OpenSSL 1.1.1w |

---

## 2. 项目整体架构

### 2.1 目录结构

```
ijkplayer/
├── android/                      # Android平台相关代码
│   ├── ijkplayer/               # 各ABI的播放器模块
│   │   ├── ijkplayer-arm64/     # arm64-v8a架构
│   │   ├── ijkplayer-armv7a/    # armeabi-v7a架构
│   │   ├── ijkplayer-x86/       # x86架构
│   │   ├── ijkplayer-x86_64/    # x86_64架构
│   │   ├── ijkplayer-java/      # Java层播放器封装
│   │   ├── ijkplayer-example/   # 示例应用
│   │   └── ijkplayer-exo/       # ExoPlayer后端
│   ├── contrib/                  # 第三方库编译脚本
│   └── patches/                 # 补丁文件
├── ijkmedia/                     # 核心媒体处理模块
│   ├── ijkplayer/                # C语言播放器核心
│   ├── ijkj4a/                   # Java-JNI桥接层
│   ├── ijksdl/                   # SDL封装层
│   └── ijksdl/android/           # Android平台SDL实现
├── config/                       # FFmpeg配置模块
├── ijkprof/                      # 性能分析模块
└── doc/                          # 文档
```

### 2.2 架构层次图

```
┌─────────────────────────────────────────────────────────────┐
│                    Java/Kotlin 应用层                        │
│                  (IjkMediaPlayer.java)                       │
├─────────────────────────────────────────────────────────────┤
│                    JNI 桥接层                                │
│                  (ijkplayer_jni.c)                          │
├─────────────────────────────────────────────────────────────┤
│                    ijkplayer 核心层                          │
│  ┌─────────────┬──────────────┬──────────────┐              │
│  │ ijkplayer.c │ ff_ffplay.c  │ pipeline/    │              │
│  └─────────────┴──────────────┴──────────────┘              │
├─────────────────────────────────────────────────────────────┤
│                    FFmpeg 编解码层                           │
│  ┌──────────┬──────────┬──────────┬──────────┐             │
│  │avformat  │ avcodec  │ swscale  │swresample│             │
│  └──────────┴──────────┴──────────┴──────────┘             │
├─────────────────────────────────────────────────────────────┤
│                    SDL 输出层                                │
│  ┌─────────────────────┬─────────────────────┐              │
│  │   ijksdl_vout       │    ijksdl_aout      │              │
│  │ (视频渲染/Overlay)   │   (音频输出)         │              │
│  └─────────────────────┴─────────────────────┘              │
├─────────────────────────────────────────────────────────────┤
│                    硬件加速层                                │
│  ┌─────────────────────┬─────────────────────┐              │
│  │  MediaCodec (Android)│ VideoToolbox (iOS)  │              │
│  └─────────────────────┴─────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 主要模块职责

### 3.1 ijkplayer模块 (`ijkmedia/ijkplayer/`)

**核心职责**：媒体播放控制、FFmpeg封装、媒体元数据管理

**核心文件**：

| 文件 | 职责 |
|------|------|
| `ijkplayer.c` | 播放器对外API实现（创建、销毁、播放控制） |
| `ijkplayer.h` | 对外API声明、播放器状态定义 |
| `ijkplayer_internal.h` | 内部数据结构`IjkMediaPlayer`定义 |
| `ff_ffplay.c` | 基于ffplay的播放逻辑实现 |
| `ff_ffplay.h` | FFPlayer结构及函数声明 |
| `ff_ffplay_def.h` | FFPlayer核心数据结构定义 |
| `ijkmeta.c/h` | 媒体元数据管理 |
| `ijkavformat/` | 自定义IO协议和格式支持 |

**关键数据结构**：

```c
// IjkMediaPlayer - Java层调用入口
struct IjkMediaPlayer {
    volatile int ref_count;          // 引用计数
    pthread_mutex_t mutex;           // 线程安全锁
    FFPlayer *ffplayer;              // 核心播放器指针
    int (*msg_loop)(void*);          // 消息循环函数
    SDL_Thread *msg_thread;          // 消息线程
    int mp_state;                    // 播放器状态
    char *data_source;               // 数据源URL
};

// FFPlayer - 核心播放结构
struct FFPlayer {
    VideoState *is;                  // 视频状态
    SDL_Aout *aout;                  // 音频输出
    SDL_Vout *vout;                  // 视频输出
    AVDictionary *format_opts;       // 格式选项
    AVDictionary *codec_opts;        // 编解码选项
    int mediacodec_all_videos;       // MediaCodec配置
    MessageQueue msg_queue;          // 消息队列
    // ... 更多配置选项
};
```

### 3.2 ijksdl模块 (`ijkmedia/ijksdl/`)

**核心职责**：跨平台音视频输出抽象、OpenGL ES渲染、硬件Overlay支持

**目录结构**：

```
ijksdl/
├── ijksdl.h                        # 主头文件
├── ijksdl_vout.h/c                 # 视频输出抽象
├── ijksdl_aout.h/c                 # 音频输出抽象
├── ijksdl_video.h                  # 视频相关定义
├── ijksdl_audio.h                  # 音频相关定义
├── ijksdl_gles2/                   # OpenGL ES 2.0渲染
│   ├── renderer.c                  # 渲染器实现
│   ├── shader.c                    # Shader管理
│   ├── common.c                   # GLES通用函数
│   └── fsh/                        # 片段着色器
│   └── vsh/                        # 顶点着色器
├── android/                        # Android平台实现
│   ├── ijksdl_vout_android_nativewindow.c  # NativeWindow输出
│   ├── ijksdl_aout_android_audiotrack.c   # AudioTrack输出
│   ├── ijksdl_aout_android_opensles.c     # OpenSL ES输出
│   └── ijksdl_codec_android_mediacodec.c  # MediaCodec解码
└── ffmpeg/                         # FFmpeg Overlay实现
```

**关键接口**：

```c
// SDL_Vout - 视频输出抽象
struct SDL_Vout {
    SDL_mutex *mutex;
    SDL_Vout_Opaque *opaque;
    SDL_VoutOverlay *(*create_overlay)(int width, int height, int format, SDL_Vout *vout);
    void (*free_l)(SDL_Vout *vout);
    int (*display_overlay)(SDL_Vout *vout, SDL_VoutOverlay *overlay);
    Uint32 overlay_format;
};

// SDL_Aout - 音频输出抽象
struct SDL_Aout {
    SDL_mutex *mutex;
    double minimal_latency_seconds;
    int (*open_audio)(SDL_Aout *aout, const SDL_AudioSpec *desired, SDL_AudioSpec *obtained);
    void (*pause_audio)(SDL_Aout *aout, int pause_on);
    void (*flush_audio)(SDL_Aout *aout);
    void (*set_volume)(SDL_Aout *aout, float left, float right);
};
```

### 3.3 ijkj4a模块 (`ijkmedia/ijkj4a/`)

**核心职责**：Java/Android API的C语言封装，提供JNI调用接口

**目录结构**：

```
ijkj4a/
├── j4a/                            # J4A自动生成的绑定代码
│   ├── class/
│   │   ├── android/media/          # Android Media*系列
│   │   │   ├── MediaCodec.c/h     # MediaCodec JNI
│   │   │   ├── MediaFormat.c/h    # MediaFormat JNI
│   │   │   └── AudioTrack.c/h     # AudioTrack JNI
│   │   └── java/nio/              # Java NIO支持
│   └── j4a_allclasses.h            # 所有类定义汇总
├── j4au/                           # J4A工具函数
│   └── class/android/media/        # 工具函数
└── java/                          # Java源文件
    └── tv/danmaku/ijk/media/player/
        └── IjkMediaPlayer.java     # Java层播放器
```

### 3.4 Android平台特定模块 (`android/`)

| 模块 | 路径 | 职责 |
|------|------|------|
| arm64播放器 | `ijkplayer/ijkplayer-arm64/` | arm64-v8a架构的NDK代码 |
| armv7a播放器 | `ijkplayer/ijkplayer-armv7a/` | armeabi-v7a架构的NDK代码 |
| x86_64播放器 | `ijkplayer/ijkplayer-x86_64/` | x86_64架构的NDK代码 |
| Java封装 | `ijkplayer/ijkplayer-java/` | IjkMediaPlayer Java类 |
| 示例应用 | `ijkplayer/ijkplayer-example/` | 演示程序 |
| 编译脚本 | `contrib/` | FFmpeg/OpenSSL编译 |

---

## 4. 关键类与函数说明

### 4.1 Java层核心API (`IjkMediaPlayer`)

**播放器生命周期**：

```java
// 创建播放器实例
public IjkMediaPlayer()

// 设置数据源（本地文件或网络URL）
public void setDataSource(String url)

// 异步准备（推荐用于网络流）
public void prepareAsync()

// 同步准备
public void prepare()

// 开始播放
public void start()

// 暂停播放
public void pause()

// 停止播放
public void stop()

// 释放资源
public void release()

// 重置播放器状态
public void reset()
```

**播放控制**：

```java
// 跳转到指定位置（毫秒）
public void seekTo(long msec)

// 获取当前播放位置
public long getCurrentPosition()

// 获取媒体总时长
public long getDuration()

// 是否正在播放
public boolean isPlaying()

// 设置播放循环次数（-1无限循环）
public void setLooping(int looping)
```

**选项设置**：

```java
// 设置字符串选项
public void setOption(int category, String name, String value)

// 设置整数选项
public void setOption(int category, int name, long value)

// 选项类别
public static final int OPT_CATEGORY_FORMAT  = 1;  // 格式选项
public static final int OPT_CATEGORY_CODEC   = 2;  // 编解码选项
public static final int OPT_CATEGORY_SWS     = 3;  // 缩放选项
public static final int OPT_CATEGORY_PLAYER  = 4;  // 播放器选项
```

**硬件解码配置**：

```java
// 启用所有视频的MediaCodec硬解
setOption(OPT_CATEGORY_PLAYER, "mediacodec", 1);

// 启用HEVC MediaCodec
setOption(OPT_CATEGORY_PLAYER, "mediacodec-hevc", 1);

// 设置VideoToolbox（iOS）
setOption(OPT_CATEGORY_PLAYER, "videotoolbox", 1);
```

**重要属性**：

```java
// 视频解码帧率
public static final int PROP_FLOAT_VIDEO_DECODE_FRAMES_PER_SECOND = 10001;

// 视频输出帧率
public static final int PROP_FLOAT_VIDEO_OUTPUT_FRAMES_PER_SECOND = 10002;

// 丢帧率
public static final int FFP_PROP_FLOAT_DROP_FRAME_RATE = 10007;

// 视频解码器类型
public static final int FFP_PROP_INT64_VIDEO_DECODER = 20003;
public static final int FFP_PROPV_DECODER_AVCODEC     = 1;  // FFmpeg软解
public static final int FFP_PROPV_DECODER_MEDIACODEC  = 2;  // MediaCodec硬解
```

### 4.2 JNI层核心函数 (`ijkplayer_jni.c`)

**Native方法映射**：

```c
// 播放器创建与销毁
static void        ijkMediaPlayer_native_setup(JNIEnv *env, jobject thiz, ...);
static void        ijkMediaPlayer_native_finalize(JNIEnv *env, jobject thiz);

// 数据源设置
static void        ijkMediaPlayer_setDataSource(JNIEnv *env, jobject thiz, jstring path);

// 播放控制
static void        ijkMediaPlayer_prepareAsync(JNIEnv *env, jobject thiz);
static void        ijkMediaPlayer_start(JNIEnv *env, jobject thiz);
static void        ijkMediaPlayer_pause(JNIEnv *env, jobject thiz);
static void        ijkMediaPlayer_stop(JNIEnv *env, jobject thiz);

// 定位
static void        ijkMediaPlayer_seekTo(JNIEnv *env, jobject thiz, jlong msec);

// Surface绑定
static void        ijkMediaPlayer_setSurface(JNIEnv *env, jobject thiz, jobject surface);
```

### 4.3 C层核心API (`ijkplayer.h`)

**全局初始化**：

```c
// 全局初始化（必须在创建播放器前调用）
void ijkmp_global_init();

// 全局反初始化
void ijkmp_global_uninit();

// 设置日志级别
void ijkmp_global_set_log_level(int log_level);  // AV_LOG_xxx

// 注册IO统计回调
void ijkmp_io_stat_register(void (*cb)(const char *url, int type, int bytes));
```

**播放器创建**：

```c
// 创建播放器实例
// msg_loop: 消息循环函数指针
IjkMediaPlayer *ijkmp_create(int (*msg_loop)(void*));

// 引用计数管理
void ijkmp_inc_ref(IjkMediaPlayer *mp);
void ijkmp_dec_ref(IjkMediaPlayer *mp);
```

**数据源与准备**：

```c
// 设置数据源
int ijkmp_set_data_source(IjkMediaPlayer *mp, const char *url);

// 异步准备
int ijkmp_prepare_async(IjkMediaPlayer *mp);
```

**播放控制**：

```c
int ijkmp_start(IjkMediaPlayer *mp);
int ijkmp_pause(IjkMediaPlayer *mp);
int ijkmp_stop(IjkMediaPlayer *mp);
int ijkmp_seek_to(IjkMediaPlayer *mp, long msec);
```

**状态查询**：

```c
int  ijkmp_get_state(IjkMediaPlayer *mp);
bool ijkmp_is_playing(IjkMediaPlayer *mp);
long ijkmp_get_current_position(IjkMediaPlayer *mp);
long ijkmp_get_duration(IjkMediaPlayer *mp);
```

**播放器状态常量**：

```c
#define MP_STATE_IDLE              0   // 初始状态
#define MP_STATE_INITIALIZED        1   // 已设置数据源
#define MP_STATE_ASYNC_PREPARING    2   // 异步准备中
#define MP_STATE_PREPARED           3   // 已准备好
#define MP_STATE_STARTED            4   // 播放中
#define MP_STATE_PAUSED             5   // 已暂停
#define MP_STATE_COMPLETED          6   // 播放完成
#define MP_STATE_STOPPED            7   // 已停止
#define MP_STATE_ERROR              8   // 错误状态
#define MP_STATE_END                9   // 已释放
```

**选项设置**：

```c
void ijkmp_set_option(IjkMediaPlayer *mp, int opt_category, const char *name, const char *value);
void ijkmp_set_option_int(IjkMediaPlayer *mp, int opt_category, const char *name, int64_t value);

// 选项类别
#define IJKMP_OPT_CATEGORY_FORMAT FFP_OPT_CATEGORY_FORMAT
#define IJKMP_OPT_CATEGORY_CODEC  FFP_OPT_CATEGORY_CODEC
#define IJKMP_OPT_CATEGORY_SWS    FFP_OPT_CATEGORY_SWS
#define IJKMP_OPT_CATEGORY_PLAYER FFP_OPT_CATEGORY_PLAYER
```

**播放属性**：

```c
void ijkmp_set_playback_rate(IjkMediaPlayer *mp, float rate);
void ijkmp_set_playback_volume(IjkMediaPlayer *mp, float volume);
float ijkmp_get_property_float(IjkMediaPlayer *mp, int id, float default_value);
void  ijkmp_set_property_float(IjkMediaPlayer *mp, int id, float value);
```

### 4.4 FFPlayer核心函数 (`ff_ffplay.h`)

**播放器创建与销毁**：

```c
FFPlayer *ffp_create();
void ffp_destroy(FFPlayer *ffp);
void ffp_reset(FFPlayer *ffp);
```

**播放流程**：

```c
// 准备播放（异步）
int ffp_prepare_async_l(FFPlayer *ffp, const char *file_name);

// 播放控制
int ffp_start_l(FFPlayer *ffp);
int ffp_start_from_l(FFPlayer *ffp, long msec);
int ffp_pause_l(FFPlayer *ffp);
int ffp_stop_l(FFPlayer *ffp);
int ffp_wait_stop_l(FFPlayer *ffp);
```

**Seek操作**：

```c
int ffp_seek_to_l(FFPlayer *ffp, long msec);
long ffp_get_current_position_l(FFPlayer *ffp);
long ffp_get_duration_l(FFPlayer *ffp);
```

### 4.5 Pipeline机制

Pipeline是ijkplayer的插件化架构，支持替换视频解码和音频输出实现。

**Android平台支持**：

| Pipeline类型 | 描述 | 位置 |
|-------------|------|------|
| `ffpipeline_ffplay` | FFmpeg软解Pipeline | `ijkmedia/ijkplayer/pipeline/ffpipeline_ffplay.c` |
| `ffpipeline_android` | MediaCodec硬解Pipeline | `ijkmedia/ijkplayer/android/pipeline/ffpipeline_android.c` |

**Pipeline节点**：

```c
// Pipeline节点接口
struct IJKFF_Pipenode {
    SDL_Class *opaque_class;
    void *opaque;
    int (*func_destroy)(IJKFF_Pipenode *node);
    int (*func_run_sync)(IJKFF_Pipenode *node);
    int (*func_flush)(IJKFF_Pipenode *node);
};
```

**支持的视频解码器**：

```c
// 软解码（FFmpeg）
IJKFF_Pipenode *ffpipenode_create_video_decoder_from_ffplay(FFPlayer *ffp);

// MediaCodec硬解
IJKFF_Pipenode *ffpipenode_android_mediacodec_vdec_create(FFPlayer *ffp);
```

---

## 5. 依赖关系

### 5.1 项目内部依赖

```
┌─────────────────────────────────────────────────────────────┐
│                       ijkj4a (Java/JNI)                     │
│   - 提供Java到Native的桥接                                   │
│   - 依赖: IjkMediaPlayer.java, MediaCodec, MediaFormat        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    ijkplayer (C层)                          │
│   - 播放器核心逻辑                                           │
│   - 依赖: ijksdl, FFmpeg, ijkmeta                           │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    ijksdl       │ │    FFmpeg       │ │   ijkmeta       │
│   (音视频输出)    │ │   (编解码核心)   │ │   (元数据管理)   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### 5.2 外部依赖

| 依赖库 | 版本 | 用途 | 许可证 |
|--------|------|------|--------|
| FFmpeg | 4.3 (ijk-n4.3-20260301-007) | 音视频编解码、封装 | LGPL/GPL |
| OpenSSL | 1.1.1w | HTTPS/TLS加密支持 | Apache 2.0 |
| SDL | - | 跨平台音视频抽象 | zlib |
| libyuv | - | YUV图像处理 | BSD |
| soundtouch | - | 音频变速不变调 | LGPL |

### 5.3 Android平台依赖

| 组件 | 最小API | 用途 |
|------|---------|------|
| MediaCodec | API 16+ | 硬件视频解码 |
| AudioTrack | API 1+ | 音频输出 |
| OpenSL ES | API 10+ | 低延迟音频 |
| NativeWindow | API 1+ | 原生窗口渲染 |
| OpenGL ES 2.0 | API 8+ | GPU渲染 |

---

## 6. 项目构建与运行

### 6.1 环境要求

**构建环境**：

- 主机系统：macOS 10.11+ / Linux
- Android NDK：r22 (推荐)
- Android SDK：API 21+
- Android Gradle Plugin：2.x+
- Java：JDK 8+

**环境变量配置**：

```bash
# ~/.bash_profile 或 ~/.profile
export ANDROID_SDK=/path/to/android-sdk
export ANDROID_NDK=/path/to/android-ndk-r22
```

### 6.2 构建步骤

**1. 初始化子模块**：

```bash
cd ijkplayer
./init-android.sh
```

**2. 编译第三方库（FFmpeg + OpenSSL）**：

```bash
cd android/contrib
./compile-openssl.sh clean
./compile-openssl.sh all
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all
```

**3. 编译ijkplayer主项目**：

```bash
cd ../..

# 编译所有64位ABI（推荐）
./compile-ijk.sh all

# 或单独编译指定ABI
./compile-ijk.sh arm64
./compile-ijk.sh armv7a
./compile-ijk.sh x86_64
```

**4. 清理构建**：

```bash
./compile-ijk.sh clean
cd android/contrib
./compile-ffmpeg.sh clean
```

### 6.3 使用Android Studio导入

1. 打开Android Studio
2. 选择 "Open an existing Android Studio project"
3. 选择 `android/ijkplayer/` 目录
4. 同步Gradle配置
5. 选择对应的ABI模块构建

**Gradle配置示例**：

```groovy
// root build.gradle
ext {
    compileSdkVersion = 33
    buildToolsVersion = "33.0.0"
    targetSdkVersion = 33
}
```

### 6.4 集成到现有项目

**添加依赖**：

```groovy
dependencies {
    // 必需模块
    compile 'tv.danmaku.ijk.media:ijkplayer-java:0.8.8'
    compile 'tv.danmaku.ijk.media:ijkplayer-armv7a:0.8.8'
    
    // 可选模块
    compile 'tv.danmaku.ijk.media:ijkplayer-arm64:0.8.8'
    compile 'tv.danmaku.ijk.media:ijkplayer-x86_64:0.8.8'
}
```

### 6.5 配置选项

**FFmpeg配置模块**：

| 配置文件 | 描述 | 二进制大小 |
|----------|------|-----------|
| `module-lite.sh` | 基础配置 | ~2MB |
| `module-lite-hevc.sh` | 包含HEVC支持 | ~2.5MB |
| `module-default.sh` | 完整功能 | ~8MB |

**切换配置**：

```bash
cd config
rm module.sh
ln -s module-lite-hevc.sh module.sh
```

---

## 7. 播放器状态机

### 7.1 状态转换图

```
                                    ┌──────────┐
                                    │   IDLE   │
                                    └────┬─────┘
                                         │
                                    setDataSource()
                                         │
                                         ▼
                              ┌───────────────────┐
                              │   INITIALIZED     │
                              └─────────┬─────────┘
                                        │
                                 prepareAsync()
                                        │
                                        ▼
                              ┌─────────────────────┐
                         ┌────│ ASYNC_PREPARING    │────┐
                         │    └─────────────────────┘    │
                         │           │                   │
                    error│           │ prepared           │error
                         │           ▼                   │
                         │    ┌────────────┐              │
                         └───►│  ERROR     │◄─────────────┘
                              └────────────┘
                                        │
                              ┌─────────┴─────────┐
                         reset()                │
                              │                 │
                              ▼                 │
                         ┌────────┐              │
                         │  IDLE  │◄─────────────┘
                         └────────┘
                                        │
                              ┌─────────┴─────────┐
                         prepareAsync()             │
                              │                    │
                              ▼                    │
                         ┌───────────────────┐     │
                    ┌────│   PREPARED        │     │
                    │    └─────────┬─────────┘     │
                    │              │               │
               start_on_prepared=1 │    start_on_prepared=0
                    │              │               │
                    │              ▼               │
                    │         ┌────────┐          │
                    │         │ PAUSED │          │
                    │         └───┬────┘          │
                    │             │               │
                    │             │               │
                    ▼             ▼               ▼
              ┌────────┐    ┌────────┐     ┌──────────┐
              │STARTED │    │STARTED │     │COMPLETED │
              └───┬────┘    └────┬───┘     └────┬─────┘
                  │             │              │
            pause()│       start()│        seekTo()
                  │             │              │
                  ▼             │              ▼
             ┌────────┐        │         ┌────────┐
             │ PAUSED │────────┴────────►│STARTED │
             └────────┘                  └────────┘
```

### 7.2 状态说明

| 状态 | 值 | 说明 |
|------|---|------|
| MP_STATE_IDLE | 0 | 初始状态，未设置数据源 |
| MP_STATE_INITIALIZED | 1 | 已设置数据源 |
| MP_STATE_ASYNC_PREPARING | 2 | 异步准备中 |
| MP_STATE_PREPARED | 3 | 准备完成，可以调用start() |
| MP_STATE_STARTED | 4 | 正在播放 |
| MP_STATE_PAUSED | 5 | 已暂停 |
| MP_STATE_COMPLETED | 6 | 播放完成 |
| MP_STATE_STOPPED | 7 | 已停止 |
| MP_STATE_ERROR | 8 | 发生错误 |
| MP_STATE_END | 9 | 已释放 |

---

## 8. 硬件解码支持

### 8.1 Android MediaCodec

**支持的视频格式**：

| 格式 | MIME类型 | 支持情况 |
|------|----------|----------|
| H.264/AVC | video/avc | ✅ 稳定支持 |
| H.265/HEVC | video/hevc | ✅ 支持 |
| VP8 | video/x-vnd.on2.vp8 | ✅ 支持 |
| VP9 | video/x-vnd.on2.vp9 | ✅ 支持 |
| MPEG-2 | video/mpeg2 | ⚠️ 可选支持 |
| MPEG-4 | video/mp4v-es | ⚠️ 可选支持 |

**启用配置**：

```java
// 启用MediaCodec硬解
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 1);

// 启用HEVC硬解
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-hevc", 1);

// 允许所有视频使用MediaCodec（默认只有非B帧的视频）
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec_all_videos", 1);
```

### 8.2 iOS VideoToolbox

**启用方式**：

```java
// iOS专用配置
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "videotoolbox", 1);
```

---

## 9. 重要配置选项

### 9.1 播放器选项

| 选项名 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `start_on_prepared` | int | 1 | 准备完成后自动开始播放 |
| `packet_buffering` | int | 1 | 是否启用packet缓冲 |
| `framedrop` | int | 0 | 丢帧策略（0=不丢，1=丢迟帧，-1=智能丢帧） |
| `max_fps` | int | 31 | 最大帧率限制 |
| `overlay_format` | int | - | 覆盖层格式 |

### 9.2 网络选项

| 选项名 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `timeout` | int | - | 超时时间（微秒） |
| `reconnect` | int | - | 自动重连次数 |
| `dns_cache` | int | - | DNS缓存时间（秒） |

### 9.3 软解/硬解选项

```java
// FFmpeg软解配置
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_CODEC, "threads", 4);

// MediaCodec配置
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 1);
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec_handle_resolution_change", 1);
```

---

## 10. 消息回调机制

### 10.1 消息类型

```java
// 基本消息
public static final int MEDIA_PREPARED = 1;
public static final int MEDIA_PLAYBACK_COMPLETE = 2;
public static final int MEDIA_BUFFERING_UPDATE = 3;
public static final int MEDIA_SEEK_COMPLETE = 4;
public static final int MEDIA_SET_VIDEO_SIZE = 5;
public static final int MEDIA_ERROR = 100;
public static final int MEDIA_INFO = 200;

// 扩展消息
public static final int MEDIA_SET_VIDEO_SAR = 10001;
```

### 10.2 错误码

```java
public static final int MEDIA_ERROR_UNKNOWN = 1;
public static final int MEDIA_ERROR_SERVER_DIED = 100;
public static final int MEDIA_ERROR_NOT_VALID_FOR_PROGRESSIVE_PLAYBACK = 200;
```

### 10.3 信息码

```java
public static final int MEDIA_INFO_UNKNOWN = 1;
public static final int MEDIA_INFO_STARTED_AS_NEXT = 2;
public static final int MEDIA_INFO_VIDEO_RENDERING_START = 3;
public static final int MEDIA_INFO_AUDIO_RENDERING_START = 4;
public static final int MEDIA_INFO_BUFFERING_START = 701;
public static final int MEDIA_INFO_BUFFERING_END = 702;
```

---

## 11. 使用示例

### 11.1 基础播放

```java
public class VideoPlayerActivity extends AppCompatActivity {
    private IjkMediaPlayer mMediaPlayer;
    private SurfaceView mSurfaceView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_player);
        
        // 初始化播放器
        mMediaPlayer = new IjkMediaPlayer();
        
        // 设置日志级别
        IjkMediaPlayer.native_setLogLevel(IjkMediaPlayer.IJK_LOG_DEBUG);
        
        // 获取SurfaceView
        mSurfaceView = findViewById(R.id.surface_view);
        mSurfaceView.getHolder().addCallback(new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(SurfaceHolder holder) {
                mMediaPlayer.setDisplay(holder);
                mMediaPlayer.setScreenOnWhilePlaying(true);
            }
            
            @Override
            public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {}
            
            @Override
            public void surfaceDestroyed(SurfaceHolder holder) {}
        });
        
        // 配置选项
        mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 1);
        mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "dns_cache", 600);
        
        // 设置数据源并准备
        mMediaPlayer.setDataSource("https://example.com/video.mp4");
        mMediaPlayer.prepareAsync();
        
        // 设置监听器
        mMediaPlayer.setOnPreparedListener(mp -> {
            mp.start();
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mMediaPlayer != null) {
            mMediaPlayer.release();
            mMediaPlayer = null;
        }
    }
}
```

### 11.2 直播流播放

```java
// HLS直播流
mMediaPlayer.setDataSource("https://live.example.com/stream.m3u8");

// RTMP流
mMediaPlayer.setDataSource("rtmp://live.example.com/stream");

// 配置直播优化
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "fflags", "nobuffer");
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "analyzeduration", 1);
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "packet-buffering", 0);
```

---

## 12. 性能优化建议

### 12.1 首帧加载优化

```java
// 使用prepareAsync配合封面图
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "render-wait-cluster", 1);

// 设置首帧超时
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "first-frame-timeout", 30000);
```

### 12.2 内存优化

```java
// 设置视频缓冲大小
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "buffer_size", 1024 * 1024);

// 释放时清理缓存
mMediaPlayer.release();
System.gc();
```

### 12.3 播放流畅度优化

```java
// 启用丢帧策略（网络不稳定时）
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "framedrop", 1);

// 设置最大帧率
mMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "max-fps", 30);
```

---

## 13. 常见问题

### 13.1 编译问题

**Q: NDK编译报错 "aarch64-linux-android-ar crashed"**

A: 这是NDK r22在Darwin arm64上的已知问题。项目已修复，请确保使用最新的编译脚本。

**Q: OpenSSL编译失败**

A: 检查NDK路径是否正确配置，确保ANDROID_NDK环境变量指向r22版本。

### 13.2 运行问题

**Q: 播放无声音**

A: 检查是否正确配置了AudioTrack/OpenSL ES，以及音频格式是否被支持。

**Q: MediaCodec硬解不生效**

A: 确认设备Android版本>=4.1，且视频格式被设备硬件支持。

### 13.3 性能问题

**Q: 首帧加载慢**

A: 可尝试使用更小的FFmpeg配置模块，或启用MediaCodec硬解。

---

## 14. 参考资源

- [官方GitHub仓库](https://github.com/Bilibili/ijkplayer)
- [GSYVideoPlayer 16K补丁](https://github.com/CarGuo/GSYVideoPlayer/tree/master/16kpatch)
- [FFmpeg官方文档](https://ffmpeg.org/documentation.html)
- [Android MediaCodec文档](https://developer.android.com/reference/android/media/MediaCodec)

---

*文档版本：1.0*  
*最后更新：2026-05-18*  
*项目版本：ijk-n4.3-20260301-007*
