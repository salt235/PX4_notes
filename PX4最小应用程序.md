# PX4最小应用程序

PX4 应用的“构建”，简单说就是：**把你写的 PX4 应用代码，编译、链接，并打包进 PX4 固件或仿真程序里，让 PX4 能识别并运行它。**

整体的流程：

```
你写的代码
   ↓
CMake 告诉 PX4 这个应用怎么编译
   ↓
make px4_sitl 编译整个 PX4 工程
   ↓
生成可运行的 PX4 模块
   ↓
进入 PX4 shell
   ↓
输入应用名运行
```

## 1. 创建c文件

创建一个新的目录`PX4-Autopilot/src/examples/hello_sky_app`。

在该目录下创建一个名为`hello_sky_app.c`的新 C 文件：

```c
#include <px4_platform_common/log.h> // 引入PX4的日志系统

// 主函数必须按所示方式命名<module_name>_main并从模块中导出。
__EXPORT int hello_sky_app_main(int argc, char *argv[]);

int hello_sky_app_main(int argc, char *argv[])
{
	PX4_INFO("Hello Sky!");
	return OK;
}
```

## 2. 创建Cmake文件

在同一个文件夹，创建`CMakeLists.txt`文件：

```cmake
px4_add_module(
	MODULE examples__hello_sky_app  # 模块的内部名字,也是模块的路径，双下划线 __ 可以理解成把路径里的 /
	MAIN hello_sky_app # 模块主入口
	STACK_MAIN 2000 # 分配 2000 字节左右的栈空间
	SRCS # 源文件列表。也就是这个模块由哪些 .c / .cpp 文件编译而来
		hello_sky_app.c
	DEPENDS # 暂时没有依赖项
	)
```

`px4_add_module`是 PX4 自己封装的 CMake 函数。普通 CMake 里没有这个函数，它是 PX4 构建系统提供的。它的作用类似于告诉 PX4：

“我要添加一个 PX4 模块，这个模块有哪些源文件，入口函数叫什么，需要多少栈空间，依赖哪些东西。”

## 3. 创建Kconfig文件

在同一个文件夹，创建`Kconfig`文件：

```kconfig
menuconfig EXAMPLES_HELLO_SKY_APP    // 必须大写，是否启用 hello_sky_app 这个模块
	bool "hello_sky_app"  
	default n     // 默认不打开
	---help---    // help是帮助说明，给人看的，不影响编译逻辑
		Enable support for hello_sky_app
```

作用：给 `hello_sky_app` 创建一个“编译开关”，让 PX4 的配置系统知道这个应用是否应该被编译进去。

## 4. 构建（仿真中）

```bash
make px4_sitl_default boardconfig
在 examples 中启用 hello_sky_app，保存退出。

make px4_sitl_default none

在pxh>中
    输入:
        hello_sky_app
    输出：
        pxh> hello_sky_app
        INFO  [hello_sky_app] Hello Sky!
```

