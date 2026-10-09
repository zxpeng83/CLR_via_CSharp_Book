# 第22章　CLR寄宿和AppDomain

本章按八轮追问梳理“进程如何运行托管代码”和“一个进程怎样容纳多组程序集”。第四版的.NET Framework寄宿、AppDomain、Remoting与权限模型是历史主线；现代.NET的apphost、CoreCLR与AssemblyLoadContext另标范围。原问答中的站点、公寓、办公室、书架等例子各有用途，不能据类比推导内存破坏隔离或固定成本。

目录：[Framework寄宿](#c22-host) · [现代运行环境](#c22-environment) · [加载与映射](#c22-mapping) · [AppDomain能力](#c22-domain) · [跨域对象](#c22-marshaling) · [Loader与共享](#c22-loader) · [卸载](#c22-unload) · [资源监视](#c22-arm) · [IIS例子](#c22-sites) · [域与程序集](#c22-assemblies) · [现代ALC](#c22-modern) · [纠错](#c22-corrections)

<a id="c22-host"></a>
## CLR为什么需要宿主

CLR执行引擎本身需要在进程里工作；“寄宿”指宿主将运行时加载到自己的进程、初始化它并请求执行托管代码。Windows先按照PE、体系结构及相关标志创建相应32/64位进程，32位程序在适用64位系统上可使用WoW64；C#源码并不是Windows加载器直接执行的对象。

原房东/房客比喻：进程给CLR提供工作场所，CLR负责托管执行。Framework的本体常为clr.dll；dotnet.exe属于现代.NET的一种入口，xxx.vshost.exe是旧Visual Studio调试宿主例子，不能把这两个名字定义成一切C#程序必有的原生房东。Framework托管exe也能成为自身进程的入口。IIS、SQL Server、PowerShell及自己编写的原生宿主都是不同宿主用途，具体运行时与策略取决于产品年代。

<a id="c22-shim"></a>
原接力赛图保留，范围为Framework传统Windows托管exe启动，图中Main的JIT是通常IL路径示意：

```text
Windows 内核
   ↓ ① 创建进程，发现是 .NET 程序
   ↓    于是加载 MSCorEE.dll（垫片）
MSCorEE.dll（垫片 / Shim）
   ↓ ② 根据程序的目标框架版本，找到对应的 clr.dll
   ↓    调用 COM 接口把 CLR 请进来
CLR（clr.dll）
   ↓ ③ 创建默认 AppDomain
   ↓ ④ 加载入口程序集，JIT 开始编译 Main 方法
你的 Main 方法终于跑起来了
```

MSCorEE是Microsoft Component Object Runtime Execution Engine，shim是垫片/入口选择层，安装路径可见Windows系统目录，32/64位文件与重定向须按进程位数看。它不是clr.dll本体；原“前台接待”类比表示先接待exe，再转给合适运行时版本。示例目标v4.0.30319表示CLR4版本，不是说每个Framework产品版本各有一套独立CLR大版本。

<a id="c22-com"></a>
### COM接口链是怎样的合同

COM在这里解释稳定的二进制接口、标识符及调用合同；宿主取得接口后“遥控”运行时。原简化调用链修正IMetaHost拼名，保原v4.0.30319输入；这是省略真实参数/HRESULT检查的API形状，不是完整可编译C++程序：

```text
CLRCreateInstance()        // 拿到 ICLRMetaHost 工厂
  → GetRuntime()           // 指定版本（如 "v4.0.30319"），拿到 ICLRRuntimeInfo
    → GetInterface()       // 拿到 ICLRRuntimeHost（这才是真正控制 CLR 的把手）
```

CLRCreateInstance用CLSID_CLRMetaHost/IID_ICLRMetaHost取得ICLRMetaHost，GetRuntime选择运行时并取得ICLRRuntimeInfo，GetInterface取得ICLRRuntimeHost。原把CLR泛称“安装时注册的COM服务器”需限定：Framework4新增寄宿接口复用AddRef/Release、QueryInterface等COM合同，但没有CoCreateInstance注册表激活、apartment与aggregation模型；不能从接口长得像COM推出该链必须先注册激活。[真实工厂入口](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clrcreateinstance-function)、[Framework4接口范围](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clr-hosting-interfaces-added-in-the-net-framework-4-and-4-5)

<a id="c22-host-managers"></a>
原五种用途表与Q1概览的同五项对应，保其遥控器解释，限制项按历史具体接口合同理解：

<a id="c22-host-five-table"></a>
| 能力 | 通俗解释 |
| --- | --- |
| **设置宿主管理器**​ | 告诉 CLR："内存分配你别全权管，遇到 XX 情况要先问我" |
| **获取 CLR 管理器**​ | 取得CLR管理接口，按其具体合同配置保护类别、策略与通知等 |
| **初始化并启动 CLR**​ | 按下开机键 |
| **加载程序集并执行**​ | 让 CLR 开始干活 |
| **停止 CLR**​ | 关机（但其实 CLR 不支持真正卸载，只能随进程死） |

宿主管理器由宿主提供：内存分配、线程调度/同步、程序集加载、GC启停通知与超时处理等可通过特定回调合同协调。CLR管理接口供宿主调用：如保护类别、策略和AppDomain卸载、CLR停止、堆栈溢出等事件通知。原SQL Server巨大C++宿主/CLR作为插件的比喻说明为什么需协调资源；并非宿主可以任意重写每个运行时决策。

原双向图保留；上方表示提供宿主管理器，下方把取得CLR管理接口与事件回调压在同一图中，实际是不同接口/方向，不把“获取CLR管理器”本身误写成CLR把一个任意限制器回调给宿主：

```text
┌─────────────┐
        │   宿主进程   │
        └──────┬──────┘
    设置宿主管理器│
    （宿主→CLR）│
               ↓
        ┌─────────────┐
        │     CLR     │
        └──────▲──────┘
               │
    获取CLR管理器│
    （CLR→宿主）│
        ┌──────┴──────┐
        │   宿主回调   │
        └─────────────┘
```

原“System.IO.File.Delete不准用”是权限/保护用途的例子，不能据此声称所有宿主都有任意成员黑名单。Framework ICLRHostProtectionManager按受保护类别限制partial trust中的特定成员；文件访问还受具体权限/信任合同影响。现代.NET不提供同等CAS沙箱。[历史宿主保护合同](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrhostprotectionmanager-interface)

<a id="c22-runtime-lifetime"></a>
停止托管执行与卸下运行时不同。Framework ICLRRuntimeHost.Stop不释放宿主资源、不卸域也不销毁线程，资源需进程终止释放；不能把FreeLibrary或Stop作为安全移除已初始化CLR的操作。原“状态包含线程池、GC堆、类型系统、JIT缓存，交织复杂”可帮助理解，未核的“微软就是因为某风险而决定不做”不作设计者原话。[Stop合同](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrruntimehost-stop-method)

原疑惑“普通DLL不是可以FreeLibrary卸掉吗？”有实质：Windows FreeLibrary按模块引用计数处理，在计数归零时可移除该DLL映像；这并不提供CLR托管执行状态的完整终止合同，不能据普通DLL规则安全拔除已经初始化的CLR。[FreeLibrary合同](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-freelibrary)

历史Framework4以前通常只能在同一进程激活一套CLR版本；Framework4引入进程内side-by-side，可让CLR2与CLR4并存。并存不是在原地卸掉旧CLR再换新CLR，也不是任意数量/任意版本自由混用。[进程内并存范围](https://learn.microsoft.com/en-us/dotnet/framework/deployment/in-process-side-by-side-execution)。原IIS改web.config必须重启w3wp才能让CLR“重载”是误解：经典ASP.NET配置变化通常重启相关应用域，而不必移除运行时或重启整个工作进程；外部configSource等也有例外。[历史配置变化](https://learn.microsoft.com/en-us/previous-versions/aspnet/ackhksh7(v=vs.100))

<a id="c22-environment"></a>
## “dotnet环境”包括什么

用户把dotnet.exe写成dotnet.ext，核心是在问安装什么才能运行C#产物。普通依赖CLR的托管程序需要匹配的运行时与类库；框架依赖部署从机器安装寻找它们，自包含部署随程序携带它们，不能说目标机总必须另装dotnet。

原四类安装组件分别保留：.NET Runtime提供普通托管应用的基础运行组件，不只限控制台；.NET Desktop Runtime补充WinForms/WPF等Windows桌面支持；ASP.NET Core Runtime提供Web框架共享运行组件；SDK带开发所需compiler、dotnet build/run等工具及对应运行组件。IIS部署还需合适Hosting Bundle/ASP.NET Core Module。开发通常选择相应SDK，生产按应用类型和部署方式准备；安装某SDK不保证所有其它大版本/Desktop/ASP.NET依赖都齐备。

原工具箱图保留，其using说明是例子：using是命名空间导入，不直接负责安装或加载程序集，也不是所有System.*都来自一同名文件。

```text
.NET 运行环境
├── CLR（公共语言运行时）
│   ├── JIT 编译器   —— 把 CIL 翻译成本地机器码
│   ├── GC           —— 自动管理内存
│   ├── 线程池       —— 调度托管线程
│   └── 类型系统     —— 类型安全校验
│
├── BCL（基础类库）
│   └── System.IO、System.Collections、System.String……
│       你代码里 using 的那些 System.* 都在这里
│
└── 宿主入口（dotnet.exe / hostfxr.dll 等）
    └── 负责把 CLR "叫醒"并交给它你的程序
```

传统路径是C#→CIL与元数据→运行时加载→按需JIT成本机代码。JIT按方法/代码区域工作，不是CPU逐条解释IL。**补充边界**：ReadyToRun等可含预编译机器代码，NativeAOT程序可直接以本机产物部署，不能把“所有C#只能IL、每次都必须JIT与安装CLR”当普遍规则；本任务没有构建NativeAOT产物。[NativeAOT部署范围](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

<a id="c22-framework-start"></a>
### MyApp.exe与PE的托管目录

原Framework启动七步图修正目录位置：

```text
① 你双击 MyApp.exe
      ↓
② Windows 加载器检查 EXE 文件头，发现 PE可选头CLR Runtime Header目录含有效RVA/Size
   → 这说明"我是托管模块，不是普通原生程序"
      ↓
③ Windows 加载 MsCorEE.dll（这就是上一轮说的"垫片"）
      ↓
④ MsCorEE.dll 把 EXE 的入口点改成 CLR 的入口，
   然后调用 _CorExeMain
      ↓
⑤ CLR 被初始化：
   · 加载 clr.dll
   · 创建系统域、共享域、默认 AppDomain
   · 找到 MyApp.exe 里的 Main 方法
      ↓
⑥ JIT 编译器把 Main 方法的 CIL 翻译成本地机器码
      ↓
⑦ 你的代码终于开始执行！
```

图中_CorExeMain与入口修整属于传统Windows托管启动模型，系统域/共享域/默认域是该CLR实现的内部分类，不能一概当现代可创建的用户AppDomain。识别信息是PE可选头的数据目录项，不是COFF头一枚COM bit；其地址/大小指向CLR header，与CoffHeader.Characteristics是不同字段。原“微软原话”确实存在于官方_CorValidateImage Remarks；该页COFF bit表述不够准确，不能据此说原答虚构引文，文件结构以PE规范目录定义为准。[真实历史引文入口](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/corvalidateimage-function)、[PE数据目录](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#optional-header-data-directories-image-only)

作者真实PEReader读本任务产物：Framework PluginExecutable.exe与现代PluginB.dll均有CLR目录RVA8200、size72和CorHeader；COFF Characteristics另显示各自exe/dll标志。这些数字属于当前产物，不是所有托管PE固定值。

<a id="c22-modern-start"></a>
### dotnet MyApp.dll的现代链

原图各模块及Newtonsoft/System.Console输入保留，修runtime选择和依赖加载的顺序含义：

```text
① 你在命令行敲：dotnet MyApp.dll
      ↓
② dotnet.exe（位于 C:\Program Files\dotnet\dotnet.exe）
   作为 CoreHost 链的入口被启动
      ↓
③ dotnet.exe 调用 hostfxr.dll
   （Host FX Resolver，负责解析用哪个版本的运行时）
      ↓
④ hostfxr.dll 根据 MyApp.runtimeconfig.json等运行时配置，
   选定合适的 .NET 版本，调用 hostpolicy.dll
      ↓
⑤ hostpolicy.dll使用.deps.json等建立依赖解析/探测信息：
   · 你的 MyApp.dll（入口）
   · Newtonsoft.Json.dll等第三方库（需要时加载）
   · System.Console.dll等运行时库（不是先全部载入）
      ↓
⑥ 最终加载 coreclr.dll —— CLR 本体正式启动
      ↓
⑦ JIT 把 Main 的 CIL 翻译成本地机器码
      ↓
⑧ 你的代码开始执行
```

runtimeconfig提供目标framework/version等选择信息，deps提供依赖图/解析数据，两者不互换。hostfxr选择并初始化合适运行时，hostpolicy参与依赖解析、启动CoreCLR；原“先把所有程序集加载完再加载coreclr”不是实际普遍流程。Framework COM寄宿与现代跨平台hostfxr合同不可混套。[现代原生寄宿入口](https://learn.microsoft.com/en-us/dotnet/core/tutorials/netcore-hosting)

dotnet MyApp.dll这一命令确以dotnet为宿主，典型Windows安装位置是C:\Program Files\dotnet，但可有其它位置/架构。现代框架依赖发布也可产生原生apphost MyApp.exe，直接运行/双击，不必总显式敲dotnet；自包含产物包含运行时但仍依赖目标OS、RID与本机必要组件，不是绝对“零依赖”，体积也通常更大。Framework常由Windows预装并不保证任意程序要求的版本已安装。[两部署及apphost](https://learn.microsoft.com/en-us/dotnet/core/deploying/)

原开发/部署动作：写代码、dotnet build、dotnet run，或复制发布产物到目标机器。框架依赖先准备相应Runtime，自包含按目标平台发布。四宿主用途仍是dotnet.exe、IIS w3wp.exe、SQL sqlservr.exe和自己的原生程序；各自具体年代/API范围依前述条件。

<a id="c22-process-diagram"></a>
原Windows进程总图保留为现代CoreCLR场景，默认AppDomain只是兼容观察视图，不在现代提供多域安全隔离：

```text
┌─────────────────────────────────────────────┐
│               Windows 操作系统                │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │           你的进程 (MyApp)           │   │
│   │                                     │   │
│   │   宿主 (dotnet.exe 或 你的 exe 自身)  │   │
│   │            ↓ 加载                     │   │
│   │   ┌─────────────────────────────┐    │   │
│   │   │         CLR (coreclr.dll)   │    │   │
│   │   │   ┌─────────────────────┐   │    │   │
│   │   │   │   JIT 编译器         │   │    │   │
│   │   │   │   GC                 │   │    │   │
│   │   │   │   线程池 类型系统 ...  │   │    │   │
│   │   │   └─────────────────────┘   │    │   │
│   │   │                             │    │   │
│   │   │   默认 AppDomain             │    │   │
│   │   │   ┌─────────────────────┐   │    │   │
│   │   │   │   你的程序集 (IL)     │   │    │   │
│   │   │   │   System.* (BCL)     │   │    │   │
│   │   │   └─────────────────────┘   │    │   │
│   │   └─────────────────────────────┘    │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

<a id="c22-mapping"></a>
## “复制到进程里”怎样理解

用户“复制”可能指把运行时载入进程，而不是把磁盘文件复制到exe目录；先区分这两个意思。DLL映像可映射到虚拟地址空间，代码与数据形成进程内可执行/可访问映像；映射并不表示磁盘文件被移走，也不必先复制到程序目录。OS可让适合共享的只读映像页在进程间共享，进程各自持有自己的地址空间、运行时状态、托管堆、GC和线程池等。

原“系统盘里只有一份clr.dll，几万个程序都共享同份代码”过强：可以装多版本、多架构、多位置，现代自包含也能带自己的CoreCLR；页共享是否实际发生要看映像/映射/页状态，模块列表不能证明所有物理页共享。Framework示例路径C:\Windows\Microsoft.NET\Framework64\v4.0.30319与现代C:\Program Files\dotnet都是常见位置，不是唯一规则；现代Windows本体通常是coreclr.dll，不能仍统一称clr.dll。

<a id="c22-modules"></a>
原观察练习保留步骤：准备适配版本的Framework，编译最简单的exe并运行；任务管理器→详细信息→列标题→命令行，看实际进程入口。若使用已有Process Explorer，选择该进程查看DLL列表，可见运行时/其它模块；具体mscoreei.dll、System*.dll是否出现取决于版本与载入状态。模块列表支持“该进程载入了模块”的观察，不单独证明没有磁盘副本或所有只读页物理共享。本任务未安装或启动新观察软件。

作者改用自己受控子进程的Process.Modules实际观察：Framework程序MainModule是自己的exe、runtime为clr.dll；通过dotnet启动的NET10程序MainModule是dotnet.exe、runtime为coreclr.dll。两者CurrentDomain默认、BaseDirectory存在、GetAssemblies有记录。kernel32.dll/user32.dll原类比说明其它Windows模块也能在多进程有映像映射与各自状态，不能把“DLL文件”和“进程中的执行状态”混成一个对象。

<a id="c22-domain"></a>
## Framework AppDomain的四种能力

AppDomain装一组程序集，提供隔离、卸载、权限和配置等管理范围；“唯一作用就是隔离”是目的概括，不该否定其它机制。CLR启动时建立默认域，宿主或托管API可创建额外域。默认域不能用普通Unload移除；原“Main总在默认域”限通常exe启动，用ExecuteAssembly也可在指定非默认域执行入口。

原四能力表保留，安全条件与默认域例外明确：

<a id="c22-domain-four-table"></a>
| 能力 | 说明 |
| --- | --- |
| **代码隔离**​ | 类型安全托管代码的跨域对象访问需适当封送；不是OS地址空间屏障 |
| **可卸载**​ | 可卸载额外域及其可卸载内容；默认域/域中立代码有例外，不能任选单个程序集 |
| **可保护**​ | Framework特定partial-trust/CAS/宿主条件可施加权限集；现代无同等沙箱 |
| **可配置**​ | 可按该域配置搜索路径、版本绑定重定向、卷影复制、加载器优化 |

原10个ASP.NET站点故事用来问共享进程里怎样独立更新/限制工作负载。多个应用可共用进程；一个进程更新应用不必将其它应用全停。若进程被unsafe/native内存破坏或有致命错误，所有域都可能受影响；建立AppDomain并不会挡住该类损坏。未处理异常在通常Framework策略下也可能终止整个进程，不能说任意“站点崩溃”都只伤该域。原“不可信插件不能随意看邻居数据”需验证类型安全与适当信任/权限前提，不是把任意全信任恶意代码放一域就安全。

<a id="c22-apartments"></a>
原独立房子/同楼公寓表保留，修绝对成本/崩溃保证：

<a id="c22-apartments-table"></a>
| 真实进程 | AppDomain |
| --- | --- |
| 一栋独立的房子 | 同一栋楼里的不同公寓 |
| 有自己的地址空间 | 共享进程地址空间，但逻辑隔离 |
| 创建/维护另进程有相应成本，具体需测量 | 共享运行时可减少某些成本，非固定性能保证 |
| 地址空间分隔有助故障隔离；仍有共享外部资源 | 只隔离合适逻辑状态；进程级故障可波及所有域 |
| 卸载 = 杀掉进程 | 卸载 = `AppDomain.Unload()` |

“逻辑进程”是教学比喻：所有域共享该进程地址空间，不等OS级隔离。进程本身也能由OS终止/回收，原“域能卸载但进程不能”应改为可在保留宿主进程的同时卸载额外域。

<a id="c22-create-execute"></a>
原MyPluginDomain/Plugin.dll三步代码保留，修“所有内存全部回收”注释，调用须放方法内并有System等上下文：

```csharp
// 在当前进程里再建一个 AppDomain
AppDomain newDomain = AppDomain.CreateDomain("MyPluginDomain");

// 在新 AppDomain 里加载一个程序集并执行
newDomain.ExecuteAssembly("Plugin.dll");

// 用完就扔——整个 AppDomain 被卸载，发起该域的卸载；域中立内容/GC及失败条件见下文
AppDomain.Unload(newDomain);
```

ExecuteAssembly要求入口点；扩展名.dll不妨碍有入口的PE，但任意类库不能用它执行。作者创建自己的安全入口夹具并命名为Plugin.dll，在MyPluginDomain真输出entry-domain:MyPluginDomain、返回37；普通PluginAssembly.dll真抛MissingMethodException。NET10不支持CreateDomain的事实另列，不倒判这段Framework历史API错误。

<a id="c22-object-boundary"></a>
原MyClass局部片段保留，它表达两个执行范围，不是给出了完整跨域调度程序：

```csharp
// 在默认 AppDomain 里
var obj = new MyClass();  // MyClass 在默认 AppDomain

// 在另一个 AppDomain 里
// 无法直接拿到上面那个 obj！
```

普通类型安全引用不能随意跨域直接当本地对象用；框架会按对象合同封送。各域搜索路径、版本绑定重定向、shadow copy允许更新适当被使用DLL、LoaderOptimization可分别配置，这些不保证任何文件锁/任何依赖冲突都自动解决。权限集可在历史适当partial trust合同下限制文件、网络及UnmanagedCode/PInvoke权限；宿主全信任例外和不安全本机代码须分开。现代ALC不能当这些权限隔离的等价替换。

**当前边界补充**：第四版所述Framework受限域/权限集是历史机制；当前微软资料也不再认可CAS为.NET Framework恶意代码的安全边界。不能仅因使用Framework就承诺可靠沙箱，需要按适当OS边界与最小权限运行；这不倒判原历史权限 API/限制例无效。[CAS当前定位](https://learn.microsoft.com/en-us/dotnet/core/porting/net-framework-tech-unavailable#code-access-security-cas)

<a id="c22-marshaling"></a>
## 跨域调用：原Plugin行为与Data副本

按引用封送对象派生MarshalByRefObject，CreateInstanceAndUnwrap返回用于访问远端对象的透明代理。原PluginAssembly/Plugin/DoWork输入和“插件干活中...”输出保留；表达式体为C#6，第四版C#5可改普通方法体返回同字符串。类型声明与调用语句各置适当类/方法，程序集/类型必须实际存在并可解析：

```csharp
// 在插件程序集里
public class Plugin : MarshalByRefObject
{
    public string DoWork() => "插件干活中...";
}

// 在主程序里
AppDomain pluginDomain = AppDomain.CreateDomain("Plugin");
Plugin proxy = (Plugin)pluginDomain.CreateInstanceAndUnwrap(
    "PluginAssembly", 
    "Plugin");
    
string result = proxy.DoWork();  // 跨 AppDomain 调用
```

五步语义是拦截代理调用→封送参数→插件域执行真正DoWork→封送返回值→调用方得到结果。字符串返回值按其可封送合同处理，不是把真正Plugin对象移到主域。代理仍代表远端对象，其具体寿命、异常和参数合同需遵守Remoting。

跨边界调用有代理/封送开销，但不应说每次都发生OS线程切换、每个参数都完整序列化。作者补充方法ThreadId观察到主域调用与插件域执行ManagedThreadId相同；线程可跨域，AppDomain本身不要求创建专属线程。真实DoWork打印原字符串，RemotingServices.IsTransparentProxy=True，插件域名Plugin；这只是受控调用，不是性能基准。

<a id="c22-byvalue"></a>
按值封送适当可序列化且不按引用封送的对象，接收方得到独立副本，需要相应类型/程序集解析。原Data.Value=42片段原样保留；它只在主域new Data，并用注释说明将来传递，没有实际调用把data送过去：

```csharp
[Serializable]
public class Data
{
    public int Value { get; set; }
}

// 主程序里
AppDomain pluginDomain = AppDomain.CreateDomain("Plugin");
Data data = new Data { Value = 42 };

// 跨 AppDomain 传递 data 时，CLR 会：
// 1. 把 data 序列化成字节流
// 2. 在目标 AppDomain 里反序列化
// 3. 创建 data 的一个"精准复制"对象
// 4. 把这个副本交给你
```

不能把这片段声称已实证跨域复制。**补充安全夹具**Bridge派生MarshalByRefObject，Accept(Data)在插件域把42改43并保留，主域原Data仍42；Get返回43的副本，主域改它为99后插件域Saved仍43。原Value=42输入保留，新增Bridge只是把教学意图变成可观测传递，不替代原示例。[域与封送合同](https://learn.microsoft.com/en-us/dotnet/framework/app-domains/application-domains)

<a id="c22-marshaling-choice"></a>
原四场景表保留，大/小只是需要权衡的输入，非固定最佳答案：

<a id="c22-marshaling-choice-table"></a>
| 场景 | 选择 |
| --- | --- |
| 对象有行为（方法），想远程调用 | 按引用（`MarshalByRefObject`） |
| 对象只是数据，想传过去自己用 | 按值（`[Serializable]`） |
| 对象很大，不想复制 | 可考虑按引用避免重复大副本；还须衡量调用次数/寿命/副作用 |
| 对象很小，且不需要跨域共享状态 | 可考虑按值复制以本地使用，仍按实际合同选择 |

<a id="c22-loader"></a>
## Loader堆、域中立代码与静态状态

第四版用Loader堆解释运行时类型/加载信息的生命周期。按非域中立方式载入的程序集，其域相关加载数据与JIT代码可随域管理；类型内部数据结构、元数据、代码与托管System.Type对象不是可随意互换的术语。原System.dll两域各份、卸一域不伤另一域，只能作为指定非中立加载的情况，不能说所有非mscorlib程序集永远不共享。

Framework根据LoaderOptimization/宿主、加载方式、信任合同等决定域中立共享。mscorlib始终域中立，其可共享代码/相关类型数据不能随一个用户域完全卸掉；其它符合条件程序集也可域中立。MultiDomain、MultiDomainHost、SingleDomain等策略各有范围，LoadFrom/字节数组加载不能简单套同样JIT共享。域中立代码共享不等静态数据跨域共享。[域中立与线程范围](https://learn.microsoft.com/en-us/dotnet/framework/app-domains/application-domains)

<a id="c22-string-counterexample"></a>
下面原DomainA/DomainB string图作为**原错误对照**保留，不能当正例记忆：

```text
进程里有两个 AppDomain：DomainA 和 DomainB

DomainA 里第一次用到 string → CLR 在 DomainA 的 Loader 堆里
                                  创建 System.String 的类型对象
                                  
DomainB 里第一次用到 string → CLR 在 DomainB 的 Loader 堆里
                                  也创建 System.String 的类型对象
```

string正来自mscorlib，随后原答自己列出的mscorlib例外已否定“两份String/JIT必各一套”。若要说明非域中立加载，须换成符合该前提的自定义类型，不能新增类型后把它说成原string例。作者安全Counter补充观察：远端Increment为1，主域Counter预置10仍10，说明该受控非中立程序集的静态状态分开；不拿它证明全部内置类型的MethodTable布局。

<a id="c22-domain-diagram"></a>
原两域/四程序集/共享mscorlib大图保留，“堆+栈”栏修为该域可访问对象的逻辑归属：

```text
┌──────────────────────────────────────────────┐
│                  进程 (w3wp.exe)               │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │              CLR 实例                    │  │
│  │                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │  AppDomain A  │  │  AppDomain B  │  │  │
│  │  │  (默认域)     │  │  (插件域)     │  │  │
│  │  │              │  │              │  │  │
│  │  │  Loader 堆 A  │  │  Loader 堆 B  │  │  │
│  │  │  ┌─────────┐  │  │  ┌─────────┐  │  │  │
│  │  │  │Assembly1│  │  │  │Assembly3│  │  │  │
│  │  │  │Assembly2│  │  │  │Assembly4│  │  │  │
│  │  │  └─────────┘  │  │  └─────────┘  │  │  │
│  │  │              │  │              │  │  │
│  │  │  对象归属      │  │  对象归属      │  │  │
│  │  └──────────────┘  └──────────────┘  │  │
│  │                                        │  │
│  │  ┌──────────────────────────────────┐  │  │
│  │  │  MSCorLib（AppDomain 中立加载）   │  │  │
│  │  │  所有域共享同一份 System.Object 等  │  │  │
│  │  └──────────────────────────────────┘  │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

各域不是各自拥有一套完全独立GC堆和线程栈。托管堆由运行时/GC组织，线程有自己的栈并能跨域，域的Loader数据也不同于托管对象堆。“mscorlib所有域共享System.Object等”解释共享运行时类型/代码范围，不是每个域的静态状态与对象也自动共一份。

<a id="c22-unload"></a>
## 卸载额外域的合同与限制

原列举的四类内容分别保留：程序集、类型加载数据、已创建的托管实例、分配的托管堆内存。非域中立程序集关联的Loader数据/JIT代码释放与托管实例及其堆空间的GC回收不是同一个动作；不能把四类都说成由一次GC立即全部回收。域中立共享内容和仍在其它域使用的内容另有寿命限制。

Framework AppDomain.Unload同步请求卸载目标域，可影响在域内执行或未来返回域的调用链。原Q1七步概览依序为：挂起适当托管线程→检查栈/域边界→中止相关执行并展开finally→finally/catch/类构造器/CER/本机执行等区域可能延迟中止→跨域代理失效→相关终结/回收处理→恢复其余线程、调用方继续。这里保留第四版实现叙述的各个阶段，不把它升级成所有运行时逐字遵守的公共算法。

原Q5六步图有不同组织粒度，完整保留为**传统实现的教学流程**；其中“一次GC/所有Finalize/等待所有清理”不能当逐字API保证：

```text
① 挂起进程中所有曾执行过托管代码的线程
② 检查所有线程栈，找出哪些线程：
   - 正在该 AppDomain 中执行，或
   - 将来可能返回到该 AppDomain
③ 对这些线程强制抛出 ThreadAbortException
   → 线程展开调用栈，执行 finally 块做清理
   → 但如果在执行 finally/catch/类构造函数/非托管代码，
     则等这些代码块执行完再抛
④ 所有相关线程离开该 AppDomain 后：
   → 给所有指向该 AppDomain 对象的代理打上"失效"标记
   → 之后再调用这些代理的方法会抛 AppDomainUnloadedException
⑤ 强制执行一次 GC，调用待回收对象的 Finalize 方法
⑥ 恢复其余线程
```

实际可靠边界：卸载涉及目标域线程中止，native执行或finally等情况可能使请求失败/延迟；可能抛CannotUnloadAppDomainException，不保证任意时长最终完成，也不保证所有域中立代码、进程资源或对象字节马上释放。已卸域代理再调用可抛AppDomainUnloadedException。垃圾回收和终结器时序不能替代正确资源Dispose合同。[Framework Unload范围](https://learn.microsoft.com/en-us/dotnet/api/system.appdomain.unload?view=netframework-4.8.1)

原ThreadAbortException提醒保留：它在Framework具有特殊重新抛出/线程退出语义，未处理该中止异常与普通未处理业务异常的进程政策不同；不能据此说任何其它异常都只结束某个域。CER是历史可靠性机制，现代.NET不是等价保留其全部语义。本任务未运行挂死、native损坏、堆栈溢出或强行终结进程的实验。

受控输出：卸掉Plugin域后原proxy.DoWork抛AppDomainUnloadedException；尝试Unload当前默认域抛CannotUnloadAppDomainException。只是两个API输入边界，不证明原六/七步每条内部动作都已独立观测。

<a id="c22-arm"></a>
## ARM测量的是累计与上次完整回收后的数据

应用域资源监视可帮助宿主识别消耗CPU/内存的应用，再按策略选择处理。Q1独立开关片段保留：

```csharp
AppDomain.MonitoringIsEnabled = true;
```

Q5另一个片段包含三条实际读取，不能用前一个只设开关的短例代替：

```csharp
// 打开全局监视开关
AppDomain.MonitoringIsEnabled = true;

// 读取当前 AppDomain 的资源使用
TimeSpan cpu = AppDomain.CurrentDomain.MonitoringTotalProcessorTime;
long memoryUsed = AppDomain.CurrentDomain.MonitoringSurvivedMemorySize;
long memoryAllocated = AppDomain.CurrentDomain.MonitoringTotalAllocatedMemorySize;
```

启用后收集进程内所有域的数据，若域早已存在，则累计从启用时开始；Framework开关开启后不能再关闭。MonitoringTotalProcessorTime为归于该域的线程CPU时间，不是钟表耗时；阻塞/睡眠不积同等CPU，域内进入本机的时间仍可计入。MonitoringTotalAllocatedMemorySize是累计分配，不等当前活对象大小；MonitoringSurvivedMemorySize是最近完整阻塞GC后存活且归域的数据，不是任意时刻精确“内存使用”。[ARM各量的定义](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/app-domain-resource-monitoring)

原AppDomainMonitorDelta实现IDisposable，在静态构造器启用监视，以m_thisADCpu(TimeSpan)、m_thisADMemoryInUse和m_thisADMemoryAllocated记录基准，执行前后作差帮助诊断泄漏/瓶颈。原问答没有该类完整实现，不补一份外部源码冒原例；“InUse”字段名也须按实际存活计数定义理解。

作者Framework安全读取true/三项非负，在显式完整GC后读取存活量；尝试设false实际ArgumentException。没有承诺具体CPU毫秒/内存字节，也不以单次非负数据证明没有泄漏。现代兼容API的process范围不能代替历史多域ARM解释。

<a id="c22-sites"></a>
## 原“站点A/B/C”具体是什么

原IIS例子分别为博客www.myblog.com、商店www.shop.com、API api.myservice.com，程序集分别Blog.dll、Shop.dll、Api.dll并使用EntityFramework等依赖。IIS是Web服务器，网站可有多个application/虚拟路径，应用可配置app pool；一台服务器不必只有一个w3wp，也不是site↔application↔AppDomain的一一关系。

原三域图保留为**三个经典ASP.NET应用恰共用一个工作进程的例子**，图中站点名称指这三个应用，不能推广为全服务器固定结构：

```text
w3wp.exe 进程
└── CLR
    ├── AppDomain: 站点A (www.myblog.com)
    │   └── 加载 Blog.dll, EntityFramework.dll ...
    ├── AppDomain: 站点B (www.shop.com)
    │   └── 加载 Shop.dll, EntityFramework.dll ...
    └── AppDomain: 站点C (api.myservice.com)
        └── 加载 Api.dll ...
```

受控更新/配置变化可以重启某应用域而保留工作进程及其它应用；shared外部状态、宿主策略、进程故障仍可能影响邻居。历史partial trust可限制适当应用，并非默认每站都沙箱化。原现实三句话“站点A更新dll、站点A崩溃、可疑第三方代码”分别按应用重启、异常/进程故障、具体信任合同判断，不能混成一条安全承诺。

原写字楼/办公室/三家公司类比保留：水电代表共享进程资源，办公室代表逻辑状态，装修代表更新应用；“一家着火必不烧邻居”应换成有限受控错误不必导致其它应用重启，真实进程级火灾可以波及整栋楼。它和前面的独立房子/公寓表是不同解释例，不能简单说已用同类比覆盖掉。

<a id="c22-console-domain"></a>
原普通MyApp/Newtonsoft单域图保留，默认域后“唯一”限没有额外域的这个程序形状：

```text
MyApp.exe 进程
└── CLR
    └── 默认 AppDomain（唯一一个）
        └── 加载 MyApp.exe, Newtonsoft.Json.dll ...
```

桌面、控制台/WPF程序通常不需自己创建额外域，但宿主或库也可能调用CreateDomain；不能说只有用户显式创建时才可能有第二域。现代.NET单默认域则是平台限制，与Framework通常只用一域的习惯不同。

<a id="c22-assemblies"></a>
## 一个域能装多个程序集，同程序集也可进多个域

AppDomain与程序集不是一一关系；一个域常载入主程序集与若干依赖，同一程序集可在多个域有加载状态，域中立时还涉及前述共享。原多域关系图完整保留MyApp/System/System.Core/Newtonsoft与PluginX/System名称：

```text
进程 (w3wp.exe / MyApp.exe)
│
└── CLR
    │
    └── AppDomain A（默认 AppDomain）
    │   ├── MyApp.exe          ← 你的主程序集
    │   ├── System.dll         ← 基础类库
    │   ├── System.Core.dll
    │   ├── Newtonsoft.Json.dll ← 第三方库
    │   └── ...                ← 还有其他被引用的程序集
    │
    └── AppDomain B（你手动创建的）
        ├── PluginX.dll        ← 插件程序集
        └── System.dll         ← 注意：这个 System.dll 
                                 在 B 域里是另一份
```

原加载次序“进程→CLR→默认域→程序集→Main”是典型入口形状，依赖通常按需解析/加载，不能说所有引用DLL在启动时必全进入默认域。程序集是逻辑加载/身份单元，常有磁盘dll/exe；动态Reflection.Emit程序集可没有现成文件，NativeAOT也不必遵循每个方法的JIT示意。域中立代码卸载边界见前述，原默认System.dll多份须标非中立条件。

原三场景表保持普通程序/插件系统/ASP.NET不同用途：

<a id="c22-assembly-relation-table"></a>
| 场景 | 关系 |
| --- | --- |
| 普通控制台程序 | 1个默认域←→主程序集与需要时加载的多个依赖 |
| 插件系统 | 同一个程序集可被多个域加载；每域也可有多程序集 |
| 简单 ASP.NET 站点 | 一个简单ASP.NET应用恰用1个域←→应用dll、Framework类库及需要时载入的依赖 |

原现代名称的另一张默认域图保留，System.Runtime、System.Console、System.Collections是现代例子，不把它当Framework所有程序固定文件清单；只展示单域多个程序集，不表示只有一个ALC：

```text
MyApp.exe 进程
└── CLR
    └── 默认 AppDomain（唯一一个）
        ├── MyApp.exe
        ├── System.Runtime.dll
        ├── System.Console.dll
        ├── System.Collections.dll
        └── ...（十几个甚至几十个程序集）
```

原书架/书类比依次是一个架放多本书、同本书可在多架有加载表示、CLR翻书才使用代码、要卸某个域的内容可移除架。不能据此说单域随时能抽任意一程序集，或移架必清掉域中立共享代码。程序集与ALC也不是“一程序集一context”；每个context都可包含多个程序集，作者受控PluginB+Version库在同一ctx实际成立。

<a id="c22-modern"></a>
## 现代.NET保留AppDomain观察入口，使用ALC管理加载

AppDomain类型没有完全删除；CurrentDomain、BaseDirectory、GetAssemblies等仍可用。现代只支持单默认域，不支持创建额外域/传统跨域Remoting，不提供旧多域隔离/权限/卸载机制。原API/能力表保留并避免把两种失败异常写成一样：

<a id="c22-modern-api-table"></a>
| 能力 | .NET Framework | .NET 5+ |
| --- | --- | --- |
| `AppDomain.CurrentDomain` | ✅ 可用 | ✅ 可用（只有这一个） |
| `AppDomain.CreateDomain()` | ✅ 创建新域 | ❌ 抛出 `PlatformNotSupportedException` |
| `AppDomain.Unload()` | ✅ 卸载指定域 | 不支持；本机.NET10卸CurrentDomain抛CannotUnloadAppDomainException |
| 隔离/安全边界 | 历史适当type-safe/partial-trust条件下的逻辑/权限范围，非OS屏障 | ❌ 不提供 |
| 动态加载程序集 | AppDomain 内加载 | 可用普通加载API及ALC；不是只有一种新入口 |

受控NET10真实输出：CurrentDomain默认、GetAssemblies有记录；CreateDomain(MyPluginDomain)为PlatformNotSupportedException，Unload(CurrentDomain)为CannotUnloadAppDomainException。不能因现代失败就判Framework相同原代码错误。[不再支持的Framework技术](https://learn.microsoft.com/en-us/dotnet/core/porting/net-framework-tech-unavailable)

<a id="c22-remoting-cost"></a>
原三种背景分别保留：Framework跨域对象合同依赖Remoting体系；额外域需要运行时支持及资源；需要更强隔离时可用独立进程/容器。官方移植资料确说明不添加更多域、Remoting不受支持以及相关成本；原Stack Overflow团队人引句没有本次一手身份/完整语境核对，不能作为确定官方原话。不能从“现代重视跨平台/性能”反推某一机制已是唯一设计因果，也没有本任务成本基准。“独立GC视角”只是监视/归属维度，不是各域独立GC堆。

<a id="c22-alc-version"></a>
### ALC的版本缓存与加载身份

AssemblyLoadContext负责查找、加载、缓存程序集与依赖；每个实例按简单名称缓存一个程序集版本。需要同简单名称不同版本并存时，可放不同context；版本解析请求是否满足还受该context已加载版本/解析合同影响。原Newtonsoft.Json v13.0.1/PluginA与v12.0.3/PluginB图完整保留为设计例，不声称本任务安装并实跑了这两个第三方版本：

```text
默认 ALC
└── Newtonsoft.Json v13.0.1
└── PluginA.dll（依赖 Json v13）

插件 ALC
└── Newtonsoft.Json v12.0.3
└── PluginB.dll（依赖 Json v12）
```

原13.0.1与12.0.3是NuGet包发布版本；ALC比较的是程序集的AssemblyName.Version，包版本不自动等于这个四段身份版本，须读取实际程序集确认。[原13.0.1包](https://www.nuget.org/packages/Newtonsoft.Json/13.0.1)、[原12.0.3包](https://www.nuget.org/packages/Newtonsoft.Json/12.0.3)、[包版本与程序集版本](https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/versioning)。Framework多AppDomain也能解决适当版本隔离，原“旧时代很难优雅”是主观评价，不能改成历史完全不支持。作者**补充同形实验**用自己C22Version简单名1.0.0.0/2.0.0.0，分别载入A/B ctx，实际Type身份不同、方法返回1/2；这是平台合同的受控例，不取代原Json版本输入。[ALC身份与版本合同](https://learn.microsoft.com/en-us/dotnet/core/dependency-loading/understanding-assemblyloadcontext)

<a id="c22-alc-collectible"></a>
原PluginContext collectible短片段修成绝对路径，须有System.Reflection/System.IO/System.Runtime.Loader上下文并存在目标文件：

```csharp
var context = new AssemblyLoadContext("PluginContext", isCollectible: true);
Assembly plugin = context.LoadFromAssemblyPath(Path.GetFullPath("PluginB.dll"));
// ... 使用插件 ...
context.Unload();  // 发起合作卸载；释放强根/活动调用栈等后由GC完成
```

Unload只启动合作卸载：没有活动方法栈引用，没有外部强根或相关强GCHandle等时，GC才可完成回收；不保证下一GC、一次Unload或立即全资源释放。CLR并不替插件停止任意线程/订阅或Dispose所有资源。[现代合作卸载条件](https://learn.microsoft.com/en-us/dotnet/standard/assembly/unloadability)

<a id="c22-alc-compare"></a>
原六维对照表逐项保留，修跨context调用与沙箱/成本绝对说法：

<a id="c22-alc-compare-table"></a>
| 维度 | AppDomain（.NET Framework） | AssemblyLoadContext（.NET 5+） |
| --- | --- | --- |
| 隔离级别 | 适当类型安全/历史权限合同的进程内逻辑隔离，非进程内存故障屏障 | 仅程序集加载隔离 |
| 跨边界调用 | 传统透明代理/值封送，按Remoting合同 | 可普通对象/接口直接调用，需共享合同类型身份；无自动Remoting代理 |
| 卸载 | Unload可强制影响域中执行；可能失败/延迟 | collectible上下文合作回收；需清强根/活动栈等 |
| 安全边界 | 历史Framework partial-trust/CAS等条件可用 | 无安全边界；需要时选择进程/适当配置的容器 |
| 版本共存 | 可在不同域按加载/绑定合同共存 | 不同context可各载同简单名称的不同版本 |
| 开销 | 有额外运行时/代理等成本；未测固定差值 | 加载context也有成本；不承诺总更小 |

ALC隔离加载名称/类型身份/依赖选择，不限制同进程恶意代码的内存/文件权限，且不同context可正常对象调用，不是不能跨界。原“重武器/轻武器”类比只表示机制范围差异，不能作为性能排名或安全边界证明。

<a id="c22-modern-layers"></a>
原三层ALC/进程/Docker图完整保留，图中故障隔离指OS边界内的合适故障；容器的资源/依赖限制还需实际配置，不能绝对“所有崩溃互不影响”：

```text
层级 1：AssemblyLoadContext
   → 解决"动态加载/卸载程序集"、"多版本共存"
   → 轻量级，无安全边界

层级 2：进程隔离
   → 解决"代码崩溃互不影响"
   → 操作系统级保护

层级 3：容器隔离（Docker）
   → 解决"依赖隔离、资源限制、部署一致性"
   → 云原生时代的主流方案
```

现代IIS有in-process（CoreCLR在w3wp内）与out-of-process（模块代理到独立后端exe/dotnet）模型，不能说每个站点总拥有一个独立dotnet进程；两种都有app pool/托管约束。历史也不是全服一个w3wp、每站一个域。[IIS进程内模型](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/in-process-hosting?view=aspnetcore-10.0)

<a id="c22-migration"></a>
原四迁移方向表保持插件加载、旧沙箱、跨域调用和多版本共存四个用途，方案按真正需要选择：

<a id="c22-migration-table"></a>
| 《CLR via C#》讲的 | .NET 5+ 中改用 |
| --- | --- |
| AppDomain 隔离插件 | 需要动态加载/合作卸载时选collectible ALC；不是同等安全隔离 |
| AppDomain 沙箱权限 | 按威胁/权限要求选独立进程/配置容器等 |
| 跨 AppDomain 调用 | 跨进程可选gRPC/HTTP等显式通信；进程内共享实际接口身份+ALC普通调用 |
| 多版本程序集共存 | 不同ALC可各载同简单名称的不同版本；具体按加载/依赖解析合同 |

不能说所有第四版原理仅“换一个API”便完整保留：JIT/GC/类型系统还需分实现，现代没有原多域Remoting/CAS合同。维护Framework项目时旧域机制仍相关，新项目按平台用途选择，原“忘掉AppDomain”只适于新多域设计，兼容观察API仍有用。

<a id="c22-plugin-alc"></a>
### 原完整PluginLoadContext示例的适用上下文

下面保原Load返回null、共享IPlugin、PluginB.Plugin、Execute和Unload，各片段需放完整方法/类上下文；Assembly?为nullable-reference注解，需相应语言版本与上下文。补using、绝对路径及Load注释，不把不存在的目录resolver当已实现：

```csharp
using System;
using System.IO;
using System.Reflection;
using System.Runtime.Loader;

// 1. 定义一个可收集的 ALC
class PluginLoadContext : AssemblyLoadContext
{
    public PluginLoadContext() : base(isCollectible: true) { }
    
    // 自定义程序集解析逻辑
    protected override Assembly? Load(AssemblyName name)
    {
        // 本例不实现插件目录解析；null请求运行时按默认机制继续
        return null;  // 交给默认逻辑
    }
}

// 2. 使用
var alc = new PluginLoadContext();
Assembly plugin = alc.LoadFromAssemblyPath(Path.GetFullPath(@"plugins\PluginB.dll"));

// 3. 通过共享接口创建实例（主程序和插件都引用同一个接口程序集）
Type pluginType = plugin.GetType("PluginB.Plugin");
var instance = (IPlugin)Activator.CreateInstance(pluginType);

instance.Execute();

// 4. 卸载
alc.Unload();
// 注意：还要确保所有对 instance、pluginType、plugin 的引用都被释放
// 否则 ALC 不会被 GC 回收
```

原没有定义IPlugin或PluginB实际文件，不能声称复制该片段就完整可跑。主程序/插件应使用同一真实合同程序集身份；若把接口各载成不同Type，名字相同也不能随便强转。GetType名称必须存在，Activator要求适当可实例化类型/构造器，生产调用应检查这些失败。若需插件私有依赖解析，可另外使用AssemblyDependencyResolver等；原return null本身只回退，不自动“从插件目录加载所有依赖”。

<a id="c22-alc-evidence"></a>
作者补充夹具实跑：共享SharedContract/IPlugin由Default持有，PluginB在PluginContext普通调用Execute打印“插件干活中...”；同ctx再载Version库证明可多个程序集。原两条相对路径PluginB.dll与plugins\PluginB.dll分别真ArgumentException，绝对化后成功。独立不内联加载方法返回WeakReference；Unload后保留插件实例强根，10轮受控GC后仍alive=True；清根后False，无根路径也False。这个观察支持强根边界，不保证任意GC次数或环境。初轮作者新增Thread方法名遮蔽BCL Thread导致Framework夹具库失败，仅修补充名ThreadId并重跑三条相关记录，首轮源码/日志按SHA保全，原例/输入不改。

<a id="c22-source-notes"></a>
原Q1/Q3/Q7/Q8 PDF参考目录各13/18/10/11题目保存在只读提取底稿相应PDF消息及差异context，供追溯，不能当本次全部外部链接已核。原T明确第一次GitHub解析失败而参考其它读书页，故正文不沿用“已读被问链接”的声称；T中的现代不可卸载错误文章已被原T拒绝，不捏成A原主张。无PDF独有完整程序遗漏，8表的不同用途分别保留；纯写作安排/未展开邀约不变成新技术章节。

<a id="c22-corrections"></a>
## 重要纠错与版本范围

| 编号 | 原需要纠正或限定的说法 | 本章处理 |
| --- | --- | --- |
| C22-F01 | dotnet/vshost是所有C#宿主；所有exe/现代clr.dll；必另装CLR/所有C#必IL/JIT/自包含零依赖 | Framework传统exe、现代dotnet/apphost/CoreCLR和部署范围分开，NativeAOT等仅补边界。 |
| C22-F02 | Framework4 COM接口必注册激活；IMetaHost；获取manager和回调混同、任意File.Delete黑名单 | 真实ICLRMetaHost与工厂/COM样式合同、管理接口/回调方向与历史partial-trust保护类别分层。 |
| C22-F03 | COM descriptor是COFF一bit；deps负责选runtime；先加载所有依赖再coreclr | PE目录RVA/Size与COFF不同；runtimeconfig选择与deps解析分工，依赖按需加载。 |
| C22-F04 | 复制必指拷到exe目录；系统只有一CLR文件/所有代码物理共享；模块列表证明这些 | 区分用户进程载入意图/磁盘副本/映像映射/可共享页和私有状态；实际模块观察仅限载入。 |
| C22-F05 | web.config变动必须重启w3wp；全服务器一个进程/每site一个域；Core每site总独立dotnet | 经典应用域重启与site/application/app pool区别；现代IIS in/out process分别。 |
| C22-F06 | AppDomain挡住所有内存破坏/崩溃、恶意全信任代码；每域独立GC堆/线程栈、必性能更优 | type-safe/partial-trust历史逻辑范围；共享进程/线程跨域/GC组织与真实OS故障边界，未做成本基准。 |
| C22-F07 | ExecuteAssembly能执行任意Plugin.dll；Main总默认；所有内存Unload立刻全回收且不会失败 | 入口点/指定域入口实测，额外域与默认域/域中立内容、失败与终结GC边界。 |
| C22-F08 | 除mscorlib全部不共享；string两域必两份；共享代码就是共享所有静态状态 | 域中立策略与mscorlib/String真实归属；受控静态隔离，不泛化内部Type对象布局。 |
| C22-F09 | 每次代理调用都序列化+线程切换；所有程序集必文件、依赖全启动加载 | 保Remoting五步，补同线程观察；动态程序集与按需加载分开。原Data42片段范围和Bridge实证属于正文补充，不列为源曾声称实跑的错误。 |
| C22-F10 | ARM InUse是任意当前精确占用；累计分配等于现存、可随时关 | 最近完整阻塞GC存活与累计/CPU定义、启用时点/不能关闭及真实异常。 |
| C22-F11 | ALC一程序集一ctx、不能普通跨ctx调用、只提供卸载；Load null实现目录resolver | 查找/加载/缓存/版本/多程序集/共享实际接口身份/默认回退分别。原Json包版本和AssemblyName.Version分开，保输入与补充自有库；旧AD“困难”的主观评价只在正文保，不列已证技术错误。 |
| C22-F12 | 卸ALC必下次GC全回收；relative AssemblyPath合法；无所有引用就足够，所有原理只是换API | 两绝对路径修正、活动栈/强根/GCHandle合作条件；现代Remoting/CAS不等历史，观察不作普遍时点保证。 |

正文为作者完整来源整理，实证限制与细账见整理记录；是否通过只由独立实际验收决定。下一章[程序集加载和反射](README.md#chapter-23)继续讨论加载身份与反射，而不是在这里补入原未展开的大型插件架构。
