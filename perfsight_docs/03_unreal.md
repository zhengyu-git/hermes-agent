# Unreal 接入指南

接入SDK前，请您仔细阅读PerfSight SDK个人信息处理规则，在同意个人信息处理规则后，请按下述指引接入SDK。


## 导入PerfSight UE4 Plugin

将Plugins文件夹的PerfSight目录拷贝到UE4项目根目录下的Plugins目录（若没有则创建该文件夹）。
点击[文件]->[刷新Visual Studio项目]，然后在 Visual Studio 项目工程中则可以看到 PerfSight plugin 代码以及目录结构，如图所示。 
将 PerfSight 插件源码和项目源码一起编译，编译完成后则可以在 UE4 编辑器中 的[编辑]->[插件]中看到 PerfSight 插件，如图所示。 
注意:PerfSight 默认是不启用的，当勾选 Enabled 选项时可能需要重新启动 Editor。
如果需要在 C++模块中调用代码，需要配置添加相应的模块依赖，如下所示。

PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore","PerfSight" });

## 接口说明

### 1.初始化PerfSight（必选）

static void InitContext(const char* AppID)

功能：初始化PerfSight
参数 AppID: 联系PerfSight获取
返回值：无


### 2. 用户标识（必选）

static void SetUserId(const char* usrId)

功能：设置usrId
参数 usrId: usrId信息
返回值：无

PerfSight既能反映整体大盘的外网性能状况，也提供接口进行单用户的性能回溯，通过UPerfSightHelper::SetUserId设置玩家OpenId之后，在Web端可以通过玩家OpenId查看其性能数据，准确还原玩家游戏性能，此接口可在程序任意位置调用。

### 3.场景标记函数

#### 3.1 场景开始标记（必选）

static void MarkLevelLoad(const char* sceneName);

功能：标记场景开始
参数 sceneName: 场景标识
返回值：无

PerfSight提供按需采集的功能。默认情况下， PerfSight将不会进行数据采集，只有当场景开始标记之后才开始采集数据。PerfSight中的场景是逻辑意义上的表示，和实际的物理场景没有直接关系，因此场景标记的接口可在任意位置调用.

#### 3.2 场景加载结束（可选）

static void MarkLevelLoadCompleted();

功能：标识场景加载结束
参数：无
返回值：无

在场景加载完毕之后通过调用UPerfSightHelper::MarkLoadlevelCompleted()进行场景加载结束的标记，通过调用此接口可计算场景的加载时间。在同一个逻辑场景中(MarkLeveLoad到MarkLevelFin)多次调用此接口无效，其只会执行一次。


#### 3.3 场景结束标记（必选）

static void MarkLevelFin()

功能：标识场景加载结束
参数：无
返回值：无

通过UPerfSightHelper::MarkLevelFin ()函数来标记场景结束（非必要），场景的性能数据将异步发送至PerfSight的服务器，如果没有通过调用MarkLevelFin()函数来结束场景，整个场景的时间以下一次调用MarkLoadlevel ("LevelName ")的时间为场景结束时间，但这可能会对标记区间的FPS计算有影响，为了提高计算的精度最好手动标记结束。默认情况下，PerfSight 只在 MarkLoadlevel 启动后收集数据，并在调用 MarkLevelFin 后完成数据收集。

#### 3.4 标签（可选）

有时我们需要关注场景中一些特殊区域的表现，如团战或与 BOSS 的 PVE 战斗，或者需要排除它们的表现数据，以免影响整体表现数据。

static void BeginTag(const char* TagName)

功能：开始场景内区域标记
参数tagName：标记标识符
返回值：无


static void EndTag()

功能：结束区域标记
参数：无
返回值：无

通过BeginTag接口来标记一个标签的开始，标签和场景是依附的关系，如果当前没有MarkLoadlevel进行标记，则BeginTag无效。如果在场景中调用了 SetQuality，BeginTag中继承MarkLoadlevel中的画质值。通过EndTag结束当前的标签，当 MarkLevelFin 被调用，但标签尚未完成（EndTag 尚未被调用）时，SDK 默认会在 MarkLevelFin 之前自动调用 EndTag 函数。
场景和标签之间的关系如下图所示。

注：标签只是用于对逻辑场景的某段区域进行重点标记，不能视BeginTag对MarkLevelLoad简单的替代，Tag的统计分析行为也会根据实际需要做调整。

#### 3.5 区域剔除(可选)

static void BeginExclude()

功能：开始区域剔除，MMO游戏可以用于剔除挂机
参数：无
返回值：无


static void EndExclude()

功能：结束区域剔除
参数：无
返回值：无

例如，可能需要排除挂机行为或设备在省电模式下的性能数据，这可能会影响整个场景的性能。
场景和剔除区域之间的关系如下图所示。


### 4. 帧率上报（必选）

static void PostFrame(float deltatime)

