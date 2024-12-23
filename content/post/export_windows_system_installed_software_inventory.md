---
date: 2024-12-18 10:57:10
title: 通过命令行导出 Windows 系统安装软件清单
tags:
  - "Windows"
  - "PowerShell"
draft: false
---

通过 PowerShell 命令导出 Windows 系统安装软件清单。

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

直接通过 PowerShell 执行以下命令即可：

```powershell
Invoke-Expression (new-object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/mosserlee/PSTips/master/Functions/Get-InstalledSoftwares.ps1')
Get-InstalledSoftwares | Export-Csv -Path 'D:\softwarelist.csv' -NoTypeInformation -Encoding UTF8
```

`Export-Csv` 的用法参考了[这里](https://www.hantaosec.com/3764.html)，输出 CSV 文件然后保存在 D 盘之中，具体的执行内容可以参考[源文件](https://github.com/mosserlee/PSTips/blob/master/Functions/Get-InstalledSoftwares.ps1)：

```powershell
<#
.Synopsis
   Get installed software list by retrieving registry.
.DESCRIPTION
   The function return a installed software list by retrieving registry from below path;
   1.'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall'
   2.'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall'
   3.'HKLM:SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall'
   Author: Mosser Lee (http://www.pstips.net/author/mosser/)

.EXAMPLE
   Get-InstalledSoftwares
.EXAMPLE
   Get-InstalledSoftwares  | Group-Object Publisher
#>
function Get-InstalledSoftwares
{
    #
    # Read registry key as product entity.
    #
    function ConvertTo-ProductEntity
    {
        param([Microsoft.Win32.RegistryKey]$RegKey)
        $product = '' | select Name,Publisher,Version
        $product.Name =  $_.GetValue("DisplayName")
        $product.Publisher = $_.GetValue("Publisher")
        $product.Version =  $_.GetValue("DisplayVersion")

        if( -not [string]::IsNullOrEmpty($product.Name)){
            $product
        }
    }

    $UninstallPaths = @(,
    # For local machine.
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall',
    # For current user.
    'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall')

    # For 32bit softwares that were installed on 64bit operating system.
    if([Environment]::Is64BitOperatingSystem) {
        $UninstallPaths += 'HKLM:SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall'
    }
    $UninstallPaths | foreach {
        Get-ChildItem $_ | foreach {
            ConvertTo-ProductEntity -RegKey $_
        }
    }
}
```

实际上是通过对卸载软件的注册表内容进行遍历，获取需要的信息。
