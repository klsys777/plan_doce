# 15mot灵巧手V2\(手指驱动器\) CANFD通信协议

|版本|编制人|信息|
|---|---|---|
|v1\.0|唐杰|初稿|

## 协议说明

1. CAN通讯协议：波特率5Mbps 采样点75%，采⽤标准帧格式。

2. CAN ID\(11 Bit\)：Bit0\~Bit3为手指索引，Bit4\~Bit10命令码。

3. CAN DLC\(4 Bit\)：数据字节数，根据不同命令携带不同的字节数。

4. CAN DATA\(0\-64 Byte\)：帧包携带的数据，根据不同命令携带相应的数据，数据采⽤⼤端模式。

5. 大拇指到小拇指依次CANID为1\-5。



CAN FD 的格式中，DLC 的含义和普通的 CAN 格式有些不同，DLC 的值是 0 到 8 的时候含义和CAN2\.0的是⼀样的，9 到 15 的值的含义如下表所⽰：

|DLC|9|10|11|12|13|14|15|
|---|---|---|---|---|---|---|---|
|LEN|12|16|20|24|32|48|64|

## 命令说明

### CAN\_CMD\_MOTOR\_DISABLE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|BIT0\~BIT3||BIT4\~BIT10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||1|0||

### CAN\_CMD\_MOTOR\_ENABLE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||2|1|Byte0：0x07 低3bit作电机选择|
|驱动器返回|手指索引||2|1|Byte0: <br>数据类型: uint8 <br>0：使能成功<br>\!0：使能失败（读取错误码）|

### CAN\_CMD\_MOTOR\_CALIBRATION

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||3|0||
|驱动器返回|手指索引||3|1|Byte0: <br>数据类型: uint8 <br>0：开始校准 <br>\!0：校准失败（读取错误码）|

### CAN\_CMD\_DRIVE\_RESET

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||4|0|注释：软件复位|

### CAN\_CMD\_SET\_HOME

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||5|0||

### CAN\_CMD\_SET\_ZERO

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||6|0||

### CAN\_CMD\_SET\_ID

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引<br>||7|1<br>|Byte0 <br>数据类型：uint8 <br>数据：控制器期望ID|
|驱动器返回|手指索引||7|1|Byte0 <br>数据类型：uint8 <br>数据：控制器当前ID|

### CAN\_CMD\_SET\_CAN\_TIMOUT

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||8|2|Byte0\~Byte1<br>数据类型：uint16<br>数据：CAN超时保护时间<br>单位：us|
|驱动器返回|手指索引||8|2|Byte0\~Byte1<br>数据类型：uint16<br>数据：当前CAN超时保护时间<br>单位：us|

### CAN\_CMD\_SET\_BANDWIDTH

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||9|2|Byte0\~Byte1: <br>数据：电流环带宽<br>单位：HZ|
|驱动器返回|手指索引||9|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前电流环带宽 <br>单位：Hz|

### CAN\_CMD\_SET\_LIMIT\_CURRENT

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||10|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：电流限制 \*10 <br>单位：A|
|驱动器返回|手指索引||10|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前电流限制 \*10 <br>单位：A|

### CAN\_CMD\_SET\_LIMIT\_VEL

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||11|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：速度限制 \*10 <br>单位：turn/s|
|驱动器返回|手指索引||11|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前速度限制 \*10 <br>单位：turn/s|

### CAN\_CMD\_SET\_LIMIT\_PPOS

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引<br>||12|6|Byte0\~Byte5: <br>数据类型： 3\*uint16 <br>数据：关节1\-3位置正极限 \*10 <br>单位：turn|
|驱动器返回|手指索引||12|6|Byte0\~Byte5: <br>数据类型： 3\*uint16 <br>数据：当前关节1\-3位置正极限 \*10 <br>单位：turn|

