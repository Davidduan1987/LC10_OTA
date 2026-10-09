# LC10_OTA — nRF54LC10A 单槽（single_slot）蓝牙 OTA 示例

基于 **nRF Connect SDK v3.4.0** 的 `nrf/samples/dfu/single_slot` 方案，在 **nRF54LC10 DK**（`nrf54lc10dk/nrf54lc10a/cpuapp`）上为 LBS（LED Button Service）应用增加 BLE OTA（DFU）功能。

## 方案简介：single_slot OTA

传统 MCUboot 是双槽（A/B）：应用占两份空间，一份运行、一份存放新固件。
**single_slot** 只给应用留一个槽（`image-0`），另有一个很小的 **firmware loader**（固件加载器，`image-1`）专门负责升级：

- 正常启动：MCUboot 校验并跳转到 `image-0` 里的应用。
- 进入 DFU 模式：MCUboot 启动 `image-1` 里的 BLE MCUmgr loader（广播名 **`FW loader`**），手机通过 SMP 协议把新的应用直接写进 `image-0`。
- 好处：应用可用空间接近整个 RRAM，不必为升级预留一份和应用等大的备份区。
- 代价：没有回滚备份；升级过程中断电，需要重新进入 DFU 模式再次升级。

本工程由三个镜像组成（由 sysbuild 一起构建）：

| 镜像 | 作用 |
|---|---|
| `mcuboot` | 引导，决定启动应用还是 firmware loader |
| `LC_10dfu`（应用） | LBS 示例应用，含免按键进入 DFU 的逻辑 |
| `ble_mcumgr`（firmware loader） | BLE MCUmgr 升级镜像，广播名 `FW loader` |

## 分区布局（1012 KB RRAM）

定义在 [`boards/nrf54lc10dk_nrf54lc10a_cpuapp.overlay`](boards/nrf54lc10dk_nrf54lc10a_cpuapp.overlay)。

| 分区 | 标签 | 起始地址 | 大小 | 说明 |
|---|---|---|---|---|
| `boot_partition` | mcuboot | `0x000000` | 24 KB | MCUboot |
| `slot0_partition` | image-0 | `0x006000` | 860 KB | 应用 |
| `slot1_partition` | image-1 | `0x0DD000` | 116 KB | BLE firmware loader |
| `storage_partition` | storage | `0x0FA000` | 4 KB | 仅供 MCUmgr 的 storage erase 命令使用 |
| `setting_storage_partition` | setting_storage | `0x0FB000` | 8 KB | Settings（蓝牙绑定信息等），通过 `zephyr,settings-partition` 指定 |

结束地址 `0x0FD000` = 1012 KB，即芯片 RRAM 全部容量。分区都按 4 KB（RRAM 擦除粒度）对齐。

当前各镜像占用：应用约 188 KB / 860 KB，MCUboot 约 22.2 KB / 24 KB，loader 约 99 KB / 116 KB。
**MCUboot 只剩约 1.8 KB 余量**，如果再给 MCUboot 加功能，需要同时调整 `mcuboot` 与 `image-0` 的大小。

## 进入 DFU 的两种方式

1. **按键方式**：按住 **Button 0** 的同时复位（或重新上电），MCUboot 启动时检测到按键，进入 firmware loader。
2. **免按键方式**：应用运行时按 **Button 3**（`sw3`，P0.04）。应用会把「boot mode = bootloader」写进 GPREGRET1（软复位后保留），然后冷复位；MCUboot 读到该标志后清除并进入 firmware loader。
   相关配置：`prj.conf` 的 `CONFIG_RETENTION_BOOT_MODE=y` 等；MCUboot 的 `CONFIG_BOOT_FIRMWARE_LOADER_BOOT_MODE=y`；设备树里的 `zephyr,boot-mode`。
   这一功能约增加应用 1.6 KB、MCUboot 0.4 KB 的 flash。

## 构建与烧录

在 NCS v3.4.0 环境中：

```bash
west build -b nrf54lc10dk/nrf54lc10a/cpuapp --pristine
west flash --erase
```

构建产物：

| 文件 | 用途 |
|---|---|
| `build/dfu_application.zip` | 手机端 OTA 升级包 |
| `build/LC_10dfu/zephyr/zephyr.signed.hex` | 已签名的应用 |

## DFU 测试步骤

准备：手机安装 **nRF Connect Device Manager**（Android / iOS）。

1. 构建并烧录工程，复位后设备正常广播（默认名 `Nordic_LBS`）。
2. 修改一处代码（例如 `main.c` 里 LED 闪烁间隔或 printk 内容）后重新 `west build`，得到新的 `build/dfu_application.zip`，把它拷到手机。
3. **进入 DFU 模式**（任选一种）：
   - 按键方式：按住 Button 0，再按 RESET，等 1~2 秒后松开；
   - 免按键方式：应用运行时按一下 Button 3。
