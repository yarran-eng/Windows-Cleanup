---
name: windows-cleanup
description: '安全清理 Windows C 盘空间与彻底卸载应用。采用"三重确认 + 零残留 + 举证倒置"机制。两大模式：(1) C 盘空间清理——按 9 类缓存/临时文件规则安全释放空间；(2) 应用彻底卸载——跨 8 个维度（文件系统、注册表、AppX、Prefetch、防火墙、服务、计划任务、沙箱账户）清除一切痕迹。本技能仅定义规则和判断标准，不预列具体路径，由 agent 自主扫描发现。'
---

# Windows 安全清理：C 盘空间释放与应用彻底卸载

**两大工作模式：**

| 模式 | 触发方式 | 目标 |
|------|---------|------|
| **C 盘空间清理** | 用户说"清理 C 盘 / 释放空间 / 磁盘不足" | 按第 3 节 9 类缓存规则安全释放磁盘空间 |
| **应用彻底卸载** | 用户说"彻底卸载 / 清除某应用一切痕迹" | 跨 8 个维度（文件、注册表、AppX、Prefetch、防火墙、服务、任务、沙箱）零残留 |

**共享核心原则：宁可不删，不可错删。宁可漏清十个缓存目录，不可误删一个用户文件。**

**举证责任：默认一切目录受保护。agent 必须通过内容验证来证明一个目录可以安全删除，而非证明它危险才保护。凡有疑问，一律不删。**

**设计理念：本技能仅定义规则、约束和判断标准，不预列具体目录路径。每台电脑的应用、版本、路径各不相同——执行时由 agent 自主扫描磁盘结构，发现占用空间大的目录，运用规则进行分类和判断，最终由用户确认后执行。**

---

## 1. 零残留机制（最高优先级）

**本技能在执行全过程中不得在用户磁盘上留下任何新文件或产物，也不得修改系统注册表或系统设置。**

### 1.1 禁止创建的文件类型
- ❌ `.ps1` 脚本文件
- ❌ `.bat` / `.cmd` 批处理文件
- ❌ `.txt` / `.log` 临时日志
- ❌ `.csv` / `.json` 数据导出
- ❌ 任何其他临时文件

### 1.2 合规的执行方式
- **所有 PowerShell 命令必须通过终端直接执行**，不得写入脚本文件再调用
- 如需执行复杂逻辑，使用 PowerShell 内联命令（`powershell -Command "..."`），单引号包裹防转义
- 所有中间数据（目录大小、扫描结果、分类判断过程等）仅在对话上下文中记录，不落盘
- 如果因工具限制必须创建临时文件，**必须在同一条命令链中创建并使用后立即删除**（`; Remove-Item ...`）

### 1.3 残留检查（执行完毕后）
清理完成后，执行以下命令验证无残留：
```powershell
Get-ChildItem "$env:TEMP\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:USERPROFILE\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:LOCALAPPDATA\cleanup_*" -ErrorAction SilentlyContinue
```
若发现残留文件，立即删除。

---

## 2. 扫描与发现规则

**agent 不携带任何"应该清理哪些目录"的预知清单。扫描是一个纯粹的信息收集过程——发现磁盘上的实际结构，而非验证一个预设的清理列表。**

### 2.1 扫描策略

| 阶段 | 操作 | 目的 |
|------|------|------|
| **阶段 A：获取磁盘概览** | 使用 `Get-PSDrive` 获取 C 盘总容量和已用空间 | 建立基线，判断清理是否有必要 |
| **阶段 B：顶层目录排序** | 使用 `Get-ChildItem` 对各顶层目录按大小排序 | 发现占用空间最大的区域，确定深入方向 |
| **阶段 C：逐层深入** | 对占用大的目录，逐层展开子目录并按大小排序 | 定位具体的空间消耗源 |
| **阶段 D：内容抽样** | 对候选大目录，抽样查看文件类型和名称 | 判断目录用途，为后续分类提供依据 |

### 2.2 扫描范围

**磁盘维度（常规清理与卸载均适用）：**

agent 应重点关注以下区域的非系统目录（这些都是用户数据和应用数据的常规位置）：
- 用户配置文件下的应用数据目录（通过环境变量 `$env:LOCALAPPDATA`、`$env:APPDATA` 定位）
- 用户主目录的直接子目录（通过 `$env:USERPROFILE` 定位），包括以 `.` 开头的隐藏配置目录（如 `~\.appname\`）
- 用户文档目录（`$env:USERPROFILE\Documents`），部分应用在此创建设置/工作空间目录
- 非系统盘的应用程序安装目录（如 `D:\02_Engineering_App\` 等），这些目录可能包含已卸载应用的残留
- 系统临时目录（通过 `$env:TEMP` 和 `$env:SystemRoot\Temp` 定位）

**不应扫描或触碰的区域**（除非有明确的系统级安全例外且已验证内容）：
- 系统根目录（`$env:SystemRoot`，即通常的 `C:\Windows`）——除临时目录和 Prefetch 目录（见下方例外）外
- 程序安装目录（`$env:ProgramFiles`、`${env:ProgramFiles(x86)}`）
- 程序数据目录（`$env:ProgramData`）

**非磁盘维度（仅当用户明确要求"彻底卸载/清除某应用的一切痕迹"时扫描）：**

当用户要求彻底卸载某个应用时，除了上述磁盘扫描外，还必须在以下维度进行搜索：

| 维度 | 扫描目标 | 搜索命令 |
|------|---------|---------|
| AppX 包 | 检查应用是否通过 AppX 安装 | `Get-AppxPackage -Name "*AppName*"`；管理员权限下还需 `Get-AppxPackage -AllUsers` |
| AppX 预置包 | 系统预置的 AppX 包（需要管理员） | `Get-AppxProvisionedPackage -Online \| Where-Object DisplayName -match "AppName"` |
| 传统安装注册表 | 检查 HKCU/HKLM Uninstall 注册表键 | 扫描 `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`、`HKLM:\SOFTWARE\WOW6432Node\...`、`HKCU:\SOFTWARE\...` |
| 防火墙规则 | 应用可能添加的入站/出站规则 | `netsh advfirewall firewall show rule name=all dir=in \| Select-String "AppName"`（同时检查 in/out） |
| Prefetch 文件 | Windows 预加载痕迹 | `Get-ChildItem "$env:SystemRoot\Prefetch" -Force \| Where-Object Name -match "appname"` |
| 计划任务 | 应用注册的后台任务 | `Get-ScheduledTask \| Where-Object TaskName -match "appname"` |
| Windows 服务 | 应用注册的系统服务 | `Get-Service \| Where-Object Name -match "appname"` |
| 环境变量 | 应用可能添加的 PATH 或自定义变量 | `Get-ChildItem Env: \| Where-Object Value -match "appname"` |
| 开始菜单快捷方式 | 当前用户和所有用户的快捷方式 | 检查 `$env:APPDATA\Microsoft\Windows\Start Menu` 和 `$env:ProgramData\Microsoft\Windows\Start Menu` |
| 桌面快捷方式 | 当前用户和公共桌面 | 检查 `$env:USERPROFILE\Desktop` 和 `C:\Users\Public\Desktop` |
| 沙箱用户账户 | 应用创建的隔离用户（见第 9.12 节） | `Get-ChildItem "C:\Users"` 查找非常规用户名；`Get-WmiObject Win32_UserAccount` 验证 |
| 注册表 ProfileList | 沙箱用户的配置文件注册（需要管理员） | `Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList"` 匹配 ProfileImagePath |

