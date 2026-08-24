可以，我们就按 **Windows 个人电脑 + PowerShell 脚本** 来做。整个过程其实只有：

**新建 `.ps1` 文件 → 粘贴代码 → 运行脚本 → 用 `Ctrl + C` 停止。**

我下面按你可以直接跟着点的方式写。

## 1. 在桌面新建一个 `.ps1` 文件

先回到 Windows 桌面。

### 先让 Windows 显示文件扩展名

这个很重要，不然你可能最后建成：

```text
keep_awake.ps1.txt
```

而不是 `.ps1` 文件。

打开任意一个文件夹，例如“此电脑”，然后：

```text
查看
→ 显示
→ 文件扩展名
```

把 **“文件扩展名”** 勾上。

Windows 11 通常就是：

```text
文件资源管理器顶部
→ 查看
→ 显示
→ 文件扩展名
```

---

然后回到桌面：

```text
右键桌面空白处
→ 新建
→ 文本文档
```

你会看到：

```text
新建 文本文档.txt
```

把它重命名成：

```text
keep_awake.ps1
```

Windows 会弹出：

> 如果改变文件扩展名，可能会导致文件不可用，确实要更改吗？

点击：

**是**

这样桌面上就有：

```text
keep_awake.ps1
```

---

## 2. 编辑这个文件

右键：

```text
keep_awake.ps1
```

选择：

```text
打开方式
→ 记事本
```

如果右键菜单里直接有：

```text
编辑
```

也可以直接点“编辑”。

然后把里面原来的内容全部删掉，复制下面这段进去：

```powershell
Add-Type @"
using System;
using System.Runtime.InteropServices;

public class MouseJiggler
{
    [DllImport("user32.dll")]
    public static extern void mouse_event(
        uint dwFlags,
        uint dx,
        uint dy,
        uint dwData,
        UIntPtr dwExtraInfo
    );

    public const uint MOUSEEVENTF_MOVE = 0x0001;
}
"@

Write-Host "Keep Awake 已启动"
Write-Host "每 30 秒模拟一次鼠标移动"
Write-Host "按 Ctrl+C 即可停止"
Write-Host ""

$direction = 1

try {
    while ($true) {

        [MouseJiggler]::mouse_event(
            [MouseJiggler]::MOUSEEVENTF_MOVE,
            $direction,
            0,
            0,
            [UIntPtr]::Zero
        )

        $direction = -$direction

        Start-Sleep -Seconds 30
    }
}
finally {
    Write-Host "Keep Awake 已停止"
}
```

然后：

```text
Ctrl + S
```

保存。

保存完以后关闭记事本。

---

## 3. 第一次运行

现在桌面应该有：

```text
keep_awake.ps1
```

### 方法最简单：从终端运行

在桌面空白位置按住：

```text
Shift
```

然后右键。

如果菜单里面有：

```text
在终端中打开
```

点它。

如果没有，也可以：

```text
右键开始菜单
→ 终端
```

或者搜索：

```text
PowerShell
```

打开 **Windows PowerShell**。

---

如果你是从桌面“在终端中打开”的，正常会看到类似：

```powershell
PS C:\Users\你的用户名\Desktop>
```

这说明你现在就在桌面目录。

输入：

```powershell
powershell -ExecutionPolicy Bypass -File .\keep_awake.ps1
```

按 Enter。

正常情况下会出现：

```text
Keep Awake 已启动
每 30 秒模拟一次鼠标移动
按 Ctrl+C 即可停止
```

然后窗口看起来像“卡住”了一样。

这是**正常的**。

因为程序正在：

```text
运行
↓
等待 30 秒
↓
模拟鼠标移动
↓
再等待 30 秒
↓
一直循环
```

这个 PowerShell 窗口不要关闭。

---

## 4. 怎么判断它是否真的在工作

你可以先测试一下。

运行脚本以后，不碰鼠标键盘。

脚本大概每 30 秒会让鼠标发生一次极小的移动。

理论上鼠标会：

```text
右移 1
↓
30 秒
↓
左移 1
↓
30 秒
```

所以整体位置不会一直跑偏。

你甚至可能肉眼完全看不出来，因为只有 1 个单位。

程序运行以后，你就可以：

```text
把 KVM 切到另一台电脑
```

即使这台电脑已经没有物理鼠标、键盘连接，脚本仍然应该继续运行，因为这是 Windows 自己通过 API 生成输入事件。

---

# 5. 怎么随时停止

等你把 KVM 切回这台电脑以后：

先点击正在运行脚本的 PowerShell 窗口。

然后按：

```text
Ctrl + C
```

程序就会结束。

通常你会重新看到：

```powershell
PS C:\Users\你的用户名\Desktop>
```

这就说明脚本已经停止。

你也可以直接关闭 PowerShell 窗口，效果也是停止程序。

---

## 6. 如果双击 `.ps1` 文件没反应

这个也是正常的。

Windows 默认并不会像 `.exe` 一样直接执行 `.ps1`。

所以以后每次运行，推荐继续使用：

```powershell
powershell -ExecutionPolicy Bypass -File .\keep_awake.ps1
```

不需要修改整个系统的 PowerShell 安全策略。

这里的：

```text
-ExecutionPolicy Bypass
```

只针对**这一次 PowerShell 进程**。

关闭窗口以后就结束了。

---

## 7. 可以让以后启动更方便

如果你确认上面的脚本已经成功工作，我们下一步可以把它改成：

```text
桌面上有一个“开始防锁屏.bat”
```

以后只需要**双击一次**：

```text
开始防锁屏.bat
```

就自动打开 PowerShell 并运行。

甚至还可以再做一个：

```text
停止防锁屏.bat
```

这样就不用每次输入命令了。

**你先做到第 3 步，运行以后如果看到 `Keep Awake 已启动`，就说明基础版本已经成功。**
