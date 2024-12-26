# DeletePrograms
已知：
1.  get-package在powershell 7无法正常工作(win10)
1.  Get-AppxPackage powershell 7无法正常工作(win10)
因此，使用注册表来删除程序

```powershell 
$Uninstall_path = 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall'
$res = dir $Uninstall_path #dir -> get-item
#显示软件列表
$res | select @{n = 'name'; e = { $_.name.split("\")[-1] } }, @{n = 'DisplayName'; e = { $_.getvalue("displayname") } }
#运行删除程序
Start-Process ($res | ? { $_.getvalue("displayname") -eq "哔哩哔哩" }).getvalue("UninstallString")
```