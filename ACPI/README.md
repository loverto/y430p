# SSDT-EC 仿冒补丁

## 描述

OpenCore 要求不要改变 EC 控制器名称，但是为了加载 USB 电源管理，可能需要仿冒另一个 EC。

## 使用说明

DSDT中搜索 `PNP0C09`，查看其所属设备名称。如果名称不是 `EC`，使用本补丁；如果是 `EC`，忽略本补丁。

Y430p 在DSDT中搜索后发现设备名称不是 EC ，叫做EC0

[SSDT-EC](Image/SSDT-EC.jpg)


## 注意

- 如果搜索到多个 `PNP0C09`，应确认真实有效的 `PNP0C09` 设备。
- 补丁里使用了 `LPCB`，非 `LPCB` 的请自行修改补丁内容。

什么是LPCB ?


# 禁用EHCx

什么是EHCx

增强主机控制器接口手动关闭/启用/禁用

什么是XHCI

XHCI英文全称eXtensible Host Controller Interface,是一种可扩展的主机控制器接口,是Intel开发的USB主机控制器。

## 描述

下列情况之一需要禁用 EHC1 和 EHC2 总线：

- ACPI 包含 EHC1 或者 EHC2，而机器本身不存在相关硬件。
- ACPI 包含 EHC1 或者 EHC2，机器存在相关硬件但并没有实际输出端口（外置和内置）。

## 补丁

- ***SSDT-EHC1_OFF***：禁用 `EHC1`。
- ***SSDT-EHC2_OFF***：禁用 `EHC2`。
- ***SSDT-EHCx_OFF***：是 ***SSDT-EHC1_OFF*** 和 ***SSDT-EHC2_OFF*** 的合并补丁。

## 使用方法

- 优先 BIOS 设置：`XHCI Mode` = `Enabled`。
- 如果 BIOS 没有 `XHCI Mode`选项，同时符合 **描述** 的情况之一，使用上述补丁。

### 注意事项

- 适用于 7, 8, 9 系机器，且 macOS 是 10.11 以上版本。
- 对于 7 系机器，***SSDT-EHC1_OFF*** 和 ***SSDT-EHC2_OFF*** 二者不可同时使用。
- 补丁在 `Scope (\)`   下添加了 `_INI` 方法，如果和其他补丁的 `_INI` 发生重复，应合并 `_INI` 里的内容。

# 添加缺失的部件

## 描述

添加缺失的部件只是一种完善方案，非必要！

### 使用说明

**DSDT中:**

- 搜索 `PNP0200`，如果缺失，可添加 ***SSDT-DMAC***。

- 搜索 `MCHC`，如果缺失，可添加 ***SSDT-MCHC***。

- 搜索 `PNP0C01`，如果缺失，可添加 ***SSDT-MEM2***。

- 搜索 `0x00160000`，如果缺失，可添加 ***SSDT-IMEI***。

- 6 代以上机器，搜索 `0x001F0002`，如果缺失，可添加 ***SSDT-PPMC***。

- 6 代以上机器，搜索 `PMCR` 、 `APP9876`，如果缺失，可添加 ***SSDT-PMCR***。

  说明：@请叫我官人 提供方法，目前已成为 OpenCore 官方的 SSDT 示例。
  > Z390 芯片组 PMC (D31:F2) 只能通过 MMIO 启动。由于 ACPI 规范中没有 PMC 设备，苹果推出了自己的命名 `APP9876`、从 AppleIntelPCHPMC 驱动中访问这个设备。而在其它操作系统中，一般会使用 `HID: PNP0C02`、`UID: PCHRESV` 访问这个设备。  
  > 包括 APTIO V 在内的平台，在初始化 PMC 设备之前不能读写 NVRAM（在 SMM 模式中被冻结）。  
  > 虽然不知道为什么会这样，但是值得注意的是 PMC 和 SPI 位于不同的内存区域，PCHRESV 同时映射了这两个区域，但是苹果的 AppleIntelPCHPMC 只会映射 PMC 所在的区域。  
  > PMC 设备与 LPC 总线之间毫无关系，这个 SSDT 纯粹是为了加快 PMC 的初始化而把该设备添加到 LPC 总线下。如果将其添加到 PCI0 总线中、PMC 只会在 PCI 配置结束后启动，对于需要读取 NVRAM 的操作来说就已经太晚了。

- 搜索 `PNP0C0C`，如果缺失，可添加 ***SSDT-PWRB***。

- 搜索 `PNP0C0E`，如果缺失，可添加 ***SSDT-SLPB***，《PNP0C0E睡眠修正方法》需要这个部件。

### 注意

使用以上部分补丁时，注意 `LPCB` 名称应和原始ACPI名称一致。

# 注入X86

## 描述

注入 X86，实现 CPU 电源管理。

## 使用说明

