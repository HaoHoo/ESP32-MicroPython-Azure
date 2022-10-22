ESP32-2:从VS Code分发MicroPython代码



​    【尽管本篇基本独立，但仍建议先查阅 [ESP32-1]() 】

​    上一篇文，刷完固件之后，算是让ESP32板子连上网络跑起来了。可…

​    总不能一直使用交互式的方式在ESP32上写MicroPython的代码吧？虽然不是程序员，但VS Code还是经常用的，那能不能直接从VS Code编辑MicroPython代码然后发送到板子上运行呢？



#### VS Code开发环境准备

​    VS Code写Python的代码肯定大家都很熟悉了，但是MicroPython和标准的Python还是有差别的，如果希望在VS Code里继续享受代码智能和自动完成的好处，那自然再好不多。为了实现这个目的，我们需要安装一个micropy_cli的库 [[https://GitHub/BradenM/micropy-cli](https://github.com/BradenM/micropy-cli)]。运行如下命令行：

```
pip install micropy_cli
```

​     安装完毕之后，可以执行命令行看看ESP32板子相关的更新。

```
C:\Windows\system32>micropy stubs search esp32

MicroPy  Update Available!
MicroPy  Version v3.6.0 is now available
MicroPy  You can update via: pip install --upgrade micropy-cli


MicroPy  Searching Stub Repositories...

MicroPy  Results for esp32:
MicroPy  esp32-micropython-1.10.0
MicroPy  esp32-micropython-1.11.0
MicroPy  esp32-micropython-1.12.0
MicroPy  esp32-micropython-1.15.0
MicroPy  esp32-micropython-1.9.4
MicroPy  esp32-pycopy-1.11.0
MicroPy  esp32-pycopy-2.11.0.1
MicroPy  esp32-pycopy-2.11.0.5
MicroPy  esp32-pycopy-3.0.0
MicroPy  esp32_LoBo
MicroPy  esp32_LoBo-esp32_LoBo-3.2.24
MicroPy  esp32s2-micropython-1.15.0
```

​    顺手更新一下库。

```
pip install --upgrade micropy-cli
```

​    然后创建一个Stub，将需要的固件加上。

```
C:\Windows\system32>micropy stubs add esp32-micropython-1.12.0

MicroPy  Update Available!
MicroPy  Version v3.6.0 is now available
MicroPy  You can update via: pip install --upgrade micropy-cli


MicroPy  Adding esp32-micropython-1.12.0 to stubs

MicroPy  Resolving stub...
MicroPy  esp32-micropython-1.12.0: 100%|█████████████████████████████████████████████████████████| [10.5k/10.5k @ ?B/s]
MicroPy  Detected Firmware: micropython
MicroPy  Firmware not found locally, attempting to install it...

MicroPy  Resolving stub...
MicroPy  micropython: 100%|██████████████████████████████████████████████████████████████████████| [43.6k/43.6k @ ?B/s]
MicroPy  ✔ micropython firmware added!
MicroPy  ✔ esp32-micropython-1.12.0 added!
```

​    接下来为后续的尝试创建一个文件夹，然后就可以创建开发项目了。命令行转到这个文件夹下，运行如下命令。

```
C:\ESP\code\micropython\esp32-lab>micropy init

MicroPy  Update Available!
MicroPy  Version v3.6.0 is now available
MicroPy  You can update via: pip install --upgrade micropy-cli


MicroPy  Creating New Project
? Project Name esp32-lab
? Choose any Templates to Generate (Use arrow keys to move, <space> to select, <a> to toggle, <i> to invert)
   ● VSCode Settings for Autocompletion/Intellisense
   ○Pymakr Configurationn
   ● Pylint MicroPython Settings
   ● Git Ignore Template
 » ● main.py & boot.py files
? Which stubs would you like to use? (Use arrow keys to move, <space> to select, <a> to toggle, <i> to invert)
 » ● esp32-micropython-1.12.0

MicroPy  Initiating esp32-lab
MicroPy  Stubs: esp32-micropython-1.12.0

MicroPy  Rendering Templates
MicroPy  Populating Stub Info...
MicroPy  Vscode File Generated!
MicroPy  Pylint File Generated!
MicroPy  Vsextensions File Generated!
MicroPy  Main File Generated!
MicroPy  Boot File Generated!
MicroPy  Gitignore File Generated!
MicroPy  ✔ Stubs Injected!
MicroPy  ✔ Project Created!

MicroPy  Created esp32-lab at ./.
```

​    在这个项目初始化过程里，我选择了VS Code自动完成和代码智能，以及适用于MicroPython的Pylint代码检查。实际上初始化过程会创建包含很多必须引用代码的文件夹，然后重定向给VS Code使用。在项目目录下的 .vscode 子目录里，就可以找到VS Code的配置文件。

```json
{
    "python.linting.enabled": true,
    "python.jediEnabled": false,

    // Loaded Stubs:  esp32-micropython-1.12.0 
    "python.autoComplete.extraPaths": [".micropy\\BradenM-micropy-stubs-adb6f18\\frozen", ".micropy\\BradenM-micropy-stubs-4f5a52a\\frozen", ".micropy\\BradenM-micropy-stubs-adb6f18\\stubs", ".micropy\\esp32-lab"],
    "python.autoComplete.typeshedPaths":  [".micropy\\BradenM-micropy-stubs-adb6f18\\frozen", ".micropy\\BradenM-micropy-stubs-4f5a52a\\frozen", ".micropy\\BradenM-micropy-stubs-adb6f18\\stubs", ".micropy\\esp32-lab"],
    "python.analysis.typeshedPaths":  [".micropy\\BradenM-micropy-stubs-adb6f18\\frozen", ".micropy\\BradenM-micropy-stubs-4f5a52a\\frozen", ".micropy\\BradenM-micropy-stubs-adb6f18\\stubs", ".micropy\\esp32-lab"],

    "python.linting.pylintEnabled": true
}
```

​    同样，在项目目录下的 .pylintrc 文件中，对导入例如 sys 进行了修改，也禁用了一些检查项，避免报错和运行错误。

```
[MASTER]
# Loaded Stubs:  esp32-micropython-1.12.0 
init-hook='import sys;sys.path.insert(1, ".micropy\BradenM-micropy-stubs-adb6f18\frozen");sys.path.insert(1, ".micropy\BradenM-micropy-stubs-4f5a52a\frozen");sys.path.insert(1, ".micropy\BradenM-micropy-stubs-adb6f18\stubs");sys.path.insert(1, ".micropy\esp32-lab");'

[MESSAGES CONTROL]
# Only show warnings with the listed confidence levels. Leave empty to show
# all. Valid levels: HIGH, INFERENCE, INFERENCE_FAILURE, UNDEFINED.
confidence=

# Disable the message, report, category or checker with the given id(s). You
# can either give multiple identifiers separated by comma (,) or put this
# option multiple times (only on the command line, not in the configuration
# file where it should appear only once). You can also use "--disable=all" to
# disable everything first and then reenable specific checks. For example, if
# you want to run only the similarities checker, you can use "--disable=all
# --enable=similarities". If you want to run only the classes checker, but have
# no Warning level messages displayed, use "--disable=all --enable=classes
# --disable=W".

disable = missing-docstring, line-too-long, trailing-newlines, broad-except, logging-format-interpolation, invalid-name, empty-docstring,
        no-method-argument, assignment-from-no-return, too-many-function-args, unexpected-keyword-arg
        # the 2nd  line deals with the limited information in the generated stubs.
```

​     初始化的时候，我还勾选了gitignore的选项，因为暂时没打算提交代码到github，所以看看就算了。

​     真正我们要写的代码是在 src 文件夹里的 boot.py 和 main.py 。



#### VS Code 代码上载

​    写了代码还得发到ESP32的板子上啊，继续折腾VS Code。搜了一下，通过VS Code直接上载代码到ESP32的板子有几种不同的做法：

##### 0、官方Espressif IDF扩展

​    最为全面，很多独有的支持例如通过DFU、UART、JTag等不同方式刷代码。还可以加密、擦除Flash等等，感觉有点小复杂，再加上淘宝的板子貌似不是官方的型号，所以果断放一边了。

##### 1、使用MicroPython IDE扩展集成

​    Micropython IDE，支持刷ESP8266的板子，ESP32板子使用esptool.py。需要安装Ampy和rshell。

```
pip install adafruit-ampy
```

​    安装之后就可以使用ampy命令行来操作板子上的文件。

```
ampy.exe -p COM3 ls
>
/boot.py
```

​    以下命令行就可以将写好的main.py通过USB转换的COM3端口分发到板子上。

```
ampy.exe -p COM3 put .\main.py
ampy.exe -p COM3 ls
>
/boot.py
/main.py
```

​    基本就是现在做的手动工作集成到VS Code里了。

##### 2、使用Pymakr扩展集成Pycom

​    这个方式需要依赖Node.js。需要先安装node.js，然后在VS Code中查找并安装Pymakr扩展。之后修改pymakr.json配置文件，按照板子实际使用端口修改。

```json
{
  "address": "COM3",
  "username": "micro",
  "password": "python",
  "sync_folder": "/flash",
  "open_on_start": false,
  "sync_file_types": "py,txt,log,json,xml,html,js,css,mpy",
  "ctrl_c_on_connect": false,
}
```

​     完成之后就可以通过底下状态栏里的PyCom运行或上下载代码到板子上了。因为需要安装node.js，先放一边了（所以之前用micropy初始化项目的时候也没勾选Pymakr）。

##### 3、使用RT-Thread MicroPython扩展

​     看了几个方法之后，感觉这个扩展最简单，所以直接安装试试。安装完毕之后，就能在VS Code底下状态栏左侧看到一个插头样式的图标，点击图标就可以选择连接端口。我的ESP32板子通过USB连接在COM3上，所以选择COM3。
![image-20221003121726492](../main/assets/image-20221003121726492.png)

​    这时候扩展会在PowerShell终端运行自己的脚本初始化，然后连接到ESP32的REPL上。所以VS Code需要使用PowerShell作为默认终端类型，保证成功运行初始化。

![image-20221003121938145](../main/assets/image-20221003121938145.png)

​    我首先修改了板子启动时会运行的 boot.py 程序。

```python
# boot.py - - runs on boot-up
import network

ap_if = network.WLAN(network.AP_IF) 
ap_if.active(True)
```

​    这段代码将会把ESP32板子启动到AP模式。保存后右键点击 boot.py 然后选择“下载该文件/文件夹到设备上”。

![image-20221003122203725](../main/assets/image-20221003122203725.png)

​    然后可以在COM3终端里按下 Ctrl+D 软重置ESP32板子，马上就能够在无线网络中看到这个AP的信号。

<img src="../main/assets/image-20221003122302587.png" alt="image-20221003122302587" style="zoom:50%;" />

​    接下来写一段非常简单的代码。仔细看了下这块ESP32板子，板载了一颗贴片LED。因为没有任何引脚说明，只能自己看线路了。看着LED是连在 Pin 22 上的，于是就按照这个写几句让LED闪烁吧。 

```python
# main.py
import machine
import time

led = machine.Pin(22, machine.Pin.OUT)
while True:
    time.sleep(0.75)
    led.on()
    time.sleep(0.25)
    led.off()
```

​    保存，然后右键点击，选择“直接在设备上运行该 MicroPython 文件”。

![image-20221003123221549](../main/assets/image-20221003123221549.png)

​    直接运行需要先使用扩展创建项目，否则无法成功。可是我之前不是已经使用micropy创建了项目吗？那就直接在现有项目目录里继续创建项目…野蛮粗暴…

​    点击VS Code状态栏底下的方框加号图标，选择新建项目，然后选择之前已经创建的项目。这个扩展会修改已经在 .vscode 目录下的settings.json文件，例如加入运行代码和同步文件的图标，（部分）如下所示：

```json
    "MicroPython.executeButton": [
        {
            "text": "▶",
            "tooltip": "运行",
            "alignment": "left",
            "command": "extension.executeFile",
            "priority": 3.5
        }
    ],
    "MicroPython.syncButton": [
        {
            "text": "$(sync)",
            "tooltip": "同步",
            "alignment": "left",
            "command": "extension.execute",
            "priority": 4
        }
    ],
    "python.linting.pylintArgs": [
        "--init-hook",
        "import sys; sys.path.append('c:/Users/Hao Hu/.vscode/extensions/rt-thread.rt-thread-micropython-1.0.11/microExamples/code-completion')"
    ]
```

​    创建（或者叫更新？）项目完成后，就可以直接在ESP32板子上运行刚写的代码了。可以批量同步文件到板上，感到美中不足的是不能直接操作板子上的文件，例如批量删除啥的。

​    折腾软件的环境搞好了，接下来准备折腾硬件了。