**注意：** 非磁盘维度扫描可能涉及多个注册表 hive 和系统级配置。若用户当前没有管理员权限，应在报告中标注哪些维度因权限不足未能扫描。

### 2.3 扫描工具使用规范

- 使用 `Get-ChildItem -Directory` 获取目录列表，配合 `Measure-Object` 计算大小
- 使用 `-ErrorAction SilentlyContinue` 处理权限不足的目录
- 可并行扫描多个独立区域（使用 Explore 子代理），加速信息收集
- 扫描结果必须在对话中报告，包含：路径、大小、目录内容抽样（前 5-10 个最大/最常见的文件类型）

### 2.4 符号链接与重解析点处理

- 遇到目录重解析点（Junction / SymbolicLink / MountPoint）时，**仅记录其存在，不递归进入、不计入大小**
- 可使用 PowerShell 的 `Attributes -match 'ReparsePoint'` 检测
- 这是为了防止：通过链接进入系统目录、无限循环、重复计算大小

---

## 3. 分类判断标准

**本节定义"怎样的目录可以安全删除"的判断准则。agent 必须将每个发现的候选目录与以下类别进行匹配。只有明确匹配某一类别的目录，才允许进入清理候选清单。无法匹配任何类别的目录，默认跳过。**

**通用前提：在应用以下任何分类判断之前，agent 必须先完成内容抽样（第 3.10 节）。仅凭目录名不足以做出分类判断。**

### 3.1 第 1 类：临时文件

**安全评级：高（可安全删除）**

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：目录名包含 `Temp`、`tmp`、`.tmp`、`Temporary` 等临时文件关键词，或者位于系统临时目录（通过环境变量获取的临时目录路径）下
2. **内容验证**：目录内主要为临时性质的文件——
   - 扩展名为 `.tmp`、`.temp`、`.log`（旧日志）、`.dmp`（旧转储）
   - 文件修改时间普遍较早（大部分文件距今超过 7 天）
   - 不包含 `.db`、`.sqlite`、`.json`、`.xml` 等配置/数据库格式文件
3. **进程安全**：目录路径中不包含任何当前正在运行进程的名称（通过 `Get-Process` 获取进程列表与之交叉验证）

**排除条件（满足任一即排除，不得删除）：**
- 目录被当前运行进程占用
- 目录内包含配置文件（`.ini`、`.conf`、`.config`、`.yaml`、`.json` 配置文件）
- 目录修改时间在最近 1 小时内（可能是正在使用的临时目录）
- 目录位于 `$env:ProgramFiles` 或 `$env:ProgramData` 下

### 3.2 第 2 类：包管理器下载缓存

**安全评级：高（可安全删除，删除后包管理器可在下次需要时重新下载）**

**判断标准（须同时满足以下所有条件）：**

1. **结构特征**：目录位于应用数据区域（`$env:LOCALAPPDATA` 或 `$env:USERPROFILE` 下），路径或名称中包含包管理工具的名称片段
2. **内容验证**：目录内主要为可分发的包文件——
   - 以 `.tgz`、`.tar.gz`、`.whl`、`.nupkg`、`.vsix` 等包格式为主
   - 或包含 `cache`、`tmp` 子目录且其内为上述包文件
   - **关键区分**：确认这些是下载缓存而非已安装的包（已安装的包通常有 `node_modules`、`packages` 等目录结构，而非单一的包文件）

**排除条件（满足任一即排除，不得删除）：**
- 目录内存在 `package.json`、`node_modules`、`bin/` 等表示已安装包的结构
- 目录同时也是包管理器的全局安装目录（已安装的 CLI 工具所在地）
- 路径中包含 `global`、`global-packages` 等表示全局安装的关键词

### 3.3 第 3 类：应用更新器残留

**安全评级：中高（可安全删除，但删除后可能需要手动检查更新）**

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：目录名包含 `updater`、`update`、`installer`、`patch` 等更新相关关键词，且位于应用数据目录下
2. **内容验证**：目录内主要为安装程序文件——
   - 以 `.exe`、`.msi`、`.msp` 安装程序为主
   - 或包含 `pending`、`downloads` 子目录存放下载的更新包
   - **关键区分**：确认这是更新器缓存/下载目录，而非应用本体。应用本体的可执行文件通常更大且包含 `.dll` 等运行时库

**排除条件（满足任一即排除，不得删除）：**
- 目录内包含 `.dll`、`.so` 等运行时库文件（可能是应用本体的更新程序目录）
- 目录内可执行文件正在运行
- 目录大小异常大（>5GB），可能包含完整安装包而非仅增量更新

### 3.4 第 4 类：崩溃转储文件

**安全评级：高（可安全删除，仅当用户不需要调试历史崩溃时）**

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：目录名包含 `Crash`、`Dump`、`CrashDumps`、`reports` 等关键词
2. **内容验证**：目录内主要为——
   - `.dmp`、`.mdmp`、`.hdmp` 转储文件
   - `.log` 崩溃日志文件
   - 不包含可执行文件或配置文件

**排除条件（满足任一即排除，不得删除）：**
- 目录同时包含源代码或调试符号文件（`.pdb`）
- 崩溃文件来自最近 24 小时内（可能正在调试问题）

### 3.5 第 5 类：运行时和环境缓存

**安全评级：中高（可安全删除，应用/工具下次运行时会自动重建）**

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：目录路径中包含已知可自动重建的缓存框架/运行时名称，且目录名明确为缓存目录（如 `cache`、`Cache`、`cached` 子目录）
2. **内容验证**：目录内为二进制缓存文件、编译产物或运行时自动生成的数据——
   - 浏览器渲染引擎的二进制库文件（通常数百 MB 的 `.dll` / `.so` 文件，位于 `*cache*` 目录下）
   - JIT 编译缓存（`.jsc`、编译后的字节码）
   - 字体缓存（`.dat` 缓存文件）
   - **关键区分**：确认这些文件可以被对应工具自动重新下载或生成，而非需要用户手动安装

**排除条件（满足任一即排除，不得删除）：**
- 缓存的框架不在当前系统 PATH 中（可能已卸载，但用户可能保留了使用数据）
- 缓存目录内包含用户自定义的配置文件或脚本
- 缓存目录位于应用安装目录（`$env:ProgramFiles` 下）而非用户数据目录

### 3.5.1 第 5.1 类：浏览器 AI/ML 模型缓存

**安全评级：中高（可安全删除，浏览器会在需要时自动重新下载模型文件）**

**识别方式：** 位于浏览器 User Data 目录下，目录名与已知 AI/ML 模型下载模式匹配。

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：位于浏览器 `User Data` 目录的直接子目录，目录名匹配以下模式之一——
   - `OptGuideOnDeviceModel` — Chrome 端侧优化指南模型
   - `screen_ai` — Chrome 屏幕 OCR/AI 模型
   - `optimization_guide_model_store` — Chrome 优化指南模型存储
   - 或名称包含 `model` / `Model` 关键字的 AI 模型下载目录