- DSDT中搜索 `Processor`，如：

  ```Swift
      Scope (_PR)
      {
          Processor (CPU0, 0x01, 0x00001810, 0x06){}
          Processor (CPU1, 0x02, 0x00001810, 0x06){}
          Processor (CPU2, 0x03, 0x00001810, 0x06){}
          Processor (CPU3, 0x04, 0x00001810, 0x06){}
          Processor (CPU4, 0x05, 0x00001810, 0x06){}
          Processor (CPU5, 0x06, 0x00001810, 0x06){}
          Processor (CPU6, 0x07, 0x00001810, 0x06){}
          Processor (CPU7, 0x08, 0x00001810, 0x06){}
      }
  ```

  根据查询结果，选择注入文件 ***SSDT-PLUG-_PR.CPU0***

  再如：

  ```Swift
      Scope (_SB)
      {
          Processor (PR00, 0x01, 0x00001810, 0x06){}
          Processor (PR01, 0x02, 0x00001810, 0x06){}
          Processor (PR02, 0x03, 0x00001810, 0x06){}
          Processor (PR03, 0x04, 0x00001810, 0x06){}
          Processor (PR04, 0x05, 0x00001810, 0x06){}
          Processor (PR05, 0x06, 0x00001810, 0x06){}
          Processor (PR06, 0x07, 0x00001810, 0x06){}
          Processor (PR07, 0x08, 0x00001810, 0x06){}
          Processor (PR08, 0x09, 0x00001810, 0x06){}
          Processor (PR09, 0x0A, 0x00001810, 0x06){}
          Processor (PR10, 0x0B, 0x00001810, 0x06){}
          Processor (PR11, 0x0C, 0x00001810, 0x06){}
          Processor (PR12, 0x0D, 0x00001810, 0x06){}
          Processor (PR13, 0x0E, 0x00001810, 0x06){}
          Processor (PR14, 0x0F, 0x00001810, 0x06){}
          Processor (PR15, 0x10, 0x00001810, 0x06){}
      }
  ```

  根据查询结果，选择注入文件：***SSDT-PLUG-_SB.PR00***

- 如果查询结果和补丁文件名 **无法对应** ，请任选一个文件作为样本，自行修改补丁文件相关内容。

## 注意

2 代，3 代机器无需注入 X86。


# SSDT-SBUS(SMBU) 补丁

## 设备名称

DSDT 中搜索 `0x001F0003` (6 代以前) 或者 `0x001F0004` (6 代及以后)，查看其所属设备名称。

## 补丁

- 设备名称是 `SBUS`，使用 ***SSDT-SBUS***
- 设备名称是 `SMBU`，使用  ***SSDT-SMBU***
- 设备名称是其他名称的，自行修改补丁相关内容

## 备注

TP机器多为 `SMBU`。


# 声卡 IRQ 补丁

## 描述

- 早期机器的声卡要求部件 **HPET** **`PNP0103`** 提供中断号 `0` & `8`，否则声卡不能正常工作。实际情况几乎所有机器的 **HPET** 未提供任何中断号。通常情况下，中断号 `0` & `8` 分别被 **RTC** **`PNP0B00`**、 **TIMR** **`PNP0100`** 占用。
- 解决上述问题需同步修复 **HPET**、**RTC**、**TIMR**。

## 补丁原理

- 禁用 **HPET**、**RTC**、**TIMR** 三部件。
- 仿冒三部件，即：**HPE0**、**RTC0**、**TIM0**。
- 将 **RTC0** 的 `IRQNoFlags (){8}` 和 **TIM0** 的 `IRQNoFlags (){0}` 移除并添加到 **HPE0**。

## 补丁方法

- 禁用 **HPET**、**RTC**、**TIMR**
  - **HPET**
  
    通常HPET存在 `_STA` ，因此，禁用HPET需使用《预置变量法》。如：
  
    ```Swift
    External (HPAE, IntObj) /* 或者 External (HPTE, IntObj) */
    Scope (\)
    {
        If (_OSI ("Darwin"))
        {
            HPAE =0 /* 或者 HPTE =0 */
        }
    }
    ```
  
    注意： `_STA` 内 `HPAE` 变量随机器不同可能不同。
  
  - **RTC**  
  
    早期机器的 RTC 无 `_STA`，按 `Method (_STA,` 方法禁用 RTC。如：
  
    ```Swift
    Method (_STA, 0, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            Return (0)
        }
        Else
        {
            Return (0x0F)
        }
    }
    ```
  
  - **TIMR**
  
    同 **RTC**
  
- 补丁文件：***SSDT-HPET_RTC_TIMR-fix***

  见上述 **补丁原理** ，参考示例。

## 注意事项

- 本补丁不可以和下列补丁同时使用：
  - 《二进制更名与预置变量》的 ***SSDT-RTC_Y-AWAC_N***
  - OC 官方的 ***SSDT-AWAC***
  - 《仿冒设备》或者 OC 官方的 ***SSDT-RTC0***
  - 《CMOS重置补丁》的 ***SSDT-RTC0-NoFlags***
- `LPCB` 名称、 **三部件** 名称以及 `IPIC` 名称应和原始`ACPI`部件名称一致。
- 若三合一补丁无法解决，在使用三合一补丁的前提下尝试 ***SSDT-IPIC***。按照上文禁用 ***HPET***、***RTC*** 和 ***TIMR*** 的方法禁用 ***IPIC*** 设备，然后仿冒一个 ***IPI0*** 设备，设备内容为原始 `DSDT` 中的 ***IPIC*** 或 ***PIC*** 设备内容，最后将 `IRQNoFlags{2}` 删除即可，参考示例。






