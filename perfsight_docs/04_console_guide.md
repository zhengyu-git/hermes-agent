# PerfSight 网页控制台指南

## 1. 客户端上报样例

下面是一段Unity下使用PerfSight的案例代码：

### Initialization

    // 在Awake函数中初始化PerfSight并开启Debug日志
    void Awake()
    {
        PerfSightAgent.EnableDebugMode();
        PerfSightAgent.InitContext("app id");
    }

### 场景标记

您可以在进入和退出场景时调用 MarkLoadlevel 和 MarkLevelFin 接口。

    void Start()
    {
        PerfSightAgent.MarkLoadlevel("Test");
    }

    void OnDestroy()
    {
        PerfSightAgent.MarkLevelFin();
    }


场景加载完成后，调用 MarkLoadlevelCompleted 接口。

    PerfSightAgent.MarkLoadlevelCompleted();

### 设置玩家ID

初始化 Perfsight 并获取用户信息后，调用 SetUserId 接口。设置完成后，我们可以在 "单用户数据"选项卡下的 "玩家ID查询"页面中找到相应记录。

    PerfSightAgent.SetUserId("perfsight_test");



点击数据图表下的"详细"按钮，获取详细的性能数据。

### 设置画质

我们根据以下规则配置和上报游戏画质：
画质总共包含Graphics, Frame rate, Style 三个维度。
我们使用整数进行编码，个位数代表图形维度，十位数代表帧频维度，百位数代表风格维度，最后得到一个三位整数。
我们使用整数进行编码，个位代表Graphics维度，十位代表Frame Rate维度，百位代表Style维度，最终得到一个三位数的整数。
在Graphics维度中，我们使用数字 1 至 6 表示Smooth, Balanced, HD, HDR, Ultra HD, Extreme HDR。
在Frame Rate维度中，我们使用 1 至 3 表示Power Saving, Medium, High。
在Style维度中，我们使用 1 至 5 表示 Classic, Colorful, Realistic, Soft,Movie。


我们选择 Graphics: Smooth, Frame rate: Medium, and Style: Colorful 配置进行进行上报。

    Quality value = 1(Smooth,digits representing the Graphics) + 2(Medium)*10(tens representing the Frame rate) + 2(Colorful)*100(hundreds representing the Style) = 221
    PerfSightAgent.SetQuality(221);


PerfSight后台根据上述规则配置好后，您就可以在 "单用户数据 "选项卡下的 "条件查询 "中使用自定义画质来搜索对局。 
 
### 自定义数据上报

您可以使用 PostNetworkLatency 接口报告网络延迟，也可以使用 PostValueX 系列接口报告自定义数据。在下面的示例中，游戏通过 PostValueX 接口上报了网络相关的收发包数据。

    PerfSightAgent.PostValueI("NetWork", "RecvPerSecond", recv);
    PerfSightAgent.PostValueI("NetWork", "SendPerSecond", send);
    PerfSightAgent.PostNetworkLatency(latency);


您可以在记录详情的网络数据和自定义数据下看到相应的数据。  


## 2. 最近实践

### 性能问题定位

当用户向我们反馈特定设备或场景的性能问题时，我们可以使用 "性能详情"和 "单用户数据"来分析问题。首先，我们可以通过 "性能日流水"页面选择相应的场景，查看流畅度指标最近是否有波动。
当曲线出现异常反应时，我们可以通过条件查询页面进一步筛选出存在性能问题的数据，并点击对局详情进一步分析问题。

### 如何确认新发布版本的性能？

当新版本发布时，我们可以使用版本对比页面来比较之前发布版本和新发布版本的性能，查看新版本是否存在任何性能问题。

## 3. 性能数据统计分析

### 3.1 版本分析

版本分析页面支持以版本周期或自定义时间周期为单位进行，查询特定版本的性能数据。

### 3.2 场景分析

场景分析页面支持以版本周期或自定义时间周期为单位进行，根据版本和场景维度查询性能数据. 

### 3.3 机型分析

机型分析页面支持以版本周期或自定义时间周期为单位进行，根据版本、场景、GPU、SoC 和机型维度上查询性能数据。 

### 3.4 SoC性能分析

SoC性能分析页面支持以版本周期或自定义时间周期为单位进行，根据版本、场景和 SoC维度上查询性能数据。 

## 4. 性能日流水

性能日流水页面可以在版本、场景和机型维度上查询日级和小时级的性能数据。  

## 5. 对比分析

对比分析页签包含版本对比、场景对比、多版本对比。

### 版本对比

版本对比页面支持以版本周期或自定义时间周期为单位进行任选两个版本，选择关注的核心游戏场景，进行对比分析。  

### 场景对比

场景对比页面支持以版本周期或自定义时间周期为单位进行任选两个场景，进行对比分析。 

### 多版本对比

多版本对比页面可选择多个版本，对特定机型和场景进行对比分析。

## 6. 网络延迟

### 网络延迟统计

网络延迟统计页面支持查看所选版本在不同地区、网络类型、运营商下的延迟状态，帮助开发人员排除高网络延迟对游戏性能的影响。   

## 7. 自定义数据分析

在 "自定义数据管理页面 "中，可以针对特定类别启用自定义数据分析。在本例中，我们启用了 "NetWork "类型下"SendPerSecond "和 "RecvPerSecond "的自定义数据分析。  对于启用分析后报告的自定义数据，您可以在自定义分析页面上查看相应数据的统计报表。