2. **内容验证**：目录内为浏览器自动下载的 AI 模型文件——
   - 以 `.bin`（模型权重）、`.pb`（TensorFlow 模型）、`.tflite`（TensorFlow Lite）、`.binarypb` 等 ML 模型格式为主
   - 通常包含 `manifest.json` 描述文件
   - 单个文件可极大（`weights.bin` 可能超过 4GB）
3. **进程安全**：不在浏览器进程活跃写入状态（可尝试删除，文件锁定即跳过）

**排除条件（满足任一即排除，不得删除）：**
- 目录同时包含用户训练的模型或自定义模型配置
- 目录内包含 `.py`、`.ipynb` 等代码文件（可能是用户 ML 项目目录）

**注意：** 本类目录虽然位于浏览器厂商数据目录下，但内容明确为浏览器可自动重新下载的 ML 模型文件，不作为用户配置数据处理。此类目录清理在 Rule 4.3 中有对应的例外条款。

### 3.6 第 6 类：GPU 着色器/渲染缓存

**安全评级：高（可安全删除，运行游戏或图形应用时自动重建，可能首次加载略慢）**

**判断标准（须同时满足以下所有条件）：**

1. **路径特征**：目录名包含着色器/渲染缓存关键词，如 `ShaderCache`、`GPUCache`、`DawnCache`、`VkCache`、`ProgramBinaryCache`、`D3DSCache` 等
2. **内容验证**：目录内为二进制缓存文件（`.bin`、`.dat`、`.cache`）
3. **大小合理性**：目录大小通常 < 5GB。若超过 5GB，需额外抽样确认未混入用户数据

**排除条件（满足任一即排除，不得删除）：**
- 目录内包含 `.png`、`.jpg` 等图像文件（可能是纹理/截图目录，非纯缓存）
- 目录内包含 `.blob` 文件同时夹杂用户数据特征

### 3.7 第 7 类：浏览器缓存

**安全评级：中（可安全删除缓存，但需精确指定缓存子目录，不得触碰用户数据）**

**识别方式：** agent 应在应用数据目录下扫描以下特征来发现浏览器数据目录：
- 存在 `User Data` 或 `userData` 或 `Profiles` 子目录的目录结构
- 存在 `.exe` 浏览器可执行文件且数据目录独立

**浏览器数据目录的内部结构——可清理 vs 不可清理的判定规则：**

浏览器的用户数据目录（如某个 Profile 目录）内，**仅以下子目录名称匹配的可视为"可清理缓存"**：

