---
title: "RFB协议中键盘事件（KeyEvent）的处理"
date: 2020-06-08T16:44:05+08:00
draft: false
---

> RFB（remote framebuffer）是用于图形界面远程接入的开源协议，基于C/S架构，在Windows和Linux操作系统下都有服务器和客户端实现，有较好的跨平台特性。本文从协议层入手，探究键盘事件在服务器端和客户端间的传输方式。

在RFC协议中，“按键事件”指键盘被按下（Down）或抬起（Up）的事件。一个键被按下时，该键对应的 down-flag 是非零值；当键抬起时，down-flag 是零值。按键用 X 桌面系统（X Window System）的按键符号（keysym）定义，无论协议两端的操作系统是 Windows 还是 Linux。

一个按键事件的消息结构如下表：

```
              +--------------+--------------+--------------+
              | No. of bytes | Type [Value] | Description  |
              +--------------+--------------+--------------+
              | 1            | U8 [4]       | message-type |
              | 1            | U8           | down-flag    |
              | 2            |              | padding      |
              | 4            | U32          | key          |
              +--------------+--------------+--------------+
```

每个按键事件在协议中共占4个字节。第一个字节包括分两部分，第一部分是 message-type，代表该消息的类型，其值为 0x04，长度是 8 bit，由协议固定分配；第二部分是 down-flag，用于表征该键是否被按下。down-flag 是非零值时，表示该键位处于“按下”状态；down-flag 恢复到 0 值时，该键位恢复“抬起”状态。第二个字节空置，用于位对齐，同时也为协议留下了升级的空间。

第三和第四个字节表示键位信息，键位的符号由 X Window System 定义，具体的值可以参考 [XLIBREF](https://tools.ietf.org/html/rfc6143#ref-XLIBREF) 中 <X11/keysymdef.h> 头文件的定义。X Window System 的通常简称为 X11，是用位图方式显示的软件窗口系统，现在已经是 Linux 等操作系统软件工具包和显示架构的通信协议。对于大多数普通键而言，该键的按键符号等于这个键的 ASCII 码本身，这里罗列一下常用的键：

```
                 +-----------------+--------------------+
                 | Key name        | Keysym value (hex) |
                 +-----------------+--------------------+
                 | BackSpace       | 0xff08             |
                 | Tab             | 0xff09             |
                 | Return or Enter | 0xff0d             |
                 | Escape          | 0xff1b             |
                 | Insert          | 0xff63             |
                 | Delete          | 0xffff             |
                 | Home            | 0xff50             |
                 | End             | 0xff57             |
                 | Page Up         | 0xff55             |
                 | Page Down       | 0xff56             |
                 | Left            | 0xff51             |
                 | Up              | 0xff52             |
                 | Right           | 0xff53             |
                 | Down            | 0xff54             |
                 | F1              | 0xffbe             |
                 | F2              | 0xffbf             |
                 | F3              | 0xffc0             |
                 | F4              | 0xffc1             |
                 | ...             | ...                |
                 | F12             | 0xffc9             |
                 | Shift (left)    | 0xffe1             |
                 | Shift (right)   | 0xffe2             |
                 | Control (left)  | 0xffe3             |
                 | Control (right) | 0xffe4             |
                 | Meta (left)     | 0xffe7             |
                 | Meta (right)    | 0xffe8             |
                 | Alt (left)      | 0xffe9             |
                 | Alt (right)     | 0xffea             |
                 +-----------------+————————————————————+
```

在多系统环境下，解释按键符号是一件很复杂的事情，为了实现尽可能广泛的互操作性，RFB协议建议遵循以下原则：

- Shift 状态只应该用做解释按键符号的提示（而不能修改键的表达）。例如，在美式键盘中，“#”需要按下 Shift 才能输入（shift+3），而在英式键盘中，“#”可以被直接输入（不需要Shift 键）。当使用英式键盘的客户端发送"#"时，使用美式键盘的服务器并不会收到 Shift 键被按下的消息。在这种情况下，服务器需要在内部模拟一个 Shift 按键，这样才能在服务器上得到字符 “#”，而不是字符“3”。相反的，在美式键盘的客户端输入“#”可能会发送 shift+#，这时英式键盘的服务器应该输入“#”，忽略用于解释的“shift”。
- 大写和小写的按键符号也有明显区别。在 X 系统中通常认为他们是相同的，但 RFB 协议认为他们完全不同。例如，一个服务器收到了大写的键 “A”，但没有收到任何的 SHIFT 键，服务器应该将这个按键解释为 “A”。这意味着服务器同样需要在内部模拟一个 Shift 按键。
- 服务器应该忽略 CapsLock 和 NumLock 之类的“锁定”按键符号，根据每个按键符号的大小写来解释他们。
- 跟 Shift 不同，修饰键，例如 Control 和 Alt，会修改其他键的解释。像 Ctrl+A 这种控制键组合并没有对应的按键符号来表示，客户端应该先发送 Control 被按下的消息，再发送 A 被按下的消息，来生成组合键。
- 在可以用 Control 和 Alt 等修饰键生成按键符号的客户端上，客户端需要发送额外的“抬起”事件，以便正确的解释键盘符号。例如，在德语的PC键盘上，Ctrl+Alt+Q会生成“@”字符。在这种情况下，客户端需要模拟 Control 和 Alt 的释放事件，以便正确的解释“@”字符，因为 Ctrl+Alt+@对服务器来说可能代表着完全不同的内容（这种情况下，客户端按下的是 Ctrl+Alt+Q，服务器收到的是 Ctrl+Alt+@）。
- 在 X 系统中没有定义后退标签（Back Tab）的统一标准，在一些系统中，Shift+Tab 对应的键盘符号是 “ISO_Left_Tab”；而在另一些系统中，有专用的 BackTab 符号；在其他的系统中，则给出 TAB 符号，应用程序需要根据 Shift 键的状态来判断触发的是 Back Tab 还是 Forward Tab。在 RFB协议中，优先使用后面的方式。客户端应该生成 Shift 状态下的 Tab 符号，而不是 ISO_Left_Tab。但是，为了向后兼容现有的客户端，服务器还是应该将 ISO_Left_Tab 识别为 Shift + Tab
- 现代 X 系统可以处理 unicode 字符的按键符号，包含十六进制的最高比特位被置位1（1000000）的所有字符集。但为了保证最大兼容性，如果一个键同时拥有 unicode 和旧式编码，客户端应该发送旧式编码。
- 某些系统对特定的组合键，例如 Ctrl+Alt+Delete，给出特定的解释。RFB 客户端通常提供菜单或工具栏功能来发送此类组合键，RFB 协议不对它们进行特殊处理。如果要发送 Ctrl+Alt+Delete，客户端会发送左（或右）Ctrl，左（或右）Alt，DELETE键，然后释放这些按键。许多 RFB 服务器接受 Shift+Ctrl+Alt+Delete 作为 Ctrl+Alt+Delete 的同义词，可以直接从键盘输入。