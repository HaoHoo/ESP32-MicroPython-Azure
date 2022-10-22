ESP32-3：把ESP32板和传感器硬件连起来



​    【虽然本文的内容基本相对独立，但仍建议先阅读之前的 [ESP32-1]() 和 [ESP32-2]() 】

​    板子准备好了，VS Code也准备好了，不搞点啥感觉哪都不得劲～ 

​    翻了一下，两年前手动给娃攒一套机器人考试器材的时候，还真留了一个温湿度传感器，翻出来一看，是DHT11。啊这…之前模拟器用的都是DHT22啊…能用么？



#### 准备较便宜的温湿度模块

​    查了下DHT11和DHT22引脚定义是完全一样的，只不过DHT的温湿度范围小于DHT22，并且只支持整数数据，精度不如DHT22带有一位小数。那就开整吧？找块面包板，插上插上～

![image-20221003180601772](../main/assets/image-20221003180601772.png)

​    上图的传感器DHT11/DHT22引脚都是相同的，第一脚是电源（VCC），可使用3.3-5.5V，如果接线较长，建议使用5V。因为我就插面包板，所以就接到板子上3V的引脚了；第二脚是数据（DATA），传感器已经内置处理，将从该引脚输出温湿度数据。实际使用中该引脚和VCC之间还需要一个10千欧的上拉电阻，使得数据引脚保持高电平。我把这个引脚接到了ESP32板子的第19引脚，一个可以定义为GPIO的引脚；第三脚一般空置（NC）；第四脚接地（GND），我直接接到板子上的GND引脚了。

​    我用的这个传感器已经焊在了一块小接口板上，引出了三个引脚，也不用再使用上拉电阻。



#### 开整代码

​    接下来应该找DHT11的示例MicroPython代码了，等等，还记得之前我们用 help('modules') 看到的输出吗？dht实际已经在系统里提供了支持，所以可以直接使用不用再写代码了～这简直太省事了。不过看代码是学习的重要方式不是吗？所以把 dht.py 也贴出来看看吧。

```python
# DHT11/DHT22 driver for MicroPython on ESP8266
# MIT license; Copyright (c) 2016 Damien P. George

try:
    from esp import dht_readinto
except:
    from pyb import dht_readinto


class DHTBase:
    def __init__(self, pin):
        self.pin = pin
        self.buf = bytearray(5)

    def measure(self):
        buf = self.buf
        dht_readinto(self.pin, buf)
        if (buf[0] + buf[1] + buf[2] + buf[3]) & 0xFF != buf[4]:
            raise Exception("checksum error")


class DHT11(DHTBase):
    def humidity(self):
        return self.buf[0]

    def temperature(self):
        return self.buf[2]


class DHT22(DHTBase):
    def humidity(self):
        return (self.buf[0] << 8 | self.buf[1]) * 0.1

    def temperature(self):
        t = ((self.buf[2] & 0x7F) << 8 | self.buf[3]) * 0.1
        if self.buf[2] & 0x80:
            t = -t
        return t
```

​    从代码可以看到，MicroPython自带的 dht 库支持DHT11和DHT22模块，代码基于 esp 和 pyb 库，通过 measure() 读取引脚，然后将数据存放到缓冲。通过 humidity() 和 temperature() 从缓冲提取数据并转换。这样我们就压根不用写这部分的代码了。可以看到MicroPython文档中的示例：

​    http://micropython.com.cn/en/latet/esp32/quickref.html#dht-driver

​    也因此，我的 main.py 代码简单如下：

```python
# main.py
import machine
import dht
import time

led=machine.Pin(22, machine.Pin.OUT)
d=dht.DHT11(machine.Pin(19))
while True:
    led.off()
    d.measure()
    print("temperature: ", d.temperature())
    print("humidity: ", d.humidity())
    led.on()
    time.sleep(5)
```

