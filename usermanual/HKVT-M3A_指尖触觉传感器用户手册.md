![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/0855efd5baf58578802b88e5822ae794aa41bf5fa769bc8c8e6af9d4f305c0bf.png)

HKVT-M3A  指尖触觉传感器

用 户 手 册

(版本号：Ver.01)

<a id="_Toc6384"></a>
# 前 言

本手册主要介绍了HKVT-M3A型指尖触觉传感器的功能、尺寸规格、使用方法、通讯接口及注意事项等内容。在使用本产品之前，请务必认真阅读本手册，以免出现因操作不当导致的产品故障及安全事故。

本手册所包含的内容归苏州航凯微电子技术有限公司所有，未经我司书面许可的前提下，任何公司或个人不得以任何方式复制或修改本手册的内容。我司保留本手册的修改权，本手册定期更新，如有更改，本公司不另行通知。对于本手册中的错误用词，本公司具有免责权力。

- **请注意**：

1. 不同批次的产品可能会有微小色差，因包装或运输原因，个别产品外观可能会有轻微划痕，不影响产品性能。
2. 如果用户将该型号传感器用于电磁辐射、腐蚀性化学品环境等特殊环境，请联系售后人员。
3. 对于非用户手册规定方法操作和使用传感器所造成的经济损失，本公司概不负责。
4. 因不可抗力（如地震、海啸、雪灾等自然灾害），以及第三方行为造成的传感器损坏，本公司概不负责。
5. 本产品的保修期为发货之日起12个月。擅自拆卸，修改传感器内部电路与机械结构、使用传感器方法不当造成的传感器损坏均不在保修范围之内。

- **目 录**
- I
- 1
- 1
    - 1
    - 1
    - 2
- 2
    - 2
    - 3
    - 3
    - 3
    - 4
- 4
- 6

<a id="_Toc26119"></a>
# 1 **产品简介**

HKVT-M3A型指尖触觉传感器，可实现正压力Fz以及切向力Fx、Fy三个方向力的检测，可将检测数据通过I2C协议传输至上位机。本产品具有结构简单，体积小，灵敏度高，可靠性高等特点，可应用于机器人灵巧手等领域。

<a id="_Toc20189"></a>
# 2 **外观尺寸及性能参数**

<a id="_Toc31866"></a>
## **2.1外观及坐标定义**

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/7b032029be2d178556bb06b5eb4201043f3d5f7575e26f9171b932f2c136e4c1.jpg)

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/0ca6f2c985a1e027cb72b7d617c9b802dc31b053a37ac19170d099268c0bbad6.jpg)

<a id="_Toc29178"></a>
## **2.2安装尺寸示意图**

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/c33048c1ff655e4326ff858e467d385fddb07dc6ac4c489d80625d5edcca3eb8.jpg)

<a id="_Toc12903"></a>
## **2.3性能参数**

<table><tr><td><p><strong>项目</strong></p></td><td><p><strong>参数</strong></p></td></tr><tr><td><p>颜色</p></td><td><p>黑色</p></td></tr><tr><td><p>表面材质</p></td><td><p>硅胶</p></td></tr><tr><td><p>法向力</p></td><td><p>15N</p></td></tr><tr><td><p>切向力</p></td><td><p>10N</p></td></tr><tr><td><p>精度</p></td><td><p>2%Fs</p></td></tr><tr><td><p>工作温度</p></td><td><p>-20℃~85℃</p></td></tr><tr><td><p>工作湿度</p></td><td><p>0~95%RH</p></td></tr><tr><td><p>工作电压</p></td><td><p>2.5~3.3V</p></td></tr><tr><td><p>通信方式</p></td><td><p>IIC（从机）</p></td></tr><tr><td><p>采样频率</p></td><td><p>  200Hz</p></td></tr><tr><td><p><a></a>安全过载</p></td><td><p>400%</p></td></tr></table>

# 3 **通讯协议**

<a id="_Toc352"></a>
## 3.1 **使用方法**

I2C总线自带上拉电阻，可直连MCU端口

注：传感器底部朝上，输出端连接器为下翻盖式，FPC为上接，线间距0.5mm，PIN脚定义如下：

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/ec0fb26bd3bd5ef1b9de8d6f6b21038fbafcab10071441d27803675bca6abbfc.jpg)

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/30fcdff41170b47b2730be0cb395ec20d9c515d3d0bc0290730a0ed82d65b696.png)

接插件规格：

<table><tr><td><p><strong>型号</strong></p></td><td><p><strong>AFC42-S04FMA-1H</strong></p></td></tr><tr><td><p>锁定特性</p></td><td><p>翻盖式</p></td></tr><tr><td><p> 触点类型 </p></td><td><p>双侧触点/上下接</p></td></tr><tr><td><p>触点数量</p></td><td><p>4P</p></td></tr><tr><td><p>安装方式</p></td><td><p>卧贴</p></td></tr><tr><td><p>接入柔性电缆厚度</p></td><td><p>0.3mm</p></td></tr><tr><td><p>板上高度</p></td><td><p>1mm</p></td></tr></table>

<a id="_Toc7387"></a>
## 3.2 **通讯规格**

