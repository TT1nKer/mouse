# 如何将 report.md 转换为 DOCX 文件

## 方法一：使用 Pandoc（推荐）

### 安装 Pandoc：
```powershell
# 方法1：使用 winget
winget install --id JohnMacFarlane.Pandoc

# 方法2：使用 Chocolatey
choco install pandoc

# 方法3：手动下载
访问：https://pandoc.org/installing.html
```

### 转换命令：
```powershell
cd plan
pandoc report.md -o report.docx
```

## 方法二：使用在线工具

1. 访问：https://www.markdowntoword.com/
2. 上传 `report.md` 文件
3. 下载生成的 `.docx` 文件

或者：
1. 访问：https://cloudconvert.com/md-to-docx
2. 上传文件并转换

## 方法三：使用 VS Code 插件

1. 安装 "Markdown PDF" 插件
2. 打开 report.md
3. 右键选择 "Markdown PDF: Export (docx)"

## 方法四：手动复制

1. 打开 report.md
2. 复制全部内容 (Ctrl+A, Ctrl+C)
3. 打开 Microsoft Word
4. 粘贴 (Ctrl+V)
5. 保存为 .docx 格式

---

**文件位置**：`plan/report.md`

