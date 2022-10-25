ESP32-5：把ESP32连接到Azure IoT云端



​    【虽然本文的内容基本相对独立，但仍建议先阅读之前的 [ESP32-1]()、[ESP32-2]()、[ESP32-3]()和[ESP32-4]() 】

​     在一块带有WiFi的板子上，不开启网络，让板子连接到云端，简直是浪费啊。云端的IoT服务最容易上手的，我估计是Azure的IoT Hub了。微软提供了非常好的文档，而且在文档里和GitHub上都有不少示例代码。最方便的莫过于使用Azure IoT SDK来完成开发。

​     可是…Azure IoT SDK并不是为MicroPython而开发的，因此无法直接在类似ESP32的板子上直接使用。所以，就只能手工完成代码了。



#### 可惜用不上Azure IoT SDK

​    微软实际上为两类不同的设备提供了开发的SDK。

| 设备分类   | 说明                                            | 示例                         | SDK                                                          |
| :--------- | :---------------------------------------------- | :--------------------------- | :----------------------------------------------------------- |
| 嵌入式设备 | 具有计算和内存限制的基于 MCU 的特殊设备         | 传感器                       | [嵌入式设备 SDK](https://learn.microsoft.com/zh-cn/azure/iot-develop/about-iot-sdks#embedded-device-sdks) |
| 其他       | 包括具有较大计算和内存资源的基于 MPU 的常规设备 | 电脑、智能手机、Raspberry Pi | [设备 SDK](https://learn.microsoft.com/zh-cn/azure/iot-develop/about-iot-sdks#device-sdks) |

​    在以往折腾Python来连接Azure IoT平台的时候，我使用过`设备 SDK`，从我的电脑上直接运行代码和IoT Hub通信。

| 语言        | 程序包                                                       | Source                                                  | 快速入门                                                     | 示例                                                         | 参考                                                         |
| :---------- | :----------------------------------------------------------- | :------------------------------------------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **.NET**    | [NuGet](https://www.nuget.org/packages/Microsoft.Azure.Devices.Client) | [GitHub](https://github.com/Azure/azure-iot-sdk-csharp) | [IoT Hub](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-csharp) / [IoT Central](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-central?pivots=programming-language-csharp) | [示例](https://github.com/Azure/azure-iot-sdk-csharp/tree/main/iothub/device/samples) | [引用](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.azure.devices.client) |
| **Python**  | [pip](https://pypi.org/project/azure-iot-device/)            | [GitHub](https://github.com/Azure/azure-iot-sdk-python) | [IoT Hub](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-python) / [IoT Central](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-central?pivots=programming-language-python) | [示例](https://github.com/Azure/azure-iot-sdk-python/tree/main/samples) | [引用](https://learn.microsoft.com/zh-cn/python/api/azure-iot-device) |
| **Node.js** | [npm](https://www.npmjs.com/package/azure-iot-device)        | [GitHub](https://github.com/Azure/azure-iot-sdk-node)   | [IoT Hub](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-nodejs) / [IoT Central](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-central?pivots=programming-language-nodejs) | [示例](https://github.com/Azure/azure-iot-sdk-node/tree/main/device/samples) | [引用](https://learn.microsoft.com/zh-cn/javascript/api/azure-iot-device/) |
| **Java**    | [Maven](https://mvnrepository.com/artifact/com.microsoft.azure.sdk.iot/iot-device-client) | [GitHub](https://github.com/Azure/azure-iot-sdk-java)   | [IoT Hub](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-java) / [IoT Central](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-central?pivots=programming-language-java) | [示例](https://github.com/Azure/azure-iot-sdk-java/tree/master/device/iot-device-samples) | [引用](https://learn.microsoft.com/zh-cn/java/api/com.microsoft.azure.sdk.iot.device) |
| **C**       | [包](https://github.com/Azure/azure-iot-sdk-c/blob/master/readme.md#getting-the-sdk) | [GitHub](https://github.com/Azure/azure-iot-sdk-c)      | [IoT Hub](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-ansi-c) / [IoT Central](https://learn.microsoft.com/zh-cn/azure/iot-develop/quickstart-send-telemetry-central?pivots=programming-language-ansi-c) | [示例](https://github.com/Azure/azure-iot-sdk-c/tree/master/iothub_client/samples) | [引用](https://learn.microsoft.com/zh-cn/azure/iot-hub/iot-c-sdk-ref/) |

​    我记得在模拟器里用过基于 C 的SDK的，当时还提交了个字串处理的issue给开发者。

​    如今既然打算折腾ESP32的板子，那只能看看`嵌入式设备 SDK`了。只可惜这个SDK仅支持三种系统：Azure RTOS、FreeRTOS（ESP32原生系统好像是）、裸机（支持嵌入式 C 语言）。因为我们想折腾MicroPython，就没法用现成的SDK了。

​    以上及更多Azure IoT 设备 SDK 的文档，可访问：[https://learn.microsoft.com/zh-cn/azure/iot-develop/about-iot-sdks?WT.mc_id=Azure-MVP-33253](https://learn.microsoft.com/zh-cn/azure/iot-develop/about-iot-sdks?WT.mc_id=Azure-MVP-33253)



#### 自己完成IoT Hub通信的代码

​    只能手撸代码了。还好是非生产的测试，不用写像SDK那样全套的代码…先上搜索引擎…

##### 从第一篇示例代码开始

​    找到篇示例代码，是一篇个人博客：[https://www.jiaweihuang.com/post/2020-03-24-esp32-azure-iothub-micropython](https://www.jiaweihuang.com/post/2020-03-24-esp32-azure-iothub-micropython)

​    其实很久之前就搜到这篇文章了，那时候是想用模拟器跑MicroPython。事情繁杂，就放一边了。这回打算搞搞。这篇文章其实也是按照微软文档来准备代码的，特别是需要准备的各类参数。所以先把微软文档搬出来看看吧。

​    从设备连接Azure IoT Hub，可以使用不同的协议。因为MQTT相对容易实现，打算折腾这个。如前所属，因为不能使用Azure IoT SDK里写好的客户端，所以只能自己来按照微软要求构造MQTT请求了。

​    常见的MQTT客户端请求会包括：设备ID、连接的服务器地址、使用的端口、用户名、密码、是否使用SSL/TLS和持续活动时间。先不论我们使用何种Python包，第一步，需要按照微软要求准备好这些信息。

​    按照微软官方文档 [https://learn.microsoft.com/azure/iot-hub/iot-hub-mqtt-support?WT.mc_id=Azure-MVP-33253](https://learn.microsoft.com/azure/iot-hub/iot-hub-mqtt-support?WT.mc_id=Azure-MVP-33253) 的描述，所有通过Azure IoT Hub的通信都必须使用SSL/TLS保护，设备可以在端口8883上使用 MQTT v3.1.1协议，或者在端口443上使用基于WebSocket的MQTT v3.1.1协议。

​    既然不使用SDK，就只能通过MQTT协议直接连接到公共设备端点了。 在 MQTT的CONNECT 数据包中，设备应使用以下值：

- **ClientId** 字段使用 **deviceId**。

- “用户名”字段使用 `{iotHub-hostname}/{device-id}/?api-version=2021-04-12`，其中 `{iotHub-hostname}` 是 IoT 中心的完整 CName。

  例如，如果 IoT 中心的名称为 **contoso.azure-devices.net**，设备的名称为 **MyDevice01**，则完整“用户名”字段应包含：

  `contoso.azure-devices.net/MyDevice01/?api-version=2021-04-12`

  建议在字段中包含 api-version。 否则，可能会导致意外行为。

- “**密码**”字段使用 SAS 令牌。 对于 HTTPS 和 AMQP 协议，SAS 令牌的格式是相同的：

  `SharedAccessSignature sig={signature-string}&se={expiry}&sr={URL-encoded-resourceURI}`

​    如果使用X.509证书来验证设备，就不需要SAS令牌作为密码了。不过折腾证书貌似也挺麻烦，所以我还是打算用SAS令牌。

​    有两种方法可以使用 SAS 令牌来获取 IoT 中心的 **DeviceConnect** 权限：使用[标识注册表中的对称设备密钥](https://learn.microsoft.com/zh-cn/azure/iot-hub/iot-hub-dev-guide-sas?tabs=node#use-a-symmetric-key-in-the-identity-registry)，或者使用[共享访问密钥](https://learn.microsoft.com/zh-cn/azure/iot-hub/iot-hub-dev-guide-sas?tabs=node#use-a-shared-access-policy-to-access-on-behalf-of-a-device)。

请记住，可从设备访问的所有功能都故意显示在前缀为 `/devices/{deviceId}` 的终结点上。

​    面向设备的终结点包括（无论任何协议）：

| 端点                                                         | 功能                 |
| :----------------------------------------------------------- | :------------------- |
| `{iot hub host name}/devices/{deviceId}/messages/events`     | 发送设备到云的消息。 |
| `{iot hub host name}/devices/{deviceId}/messages/devicebound` | 接收云到设备的消息。 |

​    请注意设备到Azure IoT云端的通信也是有方向的，分为设备到云（D2C）和云到设备（C2D）。我们后面将尝试从设备发送消息到云端。

​    在这之前，首先得计算出用作密码的SAS令牌。



##### 使用标识注册表中的对称密钥

​    使用设备标识的对称密钥生成令牌时，将省略令牌的 policyName (`skn`) 元素。例如，创建的用于访问所有设备功能的令牌应具有以下参数：

- 资源 URI： `{IoT hub name}.azure-devices.net/devices/{device id}`，
- 签名密钥： `{device id}` 标识的任何对称密钥，
- 无策略名称；
- 任何过期时间。



​    通过在Azure里创建IoT Hub资源，就可以生成用于连接IoT Hub的连接字串。该字串类似如下：

```xml
HostName={iot-hub-name}.azure-devices.net;DeviceId={MyIoTDevice};SharedAccessKey={MySharedAccessKey=}
```

​    上例中`{iot-hub-name}`为创建IoT Hub资源时指定的IoT Hub名称，而`{MyIoTDevice}`是在这个IoT Hub里创建的设备名称， `{MySharedAccessKey=}`表示以“=“结尾的编码字串，用做给SAS令牌签名。

​    只要在Azure门户站点打开IoT Hub资源，在左边点击”共享访问策略“就能找到连接字符串，复制备用～



##### 从对称密钥生成SAS令牌

​    拿到连接字符串之后，就可以拆除需要的部分，然后计算SAS令牌了。怎么做呢？官方文档有介绍：[https://learn.microsoft.com/zh-cn/azure/iot-hub/iot-hub-dev-guide-sas?tabs=python&WT.mc_id=Azure-MVP-33253](https://learn.microsoft.com/zh-cn/azure/iot-hub/iot-hub-dev-guide-sas?tabs=python&WT.mc_id=Azure-MVP-33253)

​    前面我们说过SAS令牌的格式如下：

```
SharedAccessSignature sig={signature-string}&se={expiry}&skn={policyName}&sr={URL-encoded-resourceURI}
```

​    以下是相关预期值的说明：

| 值                        | 说明                                                         |
| :------------------------ | :----------------------------------------------------------- |
| {signature}               | HMAC-SHA256 签名字符串的格式为： `{URL-encoded-resourceURI} + "\n" + expiry`。 **重要说明**：密钥是从 base64 解码得出的，用作执行 HMAC-SHA256 计算的密钥。 |
| {resourceURI}             | 使用此令牌可以访问的终结点的 URI 前缀（根据分段）以 IoT 中心的主机名开头（无协议）。 授予后端服务的 SAS 令牌的范围限定为 IoT 中心级别；例如，`myHub.azure-devices.net`。 授予设备的 SAS 令牌必须限定为单个设备；例如，`myHub.azure-devices.net/devices/device1`。 |
| {expiry}                  | 从纪元 1970 年 1 月 1日 00:00:00 UTC 时间至今秒数的 UTF8 字符串。 |
| {URL-encoded-resourceURI} | 小写资源 URI 的小写 URL 编码                                 |
| {policyName}              | 此令牌所引用的共享访问策略名称。 如果此令牌引用设备注册表凭据，则空缺。 |

​    光看文字是看不出怎么实现的。所以微软给了一段Python的示例代码。

```python
from base64 import b64encode, b64decode
from hashlib import sha256
from time import time
from urllib import parse
from hmac import HMAC

def generate_sas_token(uri, key, policy_name, expiry=3600):
    ttl = time() + expiry
    sign_key = "%s\n%d" % ((parse.quote_plus(uri)), int(ttl))
    print(sign_key)
    signature = b64encode(HMAC(b64decode(key), sign_key.encode('utf-8'), sha256).digest())

    rawtoken = {
        'sr' :  uri,
        'sig': signature,
        'se' : str(int(ttl))
    }

    if policy_name is not None:
        rawtoken['skn'] = policy_name

    return 'SharedAccessSignature ' + parse.urlencode(rawtoken)
```

​    和文档描述一致，首先把设备的资源地址进行HTTP地址安全化处理，在示例代码中使用了urllib包里的parse，调用了qoute_plus方法。然后处理过的字符串加上“\n"换行再加上过期时间串一串，再用base64来解码，得到用于HMAC的SHA256计算密钥。

​    接下来使用导入的hmac包的HMAC方法，利用上一步计算的密钥，加上utf-8编码的IoT Hub连接字符串中的共享密钥，指定sha256算法计算，再将结果进行base64的编码，从而得到用于云端验证的SAS令牌。

​    这还没有完，密码必须包括资源地址、SAS令牌和过期时间，以规定的格式提供。还需对生成的这个格式字符串再次使用HTTP地址安全化处理。这才是最终可以用于MQTT连接的密码。

​    道理我都懂，应该怎么做？



##### 生成SAS令牌的第一次尝试

​    第一次尝试就是简单的搬代码。

​    由于MicroPython使用的Python包与标准Python有区别，所以很不幸，大部分前文示例代码中的包估计都不适合在MicroPython上使用。看过之前文章的小伙伴可能还记得，MicroPython在ESP32上的固件里已经包含了不少经过裁剪的包（模块），可以通过 `help('modules') `查看。所以像 `base64` 、 `time` 等，可以找到对应的 `ubinascii`、`utime` 等进行“平替”。

​    但其他的呢？比如 `hmac`、`urllib` 等？实际上这些常见的包在PyPI上也有发布，可以通过带有“micropyhon-”前缀的名字搜索。我很快就找到了这些包。怎么安装呢？

​    MicroPython不同于Python，可以在类似Windows、Mac OS的命令行下进行 PIP 安装，因为运行MicroPython的系统可能并没有自己的命令行，只提供了REPL界面。但MicroPython也提供了类似的 `pip` 安装方式，那就是`upip` 安装。upip的安装有些不一样，它是通过代码环境执行的，以下为安装的示例：

```python
>>> import upip
>>> upip.install('micropython-hmac')
Installing to: /lib/
Warning: micropython.org SSL certificate is not validated
Installing micropython-hmac 3.4.2.post3 from https://micropython.org/pi/hmac/hmac-3.4.2.post3.tar.gz
Installing micropython-hashlib 2.4.0.post4 from https://micropython.org/pi/hashlib/hashlib-2.4.0.post4.tar.gz
Installing micropython-warnings 0.1.1 from https://micropython.org/pi/warnings/warnings-0.1.1.tar.gz

>>> upip.install('micropython-urllib.parse')
Installing to: /lib/
Installing micropython-urllib.parse 0.5.2 from https://micropython.org/pi/urllib.parse/urllib.parse-0.5.2.tar.gz
Installing micropython-collections 0.1.2 from https://micropython.org/pi/collections/collections-0.1.2.tar.gz
Installing micropython-collections.defaultdict 0.3 from https://micropython.org/pi/collections.defaultdict/collections.defaultdict-0.3.tar.gz
Installing micropython-re-pcre 0.2.5 from https://micropython.org/pi/re-pcre/re-pcre-0.2.5.tar.gz
Installing micropython-ffilib 0.1.3 from https://micropython.org/pi/ffilib/ffilib-0.1.3.tar.gz
```

​    可以看到，利用包的依存关系，upip将需要的文件都安装到了 /lib/ 目录下。

​    我开心的写(chao)了一段代码，打算运行一下，结果ESP32的板子不客气的给我个内存报错…查了一下，应该是 import 的文件比较多，ESP32可怜的RAM爆了。

​    解决的办法也不是没有…比较官方的建议是冻结代码。也就是将使用的部分代码交叉编译为二进制码，刷到固件里，这样运行时不再使用更多的内存来即时编译。如果是一个正式项目，这是个不错的选择。可是我就想试试MicroPython，搞这么复杂还不如用 C 呢不是么？所以没打算往交叉编译和刷固件方向尝试。



##### 生成SAS令牌的第二次尝试

​    既然 import 包没搞定，那就直接上文件？首先盯着hmac这个包，这个包的py文件里由 import了 hashlib和warnings，我把warnings去掉了，反正Python学会的第一句：print 也能用，又查看了 uhashlib 的源代码，发现几乎一致。

​    于是手动改了个 hmac.py，貌似没报错…

​    接下来是 urllib.parse包。这个就麻烦了，依赖了re包，而re又依赖 collections，collections.defaultdict和ffilib包……这个工作量就大了……得看好多代码还得知道怎么改……从头看这些代码貌似还要仔细学数据结构和正则表达……这两月很多事情呢。

​    【此处停更一星期】



##### 生成SAS令牌的第三次尝试

​    总归是不死心。周末本来打算准备PPT的，又手痒的打开了VS Code和Edge……又去搜索 micropython + iot + microsoft ……找到个MicroPython下IoT Central的包。

​    IoT Central是看上去和IoT Hub差不多的Azure IoT服务。不过IoT Central基于PaaS服务，使用起来更简化，但相应代码化的订制也比较少。以折腾来说，用这个服务也不是不行…不过这个服务好像没有免费定价层，但是貌似前两个设备免费…

​    我都打算创建IoT Centeral资源了，这时候好奇地点开了IoT Central给MicroPython的GitHub地址：

​    [https://github.com/Azure/iot-central-micropython-client](https://github.com/Azure/iot-central-micropython-client)

​    啊～怎么说呢～感觉太…开心了…受益很多！学到了！

​    第一，这个包也是移植了hmac代码，替换了直接安装的包。第二，在用于制备的代码里，我找到了类似生成SAS令牌的代码。第三，我学到了随时内存回收和删除不使用的包。第四，我发现他居然用了一个简单的字典替换代替之前让我头疼不已的urllib.parse！

​    简直太开心了！我又重新燃起了解决这个问题的斗志～

​    于是，我开始写我自己的SAS令牌代码了。但是，每改一点都发到ESP32的板子上太费事了，于是我决定，先用Python代码实现。

```python
import time
from hmac import HMAC
from base64 import b64encode, b64decode
from hashlib import sha256
from ssl import SSLContext, PROTOCOL_TLS_CLIENT, CERT_REQUIRED

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

def token_sas(_resourceURI, _shared_access_key, _policyName=None, _expiry=3600):
    sasToken = ''
    _URL_encoded_resourceURI = _resourceURI.lower()
    _expiry = int(time.time()) + _expiry
    _signature = '{}\n{}'.format(encode_uri(_resourceURI), _expiry)
    _signature_string = b64encode(HMAC(b64decode(_shared_access_key), _signature.encode('utf-8'), sha256).digest()).decode('utf8')
    if _policyName:
        sasToken = 'SharedAccessSignature sr={}&sig={}&se={}&skn={}'.format(encode_uri(_resourceURI), encode_uri(_signature_string), _expiry, _policyName)
    else:
        sasToken = 'SharedAccessSignature sr={}&sig={}&se={}'.format(encode_uri(_resourceURI), encode_uri(_signature_string), _expiry)
    return sasToken

connection_string = 'HostName={iot-hub-name}.azure-devices.net;DeviceId={MyIoTDevice};SharedAccessKey={MySharedAccessKey=}'

parts = dict(x.split('=', 1) for x in connection_string.split(';'))
_device_id = parts['DeviceId']
_hostname = parts['HostName']
_shared_access_key = parts['SharedAccessKey']
_resourceURI = '{}/devices/{}'.format(_hostname, _device_id)
_username = '{}/{}/?api-version=2021-04-12'.format(_hostname, _device_id)
_password = token_sas(_resourceURI, _shared_access_key)

# Show the string same as Microsoft IoT tools
print("HostName={};DeviceId={};SharedAccessSignature={}".format(_hostname, _device_id, _password))

from paho.mqtt import client as mqttc

def on_connect(client, userdata, flags, rc):
    print("Device connected with result code: " + str(rc))

def on_disconnect(client, userdata, rc):
    print("Device disconnected with result code: " + str(rc))

def on_publish(client, userdata, mid):
    print("Device sent message")

def on_message(client, userdata, msg):
    print(msg.topic + " " + str(msg.payload))

client = mqttc.Client(client_id=_device_id, protocol=mqttc.MQTTv311)

client.on_connect = on_connect
client.on_disconnect = on_disconnect
client.on_publish = on_publish
client.on_message = on_message

ssl_context = SSLContext(protocol=PROTOCOL_TLS_CLIENT)
ssl_context.load_default_certs()
ssl_context.verify_mode = CERT_REQUIRED
ssl_context.check_hostname = True
client.tls_set_context(context=ssl_context)

client.username_pw_set(username=_username, password=_password)
client.connect(_hostname, port=8883)

client.loop_start()
for i in range(0, 3):
    client.publish("devices/" + _device_id + "/messages/events/", '{"id":555}', qos=1)
    time.sleep(5)

```

​    调试代码永远是件有意思但又很枯燥很折磨人的事情，没连上IoT Hub之前，难道我永远也不确定SAS令牌对不对吗？



##### 生成SAS令牌的第四次尝试

​    实际上，如果不是要自己写代码实现，至少有三个方法可以帮助我们生成SAS令牌的。

​    第一个，是VS Code的IoT Hub扩展插件，我记得很久之前韩俊老师就做了这个插件，收归微软后，这个插件变得更加好用。使用这个插件在VS Code里找到IoT Hub下要通信的设备，右键菜单就可以选择生成SAS令牌。

​    第二个，是微软的Azure IoT Explorer。这是个跨平台可用的应用，也可以设备标识那里，选择对称密钥和过期时间，生成一个SAS令牌。

​    第三个，我找到了GitHub上的一个示例代码，经测试可以使用，因此由它生成的SAS令牌应该是没问题的。

​    于是，我就把SAS令牌是不是正确生成这个问题，变成了：能不能使用一样的信息，和微软“官方”生成的SAS令牌比较一下。

​    说干就干。VS Code开着，就从这开始吧。给一个很长的过期时间，然后从IoT Hub的扩展上生成了一个SAS令牌。然后将我的代码里生成SAS令牌的部分修改一下。

```python
def token_sas(_resourceURI, _shared_access_key, _policyName=None, _expiry=3600):
    sasToken = ''
    _URL_encoded_resourceURI = _resourceURI.lower()
#    _expiry = int(time.time()) + _expiry
    _expiry = 1666306202
    _signature = '{}\n{}'.format(encode_uri(_resourceURI), _expiry)
    _signature_string = b64encode(HMAC(b64decode(_shared_access_key), _signature.encode('utf-8'), sha256).digest()).decode('utf8')

```

​      如上，我将原来计算过期时间的代码注释掉，把刚才生成的可用SAS令牌的过期时间数字直接分配给产生SAS令牌的代码。这下终于可以确认代码生成的SAS令牌和工具生成的可用SAS令牌是一致的了。

​      不过既然是折腾，就彻底点，示例代码我也试试。示例代码地址：[https://github.com/Azure-Samples/IoTMQTTSample](https://github.com/Azure-Samples/IoTMQTTSample)

```python
import paho.mqtt.client as mqtt
from base64 import b64encode, b64decode
from time import time, sleep
from urllib import parse
from hmac import HMAC
from hashlib import sha256
from ssl import SSLContext, PROTOCOL_TLS_CLIENT, CERT_REQUIRED

IOT_HUB_HOSTNAME  = "{IoT-Hub-Name}.azure-devices.net"
IOT_HUB_DEVICE_ID = "{MyDeviceName}"
IOT_HUB_SAS_KEY   = "{MySharedAccessKey=}"

def on_connect(mqtt_client, obj, flags, rc):
    print("connect: " + str(rc))

def on_publish(mqtt_client, obj, mid):
    print("publish: " + str(mid))

def generate_sas_token():
    ttl = int(time()) + 300
    uri = IOT_HUB_HOSTNAME + "/devices/" + IOT_HUB_DEVICE_ID
    sign_key = uri + "\n" + str(ttl)
    print()
    signature = b64encode(HMAC(b64decode(IOT_HUB_SAS_KEY), sign_key.encode('utf-8'), sha256).digest())

    return "SharedAccessSignature sr=" + uri + "&sig=" + parse.quote(signature, safe="") + "&se=" + str(ttl)

mqtt_client = mqtt.Client(client_id=IOT_HUB_DEVICE_ID, protocol=mqtt.MQTTv311)
mqtt_client.on_connect    = on_connect
mqtt_client.on_publish    = on_publish

_password = generate_sas_token()
print(_password)

mqtt_client.username_pw_set(username=IOT_HUB_HOSTNAME + "/" + IOT_HUB_DEVICE_ID + "/?api-version=2021-04-12", 
                            password=_password)

ssl_context = SSLContext(protocol=PROTOCOL_TLS_CLIENT)
ssl_context.load_default_certs()
ssl_context.verify_mode = CERT_REQUIRED
ssl_context.check_hostname = True
mqtt_client.tls_set_context(context=ssl_context)

mqtt_client.connect(host=IOT_HUB_HOSTNAME, port=8883, keepalive=120)

# start the MQTT processing loop
mqtt_client.loop_start()

# send telemetry
messages = ["Accio", "Aguamenti", "Alarte Ascendare", "Expecto Patronum", "Homenum Revelio"]
for i in range(0, len(messages)):
    print("sending message[" + str(i) + "]: " + messages[i])
    mqtt_client.publish("devices/" + IOT_HUB_DEVICE_ID + "/messages/events/", payload=messages[i], qos=1)
    sleep(5)
    
```

​    也是一样的操作，但是问题来了……示例代码生成的SAS令牌居然和我的代码生成的不一样！更关键的，居然都能用！！

​    这……不科学啊……熬到快四点，我决定睡一觉再想。

​    【此处经过四小时】

​    醒了之后我在微信上问了下韩俊老师，他也觉得很稀奇——这应该跟Azure IoT Hub服务有关。我洗了把脸清醒一下准备开会。突然我注意到SAS令牌的sr部分，一个使用HTTP安全字串方法处理过，一个没有。这意味着两种方式Azure IoT Hub服务都接受了。另外，文档里提到的URI小写处理，我发现并未在生成SAS令牌时使用，可以去提交一个issue了吧。从网络传输角度来说，我觉得SAS令牌经过HTPP安全处理更加规范安全。

​    接下来，是该把代码移植到MicroPython的时候了～


