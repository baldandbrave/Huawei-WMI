# Huawei-WMI
WMI functions available for Huawei/Honor laptops, basically available on all laptops that has `PCManager.exe` preinstalled.

Date: June 2024

**Use at your own risk, I have no resposibility.**
# TL;DR
**Setting Battery Charging Threshold to 80-85 with Administrator Powershell**

`Get-CimInstance -ClassName 'OemWMIMethod' -Namespace 'root/wmi' | Invoke-CimMethod -MethodName OemWMIfun -Arguments @{u64Input=93803293775107}`

## Readings & Tools
[Linux Driver](https://github.com/aymanbagabas/Huawei-WMI)

[Huawei Battery Control](https://github.com/AceDroidX/HuaweiBatteryControl)  [(its doc in zh-cn)](https://blog.acedroidx.top/HuaweiBatteryControl/#%E5%B0%9D%E8%AF%95%E8%B0%83%E7%94%A8)

[WMI Explorer](https://github.com/vinaypamnani/wmie2)

IDA
## DLLs
WmiUtil.dll is used by `PCManager.exe` to talk to EC. Some functions with its ordinal in dll and opcode listed below.

<details>
    <summary>WmiUtils.dll</summary>

| Name                                                                                             | Ordinal | OpCode(hex) |
| ------------------------------------------------------------------------------------------------ | ------- | ----------- |
| BrightnessAdjust(char)                                                                           | 11      | 2306        |
| GetAcDurationTime(int &)                                                                         | 15      | 1403        |
| GetBiosInfo(HwSDK::WmiInputParams &,HwSDK::WmiOutputParams &)                                    | 16      |             |
| GetBiosLogoAddressV2(uint &,int &)                                                               | 17      |             |
| GetCellBatteryVoltage(HwSDK::BatteryVoltageMode,int &,int)                                       | 18      |             |
| GetDeviceBiosSwitchStatus(HwSDK::DeviceBiosSwitchNumber,bool &)                                  | 21      |             |
| GetDeviceBiosSwitchStatus(HwSDK::DeviceBiosSwitchNumber,int &)                                   | 20      |             |
| GetDpStatus(int &)                                                                               | 22      | 1b06        |
| GetFanSpeed(int &,HwSDK::FanSpeedLocateNumber)                                                   | 24      | 0802        |
| GetFnReverseStatus(bool &)                                                                       | 25      | 0604        |
| GetKeyboardLightTime(ushort &)                                                                   | 27      | 1206        |
| GetNextBootType(uchar &)                                                                         | 29      | 0706        |
| GetOsLockStatus(bool &)                                                                          | 30      | 1006        |
| GetPdFirmwareVersion(std::basic_string<char,std::char_traits<char>,std::allocator<char>> &)      | 31      | 0e06        |
| GetResumeChargingFlag(bool &)                                                                    | 41      | 1303        |
| GetSensorTemperature(int &,HwSDK::QuerySensorTemperatureIndex)                                   | 42      | 0202        |
| GetSmartChargeMode(HwSDK::SmartChargeMode &)                                                     | 43      | 1603        |
| GetTurboMode(bool &)                                                                             | 45      | 0e04        |
| GetVoltageOrCurrent(int &,HwSDK::VoltageCurrentLocateNumber)                                     | 46      | 0902        |
| GetWorkingMode(uchar &)                                                                          | 47      | 1004        |
| ReadMemoryTrainingData(HwSDK::TagMTData &)                                                       | 48      | 1306        |
| ReadPDFirmwareVersion(std::basic_string<char,std::char_traits<char>,std::allocator<char>> &,int) | 49      | 2506        |
| ReadScalerFirmwareVersion(std::basic_string<char,std::char_traits<char>,std::allocator<char>> &) | 50      | 1e06        |
| ReadStabilityLog(HwSDK::TagWmiLogStruct &)                                                       | 51      | 0406        |
| ResetTouchPanel(ulong)                                                                           | 52      | 5a06        |
| SetBatteryChargeThreshlod(uchar,uchar)                                                           | 53      | 1003        |
| SetBiosLogoV2(HwSDK::SetBiosLogoValue)                                                           | 54      | 0106        |
| SetDpStatus(int)                                                                                 | 55      | 1c06        |
| SetFnReverseStatus(bool)                                                                         | 56      | 0704        |
| SetKeyboardLightTime(ushort)                                                                     | 57      | 1106        |
| SetNextBootType(HwSDK::NextBootType)                                                             | 58      | 0806        |
| SetResumeChargingFlag(bool)                                                                      | 65      | 1203        |
| SetSmartChargeMode(HwSDK::SmartChargeMode,bool,int,int,int)                                      | 66      | 1503        |
| SetTurboMode(bool)                                                                               | 67      | 0f04        |
| SetWorkingMode(uchar)                                                                            | 68      | 1104        |
| UpdatePdFirmwareVersion(void)                                                                    | 71      | 0d06        |
</details>

HardwareHal.dll calls `GetVolageOrCurrent` from WmiUtil.dll. More details in Detail section.

## Trials
1. Calling functions through ordinal in dll

    This should be the most robust method since code is always provided by manufacturer.
    The easiest way to do this is via csharp `DllImport`. Also it's easy to try via `csi.exe` REPL environment.
    <details>
        <summary>Example</summary>
    
    ```csharp
    using System;
    using System.Runtime.InteropServices;

    path = 'full_path_to_dll'
    // use stdcall or cdecl as decompiled code suggests
    [DllImport(path, CallingConvention = CallingConvention.Cdecl, EntryPoint = "#Num")]
    // function type inferred by IDA
    public static extern Int64 SetSmartChargeMode(int mode, char smart, char smartparam, char low, char high);
    ```
    </details>
    However none of the methods could be used. Decompiling shows 2 arguments used by WMI functions are not initialized, which means they are not static methods, and should be initialized before using.
2. Calling WMI functions through powershell
    
    Method `Get-WmiObject` used by [HBC doc](https://blog.acedroidx.top/HuaweiBatteryControl/#%E5%B0%9D%E8%AF%95%E8%B0%83%E7%94%A8) essentially do the same thing as the `wmic.exe` cli, which is [deprecated  since Windows 10 21H1.](https://learn.microsoft.com/en-us/windows/win32/wmisdk/wmic)

    However, Cim-related commands are still available. Provided by WMI explorer, `OemWMIfun` is not a static method. A Cim instance is required to call this function.Here is a working example command.(Requires sudo)
    
    `Get-CimInstance -ClassName 'OemWMIMethod' -Namespace 'root/wmi' | Invoke-CimMethod -MethodName OemWMIfun -Arguments @{u64Input=FULL_COMMAND}`
## Quirks
1. According to HBC doc, input parameter of `OemWMIfun` is `u8Input`, which is a pointer to a uint64 command. Since WmiUtil.dll version 11, input param is now `u64Input`, much easier to call `OemWMIfun`.

2. Accoring to Linux Driver, 1st and 2nd argument of `SetSmartChargeMode` are unknown. Decompiled code shows 2nd argument is `0x48` when 1st argument selected from `[1,2,3]`. The most possible interpretation should be the default charging profiles on windows and the remaining battery longevity.

3. Calling `SetResumeChargingFlag` doesn't change anything on my laptop. 3rd byte from this function should be `1/2` instead of `0/1`. Mapping `{1:1,2:0}` is observed.

## Command Format
Little endian, use as uint64
```
0x0     0x7    0xF    
+-------+------+------+--+
|Command|Param1|Param2|...
+-------+------+------+--+
```
Set 80-85 threshold: 0x`55 50 48 01 1503` => `93803293775107`


`u8Output[]` result. First byte {`1`:error/not supported,`0`:success}
```
0x0             0x7
+---------------+-------+-------+--+
|State/Supported|result1|result2|...
+---------------+-------+-------+--+
```
<details>
    <summary>Other function args format by tryout</summary>

| Name                                                           | Ordinal | OpCode(hex) | 3rd byte | 4th byte  |
| -------------------------------------------------------------- | ------- | ----------- | -------- | --------- |
| GetFanSpeed(int &,HwSDK::FanSpeedLocateNumber)                 | 24      | 0802        | 0/1      | None      |
| GetSensorTemperature(int &,HwSDK::QuerySensorTemperatureIndex) | 42      | 0202        | 0?       | None      |
| GetVoltageOrCurrent(int &,HwSDK::VoltageCurrentLocateNumber)   | 46      | 0902        | \*\*     |           |
| SetFnReverseStatus(bool)                                       | 56      | 0704        | 0/1      | None      |
| SetKeyboardLightTime(ushort)                                   | 57      | 1106        | timelo   | timehi    |
| SetNextBootType(HwSDK::NextBootType)                           | 58      | 0806        | byte     |           |
| SetResumeChargingFlag(bool)                                    | 65      | 1203        | 1/2      | None      |
| SetSmartChargeMode(HwSDK::SmartChargeMode,bool,int,int,int)    | 66      | 1503        | 1/2/3    | 72/custom |
| SetTurboMode(bool)                                             | 67      | 0f04        | 0/1      | None      |
| SetWorkingMode(uchar)                                          | 68      | 1104        | byte     | None      |

From `HardwareHal.dll`, `GetVoltageOrCurrent` has following mapping.
| Func            | arg(dec) |
| --------------- | -------- |
| CurrentBattery0 | 48       |
| CurrentBattery1 | 49       |
| CurrentUSB1     | 16       |
| VoltageBattery0 | 32       |
| VoltageBattery1 | 33       |
| VoltageUSB0     | 0        |
| VoltageUSB1     | 1        |
</details>
