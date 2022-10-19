ESP32-1:搞了块能跑Python的小板子



​    我错了…明明十月十一月那么多事情，可是一放假看到前几周买的一块板子，就有点手痒痒了…

​    之前一直使用模拟器跑小板子，然后连Azure的IoT Hub来发设备到云（D2C）消息，通过消息可以收集类似温湿度数据等等…等我有空写一篇模拟器版的文吧。

​    这是我在淘宝买的一块据说支持MicroPython的板子，跟一堆传感器和433MHz射频、38KHz红外线一起买的，好像总共也没超过疯狂星期四的V我50…

​    我记得放进购物车的时候是挑的支持wifi和蓝牙的，但是在购物车里躺了太久…既然在手边了，就忍不住插上看看。看了眼，板子上有块CH340，应该可以插上USB直接串口通信。MAC下面做这件事情还是嫌麻烦，所以折腾了下，我还是选择开了个Windows的虚拟机。

​    连上虚拟机，在设备管理器里直接看COM端口，果然出现了个COM3。找了个putty，配置波特率115200，连上去，诶～怎么不太对啊，没有我期待看到的Python交互式界面啊…固件不对吧…



#### 给ESP32刷MicroPython固件

​    看来得刷个固件……我搜了下，找到个ESP8266的刷固件文章，因为印象里ESP8266是很常见的带WiFi的板子，所以就直接找到ESP8266上支持MicroPython的文章参考了。

​    此处略去如何刷固件两千字……因为刷完之后，板子连接上去乱码了……（为了节省篇幅，也节省您的时间，刷机过程放后面吧）

​    这肯定有问题啊，于是我睁大眼睛看了看芯片……不是ESP8266，是ESP32……

​    于是，从头再来一遍。首先，找一下ESP对应的固件。在micropython.org站点的download页面，有好多不同的板子，我眼睛都看花了，还好页面支持过滤，选择ESP32，很快就找到了ESP32固件的下载页面。当然，您也可以直达：https://micropython.org/download/esp32 下载。

​    固件下好了，怎么刷呢？最简单的方式还是使用esptool这个工具。连找都不用找的，直接在Python下运行：

```
pip install esptool
```

​    这就能装好了。在Windows下面，该工具会自动生成esptool.py.exe在Python安装目录下的Scripts目录里。直接运行如下命令行就可以尝试连接ESP32的板子：

```
C:\Program Files\Python310\Scripts>esptool.py read_mac
esptool.py v4.3
Found 2 serial ports
Serial port COM3
Connecting....
Detecting chip type... Unsupported detection protocol, switching and trying again...
Connecting....
Detecting chip type... ESP32
Chip is ESP32-D0WDQ6 (revision v1.0)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
Crystal is 40MHz
MAC: 78:e3:6d:19:39:5c
Uploading stub...
Running stub...
Stub running...
MAC: 78:e3:6d:19:39:5c
Hard resetting via RTS pin...
```

​    可以看到，这个板子型号ESP32-D0WDQ6 (revision v1.0)，带有WiF i、蓝牙，双核240MHz的微处理器，晶振提供的40MHz频率，以及网络MAC地址。这个地址是作为客户端模式（STA，Station），还有个MAC地址是给AP模式（AP，Access Point）。这两个模式可以通过定义network库的不同接口来操作。

​    接下来就是擦原有固件了，可惜我忘了给原来的固件截个图，等啥时候研究再刷回去吧。要擦除固件，直接运行如下命令行：

```
C:\Program Files\Python310\Scripts>esptool.py -p com3 erase_flash
esptool.py v4.3
Serial port com3
Connecting...
Failed to get PID of a device on com3, using standard reset sequence.
.
Detecting chip type... Unsupported detection protocol, switching and trying again...
Connecting...
Failed to get PID of a device on com3, using standard reset sequence.
..
Detecting chip type... ESP32
Chip is ESP32-D0WDQ6 (revision v1.0)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
Crystal is 40MHz
MAC: 78:e3:6d:19:39:5c
Uploading stub...
Running stub...
Stub running...
Erasing flash (this may take a while)...
Chip erase completed successfully in 13.0s
Hard resetting via RTS pin...
```

​    擦除Flash需要一点时间，完成后也是一样，会硬件重置板子。擦完了，就可以刷固件了。执行如下命令行，需要指定芯片类型（--chip esp32），建议使用高一点的波特率（--baud 460800），会快一点，如果报错再降低波特率。

```
C:\Program Files\Python310\Scripts>esptool.py --chip esp32 -p com3 --baud 460800 write_flash -z 0x1000 c:\ESP8266\esp32-20220618-v1.19.1.bin
esptool.py v4.3
Serial port com3
Connecting...
Failed to get PID of a device on com3, using standard reset sequence.
.
Chip is ESP32-D0WDQ6 (revision v1.0)
Features: WiFi, BT, Dual Core, 240MHz, VRef calibration in efuse, Coding Scheme None
Crystal is 40MHz
MAC: 78:e3:6d:19:39:5c
Uploading stub...
Running stub...
Stub running...
Changing baud rate to 460800
Changed.
Configuring flash size...
Flash will be erased from 0x00001000 to 0x0017efff...
Compressed 1560976 bytes to 1029132...
Wrote 1560976 bytes (1029132 compressed) at 0x00001000 in 24.3 seconds (effective 514.6 kbit/s)...
Hash of data verified.

Leaving...
Hard resetting via RTS pin...
```