​    我使用了一个循环来定时输出传感器读取的温度和湿度。首先导入DHT库，然后就简单的使用三个函数调用获取温湿度数据即可。在连接到的REPL终端窗口可以看到输出：

```powershell
PS C:\ESP\code\micropython\esp32-lab> cli.exe -p COM3 repl

>>>
temperature:  32
humidity:  64   
temperature:  32
humidity:  64   
temperature:  32
humidity:  64
```

​    在调试代码的时候，开始还遇到了奇怪的报错，类似

```
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
File "dht.py", line 24, in measure
OSError: [Errno 116] ETIMEDOUT
```

​    因为看到超时，怀疑是线没接好（因为我都没有焊上引脚）。用万用表一量，果然没电压。接好就正常了。



#### 很贵及更贵的气压温湿度模块

​    不翻不知道，一翻才发现以前还买了BMP280，博世的这个传感器支持气压和温度，淘宝的板子大概几块钱，还翻到一块BME280，因为除了气压温度，还支持湿度。

![image-20221004194752391](../main/assets/image-20221004194752391.png)

​    这个板子就贵多了，好像是几十块。这两个传感器板子都是通过I2C来通信的，板子本身通用，引脚也相同，区别在于焊上的是BMP280还是BME280。搜一下有不少MicroPython的驱动库可用，例如以下的：