### CAN\_CMD\_SET\_LIMIT\_NPOS

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||13|6|Byte0\~Byte5: <br>数据类型： 3\*uint16 <br>数据：关节1\-3位置负极限 \*10 <br>单位：turn|
|驱动器返回|手指索引||13|6|Byte0\~Byte5: <br>数据类型： 3\*uint16 <br>数据：当前关节1\-3位置负极限 \*10 <br>单位：turn|

### CAN\_CMD\_SET\_MOS\_TEMPERATURE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||14|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：MOS过温阈值 \*10 <br>单位：℃|
|驱动器返回|手指索引||14|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前MOS过温阈值 \*10 <br>单位：℃|

### CAN\_CMD\_SET\_OVER\_VOLTAGE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||15|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：过压保护 \*10 <br>单位：V|
|驱动器返回|手指索引||15|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前过压保护 \*10 <br>单位：V|

### CAN\_CMD\_SET\_UNDER\_VOLTAGE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||16|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：欠压保护 \*10 <br>单位：V|
|驱动器返回|手指索引||16|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前欠压保护 \*10 <br>单位：V|

### CAN\_CMD\_SET\_CONTROL\_MODE

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||17|1|Byte0: <br>数据类型： uint8<br>数据：<br>0: 空闲模式<br>1：转矩爬升模式<br>2: ADRC模式<br>3: 直接转矩模式<br>4：转速爬升模式<br>5：直接转速模式<br>6：位置过滤模式<br>7：轮廓位置模式<br>8：直接位置模式|
|驱动器返回|手指索引||17|1|Byte0: <br>数据类型： uint8<br>数据：当前模式|



### CAN\_CMD\_MOTOR\_CONCTOL（上位机通信使用，整机不适用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||18|14|Byte0：Motor Index\(按位 00000111，不控的手指数据发0，最高位0控关节，最高位1控电机\)<br>Byte1\~Byte6: <br>数据类型：3\*int16\_t<br>数据：关节1\-3目标位置（**放大100倍下发）**<br>数据实际范围：\-120\~120<br>单位：角度<br><br>Byte7\~Byte12: <br>数据类型：3\*int16\_t<br>数据：关节1\-3目标速度（**放大10倍下发）**<br>数据实际范围：\-700\~700<br>单位：角度/s <br><br>Byte13\~Byte18: <br>数据类型：3\*int16\_t<br>数据：关节1\-3目标加速度（**放大10倍下发）**<br>数据实际范围：0\~3000<br>单位：角度/s²|
|驱动器返回|手指索引||18|14|Byte0\~Byte5: <br>数据类型：3\*int16\_t <br>数据：关节1\-3⻆度（**放大100倍）**<br>单位：角度<br><br>Byte6\~Byte11: <br>数据类型：3\*int16\_t <br>数据：关节1\-3速度（**放大10倍）**<br>单位：角度/s <br><br>Byte12\~Byte17: <br>数据类型：3\*int16\_t<br>数据：关节1\-3电流（**放大10倍）**<br>单位：角度/s²<br><br>Byte18\~Byte23: <br>错误码<br>数据类型： 3\*uint16 <br>数据：错误码<br>注释: 关节1\-3<br><br>Byte24\~Byte31: 手指传感器数据 （需放缩成16位）<br>int8\_t x1,y1,z1；<br>int8\_t x2,y2,z2;<br>uint8\_t tof,;<br>uint8\_t tof\_confidence\_degree;<br><br>Byte32\~Byte33：手掌板数据（需放缩成16位）,hziizai<br>int8\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;<br><br>注：<br>位置范围：<br>传感器数据范围：（原始int16\_t 和 uin16\_t范围）<br>int16\_t范围：<br>\#define SENSOR\_DATA\_INT\_MIN \-3000<br>\#define SENSOR\_DATA\_INT\_MAX 3000<br><br>uint16\_t范围：<br>\#define SENSOR\_DATA\_UINT\_MIN 0<br>\#define SENSOR\_DATA\_UINT\_MAX 3000|

放缩函数：

