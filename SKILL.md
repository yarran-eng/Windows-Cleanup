---
name: windows-cleanup
description: 'Safely clean up Windows C drive space and completely uninstall applications. Adopts a "triple confirmation + zero residue + burden of proof reversed" mechanism. Two main modes: (1) C drive space cleanup — safely releases disk space according to 9 categories of cache/temporary file rules; (2) Complete application uninstall — removes all traces across 8 dimensions (file system, registry, AppX, Prefetch, firewall, services, scheduled tasks, sandbox accounts). This skill only defines rules and judgment criteria, does not pre-list specific paths, and relies on the agent to autonomously scan and discover.'
---

# Windows Safe Cleanup: C Drive Space Release and Complete Application Uninstall

**Two main working modes:**

| Mode | Trigger | Goal |
|------|---------|------|
| **C drive space cleanup** | User says "clean C drive / free up space / disk full" | Safely release disk space according to the 9-category cache rules in Section 3 |
| **Complete application uninstall** | User says "completely uninstall / remove all traces of an application" | Zero residue across 8 dimensions (files, registry, AppX, Prefetch, firewall, services, tasks, sandbox) |

**Shared core principle: Better to not delete than to delete wrongly. Better to miss ten cache directories than to mistakenly delete one user file.**

**Burden of proof: By default, all directories are protected. The agent must prove through content verification that a directory can be safely deleted, rather than proving it is dangerous to protect it. When in doubt, never delete.**

**Design philosophy: This skill only defines rules, constraints, and judgment criteria, and does not pre-list specific directory paths. Every computer has different applications, versions, and paths — during execution, the agent autonomously scans the disk structure, discovers directories that occupy large space, applies the rules for classification and judgment, and finally executes after user confirmation.**

---

## 1. Zero Residue Mechanism (Highest Priority)

**This skill must not leave any new files or artifacts on the user's disk during the entire execution process, nor must it modify the system registry or system settings.**

### 1.1 Prohibited File Types
- ❌ `.ps1` script files
- ❌ `.bat` / `.cmd` batch files
- ❌ `.txt` / `.log` temporary logs
- ❌ `.csv` / `.json` data exports
- ❌ Any other temporary files

### 1.2 Compliant Execution Methods
- **All PowerShell commands must be executed directly through the terminal**, and must not be written to script files and then invoked
- If complex logic needs to be executed, use PowerShell inline commands (`powershell -Command "..."`), wrapped in single quotes to prevent escaping
- All intermediate data (directory sizes, scan results, classification judgment processes, etc.) are only recorded in the conversation context, not written to disk
- If a temporary file must be created due to tool limitations, **it must be created and used then immediately deleted in the same command chain** (`; Remove-Item ...`)

### 1.3 Residue Check (After Execution)
After cleanup is complete, execute the following commands to verify no residue:
```powershell
Get-ChildItem "$env:TEMP\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:USERPROFILE\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:LOCALAPPDATA\cleanup_*" -ErrorAction SilentlyContinue
```
If residual files are found, delete them immediately.

---

## 2. Scanning and Discovery Rules

**The agent does not carry any pre-known list of "which directories should be cleaned." Scanning is a pure information-gathering process — discovering the actual structure on disk, rather than validating a preset cleanup list.**

### 2.1 Scanning Strategy

| Phase | Operation | Purpose |
|------|------|------|
| **Phase A: Get disk overview** | Use `Get-PSDrive` to get the C drive total capacity and used space | Establish a baseline, determine whether cleanup is necessary |
| **Phase B: Top-level directory sorting** | Use `Get-ChildItem` to sort top-level directories by size | Discover the areas occupying the most space, determine the direction to drill down |
| **Phase C: Layer-by-layer drill-down** | For large directories, expand subdirectories layer by layer and sort by size | Locate the specific space-consuming source |
| **Phase D: Content sampling** | For candidate large directories, sample to view file types and names | Determine the directory's purpose, provide basis for subsequent classification |

### 2.2 Scanning Scope

**Disk dimensions (applicable to both regular cleanup and uninstall):**