| 可清理的缓存目录名 | 内容性质 |
|-------------------|---------|
| `Cache` | 页面资源缓存，重启浏览器自动重建 |
| `Code Cache` | JavaScript/WASM 编译缓存 |
| `GPUCache` | GPU 着色器缓存 |
| `DawnWebGPUCache` | WebGPU 着色器编译缓存（Chromium 系浏览器） |
| `DawnGraphiteCache` | Graphite 渲染缓存（Chromium 系浏览器） |
| `GrShaderCache` | 全局 GPU 着色器编译缓存（Chromium 系浏览器） |
| `ShaderCache` | GPU 着色器缓存（通用变体名） |
| `component_crx_cache` | 浏览器扩展组件下载缓存 |
| `cache2` | 部分浏览器（如 Firefox）使用的页面缓存目录名 |
| 位于 `Service Worker\` 下的 `CacheStorage` | Service Worker 离线缓存 |
| `blob_storage` | Blob 临时存储 |
| `Session Storage` | 会话临时存储 |

**红线规则（以下目录名称匹配的，一律不删，无例外）：**

以下目录包含用户个人数据、登录状态或扩展数据，**名称匹配即保护**：

| 不可删除的目录/文件 | 原因 |
|---------------------|------|
| `History` | 浏览记录 |
| `Login Data` | 保存的密码 |
| `Cookies` | 登录状态和站点偏好 |
| `Bookmarks` | 用户书签 |
| `Preferences` | 浏览器设置 |
| `Web Data` | 自动填充、站点设置 |
| `Sync Data` | 同步配置 |
| `Extensions` | 已安装扩展 |
| `Local Extension Settings` | 扩展的本地设置和数据 |
| `IndexedDB` | 网站数据库（可能含离线数据） |
| `databases` | 网站数据库 |
| `storage` | 网站持久存储（可能含用户数据） |
| `File System` | 网站文件系统 API 存储 |
| `Local Storage` | 网站 localStorage 数据 |
| `Session Storage` | （仅当位于根目录而非浏览上下文中的临时会话存储时保护） |

**执行要求：**
- 必须精确指定完整子目录路径删除，不得使用宽泛通配符
- 删除前必须先用 `Get-ChildItem` 确认目录内容仅含缓存文件
- 每次只能删除一个 Profile 的一个缓存子目录，不得批量删除多个目录

### 3.8 第 8 类：通用应用缓存

**安全评级：低—中（必须逐个验证，确认是纯缓存才能删除）**

**识别方式：** 在应用数据目录（`$env:LOCALAPPDATA` 和 `$env:APPDATA`）下的各应用子目录中，查找名为 `Cache`、`cache`、`caches` 的子目录。

**判断标准（须同时满足以下所有条件）：**

1. 目录名精确为 `Cache`、`cache` 或 `caches`
2. 目录内容 100% 为缓存文件（二进制缓存、缩略图、预计算数据），不含 `.db`、`.sqlite`、`.json`、`.xml`
3. 父目录可明确识别为某一应用（非系统组件，非通信/社交应用，非云同步应用）
4. 不属于本节下方"高风险应用类别识别规则"表格中描述的任何类别

**高风险应用类别识别规则（这些应用的数据目录一律保护，仅允许清理其下明确命名的浏览器类缓存子目录）：**

| 应用类别 | 识别特征 | 可清理例外 |
|---------|---------|-----------|
| 通信/社交应用 | 数据目录位于 `$env:APPDATA`（Roaming）下；目录内包含 `Databases`、`Msg`、`Chat`、`messages` 等子目录或文件 | 仅其下的嵌入式浏览器缓存子目录（按第 3.7 节规则判断） |
| 云同步/网盘应用 | 目录内包含大量与云上同步的文件夹结构；内容为 `.docx`、`.pdf`、`.jpg` 等用户文件 | 无——一律保护，不清理 |
| 虚拟机/模拟器 | 目录大小极大（>10GB）；包含 `.vdi`、`.vmdk`、`.vhdx`、`.qcow2` 等虚拟磁盘文件 | 无——一律保护，不清理 |
| 数据库应用 | 目录内包含数据库文件（`.db`、`.mdf`、`.ndf`、数据目录） | 无——一律保护，不清理 |

### 3.9 第 9 类：IDE 和开发工具缓存

**安全评级：中（可清理编译缓存和运行时缓存，但必须保护用户设置、扩展、工作区数据）**

**识别方式：** agent 应在应用数据目录下扫描具有以下特征的目录来识别 IDE：
- 目录内同时存在用户配置子目录（`User/`、`user-data/`、`config/`）和缓存子目录（`Cache/`、`CachedData/`、`CachedExtensions/`）
- 目录内存在 `extensions/` 或 `extensions.json` 等扩展管理结构
- 目录内存在 `workspace-storage/` 或 `workspaces/` 工作区数据

**可清理的缓存目录名模式（名称匹配即可归入缓存）：**

| 模式 | 说明 |
|------|------|
| 名称包含 `Cache`（如 `Cache/`、`Code Cache/`、`GPUCache/`、`DawnGraphiteCache/`、`DawnWebGPUCache/`） | 各类渲染和编译缓存 |
| 名称以 `Cached` 开头（如 `CachedExtensions/`、`CachedData/`、`CachedProfilesData/`、`CachedConfigurations/`） | 扩展和配置缓存 |
| `Crashpad/`、`logs/` | 崩溃报告和日志 |
| `blob_storage/`、`Network/`、`Partitions/`、`Shared Dictionary/`、`WebStorage/` | 网络和存储临时数据 |

**红线规则（以下目录名称匹配的，一律不删）：**

| 不可删除的目录 | 性质 |
|---------------|------|
| `User/` 或 `user-data/` | IDE 设置、快捷键、代码片段、工作区状态 |
| `ModularData/` | 代理数据库、沙箱配置、VM 快照 |
| `Backups/` | 用户备份文件 |
| `Workspaces/` | 工作区数据 |
| `Local Storage/`、`IndexedDB/`、`Session Storage/` | 扩展存储 |
| `globalStorage/` | 扩展全局数据 |
| `extensions/` | 已安装扩展（删除后需重新下载安装，不应作为常规清理项） |
| `clp/` | 功能数据 |

### 3.10 内容抽样验证规范

**在将任何目录归入上述任何类别之前，agent 必须先执行内容抽样。以下为抽样规范：**

1. **抽样方法**：使用 `Get-ChildItem -Recurse -Depth 2 | Group-Object Extension | Sort-Object Count -Descending | Select-Object -First 10` 获取扩展名分布
2. **抽样判断**：
   - 若目录中 90% 以上的文件（按数量）为缓存/临时文件类型（`.tmp`、`.cache`、`.dmp`、`.bin`、`.dat`、`.log`）→ 可归入相应缓存类别
   - 若目录中出现任何比例的用户数据文件（`.db`、`.sqlite`、`.json`（>100KB）、`.xml`（>100KB）、`.docx`、`.xlsx`、`.pdf`、`.jpg`、`.png`、`.psd` 等）→ 停止归类，标记为"需用户决策"
   - 若目录中 30% 以上的文件无法识别类型 → 停止归类，标记为"需用户决策"
3. **大小异常检查**：缓存类目录的大小通常应在合理范围内。若某缓存目录异常大（如某浏览器缓存超过 2GB，或某 IDE 缓存超过 10GB），需额外深入抽样，确认未混入用户数据

---

## 4. 保护规则

**本节定义"一旦命中即跳过，不做任何操作"的保护规则。这些规则是安全网——即使扫描和分类阶段出现了误判，保护规则也必须最终拦截不安全的操作。**

### 4.1 举证责任规则

**默认立场：一切目录受保护。** 只有满足以下全部条件的目录才能进入清理候选清单：
1. 明确匹配第 3 节中某一种"可清理"类别
2. 已执行内容抽样验证，结果支持该分类
3. 未命中第 4 节中的任何"不可触碰"的保护规则

**任何一项不满足 → 目录排除，标记为"需用户单独决策"或"受保护"。**

### 4.2 系统目录保护

**识别方式：** 通过路径前缀与环境变量匹配。

| 保护范围 | 识别方式 | 已知安全例外 |
|---------|---------|-------------|
| 操作系统根目录下的所有子目录 | 路径为 `$env:SystemRoot` 的子目录 | `$env:SystemRoot\Temp` 下的子目录（仅清理内容）；`$env:SystemRoot\SoftwareDistribution\Download`（仅清理内容） |
| 系统旧版备份 | 路径前缀为 `$env:SystemDrive\Windows.old` | 无 |
| 64 位程序安装目录 | 路径前缀为 `$env:ProgramFiles` | 无 |
| 32 位程序安装目录 | 路径前缀为 `${env:ProgramFiles(x86)}` | 无 |
| 应用全局数据目录 | 路径前缀为 `$env:ProgramData` | 无 |

**执行规则：** 遇到路径前缀匹配上述保护范围的目录，除已列出的精确例外路径外，一律跳过。例外路径也需在执行前验证内容确为缓存/临时文件。

### 4.3 厂商应用数据保护

**这些规则基于路径结构而非具体路径值判断——无论电脑上实际路径如何，只要路径结构匹配即触发保护。**

**识别方式：** 以下规则通过检查路径中的目录名片段和位置来匹配，而非匹配完整路径。

| 保护规则 | 触发条件 | 已知安全例外 |
|---------|---------|-------------|
| Microsoft 本地应用数据 | 路径中包含 `\AppData\Local\Microsoft\` 片段 | `\Microsoft\Windows\INetCache`（网络缓存）；Microsoft Edge 浏览器 `User Data` 目录下：(a) 名称匹配第 3.7 节"可清理"列表的缓存子目录，(b) 名称匹配第 3.6 节 GPU 着色器缓存关键词的子目录，(c) 名称匹配第 3.5.1 节 AI/ML 模型缓存模式的子目录 |
| Microsoft 漫游应用数据 | 路径中包含 `\AppData\Roaming\Microsoft\` 片段 | 无 |
| Google 本地应用数据 | 路径中包含 `\AppData\Local\Google\` 片段 | Google Chrome 浏览器 `User Data` 目录下：(a) 名称匹配第 3.7 节"可清理"列表的缓存子目录，(b) 名称匹配第 3.6 节 GPU 着色器缓存关键词的子目录，(c) 名称匹配第 3.5.1 节 AI/ML 模型缓存模式的子目录 |
| Windows Store 应用 | 路径中包含 `\AppData\Local\Packages\` 片段 | 无 |

**执行规则：** 厂商数据目录允许以下三类例外（且必须精确到子目录），其他子目录一律保护，不做任何操作：
1. **浏览器缓存例外**：名称匹配第 3.7 节"可清理"列表的缓存子目录
2. **GPU 着色器缓存例外**：名称包含第 3.6 节所列 GPU 缓存关键词（如 `GPUCache`、`ShaderCache`、`DawnCache`、`VkCache` 等）的子目录
3. **AI/ML 模型缓存例外**：名称匹配第 3.5.1 节所列 AI 模型缓存模式的子目录

### 4.4 用户数据和登录状态保护

以下类别目录**一律保护，无例外**：

| 保护类别 | 识别特征 |
|---------|---------|
| 通信/社交应用的漫游数据 | 位于 `$env:APPDATA` 下，目录内包含聊天记录、联系人、数据库文件 |
| 全局包管理器的已安装目录 | 位于 `$env:APPDATA` 下，包含全局安装的 CLI 工具二进制文件（`npm` global、`yarn` global、`pip` 用户安装等） |
| IDE 用户配置目录 | 任何 IDE 目录下的 `User/`、`user-data/`、`config/` 子目录 |
| 模拟器/虚拟机数据 | 目录内包含虚拟磁盘文件或模拟器系统镜像 |
| 云同步客户端数据 | 目录内包含与云端同步的文件结构和配置数据库 |
| 数据库数据文件目录 | 目录内包含 `.db`、`.mdf`、`.ndf` 等数据库文件 |

### 4.5 无法判定的目录——兜底规则

**遇到无法通过第 3 节分类规则和第 4 节保护规则明确判断的目录（即"灰色地带"），默认策略：**

1. **不删除**
2. **记录路径、大小和不确定的原因**
3. **在清理报告中单独列出，标注为"需用户决策"**
4. **向用户解释为何无法自动判定，列出目录内容抽样结果，由用户自行决定**

**禁止行为：** 在不确定的情况下，"赌一把"删除。任何不确定性都应导向"不删除"。

---

## 5. 回收站清理

### 5.1 定位回收站

回收站位于系统盘根目录下的隐藏系统目录中：

```powershell
# 检查回收站大小（不依赖具体盘符）
$recyclePath = "$env:SystemDrive`\$Recycle.Bin"
(Get-ChildItem $recyclePath -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1MB
```

### 5.2 清空回收站

```powershell
Clear-RecycleBin -Force
```

**注意：** 回收站清空不可恢复，但回收站中的文件已经是用户"删除"过的。仍需在确认清单中单独列出回收站大小，将回收站清空作为一个独立的确认项。

---

## 6. 三重确认流程

**所有清理操作必须经过"扫描归类 → 确认 → 自审 → 再确认 → 执行"三步确认。任何一步不可跳过。**

### 6.1 第一轮确认：扫描结果与候选清单

**在完成扫描和分类后，生成结构化的清理候选清单。**

**清单格式要求：** 对每个候选目录，必须提供以下完整信息：

| 字段 | 说明 |
|------|------|
| 目录路径 | 完整的绝对路径 |
| 大小 | 当前占用空间（MB/GB） |
| 归属类别 | 匹配第 3 节中的哪种类别 |
| 安全评级 | 高 / 中高 / 中 / 低（需谨慎） |
| 内容概要 | 抽样结果——主要文件类型和数量级 |
| 删除影响 | 删除后有什么影响（"可自动重建" / "需重新下载" / "首次运行略慢" 等） |
| 判断依据 | 列出做出该分类判断的具体依据（匹配了哪些识别特征） |

**使用 `AskUserQuestion` 工具将清单呈现给用户，请求确认。用户可以选择：**
- 全部同意
- 逐类勾选
- 排除特定项目
- 要求对特定项目进一步分析

### 6.2 第二轮：AI 自我审查

**在用户第一轮确认后、执行删除前，agent 必须执行以下系统化自审。不可跳过。**

**自审清单（逐条检查，每条必须有"通过 / 排除 / 存疑"的明确结论）：**

| # | 自审项 | 检查内容 | 判断标准 |
|---|--------|----------|----------|
| 1 | 保护规则交叉检查 | 候选清单中是否有路径命中了第 4 节任何保护规则？ | 命中 → 排除该项 |
| 2 | 活跃进程关联检查 | 候选目录路径或父目录名是否与当前运行进程名（`Get-Process`）有关联？ | 有关联 → 默认排除该项。**例外**：浏览器缓存子目录（第 3.7 节）和浏览器 AI 模型缓存（第 3.5.1 节）可保留在清单中——这些缓存在浏览器运行时删除是常规操作，文件锁会自然保护活跃文件。其他类型（IDE、应用本体等）有关联则必须排除 |
| 3 | 内容验证复审 | 是否对每个候选目录都实际执行了内容抽样？（而非仅凭目录名判断） | 未做抽样 → 排除该项或回退补充抽样 |
| 4 | 用户数据误判检查 | 候选目录中是否有被错误归类的？（如将存储用户数据的 `storage/` 目录误认为缓存） | 发现误判 → 排除该项 |
| 5 | 级联影响评估 | 删除该目录是否会导致依赖它的其他应用/服务异常？ | 有风险 → 排除或降级 |
| 6 | 大小异常检查 | 候选目录大小是否远超同类目录的常规范围？（如某"缓存"目录达数十 GB） | 异常 → 深入抽样或排除 |
| 7 | 通配符/批量操作检查 | 任何删除命令是否使用了通配符或批量操作，可能误伤相邻目录？ | 有宽泛通配符 → 改为逐个精确路径 |
| 8 | 最小化检查 | 是否对每个候选目录，只删除确实可安全删除的子目录，而非整个父目录？ | 范围过大 → 缩小到精确子目录 |
| 9 | 恢复可能性检查 | 删除后是否可通过自动重建/重新下载恢复？如果无法恢复 → 排除 | 不可恢复 → 排除 |
| 10 | 时间窗口检查 | 是否存在最近被修改的缓存目录（表明正在被活跃使用）？ | 最近 1 小时内修改 → 排除 |

**自审结果处理：**
- 每发现一个问题，从候选清单中移除对应目录，记录移除原因
- 自审完成后，汇总：保留 N 项，移除 M 项（含原因）
- 将自审结果呈现给用户

### 6.3 第三轮：最终确认

**将自审结果（保留项、移除项及原因、任何存疑项）呈现给用户，使用 `AskUserQuestion` 再次请求确认。**

用户确认后方可进入执行阶段。

---

## 7. 执行规范

### 7.1 执行前准备

1. 记录磁盘基线：`(Get-PSDrive C).Used`
2. 确认所有待删除路径存在且可访问
3. 准备跳过列表和错误日志

### 7.2 执行顺序

按安全评级从高到低执行：
1. 临时文件（第 1 类）
2. 崩溃转储（第 4 类）
3. GPU 缓存（第 6 类）
4. 浏览器 AI/ML 模型缓存（第 5.1 类）
5. 包管理器缓存（第 2 类）
6. 更新器残留（第 3 类）
7. 运行时缓存（第 5 类）
8. 浏览器缓存（第 7 类）
9. 通用应用缓存（第 8 类）
10. IDE 缓存（第 9 类）
11. 回收站清空（最后执行）

### 7.3 删除操作规范

**每条删除命令必须：**
- 使用精确的完整路径（不得使用宽泛通配符）
- 单条命令只删除一个目标
- 使用 `Remove-Item -Recurse -Force -ErrorAction Stop`，配合错误捕获
- 记录每个目标的操作结果

**错误处理：**
- 文件被占用 → 跳过，记录原因，继续下一个
- 权限不足 → 跳过，记录原因，继续下一个（不尝试提权）
- 路径不存在 → 标记为"已被清理"（⏭️），记录原因（可能已被卸载程序或其他进程清理，这是正常情况），继续下一个
- 路径不存在且其他相关目录也已消失 → 标记为"卸载程序已自动清理"，视为成功而非失败

**浏览器缓存特殊要求：**
- 必须为每个 Profile 的每个缓存子目录单独执行 `Remove-Item`
- 不得使用 `*` 通配符匹配 Profile 目录

**系统临时目录特殊处理：**
删除临时目录内容时，先获取当前运行进程列表，跳过与进程名相关的子目录：
```powershell
$running = (Get-Process).ProcessName | Select-Object -Unique
Get-ChildItem $tempPath -Directory -Force | Where-Object { $running -notcontains $_.Name } | Remove-Item -Recurse -Force -ErrorAction Continue
Get-ChildItem $tempPath -File -Force | Remove-Item -Force -ErrorAction Continue
```

### 7.4 回收站清理

在整个文件清理完成后，验证阶段之前，执行：
```powershell
Clear-RecycleBin -Force -ErrorAction Continue
```

---

## 8. 验证与残留检查

### 8.1 空间回收验证

```powershell
$d = Get-PSDrive C
$freed = $baselineUsed - $d.Used
Write-Output "回收空间: $([math]::Round($freed/1MB,1)) MB ($([math]::Round($freed/1GB,2)) GB)"
Write-Output "当前剩余: $([math]::Round($d.Free/1GB,2)) GB / $([math]::Round(($d.Used+$d.Free)/1GB,2)) GB"
```

**注意：** 清理期间系统可能写入新文件（日志、更新缓存），导致"已释放"计算略有偏差。使用清理前记录的基线对比，而非实时查询。

### 8.2 残留文件检查

```powershell
Get-ChildItem "$env:TEMP\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:USERPROFILE\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:LOCALAPPDATA\cleanup_*" -ErrorAction SilentlyContinue
```
若发现残留文件，立即删除。

### 8.3 操作完整性验证

逐项检查清理清单中每个目标的操作结果：
- ✅ 成功删除
- ⏭️ 跳过（附原因：文件占用 / 权限不足 / 路径不存在）
- ❌ 失败（附错误信息）

---

## 9. 陷阱与边缘情况

### 9.1 Bash 环境下执行 PowerShell 命令

在 bash 终端中运行 PowerShell 命令时，`$()` 和 `$var` 会被 bash 解释。解决方案：
- **简单命令**：使用单引号包裹 PowerShell 参数
- **复杂逻辑**：使用 `powershell -Command '...'` 内联执行，**绝不写入 .ps1 文件**

**已知会导致 bash 截断的模式及替代方案：**

| 问题模式 | 失败原因 | 替代方案 |
|---------|---------|---------|
| `$($_.Name)` 或 `$($c.Substring(...))` | bash 将 `$()` 解释为命令替换 | 避免在 PowerShell 字符串内嵌 `$()`；改用临时变量赋值后再引用 |
| `$_.FullName` 中的反斜杠路径 | bash 将 `\` 解释为转义字符 | 使用 `$_.FullName.Replace('\','/')` 或直接用单引号包裹整条命令 |
| `foreach($c in $caches) { try { ... } catch { ... } }` | bash 将 `}` 和 `)` 解释为语法 | 将循环展开为逐条独立命令；或将 foreach 替换为 `ForEach-Object` |
| 字符串中含 `\n` 反斜杠 | bash 将 `\` 与后续字符组合 | 使用单引号包裹所有静态字符串 |

**实战验证过的安全写法模式：**

```bash
# ✅ 正确：逐条独立命令，避免循环和 $() 内嵌
powershell -Command 'Remove-Item "path1" -Recurse -Force -ErrorAction SilentlyContinue; Remove-Item "path2" -Recurse -Force -ErrorAction SilentlyContinue; Write-Output "Done"'

# ❌ 错误：在 bash -Command 内使用 foreach 循环和 $() 字符串插值
powershell -Command "foreach($c in $caches) { try { Remove-Item $c -Force } catch { Write-Output \"SKIP: $($c.Name)\" } }"
```

**通用规则：** 若 PowerShell 命令含以下任一元素，必须拆分为多条简单命令或使用 `ForEach-Object` + 单引号：
- `$()` 子表达式
- 多行 `try/catch` 块
- 字符串中含有反斜杠 `\`（Windows 路径）又同时有 `$var` 引用

### 9.2 文件占用

删除时某些文件可能被进程占用。这是正常现象——捕获错误后跳过即可，不影响清理效果，也不应导致整个清理流程中断。

### 9.3 磁盘空间基线漂移

清理期间系统可能写入新文件（日志、缓存），导致"已释放"计算略有偏差。使用清理前记录的 `$baselineUsed` 对比。

### 9.4 目录名欺骗

**这是最常见的误删原因。** 某些目录名看似缓存（如名为 `Cache` 的目录），但实际存储用户数据。**不可仅凭目录名判断，必须查看目录内容构成。**

**强制规则：**
- 如果目录内包含任何以下文件类型且占比超过总文件数的 5%，停止自动归类，标记为"需用户决策"：
  - `.db`、`.sqlite`、`.sqlite3`（数据库文件）
  - `.json`、`.xml` 且文件大小 > 100KB（可能为数据存储而非配置）
  - `.dat` 且文件大小 > 10MB（可能为用户数据文件）
  - `.jpg`、`.png`、`.svg`、`.gif` 等图像文件
  - `.docx`、`.xlsx`、`.pptx`、`.pdf` 等文档文件

### 9.5 浏览器多配置文件

浏览器可能有多个 Profile 目录（`Default`、`Profile 1`、`Profile 2` 等）。处理规则：
- 每个 Profile 独立按照第 3.7 节规则判断
- 使用精确路径逐个处理，**不得使用通配符匹配 Profiles**
- 如果 Profile 数量 > 5，只需向用户确认一次（"检测到 N 个浏览器 Profile，是否对所有 Profile 执行相同的缓存清理？"）

### 9.6 通信/社交应用的特殊处理

此类应用的数据目录是**最高风险区域**之一。处理规则：
- 默认仅报告其占用空间，不将其任何子目录加入清理候选清单
- 如果用户明确要求清理，仅当能精确到嵌入式浏览器的缓存子目录（按第 3.7 节规则），且已逐文件验证内容为浏览器缓存时，才可加入候选清单
- 对这些应用的处理必须格外保守——宁可漏过，不可误删

### 9.7 符号链接、Junction 和挂载点

- **扫描时**：使用 `Attributes -match 'ReparsePoint'` 检测，仅记录链接的存在和指向目标，不递归进入
- **删除时**：仅删除链接本身（如果它是清理目标），不递归删除链接指向的目标内容
- **绝对禁止**：通过符号链接或 Junction 进入系统目录并执行删除操作

### 9.8 权限错误

遇到权限不足的目录：
- 扫描时：使用 `-ErrorAction SilentlyContinue` 静默跳过
- 删除时：不使用管理员权限提权，正常跳过并记录
- **禁止行为**：建议用户"以管理员身份运行"来绕过权限保护

### 9.9 隐藏文件和系统文件

- 扫描时使用 `-Force` 参数包含隐藏项，确保大小计算完整
- 删除时仅删除已确认的清理候选，不因为其隐藏属性而特殊对待
- 遇到设置了"系统"属性的文件（`Attributes -match 'System'`）：不删除，标记并报告

### 9.10 应用卸载与跨维度痕迹清理

当用户明确要求"彻底卸载/清除某个应用的一切痕迹"时，agent 必须执行以下完整流程，而非仅做 C 盘空间清理。

#### 9.10.1 第一步：识别应用安装方式

执行卸载前，先确定应用的安装方式，这决定了卸载策略：

| 安装方式 | 识别特征 | 卸载方法 |
|---------|---------|---------|
| AppX/UWP 包 | `Get-AppxPackage` 能找到 | `Remove-AppxPackage`（见 9.10.2） |
| 传统安装程序（Inno Setup） | 注册表 Uninstall 键包含 `unins000.exe`，通常安装在自定义路径（如 `D:\Apps\`） | 先运行官方卸载程序（静默模式），再手动清除残留 |
| 传统安装程序（MSI） | 注册表 Uninstall 键包含 `MsiExec.exe` | `msiexec /x {ProductCode} /quiet` |
| 便携版/绿色版 | 无注册表 Uninstall 条目，无 AppX 包 | 直接删除应用目录和用户数据 |
| 用户级安装（`%LOCALAPPDATA%` 下） | 安装路径在 `$env:LOCALAPPDATA\Programs\` 下 | 检查是否有 `Update.exe` 或 `uninstall.exe` |

#### 9.10.2 AppX/UWP 应用卸载

```powershell
# 1. 先查找包名（不需要通配符时用精确名）
Get-AppxPackage -Name "*AppName*" | Select-Object Name, PackageFullName

# 2. 正规卸载
Get-AppxPackage -Name "*AppName*" | Remove-AppxPackage

# 3. 管理员权限下，检查 AllUsers 和预置包
# Get-AppxPackage -AllUsers -Name "*AppName*" | Remove-AppxPackage
# Get-AppxProvisionedPackage -Online | Where-Object DisplayName -match "AppName" | Remove-AppxProvisionedPackage -Online
```

**AppX 卸载后常见残留：**
- `$env:LOCALAPPDATA\Packages\<Publisher.AppName_*>` — AppX 用户数据目录，卸载后可能残留，可安全删除
- `$env:USERPROFILE\` 下的 `.appname` 隐藏配置目录
- `$env:USERPROFILE\.cache\appname-runtimes` — 运行时缓存
- 应用在 `$env:USERPROFILE\Documents` 下创建的工作空间/设置目录

**注意：** `Get-AppxPackage` 默认仅查询当前用户。若权限不足导致无法完全清理（如无法查询 `-AllUsers`、无法删除 `C:\Users\` 下受保护的用户目录），应在报告中说明哪些维度假定为干净但未经实际验证。

#### 9.10.3 传统安装程序卸载

**关键原则：先运行官方卸载程序，再手动清除残留。** 传统卸载程序（尤其是 Inno Setup 的 `unins000.exe`）通常只会删除安装时写入的文件，**不会清理用户数据目录和运行时生成的缓存**。

```powershell
# 1. 从注册表定位卸载程序
$uninstallKey = Get-ChildItem "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" -ErrorAction SilentlyContinue | 
  Where-Object { (Get-ItemProperty $_.PSPath -Name DisplayName -ErrorAction SilentlyContinue).DisplayName -match "AppName" }
$uninstaller = (Get-ItemProperty $uninstallKey.PSPath -Name UninstallString).UninstallString

# 2. 静默运行卸载程序（Inno Setup 参数）
# Start-Process "D:\Path\unins000.exe" -ArgumentList "/VERYSILENT /SUPPRESSMSGBOXES /NORESTART" -Wait

# 3. 卸载后验证并手动清除残留（见 9.10.4 验证清单）
```

**常见卸载程序静默参数：**
| 安装工具 | 静默卸载参数 |
|---------|-------------|
| Inno Setup | `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART` |
| NSIS | `/S` |
| MSI | `/quiet /norestart`（通过 `msiexec /x` 调用） |
| InstallShield | `/s /f1"<response_file>"` |

#### 9.10.4 卸载后跨维度验证与清理清单

卸载程序运行后，按以下清单逐维度检查并清理残留：

```powershell
# === 1. 安装目录（卸载程序可能留下空目录或配置文件）===
Remove-Item "<安装路径>" -Recurse -Force -ErrorAction SilentlyContinue

# === 2. AppData 用户数据（最常见的残留，卸载程序通常不清理）===
# Roaming: 用户配置、工作区、扩展数据
Remove-Item "$env:APPDATA\AppName" -Recurse -Force -ErrorAction SilentlyContinue
# Local: 缓存、运行时数据
Remove-Item "$env:LOCALAPPDATA\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 3. 用户主目录下的隐藏配置 ===
Remove-Item "$env:USERPROFILE\.appname" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.appname-suffix" -Recurse -Force -ErrorAction SilentlyContinue

# === 4. 文档/工作空间目录 ===
Remove-Item "$env:USERPROFILE\Documents\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 5. 桌面快捷方式 ===
Remove-Item "$env:USERPROFILE\Desktop\AppName.lnk" -Force -ErrorAction SilentlyContinue

# === 6. 开始菜单 ===
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 7. Prefetch 文件（需要管理员权限）===
Get-ChildItem "$env:SystemRoot\Prefetch" -Force | Where-Object Name -match "appname\.exe|APPNAME" | Remove-Item -Force -ErrorAction SilentlyContinue

# === 8. 防火墙规则（需要管理员权限）===
netsh advfirewall firewall delete rule name="RuleName"  # 分别对 dir=in 和 dir=out 执行

# === 9. 注册表 Uninstall 条目 ===
Remove-Item "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{GUID}" -Recurse -Force -ErrorAction SilentlyContinue
# 若有 HKLM 条目，也需要管理员权限

# === 10. 沙箱用户账户和目录（见 9.12 节）===
```

#### 9.10.5 目录复用警告

**卸载场景中的特殊风险：** 某些目录名（如 `.codex`）可能被多个不同应用使用。删除前必须确认目录内容确实属于要卸载的应用，而非其他仍在使用的应用。

**判断方法：**
1. 检查目录内文件是否引用了要卸载的应用名称（配置文件中的 `appName`、路径引用等）
2. 检查目录的修改时间是否与当前会话中其他正在运行的应用匹配
3. 若不确定，采样目录内容并报告给用户判断
4. **禁止行为：** 仅凭目录名匹配就删除，而不验证内容归属

### 9.11 完整卸载后的终端验证

**当用户要求"彻底清除某个应用的一切痕迹"时，卸载和手动清理完成后，必须执行本节定义的终端验证扫描，确认零残留。**

#### 9.12.1 验证扫描脚本

以下脚本应在所有清理步骤完成后执行，作为最终确认：

```powershell
# 完整卸载终端验证（将 <Pattern> 替换为应用名称的匹配模式）
$pattern = "<AppNamePattern>"  # 如 "codex|Codex|CODEX" 或 "trae|Trae|TRAE"

Write-Output "=== [1/8] 文件系统扫描 ==="
@("$env:USERPROFILE","$env:LOCALAPPDATA","$env:APPDATA",
  "$env:ProgramFiles","${env:ProgramFiles(x86)}","$env:ProgramData",
  "C:\Users","D:\","$env:USERPROFILE\Desktop") | ForEach-Object {
  Get-ChildItem $_ -Directory -Force -ErrorAction SilentlyContinue | 
    Where-Object Name -match $pattern | ForEach-Object { Write-Output "DIR: $($_.FullName)" }
  Get-ChildItem $_ -File -Force -ErrorAction SilentlyContinue | 
    Where-Object Name -match $pattern | ForEach-Object { Write-Output "FILE: $($_.FullName)" }
}

Write-Output "`n=== [2/8] Prefetch 文件 ==="
Get-ChildItem "$env:SystemRoot\Prefetch" -Force -ErrorAction SilentlyContinue |
  Where-Object Name -match $pattern | ForEach-Object { Write-Output "PF: $($_.Name)" }

Write-Output "`n=== [3/8] 注册表 ==="
@("HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
  "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
  "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall") | ForEach-Object {
  Get-ChildItem $_ -ErrorAction SilentlyContinue | ForEach-Object {
    $dn = (Get-ItemProperty $_.PSPath -Name DisplayName -ErrorAction SilentlyContinue).DisplayName
    if ($dn -match $pattern) { Write-Output "REG: $dn" }
  }
}
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" -ErrorAction SilentlyContinue | ForEach-Object {
  $pip = (Get-ItemProperty $_.PSPath -Name ProfileImagePath -ErrorAction SilentlyContinue).ProfileImagePath
  if ($pip -match $pattern) { Write-Output "REG-PROFILE: $pip" }
}

Write-Output "`n=== [4/8] AppX 包 ==="
Get-AppxPackage -AllUsers -ErrorAction SilentlyContinue | Where-Object Name -match $pattern |
  ForEach-Object { Write-Output "APPX: $($_.Name)" }

Write-Output "`n=== [5/8] 运行进程 ==="
Get-Process -ErrorAction SilentlyContinue | Where-Object ProcessName -match $pattern |
  ForEach-Object { Write-Output "PROC: $($_.ProcessName) PID=$($_.Id)" }

Write-Output "`n=== [6/8] 服务 ==="
Get-Service -ErrorAction SilentlyContinue | Where-Object { $_.Name -match $pattern -or $_.DisplayName -match $pattern } |
  ForEach-Object { Write-Output "SVC: $($_.Name)" }

Write-Output "`n=== [7/8] 计划任务 ==="
Get-ScheduledTask -ErrorAction SilentlyContinue | Where-Object TaskName -match $pattern |
  ForEach-Object { Write-Output "TASK: $($_.TaskName)" }

Write-Output "`n=== [8/8] 防火墙规则 ==="
netsh advfirewall firewall show rule name=all 2>$null | Select-String -Pattern $pattern
```

#### 9.12.2 验证结果判定

| 验证结果 | 含义 | 处理 |
|---------|------|------|
| 8 个维度全部为零 | 应用已彻底清除 | ✅ 报告"零残留" |
| 1-2 个维度有发现 | 少量残留 | 分析残留性质——是应用数据还是被其他应用复用的同名目录（见 9.10.5），决定处理方式 |
| 3+ 个维度有发现 | 清理不充分 | 回退到 9.10.4 清单逐项补清，然后重新验证 |
| 管理员维度假定为干净 | 因权限不足未扫描 HKLM/Prefetch/防火墙等 | 报告中标注"以下维度未验证（需管理员）：..." |

#### 9.12.3 验证报告格式

最终报告必须按维度逐条列出结果：
- **文件系统**：列出所有曾扫描的基础路径，标注"零残留"或列出发现的路径
- **Prefetch**：列出匹配的文件名（如已清理，标注"已清除 N 个"）
- **注册表**：标注 HKCU/HKLM 各 hive 的状态
- **AppX 包**：标注是否已卸载
- **运行进程**：确认无相关进程
- **服务 / 计划任务 / 防火墙**：逐一标注"零残留"或"已清除 N 条"
- **管理员权限项**：标注哪些维度需要管理员权限，以及当前是否已验证

---

### 9.12 桌面应用创建的沙箱用户账户

某些桌面应用（如 Codex、沙箱类应用）可能在系统中创建专用的低权限用户账户用于隔离运行。

- **识别方式**：`C:\Users\` 下出现非当前用户的用户目录（如 `C:\Users\CodexSandboxOffline`），且该目录内容极少或为空。可通过 `Get-WmiObject Win32_UserAccount -Filter "Name='UserName'"` 确认账户存在且状态
- **完整清理涉及三个层面**：
  1. **用户账户**：`net user <UserName> /delete`（需要管理员权限）
  2. **用户目录**：`Remove-Item "C:\Users\<UserName>" -Recurse -Force`（需要管理员权限，受 NTFS 保护）
  3. **注册表 ProfileList 条目**（需要管理员权限）：
     ```powershell
     # 查找并删除 ProfileList 中的残留条目
     Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" -ErrorAction SilentlyContinue | ForEach-Object {
       $pip = (Get-ItemProperty $_.PSPath -Name ProfileImagePath -ErrorAction SilentlyContinue).ProfileImagePath
       if ($pip -eq "C:\Users\<UserName>") { Remove-Item $_.PSPath -Recurse -Force }
     }
     ```
  4. **注意**：AppX 卸载（`Remove-AppxPackage`）可能自动删除用户账户但不清理目录和 ProfileList 注册表条目——这三个是独立的清理步骤，必须逐一验证
- **禁止行为**：删除系统内置账户（如 `Public`、`Default`、`Default User`、`All Users`）或当前活跃用户的目录

---

## 10. 交付物规范

每次执行清理后，必须输出以下交付物：

### 10.1 清理前报告
- C 盘概览：总容量 / 已用 / 可用
- 扫描发现的大目录分布（按大小排序的 Top N）
- 清理候选清单（路径、大小、归属类别、安全评级、内容概要、删除影响、判断依据）

### 10.2 自审报告
- 自审查了哪些项
- 发现了什么问题
- 移除了哪些候选及原因
- 最终保留了多少候选

### 10.3 执行报告
- 每个目录的操作结果（成功 / 跳过 / 失败及原因）
- 总回收空间（对比基线）
- 清理后磁盘状态

### 10.4 保护确认
- 列出所有被扫描检查但未触碰的目录
- 列出标记为"需用户决策"的模糊目录及不能自动判定的原因
- 列出因保护规则而跳过的目录

### 10.5 残留确认
- 确认无临时文件残留
- 确认无脚本文件残留
- 确认无配置文件修改

---

## 11. 禁止事项清单

以下是**绝对禁止**的操作：

| # | 禁止事项 | 原因 |
|---|---------|------|
| 1 | 删除任何 `.db`、`.sqlite` 文件 | 可能是用户数据 |
| 2 | 删除 `$env:APPDATA` 下的非缓存目录 | 漫游数据通常是重要配置 |
| 3 | 删除 `$env:ProgramFiles` 下的任何内容 | 应用安装目录，不属清理范围 |
| 4 | 删除 `$env:ProgramData` 下的任何内容 | 应用全局数据，不属清理范围 |
| 5 | 使用 `Remove-Item -Recurse` 配合通配符 | 可能误伤匹配的其他目录 |
| 6 | 在不确定时"赌一把"删除 | 违反核心原则 |
| 7 | 建议用户"以管理员身份运行"来绕过权限 | 权限保护是有意设计的 |
| 8 | 修改注册表或系统设置 | 不属本技能范围 |
| 9 | 写在 `.ps1` 或 `.bat` 文件再执行 | 违反零残留机制 |
| 10 | 删除任何包含用户上传/创作内容的目录 | 不可恢复的数据 |