```Python
int16_t int8_to_int16(int8_t value, float min, float max) {
    float range = max - min;

    float res = ((float)value + 128.0f) * range / 255.0f + min;

    return (int16_t)res;
}

uint16_t uint8_to_uint16(uint8_t value, float min, float max) {
    float range = max - min;

    float res = ((float)value * range / 255.0f) + min;

    if (res < 0) return 0; 

    return (uint16_t)(res + 0.5f);
}
```





**单个电机控制：**

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||xx|xx|Byte0\~Byte11: <br>关节index：uint8\_t\(0、1、2\)<br>数据类型：3\*float <br>数据：⽬标⻆度、⽬标速度、⽬标加速度 <br>|





### CAN\_CMD\_SAVE\_CONFIG

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||19|0||
|驱动器返回<br>|手指索引<br>||19|1<br>|Byte0: <br>数据类型: uint8 <br>bit0：<br>0：保存成功<br>1：保存失败|

### CAN\_CMD\_GET\_ERROR

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||20|0||
|驱动器返回<br>|手指索引<br>||20|6|Byte0\~Byte5: <br>数据类型： 3\*uint16 <br>数据：错误码<br>注释: 关节1\-3<br>Bit0 = 1: ADC采样错误<br>Bit1 = 1: 过压错误<br>Bit2 = 1: ⽋压错误<br>Bit3 = 1: 过流错误<br>Bit4 = 1: A相过流错误<br>Bit5 = 1: B相过流错误<br>Bit6 = 1: C相过流错误<br>Bit7 = 1: 飞车错误<br>Bit8 = 1: 缺相错误<br>Bit9 = 1: 硬件错误<br>Bit10 = 1: 超速错误<br>Bit11 = 1:位置超限<br>Bit12 = 1: CAN超时 |

### CAN\_CMD\_ERROR\_CLEAR

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||21|0||

### CAN\_CMD\_GET\_FW\_VERSION

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||22|0||
|驱动器返回<br>|手指索引<br>||22|4|Byte0\~Byte1: <br>数据类型： uint16 <br>数据： FW\_VERSION\_M AJOR（两位⼗进制整数\<100） <br><br>Byte2\~Byte3: <br>数据类型： uint16 <br>数据： FW\_VERSION\_M INOR（四位⼗进制整数\<10000）|

### CAN\_CMD\_ERASE\_APP\_BACK

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||23|0||
|驱动器返回<br>|手指索引<br>||23|1|Byte0: <br>数据类型: uint8 <br>bit1：<br>0：擦除成功 <br>1：擦除失败|

### CAN\_CMD\_WRITE\_APP\_BACK

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||24|4|Byte0\~Byte3: <br>数据类型: uint32<br>注释：固件写入|
|驱动器返回<br>|手指索引<br>||24|1|Byte0: <br>数据类型: uint8 <br>0：写入完成 <br>\!0：写入失败|

### CAN\_CMD\_CHECK\_APP\_BACK

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||25|4|CRC32校 验值|
|驱动器返回<br>|手指索引<br>||25|8|Byte0\~Byte3: <br>数据类型：uint32 <br>数据：固件⼤⼩ <br>单位：byte <br><br>Byte4\~Byte7: <br>数据类型：uint32 <br>数据：CRC32校 验值|

### CAN\_CMD\_DFU\_START

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||26|0||

### CAN\_CMD\_DEFAULT\_CONFIG

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||27|0|注释：还原到默认设置|



### CAN\_CMD\_POSITION\_PLAN（暂时不使用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||28|0|Byte0\~Byte11: <br>数据类型：3\*float<br>数据：关节1\-3移动速度<br>单位：turn/s<br><br>Byte12\~Byte23: <br>数据类型：3\*float<br>数据：关节1\-3加速度<br>单位：turn/s^2<br><br>Byte24\~Byte35: <br>数据类型：3\*float<br>数据：关节1\-3减速度<br>单位：turn/s^2|