功能：通过在TICK中调用进行帧率的上报。
参数：deltatime TICK 函数传入的参数。
返回值：无。
如果沒有调用该函数。如果不调用该函数，PerfSight 线程将被阻塞，数据将无法收集或报告。

### 5. 网络延迟上报（可选）

static void PostNetworkLatency(int latency)

功能：上报游戏逻辑网络延时
参数 mills：网络延时值，单位毫秒
返回值：无

PerfSight目前不具备检测网络延迟（RTT 时间）的功能，只提供网络延时的上报分析的功能。因此具体的网络延时由游戏客户端进行计算，并通过UPerfSightHelper::PostNetworkLatencyy()接口来上传。


### 6. 自定义画质信息（可选）

此类接口用于设置自定义画质分档，用于PerfSight的自定义统计、查询。此接口可以在任意位置中调用，可以调用一次，全局生效，也可以调用多次，会以最后一次为准。若使用此接口请提前联系PerfSight提供对应规则。

static void SetQuality(int quality)

功能：设定场景画质
参数 quality：画质值
返回值：无

开发人员可以将画质、图形维度或其他影响性能的维度统一编码为整数，并根据 SetQuality 函数将其传递给 PerfSight。PerfSight 服务器将根据接收到的画质值在多个维度聚合计算。
图中有多Graphics, Frame Rate和Style三种维度。
Graphics维度有六个选项： Smooth、Balanced、HD、HDR、Ultra HD 和 Extreme HDR；Frame Rate维度中有三个选项：Power Saving、Medium和High；Style维度中有五个选项：Classic、Colorful、Realistic、Soft、Movie。这些不同维度的选项组合会影响游戏的性能，因此我们需要统计每个维度组合下的性能情况。
我们使用整数进行编码，个位代表Graphics维度，十位代表Frame Rate维度，百位代表Style维度，最终得到一个三位数的整数。
在Graphics维度中，我们使用数字 1 至 6 表示"Smooth"至"Extreme HDR"。
在Frame Rate维度中，我们使用 1 至 3 表示"Power Saving"至"High"。
在Style维度中，我们使用 1 至 5 表示 "Classic"至 "Movie"。
例如，在上图中，Graphics维度选择了HD，Frame Rate维度选择了High，Style维度选择了Classic，那么最终的编码值计算如下：

Quality value = 3(HD,个位代表Graphics) + 3(High)*10(十位代表Frame rate) + 1(Classic)*100(百位代表Style) = 133
SetQuality(133);

### 7. 版本号设定（可选）

（Android/iOS 可选）

static void SetVersionIden(const char* versionName);

功能：设置游戏资源版本号
参数 versionName：游戏资源版本号
返回值：无

在 Android 或 iOS 平台上，PerfSight SDK 会自动收集应用程序的版本号。可以通过通过SetVersionIden接口进行游戏资源版本号的自主设定。

（Windows 必选） 
 
public static void SetPCAppVersion(const char* version)

功能：设定自定义版本号
参数 version：PC版本号
返回值：无

在 Windows 平台上，SDK不自动采集游戏版本号，需要通过接口设置。

### 8. 自定义上报接口系列接口

#### 8.1 玩家染色（可选）

static void PostEvent(int key, const char* info = "nA")

功能：玩家染色事件
参数 key：事件键
参数 info：事件值
返回值：无

染色事件就像编程语言中的全局变量。它可用于标记玩家的特殊事件或活动，如 moba 游戏中的 "RoomId"、玩家是否启用了作弊插件等。染色事件将随每个场景上传。开发人员可以在 PerfSight 控制台上配置、过滤和查看染色事件。

#### 8.2 自定义数据上报（可选）

static void PostValueF(const char* catgory, const char* key, float a);
static void PostValueF(const char* catgory, const char* key, float a, float b);
static void PostValueF(const char* catgory, const char* key, float a, float b, float c);
static void PostValueI(const char* catgory, const char* key, int a);
static void PostValueI(const char* catgory, const char* key, int a, int b);
static void PostValueI(const char* catgory, const char* key, int a, int b, int c);
static void PostValueS(const char* catgory, const char* key, const char* value);
static void BeginTupleWrap(const char* TupleName);
static void EndTupleWrap();
static void SetPostValueStrMaxLen(int length);

PostValue 系列函数用于报告自定义键值数据。与染色事件不同，键值系列函数支持自定义的聚合分析。
BeginTupleWrap 和 EndTupleWrap 用于定义从 BeginTupleWrap 开始到 EndTupleWrap 结束的分组数据。分组数据将以结构化方式显示，并可作为一个整体进行汇总。自定义数据的具体用法如下：


// 上传单条自定义数据
PostValueF("FuntionTime", "Update", 0.33f)

//上传一组名为 "ModuleCostAnalysis"的数据，表示函数的执行时间
BeginTupleWrap("ModuleCostAnalyse")