```python
"""
https://github.com/neliogodoi/MicroPython-BME280
Version: 0.1.0 @ 2018/06/21
"""

from time import sleep, sleep_us
from ustruct import unpack, unpack_from
from array import array

BME280_I2CADDR = 0x76

class BME280(object):

    def __init__(self,
                 temp_mode=2,
                 pres_mode=5,
                 humi_mode=1,
                 temp_scale='C',
                 iir=4,
                 address=BME280_I2CADDR,
                 i2c=None):
        osamples = [0, 1, 2, 3, 4, 5]
        msg_error = 'Unexpected {} operating mode value {0}.'
        if temp_mode not in osamples:
            raise ValueError(msg_error.format("temperature", temp_mode))
        self.temp_mode = temp_mode
        if pres_mode not in osamples:
            raise ValueError(msg_error.format("pressure", pres_mode))
        self.pres_mode = pres_mode
        if humi_mode not in osamples:
            raise ValueError(msg_error.format("humidity", humi_mode))
        self.humi_mode = humi_mode
        msg_error = 'Unexpected low pass IIR filter setting value {0}.'
        if iir not in [0, 1, 2, 3, 4]:
            raise ValueError(msg_error.format(iir))
        self.iir = iir
        msg_error = 'Unexpected temperature scale value {0}.'
        if temp_scale not in ['C', 'F', 'K']:
            raise ValueError(msg_error.format(temp_scale))
        self.temp_scale = temp_scale
        del msg_error
        self.address = address
        self.i2c = i2c
        dig_88_a1 = self.i2c.readfrom_mem(self.address, 0x88, 26)
        dig_e1_e7 = self.i2c.readfrom_mem(self.address, 0xE1, 7)
        self.dig_T1, \
        self.dig_T2, \
        self.dig_T3, \
        self.dig_P1, \
        self.dig_P2, \
        self.dig_P3, \
        self.dig_P4, \
        self.dig_P5, \
        self.dig_P6, \
        self.dig_P7, \
        self.dig_P8, \
        self.dig_P9, \
        _, \
        self.dig_H1 = unpack("<HhhHhhhhhhhhBB", dig_88_a1)
        self.dig_H2, self.dig_H3 = unpack("<hB", dig_e1_e7)
        e4_sign = unpack_from("<b", dig_e1_e7, 3)[0]
        self.dig_H4 = (e4_sign << 4) | (dig_e1_e7[4] & 0xF)
        e6_sign = unpack_from("<b", dig_e1_e7, 5)[0]
        self.dig_H5 = (e6_sign << 4) | (dig_e1_e7[4] >> 4)
        self.dig_H6 = unpack_from("<b", dig_e1_e7, 6)[0]
        self.i2c.writeto_mem(
            self.address,
            0xF4,
            bytearray([0x24]))
        sleep(0.002)
        self.t_fine = 0
        self._l1_barray = bytearray(1)
        self._l8_barray = bytearray(8)
        self._l3_resultarray = array("i", [0, 0, 0])
        self._l1_barray[0] = self.iir << 2
        self.i2c.writeto_mem(self.address, 0xF5, self._l1_barray)
        sleep(0.002)
        self._l1_barray[0] = self.humi_mode
        self.i2c.writeto_mem(self.address, 0xF2, self._l1_barray)

    def read_raw_data(self, result):
        self._l1_barray[0] = (self.pres_mode << 5 | self.temp_mode << 2 | 1)
        self.i2c.writeto_mem(self.address, 0xF4, self._l1_barray)
        osamples_1_16 = [1, 2, 3, 4, 5]
        sleep_time = 1250
        if self.temp_mode in osamples_1_16:
            sleep_time += 2300*(1 << self.temp_mode)
        if self.pres_mode in osamples_1_16:
            sleep_time += 575 + (2300*(1 << self.pres_mode))
        if self.humi_mode in osamples_1_16:
            sleep_time += 575 + (2300*(1 << self.humi_mode))
        sleep_us(sleep_time)
        while (unpack('<H', self.i2c.readfrom_mem(
                                self.address, 0xF3, 2))[0] & 0x08):
            sleep(0.001)
        self.i2c.readfrom_mem_into(self.address, 0xF7, self._l8_barray)
        readout = self._l8_barray
        raw_press = ((readout[0] << 16) | (readout[1] << 8) | readout[2]) >> 4
        raw_temp = ((readout[3] << 16) | (readout[4] << 8) | readout[5]) >> 4
        raw_hum = (readout[6] << 8) | readout[7]
        result[0] = raw_temp
        result[1] = raw_press
        result[2] = raw_hum

    def read_compensated_data(self, result=None):
        self.read_raw_data(self._l3_resultarray)
        raw_temp, raw_press, raw_hum = self._l3_resultarray
        var1 = ((raw_temp >> 3) - (self.dig_T1 << 1)) * (self.dig_T2 >> 11)
        var2 = (raw_temp >> 4) - self.dig_T1
        var2 = var2 * ((raw_temp >> 4) - self.dig_T1)
        var2 = ((var2 >> 12) * self.dig_T3) >> 14
        self.t_fine = var1 + var2
        temp = (self.t_fine * 5 + 128) >> 8
        var1 = self.t_fine - 128000
        var2 = var1 * var1 * self.dig_P6
        var2 = var2 + ((var1 * self.dig_P5) << 17)
        var2 = var2 + (self.dig_P4 << 35)
        var1 = (((var1 * var1 * self.dig_P3) >> 8) +
                ((var1 * self.dig_P2) << 12))
        var1 = (((1 << 47) + var1) * self.dig_P1) >> 33
        if var1 == 0:
            pressure = 0
        else:
            p = 1048576 - raw_press
            p = (((p << 31) - var2) * 3125) // var1
            var1 = (self.dig_P9 * (p >> 13) * (p >> 13)) >> 25
            var2 = (self.dig_P8 * p) >> 19
            pressure = ((p + var1 + var2) >> 8) + (self.dig_P7 << 4)
        h = self.t_fine - 76800
        h = (((((raw_hum << 14) - (self.dig_H4 << 20) -
                (self.dig_H5 * h)) + 16384)
              >> 15) * (((((((h * self.dig_H6) >> 10) *
                            (((h * self.dig_H3) >> 11) + 32768)) >> 10) +
                          2097152) * self.dig_H2 + 8192) >> 14))
        h = h - (((((h >> 15) * (h >> 15)) >> 7) * self.dig_H1) >> 4)
        h = 0 if h < 0 else h
        h = 419430400 if h > 419430400 else h
        humidity = h >> 12
        if result:
            result[0] = temp
            result[1] = pressure
            result[2] = humidity
            return result
        return array("i", (temp, pressure, humidity))

    @property
    def values(self):
        temp, pres, humi = self.read_compensated_data()
        temp = temp/100
        if self.temp_scale == 'F':
            temp = 32 + (temp*1.8)
        elif self.temp_scale == 'K':
            temp = temp + 273.15
        pres = pres/256
        humi = humi/1024
        return (temp, pres, humi)

    @property
    def formated_values(self):
        t, p, h = self.values
        temp = "{} "+self.temp_scale
        return (temp.format(t), "{} Pa".format(p), "{} %".format(h))

    @property
    def temperature(self):
        t, _, _ = self.values
        return t

    @property
    def pressure(self):
        _, p, _ = self.values
        return p

    @property
    def pressure_precision(self):
        _, p, _ = self.read_compensated_data()
        pi = float(p // 256)
        pd = (p % 256)/256
        return (pi, pd)

    @property
    def humidity(self):
        _, _, h = self.values
        return h
    
    @property
    def altitude(self, pressure_sea_level=1013.25):
        pi, pd = self.pressure_precision
        return 44330*(1-((float(pi+pd)/100)/pressure_sea_level)**(1/5.255))
```