4. 打开 nRF Connect Device Manager，扫描并连接广播名为 **`FW loader`** 的设备（看不到该名称就说明没进入 DFU 模式）。
5. 进入 **Image** 页，选择 `dfu_application.zip`，点击 **Upload / Start**，等待上传完成。
6. 上传完成后在 App 里选择 **Reset**，或者直接按 RESET 复位。
7. 设备重新启动并运行新应用（通过日志或 LED 行为确认新版本生效）。

注意：
- 升级过程中不要断电；中断后重新进入 DFU 模式再升级一次即可。
- 免按键方式没有做长按/防抖，按下立即复位；量产时建议改成长按触发。

## 签名与 KMU

### 默认：不使用 KMU

当前工程默认**关闭 KMU**：公钥直接编进 MCUboot，签名算法为 ED25519（`SB_CONFIG_BOOT_SIGNATURE_TYPE_PURE=y`），烧录时不需要预先写入密钥，开发调试最省事。
默认密钥由 sysbuild 自动生成。

> 开发阶段可以这样用；量产请使用 KMU 或自己管理的私钥，不要用自动生成的示例密钥。

### 如何打开 KMU

1. 编辑 [`sysbuild.conf`](sysbuild.conf)，把

   ```
   SB_CONFIG_MCUBOOT_SIGNATURE_USING_KMU=n
   ```

   改为 `=y`（或直接删掉这一行，`Kconfig.sysbuild` 里对 nRF54L 默认就是 `y`）。

2. 编辑 [`sysbuild/mcuboot/boards/nrf54lc10dk_nrf54lc10a_cpuapp.conf`](sysbuild/mcuboot/boards/nrf54lc10dk_nrf54lc10a_cpuapp.conf)，加上：

   ```
   CONFIG_PSA_CORE_LITE=y
   ```

   （不使用 KMU 时必须去掉这一项，否则链接会找不到 `psa_import_key`；使用 KMU 时需要它来减小体积。）

3. （可选）指定自己的私钥，在 `sysbuild.conf` 里加：

   ```
   SB_CONFIG_BOOT_SIGNATURE_KEY_FILE="/path/to/your_ed25519.pem"
   ```

   不指定则自动生成默认密钥。

4. 重新构建：

   ```bash
   west build -b nrf54lc10dk/nrf54lc10a/cpuapp --pristine
   ```

   构建后 `build/keyfile.json` 会生成，里面包含要写入 KMU 的公钥（slot 242）及元数据。

### 如何把密钥烧进 KMU

**开发阶段（推荐）**：构建目录里存在 `keyfile.json` 时，`west flash` 会在烧录同时自动写入 KMU：

```bash
west flash --erase
# 或者需要恢复/解锁时
west flash --recover
```

注意必须带 `--erase` 或 `--recover`，且 `build/keyfile.json` 要存在，否则密钥不会被写入。

**手动方式**：用 PEM 私钥单独写入：

```bash
west ncs-provision upload -k your_ed25519.pem --keyname UROT_PUBKEY
```

或使用 nRF Util 配合 NCS 的 generate_psa_key_attributes 脚本生成的 JSON 来烧录，详见 NCS 文档 *KMU provisioning*。

**常见现象**：如果开了 KMU 但没有把密钥写进去，MCUboot 验签失败，设备**没有任何输出**（MCUboot 的 console 已关闭）。碰到“烧录后完全无 log”，先检查这一点。
本工程的 KMU 配置在实验时遇到过上述无输出的情况，原因（是否为未烧密钥）没有进一步确认，默认配置是关闭 KMU 并已通过 OTA 测试。

> 生产环境烧录密钥请参考 Nordic 的 *nRF54L Series Production Programming* 文档，并妥善保管私钥，**不要把 `.pem` / `keyfile.json` 提交到仓库**（`.gitignore` 已忽略）。

## 目录结构

```
.
├── boards/nrf54lc10dk_nrf54lc10a_cpuapp.overlay   # 分区布局、settings 分区、boot mode 保留区
├── src/main.c                                     # LBS 应用 + Button 3 进入 DFU
├── prj.conf                                       # 应用配置（含 retention / boot mode）
├── sysbuild.conf / sysbuild.cmake / Kconfig.sysbuild
└── sysbuild/
    ├── mcuboot/                                   # MCUboot 配置（firmware loader 入口）
    └── ble_mcumgr/                                # BLE MCUmgr firmware loader 配置
```

## 参考

- NCS 示例：`nrf/samples/dfu/single_slot`
- NCS 文档：Using firmware loader mode（MCUboot）、KMU provisioning
