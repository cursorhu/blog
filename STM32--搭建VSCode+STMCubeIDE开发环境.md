# STM32--搭建VSCode+STMCubeIDE开发环境

## 用STM32CubeIDE创建工程

## 用VSCode编辑代码

### 配置.vscode使能tab补全

Stm32的HAL库默认是没有被VSCode的C/C++ intelligence检测到，自动补全功能不完整，例如HAL_UART_XXX不能tab补全到HAL_UART_Transmit，这个API定义在Drivers/Drivers/STM32FXXX_HAL_Driver/Inc里，C/C++ intelligence没有检测到这个路径，因此需要配置C/C++ intelligence的c_cpp_properties.json, 添加include和defines。

（1）打开STM32项目

注意：要配置哪个STM32项目就VSCode打开哪个目录，不要打开包括多个STM32项目的workspace，不然配置的.vscode是针对workspace目录的，不会对各项目生效。

比如以下workspace有几个STM32CubeIDE创建的项目，VSCode应该打开具体的项目serial-test-isr再配置该项目的.vscode

```
PS C:\Users\cursorhu\STM32CubeIDE\workspace_1.15.0> ls

    目录: C:\Users\cursorhu\STM32CubeIDE\workspace_1.15.0

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2024/4/29     10:03                .metadata
d-----         2024/4/29     10:34                .vscode
d-----         2024/4/24     19:45                led-test
d-----         2024/4/25     11:25                serial-test
d-----         2024/4/29     10:06                serial-test-isr


PS C:\Users\cursorhu\STM32CubeIDE\workspace_1.15.0> cd .\serial-test-isr\
PS C:\Users\cursorhu\STM32CubeIDE\workspace_1.15.0\serial-test-isr> ls

    目录: C:\Users\cursorhu\STM32CubeIDE\workspace_1.15.0\serial-test-isr

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2024/4/28     22:33                .settings
d-----         2024/4/29     10:06                .vscode
d-----         2024/4/28     16:26                Core
d-----         2024/4/28     17:51                Debug
d-----         2024/4/28     16:26                Drivers
-a----         2024/4/28     16:39          25210 .cproject
-a----         2024/4/28     16:39           8275 .mxproject
-a----         2024/4/28     16:30           1221 .project
-a----         2024/4/28     17:56          10224 serial-test-isr Debug.launch
-a----         2024/4/28     16:39           2975 serial-test-isr.ioc
-a----         2024/4/28     16:39           5306 STM32F072C8TX_FLASH.ld
```

（2）配置.vscode

VSCode左下角setting -> Command Palette -> 搜索: C/C++ Edit Configurations (UI) 或者 (JSON)

C/C++ Edit Configurations (UI) ：

在Include path添加HAL库定义的路径：这里直接用**递归搜索，类似.gitignore的语法，不需要指定到具体的Drivers/STM32FXXX_HAL_Driver/Inc路径。

![image-20240429104934058](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404291104878.png)

C/C++ Edit Configurations (JSON)方式的配置，UI配置也会反应在此 ：

```
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**",
                "Drivers/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "cStandard": "c17",
            "cppStandard": "gnu++17",
            "intelliSenseMode": "${default}"
        }
    ],
    "version": 4
}
```

（2）VSCode调用STM32 HAL 函数自动补全

现在可以tab补全STM32 HAL的所有库函数，可以彻底抛弃STM32CubeIDE的编辑界面：)

![image-20240429105530310](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404291104289.png)

## 用STM32CubeIDE和J-link调试和烧录