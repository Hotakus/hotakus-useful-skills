---
name: windows-file-links
description: Windows 文件与目录链接——创建、验证、删除符号链接(symlink)、目录联接(junction)、硬链接(hardlink)时，按权限与卷约束选择链接类型，规避 MSYS/Cygwin 的链接陷阱。覆盖 Windows 原生命令行与 PowerShell，排除 Linux/macOS 的 ln 语义。
license: MIT
metadata:
  tags: windows, symlink, junction, hardlink, filesystem
---

# Windows 文件与目录链接

在 Windows 上创建、维护、删除文件或目录链接时，必须严格遵循以下准则。目标是让链接真正指向目标而不是变成副本，并在删除时只移除链接本身。

**前置条件**：动手前必须确认三件事——链接要指向什么、当前会话有没有创建权限、目标在哪个卷。

## 1. 为什么需要这些规则

链接的失败模式有三类：**建成了副本**（以为省了空间，实际翻倍）、**建成后立即失效**（类型不匹配或目标不存在）、**删除时连目标一起删**（数据丢失）。

Windows 上三种链接的语义与权限要求各不相同，凭 Linux 的 `ln -s` 直觉操作会同时踩中前两类。

## 2. 规则 1：先判断是否真的需要链接

链接是最后手段。能让程序改配置指向新位置，就不要建链接；能直接移动目录，就不要留链接。

反例：

```
用户想把缓存目录从系统盘挪到数据盘
→ 直接建 junction 指过去
```

正例：

```
先查该程序是否支持配置缓存路径
→ 支持则改配置并移动目录
→ 路径写死没法改，才考虑 junction
```

理由：链接会被备份工具、同步工具、安装程序误判为普通目录，长期维护成本高于改配置。

## 3. 规则 2：先查权限与卷，再选类型

动手前必须确认两项：

1. **创建权限**：当前会话是否有管理员权限，或系统是否开启开发者模式
2. **卷与文件系统**：链接与目标是否在同一本地卷，文件系统是否 NTFS/ReFS

判定顺序：

```
有管理员权限 或 已开开发者模式？
├─ 是 → symlink 可用（类型最全，可跨卷、可指网络路径）
└─ 否 → 只能用 junction（目录）或 hardlink（文件）
```

反例：

```
无权限环境下先试 ln -s，失败后没有备用方案
```

正例：

```
无权限 → 目录上 junction，文件上 hardlink
```

## 4. 规则 3：按场景选链接类型

| 场景 | 类型 | 权限 | 约束 |
|---|---|---|---|
| 目录，同一本地卷 | junction | 无特权要求 | 不支持相对路径 |
| 目录，跨本地卷 | junction | 无特权要求 | 同机本地卷之间可行 |
| 目录，指向网络映射盘 | 不可行 | — | 换方案 |
| 文件，同一卷 | hardlink | 无特权要求 | 只能文件，不能跨卷 |
| 文件，跨卷 | symlink | 需管理员或开发者模式 | 仅 NTFS/ReFS |
| 目录或文件，指向网络路径 | symlink | 需管理员或开发者模式 | 仅 NTFS/ReFS |

三者的语义差异决定了哪种能删、哪种删了会丢数据：

- **hardlink**：两个目录项指向同一份数据，没有独立的"链接"实体。删任一目录项数据仍可访问，全部删除才释放空间。不能指向目录，不能跨卷。
- **junction**：目录级重解析点，指向另一个目录。删除链接只移除重解析点，目标目录不受影响。可跨同机本地卷，不能指向网络映射盘。
- **symlink**：文件或目录级重解析点。类型在创建时固定，目标类型变化后链接失效。

出处：[Win32 hard links and junctions](https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions)、[mklink](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/mklink)。

## 5. 规则 4：创建命令

cmd：

```
mklink /d <链接> <目标>    :: 目录符号链接
mklink /h <链接> <目标>    :: 硬链接（仅文件）
mklink /j <链接> <目标>    :: 目录联接
mklink <链接> <目标>       :: 不带开关 = 文件符号链接
```

不带开关时一律建文件符号链接，不会根据目标类型自动判断。

PowerShell：

```
New-Item -ItemType SymbolicLink -Path <链接> -Target <目标>
New-Item -ItemType Junction     -Path <链接> -Target <目标>
New-Item -ItemType HardLink     -Path <链接> -Target <目标>
```

`New-Item -ItemType SymbolicLink` 按目标类型自动建文件或目录链接；目标不存在时默认建文件链接。

反例：

```
mklink C:\data\link D:\data\target
想链接目录却漏了 /d 或 /j，结果建出文件链接
```

正例：

```
mklink /j C:\data\link D:\data\target
```

## 6. 规则 5：shell 陷阱

### 6.1 不用 ln -s

