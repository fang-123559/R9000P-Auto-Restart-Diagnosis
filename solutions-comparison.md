# 解决方案对比与推荐

---

## 已实施的方案（效果评估）

| 方案 | 操作 | 实施日期 | 效果 | 分析 |
|------|------|----------|------|------|
| A1 | 最小处理器状态 → 100%（AC） | 5/16 | ❌ **未解决** | OS 层策略无法约束 PSP 固件的 C-State 控制 |
| A2 | PCI-E ASPM → 关闭 | 5/16 | ✅ 生效 | 排除了 PCIe 链路状态管理干扰 |
| A3 | 硬盘关闭时间 → 0（从不） | 5/16 | ✅ 生效 | 排除了硬盘控制器休眠干扰 |
| A4 | 睡眠超时 → 15小时 | 5/16 | ✅ 生效 | 减少了系统进入睡眠的机会 |
| A5 | BIOS 更新 GKCN34WW → GKCN65WW | 5/16 | ⚠️ 部分改善 | 仍有 5/21 复发 |
| A6 | WLAN 驱动更新 22.160 → 24.20.1 | 5/16 | ✅ 解决 WLAN 崩溃 | IntelIHVRouter10.dll 不再崩溃 |

### A1 方案失败的原因分析

```
┌─────────────────────────────────────────────────────────┐
│  控制层级与约束范围                                       │
│                                                           │
│  Windows 电源策略 (已设置100%)                             │
│  ├── 可以控制：CPU P-State 频率切换                       │
│  ├── 可以控制：CPU 最小/最大频率百分比                    │
│  └── ❌ 无法控制：C-State 深度睡眠状态                    │
│                                                           │
│  AMD PSP 固件 (当前版本 5.18.0.0)                         │
│  ├── 控制：C-State 状态机（C0/C1/C6/C7/C8）              │
│  ├── 控制：CPPC 协作处理器性能控制                        │
│  └── 控制：深度睡眠状态的进入/退出时机                    │
│                                                           │
│  BIOS 固件 (当前版本 GKCN65WW)                            │
│  └── 最终仲裁：覆盖所有上层控制策略                        │
└─────────────────────────────────────────────────────────┘
```

---

## 待实施的方案

### 方案 B：更新 AMD 芯片组驱动（推荐优先）

**原理**：AMD PSP（Platform Security Processor）驱动是控制 CPU 电源状态转换的关键组件。当前版本 5.18.0.0（2022年）可能包含导致 C-State 唤醒失败的 BUG。新版 5.44.0.0 包含针对性修复。

| 项目 | 详情 |
|------|------|
| 目标版本 | AMD Ryzen Chipset Driver 8.05.04.516 |
| 下载链接 | https://drivers.amd.com/drivers/amd_chipset_software_8.05.04.516.exe |
| 官方发布说明 | https://www.amd.com/en/resources/support-articles/release-notes/RN-RYZEN-CHIPSET-8-05-04-516.html |
| 关键更新 | PSP Driver 5.18→5.44，Processor Power Management 8.0.1.13 |
| 安装方法 | 下载.exe → 双击运行 → 接受 EULA → 安装完成 → 重启 |
| 风险 | 低（WHQL 认证驱动） |
| 预期效果 | 修复 C-State 唤醒失败 BUG |

**更新后验证步骤**：
1. 重启后运行 `powercfg /a` 确认系统状态正常
2. 检查设备管理器中 AMD 驱动版本是否已更新
3. 观察 3-5 天是否仍有凌晨自动关机

---

### 方案 C：UMAF 解锁 BIOS 深度设置（方案 B 无效时使用）

**原理**：通过 UMAF（Universal AMD Form Browser）工具直接访问联想 BIOS 中隐藏的 AMD CBS（Customized BIOS Settings）菜单，禁用 CPU C-State 控制。

| 项目 | 详情 |
|------|------|
| 工具 | Universal AMD Form Browser (UMAF) |
| GitHub 仓库 | https://github.com/JamesCJ60/Universal-x86-Unlocking-Tool |
| 准备工作 | 8GB+ U 盘，制作 UEFI 启动盘 |
| 风险 | 中（修改 BIOS 设置，但不刷写 BIOS，可恢复默认） |

**操作步骤**：
1. 下载 UMAF 工具（.zip）
2. 解压到 U 盘根目录（FAT32 格式）
3. 重启 → 进入 BIOS → 关闭 Secure Boot
4. 从 U 盘启动进入 UMAF
5. 导航至：AMD CBS → SMU Common Options
6. 修改以下设置：

| 设置项 | 建议值 | 说明 |
|--------|--------|------|
| Global C-State Control | **Disabled** | 禁用所有 C-State |
| CPPC | **Disabled** | 禁用协作处理器性能控制 |
| CPPC Preferred Cores | **Disabled** | 禁用首选核心调度 |
| C-State Limit | **C0/C1** | 限制最浅 C-State |

7. 保存并退出
8. 重启进入 Windows，验证设置生效

**恢复方法**：进入 BIOS → Load Default Settings → 保存退出

---

### 方案 D：注册表禁用 CPPC（轻量级替代方案）

如果不想用 UMAF，可以通过注册表禁用 CPPC：

```
操作步骤：
1. Win + R → regedit → 回车
2. 导航至：HKLM\SYSTEM\CurrentControlSet\Control\Processor
3. 右键 → 新建 → DWORD (32位) 值
4. 命名为：DisableCpc
5. 双击设置值为：1
6. 重启电脑
```

---

## 硬件维修方案（最后手段）

| 方案 | 费用估计 | 适用情况 | 说明 |
|------|---------|----------|------|
| 联想官方售后检测 | 免费（检测） | 所有软件方案均无效 | 保修期内免费维修 |
| CPU BGA 重植 | 500-1500 元 | 确认 CPU 虚焊 | 需要专业 BGA 返修设备 |
| 更换主板 | 1500-2500 元 | 确认主板故障 | 官方报价，保修半年 |
| 更换 CPU | 800-1500 元 | 确认 CPU 损坏 | 需匹配同版本 CPU |

> 本机为 2021 年购入，**大概率已过保修期**。建议先完成所有软件层面排查后再考虑硬件维修。

---

## 推荐执行顺序

```
Step 1: 更新 AMD 芯片组驱动至 8.05.04.516 ──→ 观察 3-5 天
                                                  │
                                          ┌───────┴───────┐
                                          ▼               ▼
                                     问题解决         问题复现
                                          │               │
                                          ▼               ▼
                                     维护观察         Step 2: UMAF 禁用 C-State
                                                              │
                                                      ┌───────┴───────┐
                                                      ▼               ▼
                                                 问题解决         问题复现
                                                      │               │
                                                      ▼               ▼
                                                 维护观察         Step 3: 联想售后检测
```

---

*文档创建时间：2026年5月21日*
