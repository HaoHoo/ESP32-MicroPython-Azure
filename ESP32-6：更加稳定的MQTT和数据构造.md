

ESP32-6：更加稳定的MQTT和数据构造



​    【本文的内容接续[ESP32-5]()，也建议先阅读之前的 [ESP32-1]()、[ESP32-2]()、[ESP32-3]()和[ESP32-4]() 】



##### 生成SAS令牌的第五次尝试

​    要把代码移植到MicroPython上，最主要就是尽量使用内置模块取代安装的，首先就是前面提到的hmac.py。贴上来：

```python
"""HMAC (Keyed-Hashing for Message Authentication) Python module.

Implements the HMAC algorithm as described by RFC 2104.
"""
    
#from _operator import _compare_digest as compare_digest
import gc
import uhashlib as _hashlib
gc.collect()
PendingDeprecationWarning = None
RuntimeWarning = None

trans_5C = bytes((x ^ 0x5C) for x in range(256))
trans_36 = bytes((x ^ 0x36) for x in range(256))

def translate(d, t):
    return bytes(t[x] for x in d)

# The size of the digests returned by HMAC depends on the underlying
# hashing module used.  Use digest_size from the instance of HMAC instead.
digest_size = None

class HMAC:
    """RFC 2104 HMAC class.  Also complies with RFC 4231.

    This supports the API for Cryptographic Hash Functions (PEP 247).
    """
    blocksize = 64  # 512-bit HMAC; can be changed in subclasses.

    def __init__(self, key, msg = None, digestmod = None):
        """Create a new HMAC object.

        key:       key for the keyed hash object.
        msg:       Initial input for the hash, if provided.
        digestmod: A module supporting PEP 247.  *OR*
                   A hashlib constructor returning a new hash object. *OR*
                   A hash name suitable for hashlib.new().
                   Defaults to hashlib.md5.
                   Implicit default to hashlib.md5 is deprecated and will be
                   removed in Python 3.6.

        Note: key and msg must be a bytes or bytearray objects.
        """

        if not isinstance(key, (bytes, bytearray)):
            raise TypeError("key: expected bytes or bytearray, but got %r" % type(key).__name__)

        if digestmod is None:
            digestmod = _hashlib.sha256

        if callable(digestmod):
            self.digest_cons = digestmod
        elif isinstance(digestmod, str):
            self.digest_cons = lambda d=b'': _hashlib.new(digestmod, d)
        else:
            self.digest_cons = lambda d=b'': digestmod.new(d)

        self.outer = self.digest_cons()
        self.inner = self.digest_cons()
        #self.digest_size = self.inner.digest_size

        if hasattr(self.inner, 'block_size'):
            blocksize = self.inner.block_size
            if blocksize < 16:
                blocksize = self.blocksize
        else:
            blocksize = self.blocksize

        # self.blocksize is the default blocksize. self.block_size is
        # effective block size as well as the public API attribute.
        self.block_size = blocksize

        if len(key) > blocksize:
            key = self.digest_cons(key).digest()

        key = key + bytes(blocksize - len(key))
        self.outer.update(translate(key, trans_5C))
        self.inner.update(translate(key, trans_36))
        if msg is not None:
            self.update(msg)

    @property
    def name(self):
        return "hmac-" + self.inner.name

    def update(self, msg):
        """Update this hashing object with the string msg.
        """
        self.inner.update(msg)

    def copy(self):
        """Return a separate copy of this hashing object.

        An update to this copy won't affect the original object.
        """
        # Call __new__ directly to avoid the expensive __init__.
        other = self.__class__.__new__(self.__class__)
        other.digest_cons = self.digest_cons
        other.digest_size = self.digest_size
        other.inner = self.inner.copy()
        other.outer = self.outer.copy()
        return other

    def _current(self):
        """Return a hash object for the current state.

        To be used only internally with digest() and hexdigest().
        """
        # h = self.outer.copy()
        self.outer.update(self.inner.digest())
        return self.outer

    def digest(self):
        """Return the hash value of this hashing object.

        This returns a string containing 8-bit data.  The object is
        not altered in any way by this function; you can continue
        updating the object after calling this function.
        """
        h = self._current()
        return h.digest()

    def hexdigest(self):
        """Like digest(), but returns a string of hexadecimal digits instead.
        """
        h = self._current()
        return h.hexdigest()

def new(key, msg = None, digestmod = None):
    """Create a new hashing object and return it.

    key: The starting key for the hash.
    msg: if available, will immediately be hashed into the object's starting
    state.

    You can now feed arbitrary strings into the object using its update()
    method, and can ask for the hash value at any time by calling its digest()
    method.
    """
    return HMAC(key, msg, digestmod)
    
```