MSYS2 官方文档说明其默认模式为 `winsymlinks:deepcopy`，即 `ln -s` 会**深拷贝**目标而不是建链接（[出处](https://www.msys2.org/docs/symlinks)）。Git-for-Windows 风格的 MSYS bash 中实测更直接：`ln -s` 对文件和目录一律报 `Operation not permitted`，设不设 `MSYS=winsymlinks:nativestrict` 都一样。

两种行为都不可接受：前者白占空间且目标必须存在，后者静默失败。

反例：

```
ln -s /d/data /c/link
```

正例：

```
powershell.exe -NoProfile -Command "New-Item -ItemType Junction -Path 'C:\link' -Target 'D:\data'"
```

### 6.2 MSYS 会改写 /c 这类参数

在 MSYS bash 里调用 `cmd.exe` 时，`/c` 会被路径转换吃掉，导致 cmd 进入交互模式或报语法错误。需要 cmd 内建命令时设 `MSYS_NO_PATHCONV=1` 并验证命令确实执行，或改走 PowerShell。

反例：

```
cmd //c "rmdir C:\link"
```

正例：

```
powershell.exe -NoProfile -Command "Remove-Item -LiteralPath 'C:\link'"
```

## 7. 规则 6：删除链接

删除前必须确认：**当前路径是链接本身，不是它的目标**。

| 方式 | 目录链接 | 文件链接 | 结果 |
|---|---|---|---|
| `rm -rf <链接>` | 可用 | 可用 | 只删链接，目标内容保留（MSYS bash 实测） |
| `rd <链接>` / `del <链接>` | 可用 | 可用 | cmd 内建，官方文档给的方式 |
| MSYS `rmdir <链接>` | 不可用 | — | 报 `Not a directory`（实测） |
| `Remove-Item -LiteralPath` | 不可用 | 未测 | 抛 `NullReferenceException`（实测） |

关键事实：`rm -rf` 对 junction **不会**穿透删除目标内容——实测链接消失、目标目录内文件完好。这一点与直觉相反，已复现。

反例：

```
# 不确定路径是否为链接时直接递归删除
rm -rf /c/data/*
```

正例：

```
# 先确认是链接，再删除单个链接路径
ls -la /c/data
rm -rf /c/data/link
```

## 8. 规则 7：创建后必须验证穿透

建完链接必须立刻验证两件事：链接本身类型正确、通过链接能读到目标内容。

```
ls -la <链接>       # 确认为链接（lrwxrwxrwx）
readlink <链接>     # 核对目标路径
ls <链接>/          # 确认能列出目标内容
```

在 MSYS bash 中 junction 显示为 `lrwxrwxrwx`，`readlink` 返回的目标可能带 `Global\` 或 `\??\` 前缀，这是重解析点的正常表示，不是路径错误。

反例：

```
建完就宣布完成，没有验证
```

正例：

```
建完 → readlink 核对目标 → ls 穿透列表 → 才报告成功
```

## 9. 规则 8：报告链接类型与目标

报告必须说清三件事：链接类型、链接路径、目标路径。只报"建好了"会让后续维护者无法判断语义。

反例：

```
已创建链接。
```

正例：

```
创建 junction：C:\link → D:\data
验证：ls 穿透可列出目标内容
删除方式：rm -rf C:\link（只删链接）
```

## 10. 失败对照表

| 报错 | 原因 | 处理 |
|---|---|---|
| `You do not have sufficient privilege to perform this operation` | mklink 建 symlink 无权限 | 换 junction/hardlink，或提权，或开开发者模式 |
| `Administrator privilege required for this operation` | PowerShell 建 symlink 无权限 | 同上；也可能 PowerShell 版本过旧 |
| `The device does not support symbolic links` | 目标在 FAT32/exFAT | symlink 不可用，同机本地卷换成 junction |
| `Symbolic links are not supported for the specified path` | 同上，PowerShell 措辞 | 同上 |
| `Operation not permitted` | MSYS 的 ln -s | 改用 PowerShell 或 mklink |
| `Not a directory` | MSYS rmdir 删 junction | 改用 rm -rf |
| `NullReferenceException` | PowerShell 删 junction | 改用 rm -rf |
| 链接建好但打不开 | 文件/目录类型不匹配 | 删除重建，类型必须与目标一致 |

## 11. 禁止事项

1. 不在未确认路径语义时递归删除（`rm -rf <目录>/*`）
2. 不用 `ln -s` 在 MSYS/Cygwin 环境建链接
3. 不假设 symlink 在无权限环境可用
4. 不对网络映射盘建 junction
5. 不给 junction 传相对路径
6. 不在 FAT32/exFAT 上尝试 symlink
7. 不在创建后跳过穿透验证
8. 不把"创建成功"当作"指向正确"

## 12. 自检要求

完成链接操作后逐项复核：

- [ ] 已确认链接用途，且改配置或直接移动不可行
- [ ] 已确认当前会话权限，类型选择与权限匹配
- [ ] 已确认目标文件系统为 NTFS/ReFS（symlink 场景）
- [ ] 已确认 junction 未使用相对路径、未指向网络映射盘
- [ ] 创建命令的开关与预期链接类型一致
- [ ] `ls -la` 显示为链接而非实体目录或文件
- [ ] `readlink` 目标路径与预期一致
- [ ] 通过链接能读到目标内容
- [ ] 删除场景已确认删的是链接本身
- [ ] 报告含类型、链接路径、目标路径三项