​    等待一会，固件就刷到ESP32板子了，然后会自动重启。



#### 通过USB转换的串口访问ESP32 

   刷完之后，就可以通过putty使用USB转换的串口来连接这个板子了。连接串口的实际是MicroPython的REPL（即Python提示符）界面。

```python
MicroPython v1.19.1 on 2022-06-18; ESP32 module with ESP32
Type "help()" for more information.
>>>
```

​    输入help()可以查看帮助信息。

```python
>>> help()
Welcome to MicroPython on the ESP32!

For generic online docs please visit http://docs.micropython.org/

For access to the hardware use the 'machine' module:

import machine
pin12 = machine.Pin(12, machine.Pin.OUT)
pin12.value(1)
pin13 = machine.Pin(13, machine.Pin.IN, machine.Pin.PULL_UP)
print(pin13.value())
i2c = machine.I2C(scl=machine.Pin(21), sda=machine.Pin(22))
i2c.scan()
i2c.writeto(addr, b'1234')
i2c.readfrom(addr, 4)

Basic WiFi configuration:

import network
sta_if = network.WLAN(network.STA_IF); sta_if.active(True)
sta_if.scan()                             # Scan for available access points
sta_if.connect("<AP_name>", "<password>") # Connect to an AP
sta_if.isconnected()                      # Check for successful connection

Control commands:
  CTRL-A        -- on a blank line, enter raw REPL mode
  CTRL-B        -- on a blank line, enter normal REPL mode
  CTRL-C        -- interrupt a running program
  CTRL-D        -- on a blank line, do a soft reset of the board
  CTRL-E        -- on a blank line, enter paste mode

For further help on a specific object, type help(obj)
For a list of available modules, type help('modules')
```

​    在这里的信息给出了最基本的示例，例如设置引脚Pin的定义，设置I2C总线的定义，以及基本的网络设置等等。输入help('modules')可以看到自带的模块。

```python
>>> help('modules')
__main__          gc                ubluetooth        upysh
_boot             inisetup          ucollections      urandom
_onewire          math              ucryptolib        ure
_thread           micropython       uctypes           urequests
_uasyncio         neopixel          uerrno            uselect
_webrepl          network           uhashlib          usocket
apa106            ntptime           uheapq            ussl
btree             onewire           uio               ustruct
builtins          uarray            ujson             usys
cmath             uasyncio/__init__ umachine          utime
dht               uasyncio/core     umqtt/robust      utimeq
ds18x20           uasyncio/event    umqtt/simple      uwebsocket
esp               uasyncio/funcs    uos               uzlib
esp32             uasyncio/lock     upip              webrepl
flashbdev         uasyncio/stream   upip_utarfile     webrepl_setup
framebuf          ubinascii         uplatform         websocket_helper
Plus any modules on the filesystem
```

​    接下来试试连WiFi，先搞定AP模式。

```python
>>> import network
>>> ap_if=network.WLAN(network.AP_IF)
>>>
>>> ap_if.active(True)
True
```

​    这时用电脑扫描无线网络，就能找到名字像是ESP-xxxxxx的AP，xxxxxx是AP后半部分的MAC地址。我们加入这个网络，就能够跟ESP32通信了。默认AP的IP是192.168.4.1。



#### 通过浏览器访问ESP32

​    使用WebREPL库可创建一个基于Web Socket的REPL界面，这样就能够通过浏览器来访问了。    

```python
>>> import webrepl_setup
WebREPL daemon auto-start status: disabled

Would you like to (E)nable or (D)isable it running on boot?
(Empty line to quit)
>
```

​    可以设置启动时是否自动运行WebREPL。

```
>>> import webrepl
>>> webrepl.start(password='devpro')
WebREPL daemon started on ws://192.168.4.1:8266
Started webrepl in manual override mode

```

​    启动WebREPL可以加上密码。可以看到，WebREPL的守护进程已经在192.168.4.1的8266端口侦听了。那么怎么连接到这个端口呢？直接使用浏览器是不行的，因为WebREPL使用的不是HTTP协议，而是Web Socket，所以要找个小工具：webrepl_cli。这个工具可以在：https://github.com/micropython/webrepl 获得。下载代码包之后解压，直接用浏览器打开webrepl.html即可。

![image-20221001220538178](ESP32-1:搞了块能跑Python的小板子.assets/image-20221001220538178.png)

​    在页面的地址栏输入地址和端口，点击连接，即可连到ESP32的板载MicroPython。甚至可以通过这个工具页面上传下载文件，例如下图，下载了启动时默认运行的boot.py程序文件。

![image-20221001220651458](ESP32-1:搞了块能跑Python的小板子.assets/image-20221001220651458.png)

​    好了，先折腾到这里，板子准备好了，等空了接着折腾。