​    如果你找到BMP280/BME280的官方说明，就比较容易理解代码里接受I2C总线数据之后的处理过程，此处略去不表。在这个过程中我遇到了两个问题，第一个是搞错了ESP32板子的I2C，接到了两个GPIO引脚上，后来不知道哪里找了个Pin Out，换到I2C SCL和I2C SDA上了。第二个是最后传感器上读到的数据全是0…等有闲工夫闲钱换个板子再试吧…调用这个BME280的代码倒是很简单。

```python
import machine
import bme280

i2c = machine.SoftI2C(sda=machine.Pin(2), scl=machine.Pin(15))
bme = bme280.BME280(i2c=i2c)
print(bme.formated_values)
#这里同时显示温度气压湿度，需要显示其他的参考驱动库BME280的属性
```

​    选择这个驱动代码也是因为提供的属性多一点，懒得自己计算了。

​    BME280确实小贵……毕竟在大约3x3毫米的大小上搞定了气压温湿度三个传感，DHT11/DHT22确实大了点，所以打算回头试试AHT20+BMP280的，加一起大约不到十块，估计一个板子上有两个传感器，用了两个I2C地址，等有机会试试吧。



#### 搞个温湿度显示开关LED的网页彩蛋

​    温度湿度气压什么的，倒是很容易发送到REPL的终端查看，不过既然这块ESP32的板子有WiFi，通过网络查看岂不美哉？

​    先在 boot.py 里配置连接到家里的AP，这样同一网络都能读取了。

```python
# boot.py - - runs on boot-up
import network
import usocket as socket
from time import sleep

import esp
esp.osdebug(None)

import gc
gc.collect()

ssid = '[你AP的SSID]'
password = '[你AP的密码]'

station = network.WLAN(network.STA_IF)
station.active(True)
station.connect(ssid, password)
while station.isconnected() == False:
  pass
print('Connection successful')
print(station.ifconfig())
```

​    把代码里的ssid和password换成你的，代码就会在板子启动时连接到AP，并在REPL终端显示连接的IP。当然，如果你把ESP32板子设置为AP模式，手动去连也可以。

​    看到 boot.py 里引入的 usocket 库了吗？接下来就是重点了，我们要使用ESP32上的MicroPython的Socket，构建一个最简单的Web服务，使其能够显示DHT11的温湿度，并且可以开关板载的LED。代码如下：

