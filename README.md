# LVGL v9 PC 模拟器 (Windows + MinGW + SDL2)

本项目是将 [LVGL v9](https://github.com/lvgl/lvgl)（Light and Versatile Graphics Library）移植到 Windows PC 平台的模拟器工程，使用 **SDL2** 作为显示和输入后端，通过 **CMake + MinGW** 工具链进行构建，可在 VSCode 中直接编译、调试和运行。在 PC 上开发的 UI 代码可以直接移植到嵌入式平台使用。

---

## 项目概述

| 项目 | 说明 |
|------|------|
| LVGL 版本 | v9.5.0-dev |
| 显示后端 | SDL2 v2.32.10（本地预编译，MinGW x86_64） |
| 构建系统 | CMake 4.1.2 + Ninja |
| 编译器 | MinGW-w64 GCC（通过 `mingw64/` 目录提供） |
| IDE | Visual Studio Code |
| 操作系统 | Windows |

### 核心特性

- 使用 SDL2 模拟嵌入式 LCD 显示（默认分辨率 320x480）
- 支持鼠标、鼠标滚轮、键盘等输入设备的模拟
- 可选启用 **FreeRTOS** 以更真实地模拟嵌入式 RTOS 环境
- 内置 LVGL 全部 Demo（Widgets、Benchmark、Stress、Music 等）
- 支持可选第三方库：FreeType、libpng、libjpeg-turbo、FFmpeg、SDL2_image
- 支持 LVGL Pro 项目集成

---

## 项目目录结构

```
lvgl/                              # 项目根目录
├── lv_port_pc_vscode-master/      # LVGL PC 模拟器工程
│   ├── CMakeLists.txt             # 顶层 CMake 构建文件
│   ├── lv_conf.h                  # LVGL 配置文件
│   ├── lv_conf.defaults           # LVGL 默认配置覆盖
│   ├── simulator.code-workspace   # VSCode 工作区配置
│   ├── config/
│   │   └── FreeRTOSConfig.h       # FreeRTOS 配置
│   ├── src/
│   │   ├── main.c                 # 主入口（裸机模式）
│   │   ├── mouse_cursor_icon.c    # 鼠标光标图标数据
│   │   ├── hal/
│   │   │   ├── hal.h              # HAL 层头文件
│   │   │   └── hal.c              # HAL 层实现（SDL 显示/输入初始化）
│   │   ├── freertos_main.c        # FreeRTOS 模式主入口
│   │   └── freertos/
│   │       └── freertos_posix_port.c  # FreeRTOS POSIX 事件机制
│   ├── lvgl/                      # LVGL 库源码（Git 子模块）
│   │   ├── src/                   # LVGL 核心源码
│   │   ├── examples/              # LVGL 示例
│   │   ├── demos/                 # LVGL Demo
│   │   └── CMakeLists.txt         # LVGL 的 CMake 构建文件
│   └── SDL2-2.32.10/              # SDL2 库（本地预编译，含 MinGW 头文件和库）
├── mingw64/                       # MinGW-w64 GCC 工具链
└── cmake-4.1.2-windows-x86_64/    # CMake 工具
```

---

## 移植过程

本项目基于 LVGL 官方的 `lv_port_pc_vscode` 模板工程，移植到 Windows 平台并使用本地预编译的工具链。以下是完整的移植步骤：

### 第一步：准备工具链

1. **获取 MinGW-w64 GCC 编译器**：将 MinGW-w64 工具链放置在项目根目录的 `mingw64/` 文件夹中，确保 `mingw64/bin/` 下包含 `gcc.exe`、`g++.exe`、`gdb.exe`、`ninja.exe` 等可执行文件。

2. **获取 CMake**：将 CMake 放置在项目根目录的 `cmake-4.1.2-windows-x86_64/` 文件夹中。

3. **配置 VSCode 工作区**：在 `simulator.code-workspace` 中指定 CMake 路径和生成器：

   ```json
   "settings": {
       "cmake.cmakePath": "C:\\Users\\Lang\\Desktop\\code\\LVGL\\lvgl\\cmake-4.1.2-windows-x86_64\\bin\\cmake",
       "cmake.generator": "Ninja"
   }
   ```

### 第二步：准备 SDL2 库

本项目使用本地预编译的 SDL2 库（v2.32.10），而非通过 vcpkg 等包管理器安装：

1. 下载 SDL2 源码并使用 MinGW 进行交叉编译，或直接获取预编译的 MinGW 版本。
2. 将 SDL2 放置在 `lv_port_pc_vscode-master/SDL2-2.32.10/` 目录下。
3. 确保目录结构包含：
   - `x86_64-w64-mingw32/include/SDL2/` — 头文件
   - `x86_64-w64-mingw32/lib/` — 静态库（`libSDL2.a`、`libSDL2main.a`）
   - `x86_64-w64-mingw32/bin/SDL2.dll` — 动态链接库（运行时需要）
   - `cmake/sdl2-config.cmake` — CMake 配置文件

4. 在 `CMakeLists.txt` 中通过 `set(SDL2_DIR ...)` 指向 SDL2 的 CMake 配置目录：

   ```cmake
   set(SDL2_DIR ${PROJECT_SOURCE_DIR}/SDL2-2.32.10/cmake)
   find_package(SDL2 REQUIRED)
   ```

### 第三步：配置 LVGL

1. 将 `lv_conf.h` 放置在 LVGL 库同级目录（即 `lv_port_pc_vscode-master/lv_conf.h`）。
2. 关键配置项：
   - **颜色深度**：`LV_COLOR_DEPTH 32`（XRGB8888）
   - **内存池大小**：`LV_MEM_SIZE (1024 * 1024)`（1MB）
   - **操作系统**：`LV_USE_OS LV_OS_NONE`（裸机模式，可切换为 `LV_OS_FREERTOS`）
   - **刷新周期**：`LV_DEF_REFR_PERIOD 33`（约 30 FPS）
   - **渲染引擎**：软件渲染（`LV_USE_DRAW_SW 1`）
   - **SDL 驱动**：`LV_USE_SDL 1`（通过 `lv_conf.defaults` 启用）
   - **日志**：启用 `LV_USE_LOG`，级别为 `LV_LOG_LEVEL_WARN`

3. 编译时定义 `LV_CONF_INCLUDE_SIMPLE`，使 LVGL 可以通过 `#include "lv_conf.h"` 找到配置文件。

### 第四步：实现 HAL 层（硬件抽象层）

HAL 层是移植的核心，负责将 LVGL 的显示和输入接口对接到 SDL2：

**文件：`src/hal/hal.c`**

```c
lv_display_t * sdl_hal_init(int32_t w, int32_t h)
{
    // 1. 创建默认输入组（用于键盘/编码器导航）
    lv_group_set_default(lv_group_create());

    // 2. 创建 SDL 窗口作为 LVGL 显示设备
    lv_display_t * disp = lv_sdl_window_create(w, h);

    // 3. 注册鼠标输入设备
    lv_indev_t * mouse = lv_sdl_mouse_create();
    lv_indev_set_group(mouse, lv_group_get_default());
    lv_indev_set_display(mouse, disp);
    lv_display_set_default(disp);

    // 4. 设置鼠标光标图标
    LV_IMAGE_DECLARE(mouse_cursor_icon);
    lv_obj_t * cursor_obj = lv_image_create(lv_screen_active());
    lv_image_set_src(cursor_obj, &mouse_cursor_icon);
    lv_indev_set_cursor(mouse, cursor_obj);

    // 5. 注册鼠标滚轮输入设备
    lv_indev_t * mousewheel = lv_sdl_mousewheel_create();
    lv_indev_set_display(mousewheel, disp);

    // 6. 注册键盘输入设备
    lv_indev_t * kb = lv_sdl_keyboard_create();
    lv_indev_set_display(kb, disp);
    lv_indev_set_group(kb, lv_group_get_default());

    return disp;
}
```

移植要点：
- `lv_sdl_window_create()` 创建 SDL 窗口并绑定到 LVGL 显示缓冲区
- `lv_sdl_mouse_create()` / `lv_sdl_mousewheel_create()` / `lv_sdl_keyboard_create()` 利用 LVGL 内置的 SDL 驱动注册输入设备
- 所有输入设备绑定到同一个 `lv_group`，实现焦点导航

### 第五步：编写主循环

**裸机模式（`src/main.c`）：**

```c
int main(int argc, char **argv)
{
    lv_init();                          // 初始化 LVGL
    sdl_hal_init(320, 480);             // 初始化 HAL（显示 + 输入）
    lv_demo_widgets();                  // 运行 Demo

    while(1) {
        uint32_t sleep_time_ms = lv_timer_handler();  // 调用 LVGL 任务处理器
        if(sleep_time_ms == LV_NO_TIMER_READY)
            sleep_time_ms = LV_DEF_REFR_PERIOD;
        Sleep(sleep_time_ms);           // Windows 下休眠
    }
    return 0;
}
```

**FreeRTOS 模式（`src/freertos_main.c`）：**

```c
void lvgl_task(void *pvParameters)
{
    lv_init();
    sdl_hal_init(320, 480);
    create_hello_world_screen();

    while (true) {
        lv_timer_handler();
        vTaskDelay(pdMS_TO_TICKS(5));   // RTOS 任务延迟
    }
}

int main(int argc, char **argv)
{
    xTaskCreate(lvgl_task, "LVGL Task", 4096, NULL, 1, NULL);
    xTaskCreate(another_task, "Another Task", 1024, NULL, 1, NULL);
    vTaskStartScheduler();              // 启动 FreeRTOS 调度器
}
```

### 第六步：CMake 构建配置

`CMakeLists.txt` 中的关键配置：

- **C/C++ 标准**：C99 / C++17
- **SDL2 查找**：通过本地 `SDL2_DIR` 查找预编译库
- **FreeRTOS 可选**：通过 `option(USE_FREERTOS ...)` 控制是否编译 FreeRTOS 源码
- **可选库支持**：通过 CMake Option 控制是否启用 libpng、libjpeg-turbo、FFmpeg、FreeType 等
- **Windows 控制台**：链接 `-mconsole` 确保日志输出可见
- **Debug 模式**：启用严格的编译警告选项，可选启用 AddressSanitizer

---

## 构建与运行

### 环境要求

- Windows 操作系统
- MinGW-w64 GCC（已包含在项目中）
- CMake >= 3.12.4（已包含在项目中）
- Ninja 构建工具（已包含在 MinGW 中）
- Visual Studio Code（推荐）

### 使用 VSCode 构建

1. 用 VSCode 打开 `simulator.code-workspace` 工作区
2. 安装推荐插件：`C/C++`、`CMake Tools`
3. CMake 会自动配置（`cmake.configureOnOpen: true`）
4. 按 **F5** 或点击运行按钮开始调试

### 使用命令行构建

```bash
cd lv_port_pc_vscode-master
cmake -B build -G Ninja -DCMAKE_C_COMPILER=<mingw64>/bin/gcc.exe -DCMAKE_CXX_COMPILER=<mingw64>/bin/g++.exe
cmake --build build
```

运行时需要将 `SDL2.dll` 复制到可执行文件所在目录，或将其添加到系统 PATH 中。

---

## 运行 Demo

默认运行 `lv_demo_widgets()`。可在 `src/main.c` 中替换为其他 Demo：

| Demo 函数 | 说明 |
|-----------|------|
| `lv_demo_widgets()` | 控件展示 Demo（默认） |
| `lv_demo_benchmark()` | 性能基准测试 |
| `lv_demo_stress()` | 压力测试 |
| `lv_demo_music()` | 音乐播放器 UI |
| `lv_demo_keypad_and_encoder()` | 键盘和编码器导航 |
| `lv_demo_flex_layout()` | Flex 布局演示 |
| `lv_demo_multilang()` | 多语言支持演示 |
| `lv_demo_transform()` | 变换效果演示 |
| `lv_demo_scroll()` | 滚动效果演示 |

---

## 启用 FreeRTOS

如需在 RTOS 环境下运行 LVGL（模拟嵌入式场景）：

1. 修改 `lv_conf.h`：
   ```c
   #define LV_USE_OS  LV_OS_FREERTOS
   ```

2. 通过 CMake 启用 FreeRTOS 编译：
   ```bash
   cmake -B build -DUSE_FREERTOS=ON
   ```

3. FreeRTOS 堆大小配置为 **512 MB**（`config/FreeRTOSConfig.h`），以确保 SDL 窗口正常显示。

---

## 可选第三方库

通过 CMake 选项启用额外的图像/字体/视频支持：

```bash
cmake -B build -DLV_USE_FREETYPE=ON -DLV_USE_LIBPNG=ON -DLV_USE_LIBJPEG_TURBO=ON -DLV_USE_FFMPEG=ON
```

| CMake 选项 | 说明 |
|------------|------|
| `LV_USE_DRAW_SDL` | 使用 SDL 纹理缓存加速渲染 |
| `LV_USE_FREETYPE` | FreeType 字体渲染 |
| `LV_USE_LIBPNG` | PNG 图片解码 |
| `LV_USE_LIBJPEG_TURBO` | JPEG 图片解码 |
| `LV_USE_FFMPEG` | FFmpeg 视频播放 |

---

## 从 PC 移植到嵌入式平台

在 PC 模拟器上开发和调试完成后，将代码移植到嵌入式平台只需：

1. **替换 HAL 层**：将 `hal.c` 中的 SDL 相关调用替换为实际的 LCD 驱动（SPI/I2C/RGB 接口）和触摸/按键驱动
2. **替换 `lv_conf.h`**：根据目标芯片调整内存大小、颜色深度、DPI 等参数
3. **替换主循环**：将 `while(1)` + `Sleep()` 替换为 RTOS 任务或定时器中断中调用 `lv_timer_handler()`
4. **UI 代码无需修改**：所有使用 LVGL API 编写的 UI 代码可直接复用