​    接下来是MQTT模块的替换，要从之前用的paho.mqtt，替换成MicroPython自带的umqtt.simple。一开始我很担心ssl模块的替换，查了一下貌似ussl和ssl不大能直接换。后来发现umqtt很“simple”，因为我们的实验没有涉及到客户端证书，所以直接给参数“ssl=Ture”就可以了。

```python
import sys
import gc
gc.collect()

import ujson
from utime import time, sleep
from ubinascii import a2b_base64, b2a_base64
from uhashlib import sha256
from hmac import HMAC
from umqtt.simple import MQTTClient
gc.collect()

connection_string = 'HostName={IoT-Hub-Name}.azure-devices.net;DeviceId={MyDeviceName};SharedAccessKey={MySharedAccessKey=}'

unsafe = {
    '?': '%3F',
    ' ': '%20',
    '$': '%24',
    '%': '%25',
    '&': '%26',
    "\'": '%27',
    '/': '%2F',
    ':': '%3A',
    ';': '%3B',
    '+': '%2B',
    '=': '%3D',
    '@': '%40'
}

def encode_uri(string):
    ret = ''
    for char in string:
        if char in unsafe:
            char = unsafe[char]
        ret = '{}{}'.format(ret, char)
    return ret

def token_sas(resourceURI, shared_access_key, expiry=3600):
    sasToken = ''
    expiry = int(time()) + 946684800 + expiry
    signature = '{}\n{}'.format(encode_uri(resourceURI), expiry)
    signature_string = b2a_base64(HMAC(a2b_base64(shared_access_key), signature.encode('utf-8'), sha256).digest()).decode('utf8')[:-1]
    sasToken = 'SharedAccessSignature sr={}&sig={}&se={}'.format(encode_uri(resourceURI), encode_uri(signature_string), expiry)
    return sasToken

def cloud_to_device():
    mqttc.check_msg()

def device_to_cloud(device_id, payload):
    d2c_topic = 'devices/{}/messages/events/'.format(device_id)
    mqttc.publish(d2c_topic, ujson.dumps(payload))

def callback(topic, message):
    print('Received topic={} message={}'.format(topic, message))

try:
    from ntptime import settime
    settime()
    gc.collect()
    del sys.modules['ntptime']
    gc.collect()
except:
    pass

parts = dict(x.split('=', 1) for x in connection_string.split(';'))
_device_id = parts['DeviceId']
_hostname = parts['HostName']
_shared_access_key = parts['SharedAccessKey']
_resourceURI = '{}/devices/{}'.format(_hostname, _device_id)
_username = '{}/{}/?api-version=2021-04-12'.format(_hostname, _device_id)
_password = token_sas(_resourceURI, _shared_access_key)
print(_password)
gc.collect()

mqttc = MQTTClient(_device_id, _hostname, 8883, _username, _password, ssl=True)
mqttc.set_callback(callback)
mqttc.connect()

for i in range(0, 3):    
    d2c_topic = 'devices/{}/messages/events/'.format(_device_id)
    mqttc.publish(d2c_topic, "666")
    sleep(5)

mqttc.disconnect()
gc.collect()

```

​    这里有两个需要注意的地方。

​    第一，MicroPython固件引用的时间与其他操作系统不一样，不是从1970年01月01日0:00开始，而是从2000年01月01日0:00开始，所以计算过期时间的时候，需要加上946684800秒。

​    第二，由于使用的模块不一样，生成HMAC字符串的时候，会在后面多一个换行。使用[:-1]把最后一个字符删掉即可。



#### 查看设备到云消息

​    差点忘了交代一下怎么确认ESP32上的MicroPython发送了消息到Azure云上的IoT Hub。使用VS Code的IoT Hub扩展可以连接到终结点看D2C消息，也可以使用Azure IoT Explorer直接选择遥测查看。

![image-20221017203436494](../main/assets/image-20221017203436494.png)

​    实际折腾中发现umqtt的发送经常报“OSError: -104"错误，要么是因为家里WiFi太烂了…（电信路由的烂wifi，刷剧经常会卡）…要么是因为IoT Hub在”外面“会被”Reset“…或者参考更加韧性的代码试试…

​    【此处暂停一天】