PostValueS("ModuleCostAnalyse", "ModuleName", "ShaderModel");
PostValueI("ModuleCostAnalyse", "Parse", 23);//表示 Parse 耗时 23ms 
PostValueI("ModuleCostAnalyse", "exec", 50);// 表示 exec 耗时 50ms

// 结束ModuleCostAnalyse数据
EndTupleWrap()


在 PerfSight 控制台上，该组数据将被配置为二维表格模型，并按如下方式展示：


Timestamp	Category	Key	Value
	FuntionTime	Update	0.33
	ModuleCostAnalyse	Module	ShaderModel
	ModuleCostAnalyse	Parse	23
	ModuleCostAnalyse	exec	50

PerfSight 后端可根据项目需要，基于版本、场景和设备模型聚合和分析自定义数据的最大值、最小值、总和、平均值和分布。


注：PostValueS中的value参数默认字符串长度上限为256，可以通过SetPostValueStrMaxLen接口进行调整。

### 9 网络路由探测(可选)

void TraceRoute(const char* ipAddress)


该接口获取对应ip地址的路由，默认都使用ICMP进行路由探测，由于Android10以下设备没有icmp的发包权限，Android10以下设备使用udp协议进行路由探测，为了防止数据包被对应服务器过滤，需要在服务器部署agent服务来监听udp数据包。
路由获取信息：1、目标网址；2、路由转发路径；3、每一跳的路由IP地址；4、每一跳的网络时延



### 10 事件漏斗（Windows，可选）

static void PostStepEvent(const char *category, int stepId, int status, int code, const char *msg, const char *extDefinedKey, bool authorize, bool finish);


功能：玩家事件漏斗
参数 category：步骤事件的类别。请为所有事件设置相同的值。
参数 stepId：步骤的 id。
参数 status：步骤的状态
参数 code：步骤的错误代码。
参数 msg：步骤的错误信息。
参数 extDefinedKey、authorize：未使用。
参数 finish： 当前步骤是否为工作流的最后一步。
返回值：无


当登陆开始时调用
PostStepEvent("category", 0, 0, 0, "","", false, false);

当登陆成功时调用:
PostStepEvent("category", 1, 0, 0, "","", false, false);

当登陆失败时调用:
PostStepEvent("category", 2, 404, 0, "","", false, false);

### PerfSight上报配置

void SetUploadServerURL(const char* url)


功能：设置上报域名，初始化前调用
参数 url：上报域名，请根据平台区分上报域名
返回值：无



Android/Harmony域名配置

国内环境上报域名：https://android.perfsight.qq.com
海外环境上报域名：https://android.perfsight.wetest.net


iOS配置

国内环境上报域名：https://ios.perfsight.qq.com
海外环境上报域名：https://ios.perfsight.wetest.net


Windows/主机配置

国内环境上报域名：https://pc.perfsight.qq.com
海外环境上报域名：https://pc.perfsight.wetest.net


XBOX配置

Xbox平台还需要设置SDK工作路径，一般填写D盘即可。


void SetWorkSpace(const char* work_space)


功能：设置XBOX工作路径，初始化前调用
参数 url：路径名、一般填写D盘即可。
返回值：无


### 调试

通过EnableDebugMode接口打开调试信息，确认PerfSight接入是否成功。

Android

通过Android的logcat查看接入是否成功， 通过adb shell进入shell环境，输入"logcat | grep -i perfsight"查看debug信息，如果看到"PerfSight init finished"的字样则意味着初始化成功。 进入场景，操作游戏，结束场景，查看debug信息，如果有"file send successfully"的字样代表数据上传成功。

OpenHarmony

通过鸿蒙的hilog查看接入是否成功， 通过hdc shell进入shell环境，输入"hilog | grep -i perfsight"查看debug信息，如果看到"PerfSight init finished"的字样则意味着初始化成功。 进入场景，操作游戏，结束场景，查看debug信息，如果有"file send successfully"的字样代表数据上传成功。

iOS

iOS平台可通过NSlog查看关键字为"perfsight"的关键字获取log信息。

PC

PC平台日志在PerfSight.DLL的同级目录下查看./quality/performance 如perf.xxxx.xxx.log，查看时需要关闭游戏;初始化成功关键字：PerfSight init result: 0，场景结束后日志文件夹下有perf_data.pb_XX生成，代表数据上传成功。

在PerfSight页面中确认数据是否上报成功，国内业务请访问 https://tapm.wetest.qq.com ，国外业务请访问 https://tapm.wetest.net 。访问分栏 "单用户数据"->"玩家ID查询"，通过使用SetOpenId接口设置的id进行查询，若未设置通过"NA"进行查询，可以成功查询数据即为上报成功了。大盘数据在首次上报1小时左右刷新，刷新前请使用"玩家ID查询"查验数据。
