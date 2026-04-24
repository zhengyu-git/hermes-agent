PerfSight是腾讯的游戏性能监控SDK/平台，支持Unity、Unreal引擎，覆盖iOS/Android/主机(Xbox/PS5/Switch)平台，版本3.3.0.23081600，服务方深圳市腾讯计算机系统有限公司。专为游戏打造性能监测闭环，用于游戏研发辅助决策和运营期性能监控。官网https://perfsight.qq.com/，商务perfsighthelper@tencent.com。文档已系统学习保存在~/.hermes/perfsight_docs/（13个文档）和~/.hermes/perfsight_openapi_docs/（11个API文档）。网络搜索哪个能通用哪个，不使用Tavily。
§
WSL环境下网络搜索：当Google、Bing、百度等被限制时，360搜索(so.com)通常可用，哪个能通就用哪个。不使用Tavily。
§
UE5 性能分析（综合Epic官方/CSDN/UWA，含Nanite/Lumen/Unreal Insights）

【多线程架构】Game Thread：物理/动画/网络同步/UI/提交渲染指令。Render Thread：相机矩阵/可见性剔除/阴影/合批/RHI提交。RHI Thread：可选，解耦渲染命令生成和执行。RTHeartBeat：监控线程。

【同步】FFrameEndSync+Fence双缓冲防数据竞争。Game总耗时=Game实际耗时+Frame Sync Time（Sync小→Game瓶颈；大→等Render/GPU）。RHI/GPU：Present/SwapBuffer约占帧耗时10%。

【瓶颈判断】Frame Sync Time>1ms→Render Thread瓶颈；Present Time>3ms→GPU瓶颈；stat unit：Frame≈Draw≈GPU→GPU瓶颈；降分辨率帧率不提升→GPU瓶颈。

【Unreal Insights】
Trace录制：编辑器Trace栏→Start Trace；命令行：Trace.start/stop/pause；自定义Channels在Engine/Config/BaseEngine.ini；安卓包：UECommandLine.txt加-traceframe,cpu,gpu,log,bookmark -statnamedevents -tracehost127.0.0.1；iOS：防火墙开放1981端口同局域网。
ProfileCPU：参数-trace=cpupprofiler,default,frame,log,bookmark；快捷键Ctrl+Shift+,/。生成火焰图。
Memory Insights：参数-tracememory -LLM -LLMCSV；UnrealInsights.exe加-tracedefault,memory。
Timing Insights：右键设帧率阈值线；选中帧→Timing View+Timers；双击函数名显示全程分布。

**【重要】跨平台兼容性**：Mac 录制的 .utrace（protocol 7）与 Windows UnrealInsights 不兼容，报 "Transport buffers are not empty at end of analysis (protocol 7)"。需在同一平台（Mac→Mac，Windows→Windows）分析。UE 5.4 不支持 Mac protocol 7。

**命令行 CSV 导出（UE 5.1+）**：UnrealInsights.exe -OpenTraceFile=xxx.utrace -NoUI -AutoQuit -ExecOnCompleteCmd="@rsp_file.txt"。rsp 每行一个命令，如 TimingInsights.ExportTimerStatistics ./TimerStats.csv。注意：Mac trace 在 Windows 上导出会失败（protocol 不兼容）。