​    在继续ESP32通过WiFi网络发送数据，并从IoT Hub收集处理之前，我们还需要解决三个问题：第一，如何确保网络抖动或者闪断之后数据能恢复发送；第二，组织传感器的数据，转换成JSON格式消息发送给IoT Hub；第三，如何确保连接IoT Hub的SAS令牌到期之后能够自动生成。



#### 完善传感器到IoT Hub的代码

​    直接贴代码吧。来不及了，先上车再解释～哈哈～

```python
import sys
import gc
gc.collect()
from ujson import dumps
from utime import time, sleep
sleep(5) 
from ubinascii import a2b_base64, b2a_base64
from uhashlib import sha256
from hmac import HMAC
#from umqtt.simple import MQTTClient
gc.collect()
from mqtt_as import MQTTClient, config
import uasyncio as asyncio
gc.collect()

connection_string = 'HostName={iot-hub-name}.azure-devices.net;DeviceId={MyIotDeviceName};SharedAccessKey={MySharedAccessKey=}'

unsafe = {
    '?': '%3F',
    ' ': '%20',
    '$': '%24',
    '%': '%25',
    '&': '%26',
    "\'": '%27',
    '/': '%2F',
    ':': '%3A',
    ';': '%3B',
    '+': '%2B',
    '=': '%3D',
    '@': '%40'
}

def encode_uri(string):
    ret = ''
    for char in string:
        if char in unsafe:
            char = unsafe[char]
        ret = '{}{}'.format(ret, char)
    return ret

def token_sas(resourceURI, shared_access_key, token_time, expiry=3600):
    sasToken = ''
    expiry = token_time + 946684800 + expiry
    signature = '{}\n{}'.format(encode_uri(resourceURI), expiry)
    signature_string = b2a_base64(HMAC(a2b_base64(shared_access_key), signature.encode('utf-8'), sha256).digest()).decode('utf8')[:-1]
    sasToken = 'SharedAccessSignature sr={}&sig={}&se={}'.format(encode_uri(resourceURI), encode_uri(signature_string), expiry)
    return sasToken

_token_time = time()
_expiry = 7200
parts = dict(x.split('=', 1) for x in connection_string.split(';'))
_device_id = parts['DeviceId']
_hostname = parts['HostName']
_shared_access_key = parts['SharedAccessKey']
_resourceURI = '{}/devices/{}'.format(_hostname, _device_id)
_username = '{}/{}/?api-version=2021-04-12'.format(_hostname, _device_id)
_password = token_sas(_resourceURI, _shared_access_key, _token_time, _expiry)

from machine import Pin, I2C
from bmp280 import BMP280
from ahtx0 import AHT20
gc.collect()

i2c = I2C(scl=Pin(2), sda=Pin(15))

def mqtt_callback(topic, msg, retained):
    print((topic, msg, retained))

async def mqtt_subscribe(client):
    await client.subscribe('devices/{}/messages/devicebound/#'.format(_device_id), 1)

async def mqtt_publish(client):
    global _token_time, _expiry, _password
    try:
        await client.connect()
        print('MQTT connected.')
    except OSError:
        print('MQTT connection failed.')
        return
    while True:
        await asyncio.sleep(60)
        if time() >= _token_time + _expiry:
            _token_time = time()
            _password = token_sas(_resourceURI, _shared_access_key, _token_time, _expiry)    
        _time = gmtime()
        _payload = {
            "Date": '{}/{:0>2d}/{}'.format(_time[0],_time[1],_time[2]),
            "Time": '{:0>2d}:{:0>2d}:{:0>2d}'.format(_time[3],_time[4],_time[5]),
            "Temperature": AHT20(i2c).temperature,
            "Humidity": AHT20(i2c).relative_humidity,
            "Pressure": BMP280(i2c).Pressure
        }
        print(dumps(_payload))
        await client.publish('devices/{}/messages/events/'.format(_device_id), dumps(_payload), 1)      

config['client_id'] = _device_id
config['server'] = _hostname
config['user'] = _username
config['password'] =_password
config['ssl'] = True
config['subs_cb'] = mqtt_callback
config['ssid'] = '{SSID}'
config['wifi_pw'] = '{WiFiPassword}'

MQTTClient.DEBUG = True
client = MQTTClient(config)

try:
    asyncio.run(mqtt_publish(client))
finally:
    client.close()
    asyncio.new_event_loop()

```

