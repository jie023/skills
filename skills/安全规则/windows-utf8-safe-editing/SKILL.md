---
name: "windows-utf8-safe-editing"
description: "Use when reading, editing, or creating Chinese text/code files on Windows, especially with Chinese paths, UTF-8 requirements, mojibake, garbled output, or encoding-sensitive changes."
---
# Windows UTF-8 安全编辑

## 使用场景

- 文件路径包含中文、空格或 Windows 盘符
- 文件内容包含中文注释、中文文档、接口说明或配置文本
- 终端输出出现乱码、`���`、`涓`、`鏉` 等疑似编码错误
- 需要写入、替换、迁移或复核 UTF-8 文件

## 核心原则

- 不得基于乱码内容修改文件。
- 读取中文文件时必须显式指定 UTF-8；如果仍乱码，先判断原文件实际编码。
- 写入中文文件时必须保持 UTF-8，优先使用无 BOM UTF-8。
- 修改后必须重新读取关键片段，确认中文未乱码、格式未异常。

## Windows 路径规则

- 工具调用中优先使用正斜杠路径，例如 `C:/Users/李杰/project/file.md`。
- 路径包含空格或中文时，PowerShell 使用 `-LiteralPath`。
- 不要用未转义反斜杠拼接路径。

## 平台适配

- Claude：读取优先 `Read`，小改动优先 `Edit`，新文件或整文件重写使用 `Write`；如需确认编码，可用 PowerShell 显式 UTF-8 读取复核。
- Codex：搜索优先 `rg`，读取可用 `Get-Content -Encoding UTF8`，修改优先 `apply_patch`；跨盘外部文件可用 PowerShell/.NET 写入。
- Trae：读取优先 `SearchCodebase` / `Read`，修改优先平台编辑工具；涉及中文文件时仍需显式关注 UTF-8 和写后复核。

## PowerShell 读取

```powershell
Get-Content -LiteralPath 'C:/path/中文文件.md' -Encoding UTF8
```

带行号复核：

```powershell
$i=0; Get-Content -LiteralPath 'C:/path/中文文件.md' -Encoding UTF8 | ForEach-Object {
    $i++
    '{0,4}: {1}' -f $i, $_
}
```

## PowerShell 写入

写入前设置错误策略，避免非终止错误后继续执行写文件：

```powershell
$ErrorActionPreference = 'Stop'
$path = 'C:/path/中文文件.md'
$text = [System.IO.File]::ReadAllText($path, [System.Text.Encoding]::UTF8)
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText($path, $text, $utf8NoBom)
```

多行拼接时先定义换行符，避免 PowerShell 表达式误执行：

```powershell
$nl = if ($text.Contains("`r`n")) { "`r`n" } else { "`n" }
$replacement = @(
    '第一行中文',
    '第二行中文'
) -join $nl
```

## 修改流程

1. 确认文件绝对路径。
2. 使用 UTF-8 显式读取目标片段。
3. 如发现乱码，停止修改，重新判断编码。
4. 对小范围修改优先使用当前平台编辑工具；跨盘或外部文件可用 PowerShell/.NET 写入。
5. 写入时保持 UTF-8 无 BOM。
6. 写入后使用 UTF-8 重新读取关键片段，并检查中文、缩进、换行和代码块。

## 禁止事项

- 禁止用乱码输出作为替换依据。
- 禁止未确认影响范围就批量写入中文文件。
- 禁止用默认编码写入中文文件。
- 禁止在 PowerShell 命令中依赖非终止错误继续执行后续写入。