【MemReport】memreport -full；Saved/Profiling/MemReports/*.memreport；rhi.DumpMemory显存类型；rhi.DumpResourceMemory按实例统计；Obj List按类型降序排列。

【Nanite】GPU Driven Pipeline+Cluster剔除+Mesh Shader；软光栅化处理小三角提升3倍效率；Visibility Buffer替代G-Buffer（8-12 vs 16-32 Bytes/Pixel）。

【Lumen】动态全局光照+反射；项目设置→渲染→全局光照→Lumen；Scalability降级保帧率。

【优化方向】DrawCall：stat DrawPrimitive calls；三角面：Triangles drawn；Lumen/Nanite开启时GPU压力大；Shadow/Light最耗时；大量Actor遍历用C++替代蓝图。

【STATS】stat startfile/stopfile录制；UE5用UE4 Frontend分析（Epic Games/UE_4.27/Binaries/Win64/UnrealFrontend.exe）。
§
App Key=116a83402fa27c74fe0e571f4a32a69d（之前记录的多了两个32a，已修正）
§
PDF目录：~/.hermes/pdf_contents/（原文件名+.txt，共7个，均已提取文本）
1. 深入理解计算机系统(中文版).pdf.txt — CSAPP教材，1616页，内容：程序结构和执行、处理器/内存/网络/并发编程
2. VICKY学姐-读书笔记.pdf.txt — 笔记种类（摘要/评注/心得）、写法、形式
3. VICKY学姐-速读技巧.pdf.txt — 5个速读方法：找关键词/略读/SQ3R等
4. VICKY学姐-10种阅读技巧.pdf.txt — 10种阅读方法
5. 原则(瑞达利欧).pdf.txt — 488页，Ray Dalio的 生活+工作 原则
6. 原子习惯.pdf.txt — 285页，James Clear，好习惯累积法则
7. CSAPP第三版扫描版.pdf.txt — 775页，第三版英文翻译版
§
用户是游戏性能工程师，工作于无限暖暖(Infinity Nikki)，使用 UE5、Mac M2、PerfInsight、Unreal Insights 等工具。偏好：希望AI能自主解决问题、联网搜索新方法、自主学习创新；遇到问题时希望我先自己尝试解决，而不是直接说"不行"。vision_analyze 工具不支持本地路径（/mnt/c 或 Windows 路径），只接受 HTTP URL 或特定格式的本地路径。
§
平时多学习自然的中文表达，提升处理事情的能力。用户希望我的中文更地道、更自然，像正常人说话一样，不要太生硬。
§
用户说话风格：喜欢自然、随和的中文，像正常人聊天，可以适当调皮。用户会开玩笑（比如问"智能裤"）。
§
用户已将hermes备份从Gitee迁移到GitHub：仓库 https://github.com/zhengyu-git/hermes-agent，GitHub PAT已嵌入cron任务。Gitee仓库废弃（WSL下不通）。

【每日GitHub备份】
- 任务ID：6f4a3fec3f12，每天凌晨3点执行
- 备份内容：skills/、memories/、cron/、perfsight_docs/、perfsight_openapi_docs/、config.yaml、SOUL.md
- 同时投送到QQ Bot和当前对话

【PerfInsight网站文档导航】已整理到 ~/.hermes/perfsight_websites/README.md：
- 中文文档站：perfsight.qq.com/documents/zh/perfsight/（含Unity/Unreal/控制台/主机/云控/OpenApi/合规）
- OpenAPI文档站：perfsight.qq.com/openapi-docs/zh/（WSL下不稳定，Windows可访问）
- 本地已缓存 perfinsight_docs 和 perfinsight_openapi_docs
§
处理 Windows 电脑的任何东西（文件、系统设置、软件配置、注册表、环境变量等）都必须先询问用户，经过同意才能执行，不能自作主张。
§
WSL网络搜索可用接口（2026年4月实测）：HN Algolia API（hn.algolia.com/api）、dev.to API（dev.to/api/articles）可稳定访问；Google/必应直接被墙；Hacker News直连超时。
§
【工作原则与思维方式】
1. 行动前逐步分析边界情况
2. 优先选择可撤销的操作
3. 不确定时只问一次，不要瞎猜
4. 执行前说明推理过程
5. 遇到模糊需求主动澄清一次，之后自主完成
6. 报错时先自行排查修复，修不好再汇报
7. 简洁回复，少废话，多干活

【权限规则】
- 常规文件操作无需反复确认
- 已审查过的命令直接执行，无需再次询问
- 不要因安全扫描中断工作流，除非是明确的破坏性操作
- 所有 Windows 目录操作必须经过用户同意才能执行，未同意绝不擅自行动
- rm 命令必须先问用户，不能自作主张

【新闻汇报格式】
先发步骤：
```
1. 获取 HN Algolia 热门新闻 → 收到15条 ✓
2. 获取 dev.to AI/科技热门 → 收到20条 ✓
3. 筛选过去一周内容 → 整理中...
```
然后直接发汇总内容，不再一条条拆开发送

【WSL网络可用接口】（2026年4月实测）
- HN Algolia API（hn.algolia.com/api）→ Hacker News热门内容
- dev.to API（dev.to/api/articles）→ 开发者社区热门
- GitHub → 可访问
- Gitee → WSL完全无法访问（100%丢包，企业网络封锁）
- Google/必应 → 被墙

【GitHub备份】
- 仓库：https://github.com/zhengyu-git/hermes-agent
- GitHub PAT：ghp_REDACTED_FOR_BACKUP（已嵌入cron任务）
- 每日凌晨3点自动备份（任务ID：6f4a3fec3f12）
- WSL下GitHub完全畅通；Gitee不通（企业网络封锁）

【用户偏好】
- 喜欢自然随和的中文，像正常人聊天
- 喜欢简洁，不废话
- 希望AI自主解决问题
- 会开玩笑（如"智能裤"）
- 用户是游戏性能工程师（无限暖暖 UE5），用Mac M2
§
PerfInsight模版用"场景结束时间"标识对局，不用开始时间。detail接口参数：username/Appid(大写A)/token/Idstr，不需要env参数。username备选：traviszeng、Shen、dannyzhou，一个失效就换下一个。FPS等指标全部取PerfSight平台算好的值，不准自己算。UnrealInsights路径 D:\Program Files\Epic Games\UE_5.4 是错的，用户会提供正确路径。写Windows bat脚本要用纯英文，避免CMD编码乱码。