​    实际上MicroPython固件内置的umqtt.simple提供了简单的MQTT协议支持，本身也具备对MQTT协议3.1.1的QoS 0和QoS 1的支持。QoS 0意味着发送者只管发送消息，并不确认接收者是否收到；而QoS 1意味着接收者会确认至少收到一次消息，接收者需要应答发送者收到了消息。其实还有QoS 2，意味着确保只有一次消息被接收。这需要发送者和接收者之间两次来回消息，实现负载，开销较大。因此MicroPython和IoT Hub使用的MQTT消息都不支持QoS 2。

​    即使我们在构造MQTT发布消息的时候选择了QoS=1，实际上由于无线网络和互联网络传输质量的不可控，经常会出现拥塞，导致消息无法正常发送和确认。为了提高MQTT消息发送质量，MicroPython还提供了umqtt.robust的库，以提高可靠性。

​    但想要真的确保MQTT的可靠性，还得靠一个很厉害的模块包：micropython-mqtt。作者在这个包里考虑到了无线网络和丢包、拥塞等情况，尽一切可能来实现QoS 1的MQTT传输质量目标。为了使用这个包，可以将其mqtt_us.py直接复制到ESP32的板子上，然后引入其中的代码。使用上首先需要构造一个配置config，里面包含了需要提供的选项，例如MQTT所需要的客户端ID、服务器地址、用户名、密码、是否使用SSL连接、回调函数，以及用于恢复WiFi网络需要的SSID和密码。

​    为解决阻塞，代码使用了异步调用方式。如果代码从系统加电开始运行，可以在导入utime的sleep之后，等待5秒。

​    接下来是构造包括日期时间和温湿度气压数据的JSON消息。这部分比较简单，直接使用ujson的dumps方法处理python格式数据即可。日期时间可以使用utime中gmtime()来获得8元数组，包括年月日时分秒星期几和年度天序数。可以格式化数据变为比较易读的格式。

​    温湿度气压数据来自与之前实验使用的AHT20+BMP280的板子，通过I2C读取之后“塞”进JSON数据即可。

​    SAS令牌是会随着expiry数字过期的，之所以使用这个机制，就是不希望SAS令牌凭据被截获或者滥用。所以尽管我们可以生成一个有效期长达比如一天的SAS令牌，它第一不安全，第二始终会过期。所以需要在代码中按照过期时间自动重新生成SAS令牌。

​    这项工作其实也很简单，设置了一个用于生成token的锚点，一旦当前时间（time()得到的整数秒数）等于或者超过这个锚点记录的时间（使用time()得到的整数秒数）加上过期时间，代码重新调用生成SAS令牌的过程即可。

​    至此，上述三个问题都得到了解决。为了防止ESP32板载的时钟失准，在代码里还调用了NTP校准时间的代码。其实这部分代码放在加电运行的boot.py里可能更方便。顺手把boot.py也贴一下。

```python
# boot.py - - runs on boot-up
import network
import usocket as socket
from time import sleep, gmtime

import esp
esp.osdebug(None)

import gc
gc.collect()

ssid = '{SSID}'
password = '{WiFiPassword}'
station = network.WLAN(network.STA_IF)
station.active(True)
station.connect(ssid, password)

while station.isconnected() == False:
  pass
print('Connection successful')
print(' IP: ', station.ifconfig())

```

​    如果我们在ESP32板子的REPL接口查看，应该类似如下的输出（从板子加电开始）：

```
rst:0x1 (POWERON_RESET),boot:0x1b (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:4540
ho 0 tail 12 room 4
load:0x40078000,len:12344
ho 0 tail 12 room 4
load:0x40080400,len:4124
entry 0x40080680
Connection successful
 IP: ('192.168.1.3', '255.255.255.0', '192.168.1.1', '192.168.1.1')
NTP set time successful
UTC: (2022/10/18 15:23:03)
Checking WiFi integrity.
Got reliable connection
Connecting to broker.
Connected to broker.
MQTT connected.
RAM free 81248 alloc 29920
RAM free 81248 alloc 29920
{"Time": "15:24:18", "Temperature": 22.12887, "Humidity": 42.10873, "Pressure": 102708, "Date": "2022/10/18"}
RAM free 80192 alloc 30976
RAM free 80880 alloc 30288
RAM free 80880 alloc 30288
{"Time": "15:25:18", "Temperature": 22.10999, "Humidity": 42.06009, "Pressure": 102704, "Date": "2022/10/18"}
```

​    这块ESP32除了使用Micro USB接口供电之外，还可以使用电池接口，这样我只需要接上电源，就可以自动收集传感器数据和传输了。至此，运行MicroPyhton程序、每分钟向IoT Hub发送一次温湿度气压JSON数据的ESP32板子我们算是准备好了。

