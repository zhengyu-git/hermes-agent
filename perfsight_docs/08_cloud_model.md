# 云控机型分档

## 概述
云控机型分档功能可以快速实现针对Android和iOS的机型分档功能，支持Unity以及Unreal引擎。PerfSight（APM）提供了默认的机型分档配置文件，业务可以针对实际需要进行微调。

## Unity接口
```cpp
int CheckDeviceClass(string domainName)
int CheckDeviceClassByString(string domainName, string json)//通过传入json字符串判断机型的档位
```

## Unreal接口
```cpp
//iOS平台
int GetDeviceLevelByQcc(const char* domainName, const char* fileDir);
int GetDeviceLevelByString(const char* domainName, const char* json);//通过传入json字符串判断机型的档位

//Android平台
int GetDeviceLevelByQcc(const char* domainName, const char* gpuFamily);// gpuFamily为GPU的型号
int GetDeviceLevelByString(const char* domainName, const char* gpuFamily, const char* json);//通过传入json字符串判断机型的档位
```

## 1. 云控文件定义

### 1.1 整体结构
云控文件采用 JSON 格式定义，整体包含3个部分：
- 链接版本号定义
- 配置域列表定义
- 配置域定义

在同一云控文件中可以包含多个配置域的定义，各个配置域之间是隔离的。调用接口CheckDeviceClass(const char* domain)进行判定时，传入指定的配置域域名进行规则判定。

- 链接版本号：SDK会在初始化云控功能的同时基于此版本号到云端下载最新的配置文件
- 配置域列表：定义了此配置文件中具体的配置域，也是CheckDeviceClass(const char* domain)函数中需要传入的名字
- 配置域：JSON对象类型，每一个定义在"配置域列表"中的名字都需要有对应的配置域定义

由于Android和iOS"配置域"的定义有轻微区别，因此针对Android和iOS平台需要两个单独的配置文件。

### 1.2 配置域定义

Android和iOS的"配置域"定义稍有区别。Android由于碎片化的问题，因此设置了多个配置项：
- 机型分档数定义
- 详细分档定义
- 总控制开关定义
- 判定维度开关定义
- 正则表达式开关
- 模拟器档位开关（已废弃）
- 白名单定义
- 区间阈值判定定义

iOS中只有：
- 机型分档数定义
- 详细分档定义
- 正则表达式开关
- 机型白名单
- SoC白名单

#### 配置项说明：
- 机型分档数: 表示分档的个数，如m
- 详细分档: JSON数组(整型数组)，表示详细的分档值，如[C1, C2, …, Cm]。建议C1>=1，0作为一个保留档位，当发生内部错误时返回0
- 默认分档: 如果只配置了白名单，没有进行区间阈值判定的配置时，白名单中没有匹配则返回此值
- 控制开关: 分为总控制开关（switchops）以及区间阈值判定开关（andopts）

#### 控制开关位运算定义：
1. 机型白名单的开关值为 2^1
2. GPU白名单的开关值为 2^2
3. SoC白名单的开关值为 2^3
4. 厂商白名单的开关值为 2^4
5. 分辨率区间阈值判定维度开关值为 2^5
6. 内存区间阈值判定开关值为 2^6
7. CPU频率区间阈值判定开关值为 2^7
8. CPU核数区间阈值判定值为 2^8
9. GPU维度区间阈值判定值为 2^9
