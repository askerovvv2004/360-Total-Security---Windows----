# 360 Total Security на Windows Удаления польностью
Скачиваете скип и запускаете и все, если что можно пунк 7 убрать "<#" И "#>" для очистки эскизов 

Код

<#
.SYNOPSIS
    Полная очистка системы от остатков 360 Total Security.
.DESCRIPTION
    - Удаляет папки 360 из Program Files, ProgramData, AppData.
    - Удаляет ключи реестра 360, включая пункт в меню Корзины.
    - Удаляет службы 360 и записи автозагрузки.
    - Очищает временные файлы, корзину, кэш эскизов.
.NOTES
    Требует прав администратора. Автоматически создаёт резервную копию реестра.
#>

# Запрос прав администратора
if (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Host "Запустите PowerShell от имени администратора!" -ForegroundColor Red
    exit 1
}

Write-Host "=== ПОЛНАЯ ОЧИСТКА ПОСЛЕ 360 TOTAL SECURITY ===" -ForegroundColor Cyan

# --- 0. Резервная копия реестра ---
$backupPath = "$env:USERPROFILE\Desktop\RegistryBackup_360_$(Get-Date -Format 'yyyyMMdd_HHmmss').reg"
Write-Host "Создаю резервную копию реестра: $backupPath" -ForegroundColor Yellow
Start-Process -FilePath "regedit.exe" -ArgumentList "/e `"$backupPath`"" -Wait -NoNewWindow
Write-Host "Резервная копия создана." -ForegroundColor Green

# --- 1. Удаление папок 360 (расширенный список) ---
$pathsToDelete = @(
    "$env:ProgramFiles\360",
    "${env:ProgramFiles(x86)}\360",
    "$env:ProgramData\360safe",
    "$env:ProgramData\360TotalSecurity",
    "$env:LOCALAPPDATA\360TotalSecurity",
    "$env:APPDATA\360TotalSecurity",
    "$env:LOCALAPPDATA\360Safe",
    "$env:APPDATA\360Safe",
    "$env:ProgramData\Qihoo",
    "$env:LOCALAPPDATA\Qihoo",
    "$env:APPDATA\Qihoo"
)

foreach ($path in $pathsToDelete) {
    if (Test-Path $path) {
        Write-Host "Удаляю папку: $path" -ForegroundColor Yellow
        Remove-Item -Path $path -Recurse -Force -ErrorAction SilentlyContinue
    } else {
        Write-Host "Папка не найдена: $path" -ForegroundColor Gray
    }
}

# --- 2. Удаление ключей реестра 360 (включая пункт Корзины) ---
$regPaths = @(
    "HKCU:\Software\360",
    "HKLM:\SOFTWARE\360",
    "HKLM:\SOFTWARE\WOW6432Node\360",
    "HKCU:\Software\Qihoo",
    "HKLM:\SOFTWARE\Qihoo",
    "HKLM:\SOFTWARE\WOW6432Node\Qihoo",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\MenuOrder\Start Menu\Programs\360 Total Security",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\360*",
    # Специальный ключ для меню Корзины
    "Registry::HKEY_CLASSES_ROOT\CLSID\{645FF040-5081-101B-9F08-00AA002F954E}\shell\LОчистка"
)

foreach ($regPath in $regPaths) {
    if (Test-Path $regPath) {
        Write-Host "Удаляю ключ реестра: $regPath" -ForegroundColor Yellow
        Remove-Item -Path $regPath -Recurse -Force -ErrorAction SilentlyContinue
    } else {
        Write-Host "Ключ не найден: $regPath" -ForegroundColor Gray
    }
}

# --- 3. Удаление служб 360, QH, QHSafe, QHActiveDefense ---
$servicePatterns = @("360", "QH", "QHSafe", "QHActiveDefense", "Zhudongfangyu")
$services = Get-Service | Where-Object { 
    $name = $_.Name
    $display = $_.DisplayName
    foreach ($pattern in $servicePatterns) {
        if ($name -like "*$pattern*" -or $display -like "*$pattern*") { return $true }
    }
    return $false
}

if ($services) {
    Write-Host "Найдены потенциальные службы 360:" -ForegroundColor Yellow
    $services | ForEach-Object { Write-Host "  - $($_.Name) : $($_.DisplayName)" }
    Write-Host "Останавливаю и удаляю..." -ForegroundColor Yellow
    $services | ForEach-Object {
        Stop-Service $_.Name -Force -ErrorAction SilentlyContinue
        sc.exe delete $_.Name
    }
} else {
    Write-Host "Служб 360 не найдено." -ForegroundColor Gray
}

# --- 4. Очистка автозагрузки ---
$runPaths = @(
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
)

foreach ($runPath in $runPaths) {
    if (Test-Path $runPath) {
        $values = Get-ItemProperty -Path $runPath
        foreach ($valueName in $values.PSObject.Properties.Name) {
            if ($valueName -match "360|qihoo|total security|zhudong") {
                Write-Host "Удаляю автозагрузку: $runPath\$valueName" -ForegroundColor Yellow
                Remove-ItemProperty -Path $runPath -Name $valueName -Force -ErrorAction SilentlyContinue
            }
        }
    }
}

# --- 5. Очистка временных файлов ---
Write-Host "`n=== Очистка временных файлов ===" -ForegroundColor Cyan

$tempFolders = @(
    "$env:TEMP",
    "$env:WINDIR\Temp",
    "C:\Windows\Prefetch"
)

foreach ($folder in $tempFolders) {
    if (Test-Path $folder) {
        Write-Host "Очищаю: $folder" -ForegroundColor Yellow
        Get-ChildItem -Path $folder -Recurse -Force -ErrorAction SilentlyContinue | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
    }
}

# --- 6. Очистка корзины ---
Write-Host "Очищаю корзину..." -ForegroundColor Yellow
$Recycler = New-Object -ComObject Shell.Application
$Recycler.NameSpace(0x0a).Items() | ForEach-Object { Remove-Item $_.Path -Recurse -Force -ErrorAction SilentlyContinue }

<# --- 7. Очистка кэша эскизов ---
Write-Host "Очищаю кэш эскизов..." -ForegroundColor Yellow
ie4uinit.exe -ClearIconCache
Remove-Item -Path "$env:LOCALAPPDATA\Microsoft\Windows\Explorer\thumbcache_*.db" -Force -ErrorAction SilentlyContinue
#>

# --- 8. Запуск стандартной очистки диска ---
Write-Host "Запускаю очистку диска (cleanmgr)..." -ForegroundColor Yellow
Start-Process -FilePath "cleanmgr.exe" -ArgumentList "/verylowdisk" -Wait -NoNewWindow

Write-Host "`n=== ОЧИСТКА ЗАВЕРШЕНА! ===" -ForegroundColor Green
Write-Host "Резервная копия реестра сохранена на рабочем столе: $backupPath" -ForegroundColor Cyan
Write-Host "Перезагрузите компьютер для полного применения изменений." -ForegroundColor Yellow
