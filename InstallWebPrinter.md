# 用PowerShell安装打印机

    目前网上的资料，大致分三种：
    - wmic工具（win11中已弃用）
    - pnputil工具（较繁琐）
    - cim工具（推荐）

## wmic工具

1. 添加端口
1. 添加驱动，完成

```batch {.line-numbers}
@echo off
pushd "%~dp0"
REM 删除旧打印机
wmic printer where "Caption like '%%10.240.0.25%%'" delete
wmic printer where "Caption like '%%楼打印机%%'" delete
REM 创建端口
echo 尝试创建打印机 TCP/IP 端口： 10.245.13.215
cscript C:\Windows\System32\Printing_Admin_Scripts\zh-CN\prnport.vbs -a -r 10.245.13.215 -h 10.245.13.215 -o raw
echo 打印机 TCP/IP 端口 10.245.13.215 创建成功！
REM 通过INF文件安装驱动
echo 加载打印机驱动： 3楼打印机
rundll32 printui.dll,PrintUIEntry /if /b "3楼打印机" /r "10.245.13.215" /m "FF Apeos 3060 PCL 6" /z /f "%cd%\\Software\PCL\amd64\Common\001\FFMO3PCLA.INF"
echo 打印机驱动加载完毕
echo done...
```

## pnputil工具

1. 将所有驱动文件加载至 C:\Windows\System32\DriverStore
1. 添加端口
1. 添加打印机驱动
1. 添加打印机，完成

```powershell {.line-numbers}
#删除旧驱动，先删printerdriver
remove-printerdriver -name "FF Apeos 3060 PCL 6"
#再删windowsdriver,但pnputil 删驱动只能删以oem*.inf命名方式的驱动，先查驱动的文件名
$win_drivers = Get-WindowsDriver -Online -all
$win_drivers | ?{$_.providename -like "*fujifilm*" -and $_.classname -like "*printer*"}
#查到驱动名是oem6.inf，用pnputil删除
pnputil /delete-driver oem6.inf
#添加WindowsDriver,验证一下
pnputil /add-driver 'E:\target.inf'
get-PrinterDriver
#添加端口
add-PrinterPort -Name "10.245.13.215" -PrinterHostAddress "10.245.13.215" -PortNumber "9100"
#添加PrinerDriver,取INF文件里面找名字
add-PrinterDriver "FF Apeos 3060 PCL 6"
#添加打印机，验证一下
add-Printer -NAME "Printer_3F" -DriverName "FF Apeos 3060 PCL 6" -PortName "10.245.13.215"
get-Printer
```

## CIM工具

1. InvokeCimMethod 创建打印机驱动
1. 添加端口
1. 添加打印机
1. 打印测试页

``` powershell {.line-numbers}
#查看cim类所有的方法
(Get-CimClass -classname "Win32_PrinterDriver").CimClassMethods
#查看方法的参数,得到该方法的参数名为DriverInfo
(Get-CimClass -classname "Win32_PrinterDriver").CimClassMethods['AddPrinterDriver'].Parameters
#构造一个win32_printerdriver实例
$driver = New-CimInstance -ClassName Win32_PrinterDriver -Property @{Name="FF Apeos 3060 PCL 6";InfName = "'$home'\desktop\new\3060\ffmo3pcla.inf"} -ClientOnly
#构造一个哈希表,用来传参
$para = [System.Collections.IDictionary]@{}
$para.add("DriverInfo",$driver)
#用InvokeCimMethod命令 创建打印机驱动
invoke-CimMethod -ClassName Win32_PrinterDriver -MethodName AddPrinterDriver -Arguments $para
#添加端口
add-PrinterPort -Name "10.245.13.215" -PrinterHostAddress "10.245.13.215" -PortNumber "9100"
#添加打印机，验证一下
add-Printer -NAME "Printer_3F" -DriverName "FF Apeos 3060 PCL 6" -PortName "10.245.13.215"
get-Printer
#打印测试页
Invoke-CimMethod -InputObject (Get-CimInstance -ClassName "Win32_printer")[0] -MethodName PrintTestPage
```