### CAN\_CMD\_POSITION\_PLAN\_（暂时不使用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||29|0|Byte0\~Byte11: <br>数据类型：3\*float<br>数据：关节1\-3移动速度<br>单位：turn/s<br><br>Byte12\~Byte23: <br>数据类型：3\*float<br>数据：关节1\-3加速度<br>单位：turn/s^2<br><br>Byte24\~Byte35: <br>数据类型：3\*float<br>数据：关节1\-3减速度<br>单位：turn/s^2|



### CAN\_CMD\_SINGLE\_MOTOR\_CONCTOL（上位机通信使用，整机不适用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||30|10|Byte0：<br>数据类型：uint8\_t<br>数据：关节/电机序号\(0\~2\)，单个关节/电机控制，最高位0控关节，最高位1控电机<br><br>Byte1\~Byte2: <br>数据类型：int16\_t<br>数据：关节⽬标⻆度（**放大100倍下发）**<br>数据实际范围：\-120\~120<br>单位：角度<br><br>Byte3\~Byte4:<br>数据类型：int16\_t<br>数据：关节⽬标速度（**放大10倍下发）**<br>数据实际范围：\-700\~700<br>单位：角度/s <br><br>Byte5\~Byte6: <br>数据类型：int16\_t<br>数据：关节⽬标加速度 （**放大10倍下发）**<br>数据实际范围：0\~3000<br>单位：角度/s²<br>|
|驱动器返回|手指索引||30|10|Byte0：<br>数据类型：uint8\_t<br>数据：关节/电机序号\(0\~2\)，单个关节/电机控制，最高位0控关节，最高位1控电机<br><br>Byte1\~Byte2： 2Byte<br>数据类型：int16\_t <br>数据：关节⻆度（**放大100倍）**<br>单位：角度<br><br>Byte3\~Byte4   2Byte<br>数据类型：int16\_t <br>数据：关节速度（**放大10倍）**<br>单位：角度/s <br><br>Byte5\~Byte6：  2Byte<br>数据类型：int16\_t  <br>数据：电流（**放大10倍）**<br>单位：<br><br>Byte7\~Byte12:    6Byte<br>错误码<br>数据类型： 3\*uint16 <br>数据：错误码<br>注释: 关节1\-3<br><br>Byte13\~Byte20: 手指传感器数据   8Byte<br>int8\_t x1,y1,z1；<br>int8\_t x2,y2,z2;<br>uint8\_t tof,;<br>uint8\_t tof\_confidence\_degree;<br><br>Byte21\~Byte32: 手掌板数据     12Byte<br>int8\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4；<br><br>注：<br>传感器数据范围：（原始int16\_t 和 uin16\_t范围）<br>int16\_t范围：<br>\#define SENSOR\_DATA\_INT\_MIN \-3000<br>\#define SENSOR\_DATA\_INT\_MAX 3000<br><br>uint16\_t范围：<br>\#define SENSOR\_DATA\_UINT\_MIN 0<br>\#define SENSOR\_DATA\_UINT\_MAX 3000|



```Python
int16_t int8_to_int16(int8_t value, float min, float max) {
    float range = max - min;

    float res = ((float)value + 128.0f) * range / 255.0f + min;

    return (int16_t)res;
}


uint16_t uint8_to_uint16(uint8_t value, float min, float max) {
    float range = max - min;

    float res = ((float)value * range / 255.0f) + min;

    if (res < 0) return 0; 

    return (uint16_t)(res + 0.5f);
}
```



