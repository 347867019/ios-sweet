# Spec — 一加6 HackiOS 阶段 10（外设：Wi-Fi → GPU → 音频）

> 2026-10-26 ｜ 目标：阶段 9/SMP 之后接外设。子项顺序（本轮调研后决定）：**Wi-Fi（详案）→ GPU（概案）→ 音频（概案）**，每个子项开工前再补详案。
>
> **2026-10-27 架构修正**：Wi-Fi 路线推翻重来。OP6 的 WCN3990 挂 **SNOC**（`qcom,icnss@18800000`），**不在 PCIe 上**；驱动是 **ath10k**（ISC），不是 ath11k。此前"17cb:1101 挂 pcie0"是把 db845c（Dragonboard 845c，板载 QCA6390 ath11k 在 pcie0）误当 oneplus6 先例——ath11k 源码里 0x1101 就是 QCA6390，linux-firmware 里 ath11k 目录根本没有 WCN3990。已写的 redwood_pcie.c/run65 与 Wi-Fi 脱钩（见 §1e 归档）。

## 0. 调研结论（2026-10-26 三路并行取证）

1. **hackios 仓已有全套脚手架，阶段 10 不是从零写**：
   - [mk-ath11k.sh](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/tools/mk-ath11k.sh)：ath11k（Linux 7.0.14，BSD-3-Clause-Clear，逐文件许可校验）+ FreeBSD 15.1 LinuxKPI（带 patch）+ `wifi/lx` 适配层（lx_base/lx_bus/lx_80211/lx_wpa）→ 编成 `ath11k.o`，导出 `lx_prepare/lx_start/lx_stop/lx_transmit`（[redwood_lx.h](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/xnu/files/bsd/sys/redwood_lx.h) 接口）。
   - [enchilada.conf](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/common/devices/enchilada.conf) 已写 `WIFI_ATH11K=1`（"WCN3990, integrated on PCIe (ath11k supports it)"）+ `FIRMWARE="fw-gpu fw-wifi fw-audio fw-haptic"`（取自手机 Android /vendor/firmware_mnt，经 adb，不进仓库）。
   - 内核侧全链已存在：[redwood_lx.c](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/xnu/files/bsd/dev/redwood_lx.c)（lx KPI 落地）、[redwood_net.c](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/xnu/files/bsd/dev/redwood_net.c)（en0 经 BSD ifnet KPI）、[RedwoodWLAN.cpp](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/xnu/files/iokit/Kernel/RedwoodWLAN.cpp)（IO80211Controller/IONetworkInterface，wifid 靠服务发现）、redwood_apple80211.c、redwood_sta.c。
   - GPU/音频模板：[redwood_adreno.c](file:///home/oxjmo/oneplus_ios_sweet/repos/hackios-xnu-kernel/xnu/files/bsd/dev/redwood_adreno.c)（Adreno 642L，22 步分步唤醒：GMU+zap shader+TrustZone+RPMh+SMMU，`adreno=N` 步进 boot-arg 风格）、redwood_audio.c（/dev/redwood-audio sink+录制）、redwood_sound/speaker/mic/guard。
2. **redwood_wifi.c 的 16 步流程对 OP6 不适用**：那是 POCO X5 Pro 的 WCN6750（WPSS DSP + PIL/TrustZone 加载 + SMEM/SMP2P/GLINK/QRTR）。WCN3990 无 DSP：芯片本体挂在 SNOC 上，固件 wlanmdsp.mbn 由主机经 **MSA**（共享内存 + QMI WLFW 握手）装载。boot-arg 分步风格（每步先 print 后 verify）沿用。
3. **参考仓无用**：my-ios-sweet README 明确 "No Wi-Fi/GPU/audio"；dipa-ios-sweet 只有文档页总结。主参考 = hackios 自己的 redwood 驱动 + Linux 上游（ath10k 的 snoc/qmi 路径，mainline 先例 **db845c**）。

## 1. Wi-Fi 子案（详案，2026-10-27 SNOC 修正版）

### 1a. OP6 硬件事实（fdt.decompiled.dts L7979 + mainline sdm845.dtsi L5509 交叉验证）

- 节点：vendor `qcom,icnss@18800000` = mainline `wifi@18800000`，compatible `qcom,wcn3990-wifi`。
- **membase 0x18800000，8MB**（SNOC 直映寄存器，非 PCIe BAR）。
- **中断 SPI 414-424**（11 条，LEVEL_HIGH）：CE/WMI 等（vendor fdt 0x19e-0x1a8）。
- **SMMU sid 0x40**（vendor fdt `iommus = <apps_smmu 0x40 0x1>`）+ `qcom,smmu-s1-bypass`（vendor 特性：stage-1 旁路，主线路径只需 stage-2 映射）。
- **MSA 内存 1MB**（`qcom,wlan-msa-memory`；mainline 同义 memory-region wlan_msa_mem）——主机分给 radio 的共享内存。
- 供电四路（RPMh，phandle 待 resolve）：vdd-0.8-cx-mx(0xec)、vdd-1.8-xo(0xed)、vdd-1.3-rfa(0xee)、vdd-3.3-ch0(0xef)。
- 参考时钟：mainline `clocks = <&rpmhcc RPMH_RF_CLK2>`（cxo_ref_clk_pin）——vendor icnss 节点未列，按 mainline 处理。
- **固件（linux-firmware ath10k/WCN3990/hw1.0/ 全套已下到 repos/hackios-xnu-kernel/fw-wifi/wcn3990/）**：
  - `wlanmdsp.mbn` 3.7MB——radio 本体镜像（MSA 装载，含 WMI/HTT 全部端点代码）
  - `firmware-5.bin` 60 字节——合法的**空 IE 容器**（只有版本元数据；shikra/ 子目录同样 60B，证实 wcn3990 真代码全在 msp 镜像里）
  - `board-2.bin` 894KB——board data（含全部 bdid 表项，无需 vendor 原件）
- vendor 原件对照（/vendor/firmware_mnt image/wlanmdsp.mbn）手机回来后可 adb 抓，对齐版本；不阻塞。

### 1b. ath10k SNOC 技术要点（Linux 7.0.14 源码已抽出 /tmp/op6-phy-regs/linux-7.0.14/drivers/net/wireless/ath/ath10k/）

- 文件与许可：snoc.c/qmi.c/qmi_wlfw_v01.c + core/htc/wmi/ce/mac/desc 等全部 **ISC**（许可检查沿用 ath11k 脚本的 case 模式，ISC 形态已支持）。
- HIF = SNOC：寄存器在 membase（0x18800000 直映），**无 MHI/无 PCIe**；CE 环、doorbell 全在 SNOC 地址空间内。
- 固件装载 = **QMI WLFW 流**（ath10k/qmi.c + qmi_wlfw_v01.c）：主机分 MSA 内存 → QMI wlfw 服务协商 → msp 镜像写入 radio → fw_ready。**redwood_qmi.h/lx QMI 助手已存在**（WCN6750 用过），复用评估是 W2' 详案第一步。
- ath10k 上层 = mac80211（LinuxKPI 已覆盖 Linux 7.0 mac80211 语义——ath11k 同栈，lx_80211 层可复用）。
- 详案时必读：ath10k/qmi.c（wlfw 服务端在哪、MSA 流程精确语义）、snoc.c（probe/中断使能/irq 名称表）、ce.c（doorbell 布局）。

### 1c. 内核/构建改动清单（SNOC 版）

1. **redwood_wcn3990.c（新，或 redwood_wifi.c 加 OP6 分支 `wifi=2`）**：分步 bring-up（boot-arg 复用 wifi_step 风格）：①membase 映射+只读探针（每寄存器先打印后读，run64 教训）②四路 RPMh 供电 ③RF_CLK2 ④SMMU sid 0x40 上下文（复用 redwood_wifi.c 的 smmu_look/radio_space_on 套路）+ MSA 内存分配 ⑤SNOC 探针证实 radio 活着 ⑥wlanmdsp.mbn 装载 ⑦QMI wlfw 握手 ⑧fw ready。
2. **中断：SPI 414-424 直连**（LEVEL_HIGH），gic_v3_irq.c 加 SPI 区间参数化/分发——**不需要 MSI/GICv2m**（PCIe 版计划里的 SPI 704+ 作废）。
3. **mk-ath10k.sh（新）**：拷 mk-ath11k.sh 骨架，DRIVER 列表 = ath10k 核心栈 + snoc/qmi/qmi_wlfw_v01；许可检查改 ISC；LinuxKPI 缺口（completion/tasklet 等与 ath11k 的差异面）逐个补——这是主要工作量所在。
4. **lx_bus.c**：platform-device shim 已有（of_ 头），icnss 是 platform 语义，缺的只是 wlfw/QMI 粘合——比 PCI shim 小得多。
5. **固件**：fw-wifi = {wlanmdsp.mbn, firmware-5.bin, board-2.bin}（已就位 fw-wifi/wcn3990/）；userland/build.sh 的 fw 段加 OP6 分支；payload `fw/` 通道已有。
6. enchilada.conf 的 `WIFI_ATH11K=1` 注释更正为 SNOC/ath10k 事实。

### 1d. 验证里程碑（SNOC 版）

- **W1'** SNOC bring-up：membase 探针活、radio 响应（日志实证，分步定位）。
- **W2'** QMI wlfw 握手 + wlanmdsp 接受 → ath10k probe 完成（"fw ready"/chip id 输出等价）。
- **W3'** station scan 出网列表（redwood wifi_scan 先例语义，仅监听）。
- **W4'** join fw/network 指定网络 → en0 up → DHCP → ping 网关。
- **W5'** iOS 用户空间：wifid 经 RedwoodWLAN 发现控制器（best effort，不阻塞 W4'）。
- 回归：`wifi=0` 系统行为与 run63 完全一致。

### 1f. QMP/AOP 门铃前沿判决（2026-10-31，m30–m38 实验）

modem PIL 前置的 `qcom,signal-aop`（QMP 邮箱问 AOP 要镜像 load_state）是 W2' 的门。m30–m38 十六轮 live 取证判决：

- **desc 握手我们侧全对**：msgram 0xc300000 desc 16 词可写可读，features 版本握手完整；但 mc.link=UP 后 AOP 十秒无视（m33）。
- **0x1799000c 出局**：32 位逐位环零反应（m34）。该块是 SMP2P 门铃组（modem/adsp/dsps/cdsp doors），本来就不通 AOP——OP6 vendor fdt 的 qmp-aop irq-reg-base 写法与主线不同。
- **真门铃=apss_shared 0x1791100c bit0**（主线 sdm845 `aoss_qmp mboxes=<&apss_shared 0>`，唯一未测路径）。
- **0x17911000 页杀机只在读**：三度内核死亡全部钉死在首读（m29×2、m34）；**写安全**——m35 16 位盲写+m36/m37 bit0 盲环全部幸存，内核启动到用户空间。
- **新谜：bring-up 线程卡死**在 uc.mbox say 之后的 wait_ms(2000)（m35/36/37 一致；m34 同位置因前置 7s 忙等可返回）——系统其余照常启动。m38 以 spin_us+三段标记二分定位。
- **uc.mbox len=0**：AOP 的 QMP 任务从未给我们留消息（其任务存活性未知）。
- 后备路径（若 APCS 门铃最终不通）：SMEM doors（0x17911008）、SPI389 回铃中断初始化、TZ SMC 转发；设备侧 EDL 软件救援全灭（Sahara RESET 被无视），硬重启自愈（见 HANDOFF §10）。

### 1e. PCIe 工作归档（redwood_pcie.c，2026-10-26/27）

- redwood_pcie.c（17 步、probe 重排序版）与构建产物保留在树中，boot-arg `pcie=1` 手动触发，**不在任何默认路径**。
- 价值：①回答"OP6 pcie0 上到底有什么"（枚举空与否，一轮 run64-style 即可）②probe 拆步/先打印后读的方法论已沉淀。run64 死因的两个候选（冷块读挂 NoC / QHEE trap）未裁决——若将来跑 run65，日志会点名。
- mk-ath11k.sh 的 MHI 抓取段（`tools/mk-ath11k.sh mhi`）同样休眠保留，Wi-Fi 不再依赖。

## 2. GPU 概案（次序 2，开工前补详案）

- 目标 Adreno 630 = **a6xx gen3，无 GMU**：642L 的 22 步里 GMU/HFI/GMU-RPMh 通道全部作废，改为 gpucc@0x5090000 直接配时钟 + gpu_cx/gpu_gx GDSC（DT 已给地址 0x509106c/0x509100c）+ zap shader（a630_zap.mdt，TZ 路径照搬步 12/19）+ a630_sqe.fw 微码 + kgsl-smmu 上下文（套路同步 13/18）。
- 参考实现：Linux drivers/gpu/drm/msm/adreno 的 no-GMU 路径、freedreno/postmarketOS sdm845 社区先例。
- 内核 kgsl 节点（kgsl-3d0@5000000 等）详案时 grep fdt 补全；RedwoodGPU keep-on 服务路径照搬。
- 验证：分步推进到"3D 核写一个字进我们的页"（步 21 等价）→ RedwoodGPU → 用户空间 GL（conf 已预留 GL=1）。

## 3. 音频概案（次序 3，最重，开工前补详案）

- ADSP：`qcom,firmware-name="adsp"`（ssctl-instance-id 0x14，mbox adsp-pil）PIL 加载机制与 redwood_wifi.c WPSS 步 7-13 同源（AOP mailbox、TZ auth、SMP2P、GLINK），大量代码可复用。
- codec：Tavil（WCD934x）走 slimbus（qcom,tavil-slim-pgd、swr_master）——**slimbus 控制器驱动是全新组件**；speaker amp 型号待 fdt 详查。
- 用户空间：redwood_audio.c /dev sink + audio server 已有。
- 验证：/dev/redwood-audio 写音源 → 喇叭出声；rec_sound 录制链可作"出声"的客观证物。

## 4. 交付（全子项）

- 改动全落 xnu/files（勿直改 xnu-src）；`cp -r xnu/files/* xnu-src/...` + `./xnu-build.sh -j8 -k`；每轮 runNN 取证（logs/kl-runNN*.txt，tr -d '\000'）。
- 每子项完成：PORTING_OP6.md 阶段 10 章节 + HANDOFF 增补 + memory topics.md + git 提交（-c user.name="oxjmo" -c user.email="oxjmo@localhost"）。
