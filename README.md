# 嵌入式 C 解耦架构 Skills

适用于 **Hermes**、**Claude Code**、**Codex CLI** 的嵌入式 C 代码解耦与分层架构技能包。

## 目录

### 1. OOP 五层分层架构 (`embedded-oop-architecture`)

C 语言虚表 OOP 模式，适用于 Cortex-M 裸机项目（STM32 / GD32 / APM32）。

```
Interface → Adapter → Device → Board → App
```

- 通过 `_ops_t` 函数指针表实现多态
- `Platformdefine.h` 隔离芯片头文件
- 通过 `board_config` 依赖注入
- 完整 `.h` / `.c` 骨架模板
- 切换平台（STM32 ↔ GD32）无需改动 Device/Board/App 层代码

### 2. 函数指针注册模式 (`embedded-fnptr-register`)

轻量级回调注册模式，适用于事件驱动的嵌入式 C 项目。

- 单回调、多回调注册表、带上下文 `void* ctx` 三种模式
- 覆盖串口接收、ADC 完成、按键矩阵、定时器软调度等场景
- ISR 安全设计：中断只置标志位，主循环处理

---

## 安装

### Hermes

```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md
```

### Claude Code

对 Claude 说：

> "请安装这个 skill：读取并应用 https://github.com/gsiliconk/embedded-skills 中的编码规范，安装这个skills"

### Codex CLI

对 Codex 说：

> "请安装这个 skill：读取并应用 https://github.com/gsiliconk/embedded-skills 中的编码规范，安装这个skills"

---

## 仓库结构

```
embedded-skills/
├── README.md
└── skills/
    ├── embedded-oop-architecture/
    │   └── SKILL.md
    └── embedded-fnptr-register/
        └── SKILL.md
```

## 许可证

本仓库所有 skill 文件采用 [MIT License](./LICENSE)。