The agent should focus on non-system directories in the following areas (these are conventional locations for user data and application data):
- Application data directories under the user profile (located via environment variables `$env:LOCALAPPDATA`, `$env:APPDATA`)
- Direct subdirectories of the user home directory (located via `$env:USERPROFILE`), including hidden configuration directories starting with `.` (e.g., `~\.appname\`)
- User documents directory (`$env:USERPROFILE\Documents`), where some applications create settings/workspace directories
- Application installation directories on non-system drives (e.g., `D:\02_Engineering_App\`, etc.), which may contain residual files from uninstalled applications
- System temporary directory (located via `$env:TEMP` and `$env:SystemRoot\Temp`)

**Areas that should not be scanned or touched** (unless there is an explicit system-level safe exception and content has been verified):
- System root directory (`$env:SystemRoot`, i.e., the usual `C:\Windows`) — except for the temporary directory and Prefetch directory (see exceptions below)
- Program installation directories (`$env:ProgramFiles`, `${env:ProgramFiles(x86)}`)
- Program data directory (`$env:ProgramData`)

**Non-disk dimensions (only scan when the user explicitly requests "completely uninstall/remove all traces of an application"):**

When the user requests complete uninstall of an application, in addition to the disk scanning above, you must also search in the following dimensions:

| Dimension | Scan target | Search command |
|------|---------|---------|
| AppX packages | Check if the application was installed via AppX | `Get-AppxPackage -Name "*AppName*"`; with admin privileges also `Get-AppxPackage -AllUsers` |
| AppX provisioned packages | System-provisioned AppX packages (requires admin) | `Get-AppxProvisionedPackage -Online \| Where-Object DisplayName -match "AppName"` |
| Traditional install registry | Check HKCU/HKLM Uninstall registry keys | Scan `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`, `HKLM:\SOFTWARE\WOW6432Node\...`, `HKCU:\SOFTWARE\...` |
| Firewall rules | Inbound/outbound rules the application may have added | `netsh advfirewall firewall show rule name=all dir=in \| Select-String "AppName"` (check both in/out) |
| Prefetch files | Windows preload traces | `Get-ChildItem "$env:SystemRoot\Prefetch" -Force \| Where-Object Name -match "appname"` |
| Scheduled tasks | Background tasks registered by the application | `Get-ScheduledTask \| Where-Object TaskName -match "appname"` |
| Windows services | System services registered by the application | `Get-Service \| Where-Object Name -match "appname"` |
| Environment variables | PATH or custom variables the application may have added | `Get-ChildItem Env: \| Where-Object Value -match "appname"` |
| Start menu shortcuts | Shortcuts for current user and all users | Check `$env:APPDATA\Microsoft\Windows\Start Menu` and `$env:ProgramData\Microsoft\Windows\Start Menu` |
| Desktop shortcuts | Current user and public desktop | Check `$env:USERPROFILE\Desktop` and `C:\Users\Public\Desktop` |
| Sandbox user accounts | Isolated user accounts created by the application (see Section 9.12) | `Get-ChildItem "C:\Users"` to find non-standard usernames; `Get-WmiObject Win32_UserAccount` to verify |
| Registry ProfileList | Profile registration of sandbox users (requires admin) | `Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList"` matching ProfileImagePath |

**Note:** Non-disk dimension scanning may involve multiple registry hives and system-level configurations. If the user currently does not have admin privileges, the report should note which dimensions could not be scanned due to insufficient permissions.

### 2.3 Scanning Tool Usage Standards

- Use `Get-ChildItem -Directory` to get directory listings, combined with `Measure-Object` to calculate sizes
- Use `-ErrorAction SilentlyContinue` to handle directories with insufficient permissions
- Multiple independent areas can be scanned in parallel (using Explore subagents) to accelerate information gathering
- Scan results must be reported in the conversation, including: path, size, directory content sampling (top 5-10 largest/most common file types)

### 2.4 Symbolic Link and Reparse Point Handling

- When encountering directory reparse points (Junction / SymbolicLink / MountPoint), **only record their existence, do not recurse into them, do not count their size**
- Can use PowerShell's `Attributes -match 'ReparsePoint'` for detection
- This is to prevent: entering system directories through links, infinite loops, duplicate size counting

---

## 3. Classification Judgment Criteria

**This section defines the judgment criteria for "what kind of directory can be safely deleted." The agent must match each discovered candidate directory against the following categories. Only directories that clearly match a certain category are allowed to enter the cleanup candidate list. Directories that cannot match any category are skipped by default.**

**General prerequisite: Before applying any classification judgment below, the agent must first complete content sampling (Section 3.10). Directory name alone is not sufficient to make a classification judgment.**

### 3.1 Category 1: Temporary Files

**Safety rating: High (can be safely deleted)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Directory name contains temporary file keywords such as `Temp`, `tmp`, `.tmp`, `Temporary`, etc., or is located under the system temporary directory (temporary directory path obtained via environment variables)
2. **Content verification**: The directory mainly contains temporary nature files —
   - Extensions of `.tmp`, `.temp`, `.log` (old logs), `.dmp` (old dumps)
   - File modification times are generally old (most files older than 7 days from now)
   - Does not contain `.db`, `.sqlite`, `.json`, `.xml` and other configuration/database format files
3. **Process safety**: The directory path does not contain the name of any currently running process (cross-verify with process list obtained via `Get-Process`)

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory is occupied by a currently running process
- Directory contains configuration files (`.ini`, `.conf`, `.config`, `.yaml`, `.json` config files)
- Directory modification time is within the last 1 hour (may be a temporary directory in active use)
- Directory is under `$env:ProgramFiles` or `$env:ProgramData`

### 3.2 Category 2: Package Manager Download Cache

**Safety rating: High (can be safely deleted; after deletion, the package manager can re-download when next needed)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Structural characteristics**: Directory is located in application data area (under `$env:LOCALAPPDATA` or `$env:USERPROFILE`), path or name contains name fragments of package management tools
2. **Content verification**: The directory mainly contains distributable package files —
   - Primarily `.tgz`, `.tar.gz`, `.whl`, `.nupkg`, `.vsix` and other package formats
   - Or contains `cache`, `tmp` subdirectories with the above package files inside
   - **Key distinction**: Confirm these are download caches rather than installed packages (installed packages usually have directory structures like `node_modules`, `packages`, rather than standalone package files)

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory contains `package.json`, `node_modules`, `bin/` and other structures indicating installed packages
- Directory is also the package manager's global installation directory (where installed CLI tools reside)
- Path contains `global`, `global-packages` and other keywords indicating global installation

### 3.3 Category 3: Application Updater Residuals

**Safety rating: Medium-high (can be safely deleted, but after deletion you may need to manually check for updates)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Directory name contains update-related keywords such as `updater`, `update`, `installer`, `patch`, etc., and is located under application data directories
2. **Content verification**: The directory mainly contains installer files —
   - Primarily `.exe`, `.msi`, `.msp` installers
   - Or contains `pending`, `downloads` subdirectories storing downloaded update packages
   - **Key distinction**: Confirm this is the updater cache/download directory, not the application itself. The application's own executables are usually larger and contain runtime libraries such as `.dll`

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory contains runtime library files like `.dll`, `.so` (may be the application's own updater directory)
- Executable files in the directory are currently running
- Directory size is abnormally large (>5GB), may contain full installation packages rather than just incremental updates

### 3.4 Category 4: Crash Dump Files

**Safety rating: High (can be safely deleted, only if the user does not need to debug historical crashes)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Directory name contains keywords such as `Crash`, `Dump`, `CrashDumps`, `reports`
2. **Content verification**: The directory mainly contains —
   - `.dmp`, `.mdmp`, `.hdmp` dump files
   - `.log` crash log files
   - Does not contain executable files or configuration files

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory also contains source code or debug symbol files (`.pdb`)
- Crash files are from within the last 24 hours (may be actively debugging an issue)

### 3.5 Category 5: Runtime and Environment Cache

**Safety rating: Medium-high (can be safely deleted; the application/tool will automatically rebuild on next run)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Directory path contains names of known auto-rebuildable cache frameworks/runtimes, and the directory name is clearly a cache directory (e.g., `cache`, `Cache`, `cached` subdirectory)
2. **Content verification**: The directory contains binary cache files, compilation artifacts, or runtime auto-generated data —
   - Browser rendering engine binary library files (usually hundreds of MB of `.dll` / `.so` files, located under `*cache*` directories)
   - JIT compilation cache (`.jsc`, compiled bytecode)
   - Font cache (`.dat` cache files)
   - **Key distinction**: Confirm these files can be automatically re-downloaded or regenerated by the corresponding tool, rather than requiring manual user installation

**Exclusion conditions (if any is met, exclude, do not delete):**
- The cached framework is not in the current system PATH (may have been uninstalled, but the user may have retained usage data)
- The cache directory contains user-customized configuration files or scripts
- The cache directory is located in the application installation directory (under `$env:ProgramFiles`) rather than the user data directory

### 3.5.1 Category 5.1: Browser AI/ML Model Cache

**Safety rating: Medium-high (can be safely deleted; the browser will automatically re-download model files when needed)**

**Identification method:** Located under the browser's User Data directory, with directory names matching known AI/ML model download patterns.

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Located in a direct subdirectory of the browser's `User Data` directory, with the directory name matching one of the following patterns —
   - `OptGuideOnDeviceModel` — Chrome on-device optimization guide model
   - `screen_ai` — Chrome screen OCR/AI model
   - `optimization_guide_model_store` — Chrome optimization guide model store
   - Or an AI model download directory whose name contains the `model` / `Model` keyword
2. **Content verification**: The directory contains AI model files automatically downloaded by the browser —
   - Primarily `.bin` (model weights), `.pb` (TensorFlow model), `.tflite` (TensorFlow Lite), `.binarypb` and other ML model formats
   - Usually contains a `manifest.json` descriptor file
   - Individual files can be extremely large (`weights.bin` may exceed 4GB)
3. **Process safety**: Not in an active write state by the browser process (can attempt deletion; if file is locked, skip)

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory also contains user-trained models or custom model configurations
- Directory contains `.py`, `.ipynb` and other code files (may be a user ML project directory)

**Note:** Although these directories are located under browser vendor data directories, their content is clearly ML model files that the browser can automatically re-download, and they are not treated as user configuration data. There is a corresponding exception clause for this category of directory cleanup in Rule 4.3.

### 3.6 Category 6: GPU Shader/Render Cache

**Safety rating: High (can be safely deleted; auto-rebuilds when running games or graphics applications, may be slightly slower on first load)**

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. **Path characteristics**: Directory name contains shader/render cache keywords, such as `ShaderCache`, `GPUCache`, `DawnCache`, `VkCache`, `ProgramBinaryCache`, `D3DSCache`, etc.
2. **Content verification**: Directory contains binary cache files (`.bin`, `.dat`, `.cache`)
3. **Size reasonableness**: Directory size is usually < 5GB. If exceeding 5GB, additional sampling is needed to confirm no user data is mixed in

**Exclusion conditions (if any is met, exclude, do not delete):**
- Directory contains `.png`, `.jpg` and other image files (may be a texture/screenshot directory, not pure cache)
- Directory contains `.blob` files mixed with user data characteristics

### 3.7 Category 7: Browser Cache

**Safety rating: Medium (cache can be safely deleted, but cache subdirectories must be precisely specified, user data must not be touched)**

**Identification method:** The agent should scan for the following features in application data directories to discover browser data directories:
- Directory structures with `User Data` or `userData` or `Profiles` subdirectories
- Directories with `.exe` browser executables and independent data directories

**Internal structure of browser data directories — cleanable vs non-cleanable judgment rules:**

Within a browser's user data directory (e.g., a Profile directory), **only subdirectories with the following matching names can be considered "cleanable cache"**:

| Cleanable cache directory name | Content nature |
|-------------------|---------|
| `Cache` | Page resource cache, auto-rebuilds on browser restart |
| `Code Cache` | JavaScript/WASM compilation cache |
| `GPUCache` | GPU shader cache |
| `DawnWebGPUCache` | WebGPU shader compilation cache (Chromium-based browsers) |
| `DawnGraphiteCache` | Graphite render cache (Chromium-based browsers) |
| `GrShaderCache` | Global GPU shader compilation cache (Chromium-based browsers) |
| `ShaderCache` | GPU shader cache (generic variant name) |
| `component_crx_cache` | Browser extension component download cache |
| `cache2` | Page cache directory name used by some browsers (e.g., Firefox) |
| `CacheStorage` under `Service Worker\` | Service Worker offline cache |
| `blob_storage` | Blob temporary storage |
| `Session Storage` | Session temporary storage |

**Red line rules (directories with matching names below must never be deleted, no exceptions):**

The following directories contain user personal data, login state, or extension data, **name match means protect**:

| Non-deletable directories/files | Reason |
|---------------------|------|
| `History` | Browsing history |
| `Login Data` | Saved passwords |
| `Cookies` | Login state and site preferences |
| `Bookmarks` | User bookmarks |
| `Preferences` | Browser settings |
| `Web Data` | Autofill, site settings |
| `Sync Data` | Sync configuration |
| `Extensions` | Installed extensions |
| `Local Extension Settings` | Extension local settings and data |
| `IndexedDB` | Website databases (may contain offline data) |
| `databases` | Website databases |
| `storage` | Website persistent storage (may contain user data) |
| `File System` | Website file system API storage |
| `Local Storage` | Website localStorage data |
| `Session Storage` | (Only protected when located at root level rather than temporary session storage in browsing context) |

**Execution requirements:**
- Must precisely specify the complete subdirectory path for deletion, must not use broad wildcards
- Before deletion, must first use `Get-ChildItem` to confirm the directory contains only cache files
- Only one cache subdirectory of one Profile can be deleted at a time, must not batch delete multiple directories

### 3.8 Category 8: General Application Cache

**Safety rating: Low-medium (must verify one by one, confirm it is pure cache before deleting)**

**Identification method:** In application data directories (`$env:LOCALAPPDATA` and `$env:APPDATA`), look for subdirectories named `Cache`, `cache`, `caches` within each application's subdirectory.

**Judgment criteria (all of the following conditions must be met simultaneously):**

1. Directory name is exactly `Cache`, `cache`, or `caches`
2. Directory content is 100% cache files (binary cache, thumbnails, precomputed data), does not contain `.db`, `.sqlite`, `.json`, `.xml`
3. Parent directory can be clearly identified as a certain application (not a system component, not a communication/social application, not a cloud sync application)
4. Does not belong to any category described in the "high-risk application category identification rules" table below

**High-risk application category identification rules (data directories of these applications are protected without exception, only browser-type cache subdirectories clearly named under them are allowed to be cleaned):**

| Application category | Identification features | Cleanable exceptions |
|---------|---------|-----------|
| Communication/social applications | Data directory located under `$env:APPDATA` (Roaming); directory contains `Databases`, `Msg`, `Chat`, `messages` and other subdirectories or files | Only embedded browser cache subdirectories under them (judged per Section 3.7 rules) |
| Cloud sync/cloud drive applications | Directory contains a large number of folder structures synced with the cloud; content is `.docx`, `.pdf`, `.jpg` and other user files | None — protect all, do not clean |
| Virtual machines/emulators | Directory size is very large (>10GB); contains `.vdi`, `.vmdk`, `.vhdx`, `.qcow2` and other virtual disk files | None — protect all, do not clean |
| Database applications | Directory contains database files (`.db`, `.mdf`, `.ndf`, data directory) | None — protect all, do not clean |

### 3.9 Category 9: IDE and Development Tool Cache

**Safety rating: Medium (can clean compilation cache and runtime cache, but must protect user settings, extensions, workspace data)**

**Identification method:** The agent should scan for directories with the following features in application data directories to identify IDEs:
- Directory contains both user configuration subdirectories (`User/`, `user-data/`, `config/`) and cache subdirectories (`Cache/`, `CachedData/`, `CachedExtensions/`)
- Directory contains `extensions/` or `extensions.json` and other extension management structures
- Directory contains `workspace-storage/` or `workspaces/` workspace data

**Cleanable cache directory name patterns (name match is sufficient to classify as cache):**

| Pattern | Description |
|------|------|
| Name contains `Cache` (e.g., `Cache/`, `Code Cache/`, `GPUCache/`, `DawnGraphiteCache/`, `DawnWebGPUCache/`) | Various rendering and compilation caches |
| Name starts with `Cached` (e.g., `CachedExtensions/`, `CachedData/`, `CachedProfilesData/`, `CachedConfigurations/`) | Extension and configuration caches |
| `Crashpad/`, `logs/` | Crash reports and logs |
| `blob_storage/`, `Network/`, `Partitions/`, `Shared Dictionary/`, `WebStorage/` | Network and storage temporary data |

**Red line rules (directories with matching names below must never be deleted):**

| Non-deletable directories | Nature |
|---------------|------|
| `User/` or `user-data/` | IDE settings, keyboard shortcuts, code snippets, workspace state |
| `ModularData/` | Agent database, sandbox configuration, VM snapshots |
| `Backups/` | User backup files |
| `Workspaces/` | Workspace data |
| `Local Storage/`, `IndexedDB/`, `Session Storage/` | Extension storage |
| `globalStorage/` | Extension global data |
| `extensions/` | Installed extensions (requires re-download and re-install after deletion, should not be a regular cleanup item) |
| `clp/` | Feature data |

### 3.10 Content Sampling Verification Standards

**Before classifying any directory into any of the above categories, the agent must first perform content sampling. The following are sampling standards:**

1. **Sampling method**: Use `Get-ChildItem -Recurse -Depth 2 | Group-Object Extension | Sort-Object Count -Descending | Select-Object -First 10` to get extension distribution
2. **Sampling judgment**:
   - If more than 90% of files in the directory (by count) are cache/temporary file types (`.tmp`, `.cache`, `.dmp`, `.bin`, `.dat`, `.log`) → can be classified into the corresponding cache category
   - If any proportion of user data files appears in the directory (`.db`, `.sqlite`, `.json` (>100KB), `.xml` (>100KB), `.docx`, `.xlsx`, `.pdf`, `.jpg`, `.png`, `.psd`, etc.) → stop classification, mark as "requires user decision"
   - If more than 30% of files in the directory are of unidentifiable types → stop classification, mark as "requires user decision"
3. **Size anomaly check**: The size of cache directories should usually be within a reasonable range. If a cache directory is abnormally large (e.g., a browser cache exceeds 2GB, or an IDE cache exceeds 10GB), additional in-depth sampling is needed to confirm no user data is mixed in

---

## 4. Protection Rules

**This section defines protection rules of "once hit, skip, take no action." These rules are the safety net — even if misjudgments occur during scanning and classification, the protection rules must ultimately intercept unsafe operations.**

### 4.1 Burden of Proof Rule

**Default stance: All directories are protected.** Only directories meeting all of the following conditions can enter the cleanup candidate list:
1. Clearly matches a "cleanable" category in Section 3
2. Content sampling verification has been performed, and the result supports that classification
3. Does not hit any "untouchable" protection rule in Section 4

**If any condition is not met → directory excluded, marked as "requires individual user decision" or "protected."**

### 4.2 System Directory Protection

**Identification method:** Match via path prefix and environment variables.

| Protection scope | Identification method | Known safe exceptions |
|---------|---------|-------------|
| All subdirectories under the operating system root directory | Path is a subdirectory of `$env:SystemRoot` | Subdirectories under `$env:SystemRoot\Temp` (clean content only); `$env:SystemRoot\SoftwareDistribution\Download` (clean content only) |
| System legacy backups | Path prefix is `$env:SystemDrive\Windows.old` | None |
| 64-bit program installation directory | Path prefix is `$env:ProgramFiles` | None |
| 32-bit program installation directory | Path prefix is `${env:ProgramFiles(x86)}` | None |
| Application global data directory | Path prefix is `$env:ProgramData` | None |

**Execution rule:** When encountering a directory whose path prefix matches the above protection scope, skip all except the listed precise exception paths. Exception paths must also be verified before execution to confirm content is indeed cache/temporary files.

### 4.3 Vendor Application Data Protection

**These rules are based on path structure rather than specific path values — regardless of the actual path on the computer, protection is triggered as long as the path structure matches.**

**Identification method:** The following rules match by checking directory name fragments and location in the path, rather than matching complete paths.

| Protection rule | Trigger condition | Known safe exceptions |
|---------|---------|-------------|
| Microsoft local application data | Path contains the `\AppData\Local\Microsoft\` fragment | `\Microsoft\Windows\INetCache` (network cache); Microsoft Edge browser `User Data` directory: (a) cache subdirectories whose names match the "cleanable" list in Section 3.7, (b) subdirectories whose names match the GPU shader cache keywords in Section 3.6, (c) subdirectories whose names match the AI/ML model cache patterns in Section 3.5.1 |
| Microsoft roaming application data | Path contains the `\AppData\Roaming\Microsoft\` fragment | None |
| Google local application data | Path contains the `\AppData\Local\Google\` fragment | Google Chrome browser `User Data` directory: (a) cache subdirectories whose names match the "cleanable" list in Section 3.7, (b) subdirectories whose names match the GPU shader cache keywords in Section 3.6, (c) subdirectories whose names match the AI/ML model cache patterns in Section 3.5.1 |
| Windows Store applications | Path contains the `\AppData\Local\Packages\` fragment | None |

**Execution rule:** Vendor data directories allow the following three types of exceptions (and must be precise down to the subdirectory); all other subdirectories are protected, no action taken:
1. **Browser cache exception**: Cache subdirectories whose names match the "cleanable" list in Section 3.7
2. **GPU shader cache exception**: Subdirectories whose names contain the GPU cache keywords listed in Section 3.6 (e.g., `GPUCache`, `ShaderCache`, `DawnCache`, `VkCache`, etc.)
3. **AI/ML model cache exception**: Subdirectories whose names match the AI model cache patterns listed in Section 3.5.1

### 4.4 User Data and Login State Protection

The following categories of directories are **protected without exception**:

| Protection category | Identification features |
|---------|---------|
| Roaming data of communication/social applications | Located under `$env:APPDATA`, directory contains chat records, contacts, database files |
| Installed directories of global package managers | Located under `$env:APPDATA`, contains globally installed CLI tool binaries (`npm` global, `yarn` global, `pip` user install, etc.) |
| IDE user configuration directories | `User/`, `user-data/`, `config/` subdirectories under any IDE directory |
| Emulator/virtual machine data | Directory contains virtual disk files or emulator system images |
| Cloud sync client data | Directory contains file structures and configuration databases synced with the cloud |
| Database data file directories | Directory contains database files such as `.db`, `.mdf`, `.ndf` |

### 4.5 Undeterminable Directories — Fallback Rule

**For directories that cannot be clearly judged through the Section 3 classification rules and Section 4 protection rules (i.e., "gray areas"), the default strategy is:**

1. **Do not delete**
2. **Record path, size, and the reason for uncertainty**
3. **List separately in the cleanup report, marked as "requires user decision"**
4. **Explain to the user why automatic determination was not possible, list directory content sampling results, let the user decide**

**Prohibited behavior:** "Taking a gamble" on deletion when uncertain. Any uncertainty should lead to "do not delete."

---

## 5. Recycle Bin Cleanup

### 5.1 Locate the Recycle Bin

The recycle bin is located in a hidden system directory under the system drive root:

```powershell
# Check recycle bin size (does not depend on specific drive letter)
$recyclePath = "$env:SystemDrive`\$Recycle.Bin"
(Get-ChildItem $recyclePath -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1MB
```

### 5.2 Empty the Recycle Bin

```powershell
Clear-RecycleBin -Force
```

**Note:** Emptying the recycle bin is irreversible, but the files in the recycle bin have already been "deleted" by the user. The recycle bin size still needs to be listed separately in the confirmation list, and emptying the recycle bin should be an independent confirmation item.

---

## 6. Triple Confirmation Process

**All cleanup operations must go through "scan & classify → confirm → self-review → re-confirm → execute" triple confirmation. No step can be skipped.**

### 6.1 First Round Confirmation: Scan Results and Candidate List

**After completing scanning and classification, generate a structured cleanup candidate list.**

**List format requirements:** For each candidate directory, the following complete information must be provided:

| Field | Description |
|------|------|
| Directory path | Complete absolute path |
| Size | Current occupied space (MB/GB) |
| Category | Which category in Section 3 it matches |
| Safety rating | High / Medium-high / Medium / Low (caution needed) |
| Content summary | Sampling results — main file types and order of magnitude |
| Deletion impact | What happens after deletion ("auto-rebuildable" / "requires re-download" / "slightly slower on first run", etc.) |
| Judgment basis | List the specific basis for making this classification judgment (which identification features were matched) |

**Use the `AskUserQuestion` tool to present the list to the user for confirmation. The user can choose to:**
- Agree to all
- Select by category
- Exclude specific items
- Request further analysis of specific items

### 6.2 Second Round: AI Self-Review

**After the user's first round of confirmation and before executing deletion, the agent must perform the following systematic self-review. Cannot be skipped.**

**Self-review checklist (check item by item, each must have a clear conclusion of "pass / exclude / uncertain"):**

| # | Self-review item | Check content | Judgment criteria |
|---|--------|----------|----------|
| 1 | Protection rule cross-check | Are there any paths in the candidate list that hit any protection rule in Section 4? | Hit → exclude that item |
| 2 | Active process association check | Is the candidate directory path or parent directory name associated with currently running process names (`Get-Process`)? | Associated → exclude that item by default. **Exception**: browser cache subdirectories (Section 3.7) and browser AI model cache (Section 3.5.1) can remain in the list — deleting these caches while the browser is running is a routine operation, and file locks naturally protect active files. Other types (IDE, application itself, etc.) must be excluded if associated |
| 3 | Content verification review | Has content sampling actually been performed for each candidate directory? (Rather than judging by directory name alone) | Not sampled → exclude that item or fall back to supplement sampling |
| 4 | User data misjudgment check | Are there any misclassified items in the candidate directories? (e.g., mistaking a `storage/` directory storing user data for cache) | Misjudgment found → exclude that item |
| 5 | Cascade impact assessment | Will deleting this directory cause other applications/services that depend on it to malfunction? | Risk → exclude or downgrade |
| 6 | Size anomaly check | Is the candidate directory size far beyond the normal range of similar directories? (e.g., a "cache" directory reaching tens of GB) | Anomaly → in-depth sampling or exclude |
| 7 | Wildcard/batch operation check | Does any delete command use wildcards or batch operations that may affect adjacent directories? | Broad wildcards present → change to individual precise paths |
| 8 | Minimization check | For each candidate directory, are only the subdirectories that can be safely deleted being deleted, rather than the entire parent directory? | Scope too broad → narrow to precise subdirectories |
| 9 | Recoverability check | Can it be recovered through auto-rebuild/re-download after deletion? If not recoverable → exclude | Not recoverable → exclude |
| 10 | Time window check | Are there any cache directories modified recently (indicating active use)? | Modified within the last 1 hour → exclude |

**Self-review result handling:**
- For each issue found, remove the corresponding directory from the candidate list, record the reason for removal
- After self-review is complete, summarize: N items retained, M items removed (with reasons)
- Present the self-review results to the user

### 6.3 Third Round: Final Confirmation

**Present the self-review results (retained items, removed items and reasons, any uncertain items) to the user, and use `AskUserQuestion` to request confirmation again.**

Only after the user confirms can the execution phase begin.

---

## 7. Execution Standards

### 7.1 Pre-execution Preparation

1. Record disk baseline: `(Get-PSDrive C).Used`
2. Confirm all paths to be deleted exist and are accessible
3. Prepare skip list and error log

### 7.2 Execution Order

Execute by safety rating from high to low:
1. Temporary files (Category 1)
2. Crash dumps (Category 4)
3. GPU cache (Category 6)
4. Browser AI/ML model cache (Category 5.1)
5. Package manager cache (Category 2)
6. Updater residuals (Category 3)
7. Runtime cache (Category 5)
8. Browser cache (Category 7)
9. General application cache (Category 8)
10. IDE cache (Category 9)
11. Recycle bin emptying (execute last)

### 7.3 Deletion Operation Standards

**Each delete command must:**
- Use a precise complete path (must not use broad wildcards)
- A single command deletes only one target
- Use `Remove-Item -Recurse -Force -ErrorAction Stop`, with error handling
- Record the operation result for each target

**Error handling:**
- File occupied → skip, record reason, continue to next
- Insufficient permissions → skip, record reason, continue to next (do not attempt privilege escalation)
- Path does not exist → mark as "already cleaned" (⏭️), record reason (may have been cleaned by the uninstaller or other processes, this is normal), continue to next
- Path does not exist and other related directories have also disappeared → mark as "uninstaller has automatically cleaned", treat as success rather than failure

**Browser cache special requirements:**
- Must execute `Remove-Item` separately for each cache subdirectory of each Profile
- Must not use `*` wildcards to match Profile directories

**System temporary directory special handling:**
When deleting temporary directory contents, first get the list of currently running processes, skip subdirectories related to process names:
```powershell
$running = (Get-Process).ProcessName | Select-Object -Unique
Get-ChildItem $tempPath -Directory -Force | Where-Object { $running -notcontains $_.Name } | Remove-Item -Recurse -Force -ErrorAction Continue
Get-ChildItem $tempPath -File -Force | Remove-Item -Force -ErrorAction Continue
```

### 7.4 Recycle Bin Cleanup

After all file cleanup is complete, before the verification phase, execute:
```powershell
Clear-RecycleBin -Force -ErrorAction Continue
```

---

## 8. Verification and Residue Check

### 8.1 Space Recovery Verification

```powershell
$d = Get-PSDrive C
$freed = $baselineUsed - $d.Used
Write-Output "Recovered space: $([math]::Round($freed/1MB,1)) MB ($([math]::Round($freed/1GB,2)) GB)"
Write-Output "Current free: $([math]::Round($d.Free/1GB,2)) GB / $([math]::Round(($d.Used+$d.Free)/1GB,2)) GB"
```

**Note:** The system may write new files during cleanup (logs, update caches), causing the "freed" calculation to be slightly off. Use the baseline recorded before cleanup for comparison, rather than real-time queries.

### 8.2 Residual File Check

```powershell
Get-ChildItem "$env:TEMP\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:USERPROFILE\cleanup_*" -ErrorAction SilentlyContinue
Get-ChildItem "$env:LOCALAPPDATA\cleanup_*" -ErrorAction SilentlyContinue
```
If residual files are found, delete them immediately.

### 8.3 Operation Integrity Verification

Check the operation result of each target in the cleanup list item by item:
- ✅ Successfully deleted
- ⏭️ Skipped (with reason: file occupied / insufficient permissions / path does not exist)
- ❌ Failed (with error message)

---

## 9. Pitfalls and Edge Cases

### 9.1 Executing PowerShell Commands in a Bash Environment

When running PowerShell commands in a bash terminal, `$()` and `$var` will be interpreted by bash. Solutions:
- **Simple commands**: Use single quotes to wrap PowerShell arguments
- **Complex logic**: Use `powershell -Command '...'` for inline execution, **never write to .ps1 files**

**Known patterns that cause bash truncation and alternatives:**

| Problem pattern | Failure reason | Alternative |
|---------|---------|---------|
| `$($_.Name)` or `$($c.Substring(...))` | bash interprets `$()` as command substitution | Avoid embedding `$()` in PowerShell strings; use temporary variable assignment then reference |
| Backslash paths in `$_.FullName` | bash interprets `\` as escape character | Use `$_.FullName.Replace('\','/')` or wrap the entire command in single quotes |
| `foreach($c in $caches) { try { ... } catch { ... } }` | bash interprets `}` and `)` as syntax | Expand the loop into individual independent commands; or replace foreach with `ForEach-Object` |
| String contains `\n` backslash | bash combines `\` with following character | Wrap all static strings in single quotes |

**Battle-tested safe writing patterns:**

```bash
# ✅ Correct: individual independent commands, avoid loops and $() embedding
powershell -Command 'Remove-Item "path1" -Recurse -Force -ErrorAction SilentlyContinue; Remove-Item "path2" -Recurse -Force -ErrorAction SilentlyContinue; Write-Output "Done"'

# ❌ Wrong: using foreach loop and $() string interpolation inside bash -Command
powershell -Command "foreach($c in $caches) { try { Remove-Item $c -Force } catch { Write-Output \"SKIP: $($c.Name)\" } }"
```

**General rule:** If a PowerShell command contains any of the following elements, it must be split into multiple simple commands or use `ForEach-Object` + single quotes:
- `$()` subexpression
- Multi-line `try/catch` block
- String contains backslash `\` (Windows path) and also has `$var` references

### 9.2 File Occupation

Some files may be occupied by processes during deletion. This is normal — catch the error and skip, it does not affect the cleanup effect and should not cause the entire cleanup process to be interrupted.

### 9.3 Disk Space Baseline Drift

The system may write new files during cleanup (logs, caches), causing the "freed" calculation to be slightly off. Use the `$baselineUsed` recorded before cleanup for comparison.

### 9.4 Directory Name Deception

**This is the most common cause of accidental deletion.** Some directory names look like cache (e.g., a directory named `Cache`), but actually store user data. **Do not judge by directory name alone, must check the directory content composition.**

**Mandatory rules:**
- If the directory contains any of the following file types and they account for more than 5% of total file count, stop automatic classification, mark as "requires user decision":
  - `.db`, `.sqlite`, `.sqlite3` (database files)
  - `.json`, `.xml` and file size > 100KB (may be data storage rather than configuration)
  - `.dat` and file size > 10MB (may be user data files)
  - `.jpg`, `.png`, `.svg`, `.gif` and other image files
  - `.docx`, `.xlsx`, `.pptx`, `.pdf` and other document files

### 9.5 Browser Multiple Profiles

Browsers may have multiple Profile directories (`Default`, `Profile 1`, `Profile 2`, etc.). Handling rules:
- Each Profile is independently judged according to Section 3.7 rules
- Use precise paths to process one by one, **must not use wildcards to match Profiles**
- If the number of Profiles > 5, only need to confirm with the user once ("Detected N browser Profiles, do you want to perform the same cache cleanup for all Profiles?")

### 9.6 Special Handling of Communication/Social Applications

Data directories of such applications are **one of the highest risk areas**. Handling rules:
- By default, only report their occupied space, do not add any of their subdirectories to the cleanup candidate list
- If the user explicitly requests cleanup, only when the embedded browser's cache subdirectory can be precisely identified (per Section 3.7 rules), and the content has been verified file by file to be browser cache, can it be added to the candidate list
- Handling of these applications must be especially conservative — better to miss than to mistakenly delete

### 9.7 Symbolic Links, Junctions, and Mount Points

- **During scanning**: Use `Attributes -match 'ReparsePoint'` for detection, only record the link's existence and target, do not recurse into it
- **During deletion**: Only delete the link itself (if it is a cleanup target), do not recursively delete the target content the link points to
- **Absolutely prohibited**: Entering system directories through symbolic links or Junctions and performing deletion operations

### 9.8 Permission Errors

When encountering directories with insufficient permissions:
- During scanning: Use `-ErrorAction SilentlyContinue` to silently skip
- During deletion: Do not use admin privilege escalation, skip normally and record
- **Prohibited behavior**: Suggesting the user "run as administrator" to bypass permission protection

### 9.9 Hidden Files and System Files

- Use `-Force` parameter during scanning to include hidden items, ensuring complete size calculation
- During deletion, only delete confirmed cleanup candidates, do not treat them specially due to their hidden attribute
- When encountering files with "System" attribute set (`Attributes -match 'System'`): do not delete, mark and report

### 9.10 Application Uninstall and Cross-Dimension Trace Cleanup

When the user explicitly requests "completely uninstall/remove all traces of an application," the agent must execute the following complete process, rather than just doing C drive space cleanup.

#### 9.10.1 Step 1: Identify Application Installation Method

Before executing uninstall, first determine the application's installation method, which determines the uninstall strategy:

| Installation method | Identification features | Uninstall method |
|---------|---------|---------|
| AppX/UWP package | `Get-AppxPackage` can find it | `Remove-AppxPackage` (see 9.10.2) |
| Traditional installer (Inno Setup) | Registry Uninstall key contains `unins000.exe`, usually installed in a custom path (e.g., `D:\Apps\`) | First run the official uninstaller (silent mode), then manually clear residuals |
| Traditional installer (MSI) | Registry Uninstall key contains `MsiExec.exe` | `msiexec /x {ProductCode} /quiet` |
| Portable/green version | No registry Uninstall entry, no AppX package | Directly delete the application directory and user data |
| User-level install (under `%LOCALAPPDATA%`) | Installation path is under `$env:LOCALAPPDATA\Programs\` | Check for `Update.exe` or `uninstall.exe` |

#### 9.10.2 AppX/UWP Application Uninstall

```powershell
# 1. First find the package name (use exact name when no wildcard is needed)
Get-AppxPackage -Name "*AppName*" | Select-Object Name, PackageFullName

# 2. Standard uninstall
Get-AppxPackage -Name "*AppName*" | Remove-AppxPackage

# 3. With admin privileges, check AllUsers and provisioned packages
# Get-AppxPackage -AllUsers -Name "*AppName*" | Remove-AppxPackage
# Get-AppxProvisionedPackage -Online | Where-Object DisplayName -match "AppName" | Remove-AppxProvisionedPackage -Online
```

**Common residuals after AppX uninstall:**
- `$env:LOCALAPPDATA\Packages\<Publisher.AppName_*>` — AppX user data directory, may remain after uninstall, can be safely deleted
- `.appname` hidden configuration directory under `$env:USERPROFILE\`
- `$env:USERPROFILE\.cache\appname-runtimes` — runtime cache
- Workspace/settings directory created by the application under `$env:USERPROFILE\Documents`

**Note:** `Get-AppxPackage` only queries the current user by default. If permissions are insufficient to completely clean (e.g., cannot query `-AllUsers`, cannot delete protected user directories under `C:\Users\`), the report should explain which dimensions are assumed clean but have not been actually verified.

#### 9.10.3 Traditional Installer Uninstall

**Key principle: First run the official uninstaller, then manually clear residuals.** Traditional uninstallers (especially Inno Setup's `unins000.exe`) usually only delete files written during installation, **and do not clean up user data directories and runtime-generated caches.**

```powershell
# 1. Locate the uninstaller from the registry
$uninstallKey = Get-ChildItem "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" -ErrorAction SilentlyContinue | 
  Where-Object { (Get-ItemProperty $_.PSPath -Name DisplayName -ErrorAction SilentlyContinue).DisplayName -match "AppName" }
$uninstaller = (Get-ItemProperty $uninstallKey.PSPath -Name UninstallString).UninstallString

# 2. Run the uninstaller silently (Inno Setup parameters)
# Start-Process "D:\Path\unins000.exe" -ArgumentList "/VERYSILENT /SUPPRESSMSGBOXES /NORESTART" -Wait

# 3. Verify after uninstall and manually clear residuals (see 9.10.4 verification checklist)
```

**Common uninstaller silent parameters:**
| Installation tool | Silent uninstall parameters |
|---------|-------------|
| Inno Setup | `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART` |
| NSIS | `/S` |
| MSI | `/quiet /norestart` (invoked via `msiexec /x`) |
| InstallShield | `/s /f1"<response_file>"` |

#### 9.10.4 Post-Uninstall Cross-Dimension Verification and Cleanup Checklist

After the uninstaller runs, check and clean residuals dimension by dimension according to the following checklist:

```powershell
# === 1. Installation directory (uninstaller may leave empty directories or config files) ===
Remove-Item "<installation_path>" -Recurse -Force -ErrorAction SilentlyContinue

# === 2. AppData user data (most common residual, uninstaller usually does not clean) ===
# Roaming: user configuration, workspace, extension data
Remove-Item "$env:APPDATA\AppName" -Recurse -Force -ErrorAction SilentlyContinue
# Local: cache, runtime data
Remove-Item "$env:LOCALAPPDATA\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 3. Hidden configuration under user home directory ===
Remove-Item "$env:USERPROFILE\.appname" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.appname-suffix" -Recurse -Force -ErrorAction SilentlyContinue

# === 4. Documents/workspace directory ===
Remove-Item "$env:USERPROFILE\Documents\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 5. Desktop shortcuts ===
Remove-Item "$env:USERPROFILE\Desktop\AppName.lnk" -Force -ErrorAction SilentlyContinue

# === 6. Start menu ===
Remove-Item "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\AppName" -Recurse -Force -ErrorAction SilentlyContinue

# === 7. Prefetch files (requires admin privileges) ===
Get-ChildItem "$env:SystemRoot\Prefetch" -Force | Where-Object Name -match "appname\.exe|APPNAME" | Remove-Item -Force -ErrorAction SilentlyContinue

# === 8. Firewall rules (requires admin privileges) ===
netsh advfirewall firewall delete rule name="RuleName"  # Execute separately for dir=in and dir=out

# === 9. Registry Uninstall entries ===
Remove-Item "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{GUID}" -Recurse -Force -ErrorAction SilentlyContinue
# If there are HKLM entries, admin privileges are also required

# === 10. Sandbox user accounts and directories (see Section 9.12) ===
```

#### 9.10.5 Directory Reuse Warning

**Special risk in uninstall scenarios:** Some directory names (e.g., `.codex`) may be used by multiple different applications. Before deletion, you must confirm that the directory content indeed belongs to the application to be uninstalled, not another application still in use.

**Judgment method:**
1. Check whether files in the directory reference the application name to be uninstalled (`appName` in config files, path references, etc.)
2. Check whether the directory's modification time matches other currently running applications in this session
3. If uncertain, sample the directory content and report to the user for judgment
4. **Prohibited behavior:** Deleting based solely on directory name match without verifying content ownership

### 9.11 Post-Complete Uninstall Terminal Verification

**When the user requests "completely remove all traces of an application," after uninstall and manual cleanup are complete, the terminal verification scan defined in this section must be executed to confirm zero residue.**

#### 9.12.1 Verification Scan Script

The following script should be executed after all cleanup steps are complete, as final confirmation:

```powershell
# Complete uninstall terminal verification (replace <Pattern> with the matching pattern of the application name)
$pattern = "<AppNamePattern>"  # e.g., "codex|Codex|CODEX" or "trae|Trae|TRAE"

Write-Output "=== [1/8] File system scan ==="
@("$env:USERPROFILE","$env:LOCALAPPDATA","$env:APPDATA",
  "$env:ProgramFiles","${env:ProgramFiles(x86)}","$env:ProgramData",
  "C:\Users","D:\","$env:USERPROFILE\Desktop") | ForEach-Object {
  Get-ChildItem $_ -Directory -Force -ErrorAction SilentlyContinue | 
    Where-Object Name -match $pattern | ForEach-Object { Write-Output "DIR: $($_.FullName)" }
  Get-ChildItem $_ -File -Force -ErrorAction SilentlyContinue | 
    Where-Object Name -match $pattern | ForEach-Object { Write-Output "FILE: $($_.FullName)" }
}

Write-Output "`n=== [2/8] Prefetch files ==="
Get-ChildItem "$env:SystemRoot\Prefetch" -Force -ErrorAction SilentlyContinue |
  Where-Object Name -match $pattern | ForEach-Object { Write-Output "PF: $($_.Name)" }

Write-Output "`n=== [3/8] Registry ==="
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

Write-Output "`n=== [4/8] AppX packages ==="
Get-AppxPackage -AllUsers -ErrorAction SilentlyContinue | Where-Object Name -match $pattern |
  ForEach-Object { Write-Output "APPX: $($_.Name)" }

Write-Output "`n=== [5/8] Running processes ==="
Get-Process -ErrorAction SilentlyContinue | Where-Object ProcessName -match $pattern |
  ForEach-Object { Write-Output "PROC: $($_.ProcessName) PID=$($_.Id)" }

Write-Output "`n=== [6/8] Services ==="
Get-Service -ErrorAction SilentlyContinue | Where-Object { $_.Name -match $pattern -or $_.DisplayName -match $pattern } |
  ForEach-Object { Write-Output "SVC: $($_.Name)" }

Write-Output "`n=== [7/8] Scheduled tasks ==="
Get-ScheduledTask -ErrorAction SilentlyContinue | Where-Object TaskName -match $pattern |
  ForEach-Object { Write-Output "TASK: $($_.TaskName)" }

Write-Output "`n=== [8/8] Firewall rules ==="
netsh advfirewall firewall show rule name=all 2>$null | Select-String -Pattern $pattern
```

#### 9.12.2 Verification Result Judgment

| Verification result | Meaning | Handling |
|---------|------|------|
| All 8 dimensions are zero | Application completely removed | ✅ Report "zero residue" |
| 1-2 dimensions have findings | Minimal residue | Analyze the nature of the residue — whether it is application data or a same-name directory reused by another application (see 9.10.5), decide how to handle |
| 3+ dimensions have findings | Insufficient cleanup | Fall back to 9.10.4 checklist to clean item by item, then re-verify |
| Admin dimensions assumed clean | HKLM/Prefetch/firewall etc. not scanned due to insufficient permissions | Mark in report "The following dimensions not verified (require admin): ..." |

#### 9.12.3 Verification Report Format

The final report must list results dimension by dimension:
- **File system**: List all base paths scanned, mark "zero residue" or list discovered paths
- **Prefetch**: List matching file names (if already cleaned, mark "N removed")
- **Registry**: Mark the status of each HKCU/HKLM hive
- **AppX packages**: Mark whether uninstalled
- **Running processes**: Confirm no related processes
- **Services / Scheduled tasks / Firewall**: Mark "zero residue" or "N entries removed" one by one
- **Admin privilege items**: Mark which dimensions require admin privileges, and whether they have been verified currently

---

### 9.12 Sandbox User Accounts Created by Desktop Applications

Some desktop applications (e.g., Codex, sandbox-type applications) may create dedicated low-privilege user accounts in the system for isolated execution.

- **Identification method**: A non-current-user user directory appears under `C:\Users\` (e.g., `C:\Users\CodexSandboxOffline`), and the directory content is minimal or empty. Can be confirmed via `Get-WmiObject Win32_UserAccount -Filter "Name='UserName'"` to verify account existence and status
- **Complete cleanup involves three levels**:
  1. **User account**: `net user <UserName> /delete` (requires admin privileges)
  2. **User directory**: `Remove-Item "C:\Users\<UserName>" -Recurse -Force` (requires admin privileges, protected by NTFS)
  3. **Registry ProfileList entry** (requires admin privileges):
     ```powershell
     # Find and delete residual entries in ProfileList
     Get-ChildItem "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" -ErrorAction SilentlyContinue | ForEach-Object {
       $pip = (Get-ItemProperty $_.PSPath -Name ProfileImagePath -ErrorAction SilentlyContinue).ProfileImagePath
       if ($pip -eq "C:\Users\<UserName>") { Remove-Item $_.PSPath -Recurse -Force }
     }
     ```
  4. **Note**: AppX uninstall (`Remove-AppxPackage`) may automatically delete the user account but not clean the directory and ProfileList registry entry — these three are independent cleanup steps, each must be verified individually
- **Prohibited behavior**: Deleting built-in system accounts (e.g., `Public`, `Default`, `Default User`, `All Users`) or the directory of the currently active user

---

## 10. Deliverable Standards

After each cleanup execution, the following deliverables must be output:

### 10.1 Pre-cleanup Report
- C drive overview: total capacity / used / available
- Distribution of large directories discovered by scanning (Top N sorted by size)
- Cleanup candidate list (path, size, category, safety rating, content summary, deletion impact, judgment basis)

### 10.2 Self-Review Report
- Which items were self-reviewed
- What issues were found
- Which candidates were removed and why
- How many candidates were ultimately retained

### 10.3 Execution Report
- Operation result for each directory (success / skipped / failed and reason)
- Total recovered space (compared to baseline)
- Disk status after cleanup

### 10.4 Protection Confirmation
- List all directories that were scanned and checked but not touched
- List ambiguous directories marked as "requires user decision" and the reason they could not be automatically determined
- List directories skipped due to protection rules

### 10.5 Residue Confirmation
- Confirm no temporary file residue
- Confirm no script file residue
- Confirm no configuration file modifications

---

## 11. Prohibited Actions List

The following are **absolutely prohibited** operations:

| # | Prohibited action | Reason |
|---|---------|------|
| 1 | Deleting any `.db`, `.sqlite` files | May be user data |
| 2 | Deleting non-cache directories under `$env:APPDATA` | Roaming data is usually important configuration |
| 3 | Deleting anything under `$env:ProgramFiles` | Application installation directory, not within cleanup scope |
| 4 | Deleting anything under `$env:ProgramData` | Application global data, not within cleanup scope |
| 5 | Using `Remove-Item -Recurse` with wildcards | May affect other matching directories |
| 6 | "Taking a gamble" on deletion when uncertain | Violates core principle |
| 7 | Suggesting the user "run as administrator" to bypass permissions | Permission protection is intentional |
| 8 | Modifying the registry or system settings | Not within this skill's scope |
| 9 | Writing to `.ps1` or `.bat` files then executing | Violates zero residue mechanism |
| 10 | Deleting any directory containing user-uploaded/created content | Irrecoverable data |