<table><tr><td><p><strong>通讯参数</strong></p></td><td><p><strong>值</strong></p></td></tr><tr><td><p>通讯接口</p></td><td><p>IIC</p></td></tr><tr><td><p>IIC 速率</p></td><td><p>400kHz</p></td></tr><tr><td><p>从设备地址</p></td><td><p>0x0A</p></td></tr><tr><td><p>供电电压</p></td><td><p>2.5V-3.3V</p></td></tr></table>

<a id="_Toc4416"></a>
## 3.3 **数据格式**

IIC通信缓存数组完整数据

<table><tr><td><p>X</p></td><td><p>Y</p></td><td><p>Z</p></td></tr></table>

所有数据均存放在IIC通信缓存数组当中，三个数据连续发送，若要读取XYZ三轴数据，先写设备地址，再写相应命令，读取六个字节。注意,X,Y,Z,的数据均是由两个字节组成，先读低八位，再读高八位（小端解析）。

<a id="_Toc6850"></a>
## 3.4 **读取命令**

<table><tr><td><p><strong>命令</strong></p></td><td><p><strong>功能</strong></p></td><td><p><strong>数据类型</strong></p></td><td><p><strong>数据总长度</strong></p></td><td><p><strong>默认返回值</strong></p></td></tr><tr><td><p>0x03</p></td><td><p>读取数据</p></td><td><p>Signed short int</p></td><td><p>2x3=6</p></td><td><p>当前数据</p></td></tr></table>

（注：读取数据前，传感器校准时间约1s）

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/a687db2b1f1ee9ec448a423fcb0bfb750404e344225cea73f121b71f95fd487d.png)

I2C 协议

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/f2b5b01b5d933ae6f0664cfca4a68ecf7c7b92521245b4710ff83d1f8e74eb03.png)

读取传感器X Y Z数据命令

<a id="_Toc22681"></a>
## 3.5 **写入命令**

<table><tr><td><p><strong>命令</strong></p></td><td><p><strong>功能</strong></p></td><td><p><strong>数据类型</strong></p></td><td><p><strong>数据长度</strong></p></td><td><p><strong>数据含义</strong></p></td></tr><tr><td><p>0x1A</p></td><td><p>修改I2C地址</p></td><td><p>unsigned char</p></td><td><p>1</p></td><td><p>I2C 7bit地址</p></td></tr></table>

![](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/d2a91b64-0fdb-42e8-b9ea-e7d8e27c4586/e6356a4a7b57ed2b1c02b96eed44ab430c927facc3628d852c6e60fee6e13911.png)

修改I2C地址

（注：

1. 从机地址请勿设置成0xFF或0x00；
2. 以上命令的数据存放在Flash中，掉电存储，可读可写。

<a id="_Toc9389"></a>
# **4.注意事项**

1. 为了确保传感器的测量精度，传感器应避免在高温高湿、温度变化剧烈、高冲击环境下使用，尤其避免在强磁场环境中使用。
2. 运输前请确保传感器包装状态良好，运输过程中，避免在包装上部摆放重物，避免超出传感器量程的冲击载荷。
3. 使用中避免接触尖锐物体，若必须与尖锐物体接触，必须在指腹处增加硅胶，海绵等柔性物体，避免损坏指腹胶层。
4. 用户应避免传感器在使用中受到外部冲击(锤击、大力频繁按压等)，以免对传感器性能造成不良影响。
5. 拆卸和安装传感器时务必使传感器和执行机构处于断电状态。
6. 接线时务必按照线缆定义进行连接，避免因接线错误引起传感器读取信号故障甚至引起传感器损坏。
7. 为了降低零点漂移对传感器测量精度的影响，需要经常对传感器进行清零操作。
8. 用户不可随意拆卸和改装传感器内部的电路以及机械结构，若因改装导致传感器失效，本公司概不负责。
9. 使用过程中若出现任何本文档中未提到的异常情况，请勿擅自操作，必要时请联系售后人员。

<a id="_Toc12698"></a>
# **附 录**

Arduino例程：

//读取传感器X Y Z数据

#include <Arduino.h>

#include <Wire.h>

#define I2C\_ADDR   0x0A  //7位设备从机地址

HardwareSerial     Serialx(PA10, PA9);

TwoWire          I2C(PB7, PB6);

int16\_t           Buff\_XYZ[3];

void setup()

{

Serialx.begin(115200);

I2C.begin();

delay(500);

}

void loop()

{

I2C.beginTransmission(I2C\_ADDR); // 开始传输到设备

I2C.write(0x03);

I2C.endTransmission();         // 停止传输

I2C.requestFrom(I2C\_ADDR, 6); // 从从设备请求6个字节

while (I2C.available()) // 从设备可能发送少于请求的字节

{

// 读取X

uint8\_t lowByte1 = I2C.read();

uint8\_t highByte1 = I2C.read();

Buff\_XYZ[0] = (highByte1 << 8) | lowByte1;

// 读取Y

uint8\_t lowByte2 = I2C.read();

uint8\_t highByte2 = I2C.read();

Buff\_XYZ[1] = (highByte2 << 8) | lowByte2;

// 读取Z

uint8\_t lowByte3 = I2C.read();

uint8\_t highByte3 = I2C.read();

Buff\_XYZ[2] = (highByte3 << 8) | lowByte3;

}

}