|Byte0\~Byte5: <br>数据类型：3\*int16\_t <br>数据：关节1\-3⻆度（**放大100倍）**<br>单位：角度<br><br>Byte6\~Byte11: <br>数据类型：3\*int16\_t <br>数据：关节1\-3速度（**放大10倍）**<br>单位：角度/s <br><br>Byte12\~Byte17: <br>数据类型：3\*int16\_t<br>数据：关节1\-3电流（**放大10倍）**<br>单位：角度/s²<br><br>Byte18\~Byte23: <br>错误码<br>数据类型： 3\*uint16 <br>数据：错误码<br>注释: 关节1\-3<br><br>Byte24\~Byte31: 手指传感器数据<br>int8\_t x1,y1,z1；<br>int8\_t x2,y2,z2;<br>uint8\_t tof,;<br>uint8\_t tof\_confidence\_degree;<br><br>手掌板数据<br>int8\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;<br><br>注：<br>位置范围：<br>\#define POS\_MIN \-120\.0f<br>\#define POS\_MAX  120\.0f<br><br>速度范围：<br>\#define VEL\_MIN \-700\.0f<br>\#define VEL\_MAX  700\.0f<br><br>电流范围：<br>\#define ACC\_MIN  0\.0f<br>\#define ACC\_MAX  3000\.0f<br><br>传感器数据范围：（原始int16\_t 和 uin16\_t范围）<br>int16\_t范围：<br>\#define SENSOR\_DATA\_INT\_MIN \-3000<br>\#define SENSOR\_DATA\_INT\_MAX 3000<br><br>uint16\_t范围:<br>\#define SENSOR\_DATA\_UINT\_MIN 0<br>\#define SENSOR\_DATA\_UINT\_MAX 3000|
|---|



### CAN\_CMD\_GET\_TEMPERATURE（暂时不使用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||31|0||
|驱动器返回|手指索引||31|2|Byte0\~Byte1: <br>数据类型： uint16 <br>数据：当前MOS过温阈值 \*10 <br>单位：℃|

### CAN\_CMD\_GET\_CURRENT

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||32|0||
|驱动器返回|手指索引||32|FDCAN\_DLC\_BYTES\_12|Byte0\~Byte11: <br>数据类型： float<br>数据：关节1\-3电流|

### CAN\_CMD\_GET\_FINGER\_PRESS\_DATA

手指压力板数据，在slave上

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||33|0||
|驱动器返回|手指索引||33|FDCAN\_DLC\_BYTES\_12<br>|Byte0\~Byte15: 手指传感器数据<br>int16\_t x1,y1,z1；<br>int16\_t x2,y2,z2;<br>uint16\_t tof,;<br>uint16\_t tof\_confidence\_degree;|



整机上通信，数据需要缩放！



### CAN\_CMD\_GET\_HAND\_BOARD\_DATA

手掌板压力数据，在Master上，实际不与驱动板通信，放在此处记录使用

数据大小：24Byte

int16\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;


|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||34|0||
|驱动器返回|手指索引||34|FDCAN\_DLC\_BYTES\_24|int16\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;|





### CAN\_CMD\_GET\_FINGER\_STATUS\_DATA（上位机通信使用，整机不适用）

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||35|0||
|驱动器返回|手指索引||35||Byte0\~Byte11: <br>数据类型： float<br>数据：关节1\-3位置<br>Byte12\~Byte23: <br>数据类型： float<br>数据：关节1\-3电流<br>Byte24\~Byte29：<br>数据类型： uint16\_t<br>数据：关节1\-3错误码<br>Byte30\~Byte45: 手指传感器数据<br>int16\_t x1,y1,z1；<br>int16\_t x2,y2,z2；<br>uint16\_t tof;<br>uint16\_t tof\_confidence\_degree;<br>Byte46\~Byte69: 手掌传感器数据（只在大拇指中有数据）<br>int16\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;|

5A A5 00 00 03 00 48 23 00 

BD 80 74 5E 3D C9 3F B4 3C 03 27 53 

00 00 00 00 BE 8D 24 8A BE A2 21 C6 

00 00 00 00 00 00 

03 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 

FF 00 27 00 8C FF 5F 01 EB 00 1B FE FC 02 F6 02 34 FA B0 01 CD 02 B0 EB 

9B 27 

5A A5 00 01 03 00 30 23 00 BB 5A 91 99 BA A2 94 69 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 86 16 

5A A5 00 02 03 00 30 23 00 BB 98 6E A7 39 9D 72 1B 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 F6 88 

5A A5 00 03 03 00 30 23 00 BB 6B 8B 82 BA 79 DC 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 33 8F 

5A A5 00 04 03 00 30 23 00 BB B4 B9 D8 BB 32 D6 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 03 00 2C C5



