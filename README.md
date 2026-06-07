## 🚀 Windows C 盘安全清理与应用彻底卸载专家

### 中文描述

**功能摘要：**
该模块专为 Windows 系统设计，提供了 **“C 盘空间清理”** 与 **“应用彻底卸载”** 两大核心工作模式。模块核心围绕“宁可不删，不可错删”的红线原则展开，通过硬核的安全约束与动态扫描机制，实现工业级、零残留的系统瘦身与无痕卸载。

**核心特性：**
1. **⚖️ 举证倒置原则（默认受保护）**：Agent 不携带任何预设的清理路径清单。默认一切目录均受保护，Agent 必须通过高标准的“内容抽样验证”来证明某个目录可以安全删除，凡有疑问，一律不删。
2. **🥷 严苛的“零残留”机制**：在扫描和清理的全过程中，绝对不在用户磁盘上留下任何新文件（严格禁止生成 `.ps1`、`.bat`、`.cmd`、`.log` 等临时文件）。所有复杂逻辑均通过内存内联命令链执行，并在结束后执行残留检查。
3. **🔍 智能磁盘扫描与 9 大类深度判定**：Agent 采用纯粹的信息收集策略，逐层深入扫描磁盘结构并按大小排序。精准匹配临时文件、包管理器缓存、更新残留、崩溃转储、渲染着色器、浏览器（含 AI/ML 模型缓存）、IDE 缓存等 9 大类缓存规则。
4. **🛡️ 厂商与用户数据红线保护**：引入了基于路径结构的底层防护，对微软（AppData）、谷歌（Chrome）、社交应用（微信等聊天记录）、虚拟机/模拟器、云同步客户端及 IDE 的核心用户配置、登录状态、数据库实施绝对保护。
5. **🛑 三重确认工作流**：所有清理操作必须严格走完“扫描归类与候选清单生成 -> 10项核心指标 AI 自我审查（交叉校验保护规则与活跃进程） -> 用户最终确认”的三步卡口，绝不盲目执行。
6. **🧹 跨 8 维度彻底卸载与终检**：在应用卸载模式下，不仅清除磁盘残留，还会跨文件系统、注册表、AppX 包、Prefetch、防火墙规则、系统服务、计划任务及沙箱账户等 8 个维度清除一切痕迹。清理完成后运行自动化扫描脚本，进行终检验证并输出交付报告。

---

### English Description

**Feature Summary:**
Designed specifically for Windows systems, this module provides two core operational modes: **"C Drive Cleanup"** and **"Deep App Uninstallation"**. Bound strictly by the core principle "Better to leave ten cluttered directories than mistakenly delete a single user file," it utilizes ironclad safety constraints and dynamic scanning mechanisms to achieve industrial-grade, zero-residual system slimming and trace-wiping.

**Key Features:**
1. **⚖️ Reverse-Onus Principle (Protected by Default)**: The agent carries no rigid pre-defined cleanup path lists. All directories are protected by default; the agent must verify and prove a folder is safe to delete via file-content sampling, otherwise it will be skipped.
2. **🥷 Hardcore Zero-Residual Mechanism**: Throughout the scanning and cleanup process, the creation of any temporary script or log files (such as `.ps1`, `.bat`, `.cmd`, `.log`) on the user's disk is strictly prohibited. All complex operations are executed safely via in-memory inline command chains with post-cleanup residual verification.
3. **🔍 Smart Disk Scanning & 9-Category Deep Classification**: The agent adopts a pure information gathering strategy, expanding and sorting subdirectories layer-by-layer based on size. It accurately identifies and matches 9 specific cache types, including temporary files, package manager caches, updater residuals, crash dumps, GPU shaders, browser/AI model caches, and IDE caches.
4. **🛡️ Robust Vendor & User Data Protection**: Implemented structural path protection rules that enforce an absolute ban on deleting crucial user profiles, login states, and databases belonging to Microsoft (AppData), Google (Chrome), social apps, virtual machines/emulators, cloud sync clients, and IDEs.
5. **🛑 Triple-Confirmation Workflow**: Before any deletion, the agent must strictly pass through a three-stage gateway: "Scan & Structured Candidate List Generation -> AI Self-Audit across 10 core indicators (cross-checking protection rules and active processes) -> Final User Confirmation". No blind operations are tolerated.
6. **🧹 8-Dimensional Deep Uninstallation & Final Verification**: In uninstallation mode, the agent wipes application traces across 8 distinct dimensions (File System, Registry, AppX, Prefetch, Firewall Rules, Windows Services, Scheduled Tasks, and Sandbox Accounts), backed by an automated terminal validation script to confirm zero residuals and output formal deliverables upon completion.