```python
import machine
import dht

led = machine.Pin(22, machine.Pin.OUT)
dhd = dht.DHT11(machine.Pin(19))

led_state = "OFF"

def web_page():
  dhd.measure()
  html = """<!DOCTYPE html><html><head><title>ESP32 DHT11</title><meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" href="data:,">
  <style>
    body { text-align: center; font-family: "Trebuchet MS", Arial;}
    table { border-collapse: collapse; width:35%; margin-left:auto; margin-right:auto; }
    th { padding: 8px; background-color: #0043af; color: white; }
    tr { border: 1px solid #ddd; padding: 8px; }
    tr:hover { background-color: #bcbcbc; }
    td { border: none; padding: 8px; text-align: center; }
    .sensor { color:white; font-weight: bold; background-color: #bcbcbc; padding: 2px; }
    .button1 { background-color: #4488ff; border: none; color: white; padding: 8px; text-align: center; font-size: 12px; margin: 2px; }
    .button2 { background-color: #000000; }
  </style></head>
    <body>
     <h2>DHT11 on ESP32</h2>
     <table>
      <tr><th>Measurement</th><th>Values</span></th></tr>
      <tr><td>Temperature</td><td><span class="sensor">""" + str(dhd.temperature()) + """</span> C</td></tr>
      <tr><td>   Humidity</td><td><span class="sensor">""" + str(dhd.humidity()) + """</span> %</td></tr>
     </table>
     <p></p>
     <h2>LED on ESP32</h2>
     <p>LED state: <strong><span class="sensor">""" + led_state + """</span></strong></p>
     <p><a href=\"?led_2_on\"><button class="button1" width=60>LED ON</button></a> 
     <a href=\"?led_2_off\"><button class="button1 button2" width=60>LED OFF</button></a></p>
    </body>
  </html>"""
  return html

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('', 80))
s.listen(5)

while True:
  try:
    if gc.mem_free() < 102000: 
        gc.collect()
    conn, addr = s.accept()
    conn.settimeout(3.0)
    print('Got a connection from %s' % str(addr))
    request = conn.recv(1024)
    conn.settimeout(None)
    request = str(request)
    print('Content = %s' % request)
    led_on = request.find('/?led_2_on')
    led_off = request.find('/?led_2_off')
    if led_on == 6:
        print('LED ON -> GPIO22')
        led_state = "ON"
        led.on()
    if led_off == 6:
        print('LED OFF -> GPIO22')
        led_state = "OFF"
        led.off()
    response = web_page()
    conn.send('HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n')
    conn.sendall(response)
    conn.close()
  except OSError as e:
    conn.close()
    print('Connection closed')
```

​    在 main.py 程序里，首先对板载的LED引脚和连接的DHT11传感器引脚作出定义。然后构造了一个 web_page() 函数用于浏览器中显示的HTML代码，返回的就是代码本身。

​    MicroPython的Socket协议可以使用TCP协议 `IPPROTO_TCP` 和UDP协议 `IPPROTO_UDP` ，选择Socket类型 `SOCK_STREAM` 或 `SOCK_DGRAM` 时会自动匹配协议到TCP或UDP。因为是用做服务器，所以要把Socket绑定到80端口上，然后进行侦听。有关Socket的更多说明，可以参考MicroPython官方文档：https://docs.micropython.org/en/latest/library/socket.html

​    打开浏览器输入ESP32的IP地址，例如“http://192.168.4.1”，ESP32就会收到浏览器发来的HTTP GET请求，可使用 web_page() 函数构造的html文档发送给浏览器显示。需要说明的是，在控制LED的时候，由HTML页面的按钮提交出带有特定字串的请求，这时代码滤出开关LED的字串，然后处理不同的指令。

![image-20221004153750725](../main/assets/image-20221004153750725.png)

​    最终，我们就可以将温湿度显示在浏览器中，并通过浏览器开关LED。再往后，打算搞搞ESP32板子使用网络把温湿度数据发给Azure IoT了。

​    话说今年国庆突然高温预警和降温预警一起来，真是难顶啊…于是晚上重温了《后天》…