### CAN\_CMD\_MOTOR\_CONTROL\_IN\_ROBOT（整机通信使用，上位机不适用）



|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||18|64|Byte0\~Byte29: 五个手指目标位置   5\*3\*uint16\_t \(30Byte\)<br>大拇指=》小拇指<br>每个数据类型：3\*uint16\_t   <br>数据：关节1\-3目标位置  （放大100倍下发）<br>单位：角度<br><br>Byte30\~31：目标速度      单位：角度/s    （放大10倍下发）<br>Byte32\~33：目标加速度   单位：角度/s²   （放大10倍下发）|
|驱动器返回|手指索引||18|64|各个手指数据：大拇指=》小拇指<br>每个手指：5\*6Byte  （30Byte）<br>- 位置、关节1\-3⻆度（3\*uint16\_t）<br><br>uint8\_t sensor\_mode  \(1Byte\)  值1：Sensor\_data1，值0：Sensor\_data2<br><br>=================Start==============<br>以下26个字节，分2帧交替发送，通过sensor\_mode值来识别<br>Sensor\_data1\(26Byte\)<br>大拇指=》小拇指<br>前3个手指（24Byte）\+第4个手指x1、y1（2Byte）<br><br>Sensor\_data2\(26Byte\)<br>第4个手指z1、x2、y2、z2、tof、tof\_confidence\_degree（6Byte） \+ 第5个手指数据（8Byte） \+ 手掌板数据（12Byte）<br>=================End==============<br>错误码：uint16\_t （2Byte） 标识15个电机的某个电机有错误（第0位\~第14位，大拇指=》小拇指电机）（具体错误信息，需停止下发控制指令，下发CAN\_CMD\_GET\_ERROR查询）<br>当前控制模式：uint8 \(1Byte\)<br>|



|Byte30\~Byte45: 手指传感器数据<br>int16\_t x1,y1,z1；<br>int16\_t x2,y2,z2;<br>uint16\_t tof;<br>uint16\_t tof\_confidence\_degree;<br>Byte46\~Byte69: 手掌传感器数据（只在大拇指中有数据）<br>int16\_t x1,y1,z1,x2,y2,z2,x3,y3,z3,x4,y4,z4;|
|---|

### 37\.CAN\_CMD\_WRIT\_PARA

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||37|24<br>|Byte0\~Byte1<br>数据类型：uint16<br>数据：CAN超时保护时间<br>单位：us<br><br>Byte2\~Byte3: <br>数据：电流环带宽<br>单位：HZ<br><br>Byte4\~Byte5: <br>数据类型： uint16 <br>数据：电流限制 \*10 <br>单位：A<br><br>Byte6\~Byte7: <br>数据类型： uint16 <br>数据：速度限制 \*10 <br>单位：turn/s<br><br>Byte8\~Byte13: <br>数据类型： 3\*uint16 <br>数据：关节1\-3位置正极限 \*10 <br>单位：turn<br><br>Byte14\~Byte19: <br>数据类型： 3\*uint16 <br>数据：关节1\-3位置负极限 \*10 <br>单位：turn<br><br>Byte20\~Byte21: <br>数据类型： uint16 <br>数据：过压保护 \*10 <br>单位：V<br><br>Byte22\~Byte23 <br>数据类型： uint16 <br>数据：欠压保护 \*10 <br>单位：V<br><br>|
|驱动器返回|手指索引||37|1<br>|写入成功：1<br>写入失败：0|

### 38\.CAN\_CMD\_OPEN\_CRRENT\_LOOP

|数据域|CAN ID \(11 Bit\)|||CAN DLC|数据区|
|---|---|---|---|---|---|
|大小|Bit0\~Bit3||Bit4\~Bit10|Bit0\~Bit3|Byte0\~Byte63|
|用户发送|手指索引||38|2<br>|Byte0\~Byte1<br>数据类型：uint16<br>数据：速度<br>单位：rad/s<br>|
|驱动器返回|手指索引||38|0<br>||



