Service tidak bisa start karena ada masalah dengan konfigurasi atau service itu sendiri. Mari kita troubleshoot step by step:

## **LANGKAH 1: CHECK KONFIGURASI FILE**

```powershell
# Cek apakah config file ada
Test-Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# Lihat isi config file
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# Cek syntax JSON valid
try {
    $config = Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" | ConvertFrom-Json
    Write-Host "✓ JSON syntax valid" -ForegroundColor Green
} catch {
    Write-Host "✗ JSON syntax invalid: $_" -ForegroundColor Red
}
```

## **LANGKAH 2: TEST KONFIGURASI DENGAN AGENT CONTROL**

```powershell
# Navigasi ke folder agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Test konfigurasi dengan control script
.\amazon-cloudwatch-agent-ctl.ps1 -a config
```

## **LANGKAH 3: CHECK SERVICE DETAILS**

```powershell
# Cek detail service
sc.exe queryex "AmazonCloudWatchAgent"

# Cek konfigurasi service
sc.exe qc "AmazonCloudWatchAgent"

# Cek apakah service ada
Get-Service -Name "AmazonCloudWatchAgent" -ErrorAction SilentlyContinue
```

## **LANGKAH 4: FIX SERVICE - METHOD 1 (Uninstall/Reinstall)**

```powershell
# Stop service jika masih ada
Stop-Service -Name "AmazonCloudWatchAgent" -Force -ErrorAction SilentlyContinue

# Uninstall service
sc.exe delete "AmazonCloudWatchAgent"

# Tunggu
Start-Sleep -Seconds 5

# Reinstall service dari config
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent-ctl.ps1 -a install
```

## **LANGKAH 5: FIX SERVICE - METHOD 2 (Manual Config)**

### **5.1 Buat Config yang SUPER SIMPLE:**
```powershell
# Buat config paling sederhana
$superSimpleConfig = @'
{
  "metrics": {
    "metrics_collected": {
      "Processor": {
        "measurement": [
          "% Processor Time"
        ]
      }
    }
  }
}
'@

$superSimpleConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

### **5.2 Uninstall dan Install Ulang Service:**
```powershell
# Navigasi ke folder agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Uninstall service
.\amazon-cloudwatch-agent-ctl.ps1 -a stop
.\amazon-cloudwatch-agent-ctl.ps1 -a uninstall

# Tunggu
Start-Sleep -Seconds 3

# Install ulang
.\amazon-cloudwatch-agent-ctl.ps1 -a install

# Start
.\amazon-cloudwatch-agent-ctl.ps1 -a start
```

## **LANGKAH 6: FIX SERVICE - METHOD 3 (Windows Service Fix)**

```powershell
# Cek dependencies
Get-Service -Name "AmazonCloudWatchAgent" -DependentServices

# Repair service dengan PowerShell
$service = Get-WmiObject -Class Win32_Service -Filter "Name='AmazonCloudWatchAgent'"
if ($service) {
    $service.ChangeStartMode("Automatic")
    $service.StartService()
}

# Atau coba start via sc.exe
sc.exe start "AmazonCloudWatchAgent"
```

## **LANGKAH 7: CHECK EVENT LOGS UNTUK ERROR DETAIL**

```powershell
# Cek Windows Event Logs untuk error detail
Get-WinEvent -LogName "Application" -MaxEvents 20 | 
Where-Object {$_.ProviderName -like "*CloudWatch*" -or $_.Message -like "*CloudWatch*"} | 
Select-Object TimeCreated, LevelDisplayName, ProviderName, Message | 
Format-Table -AutoSize

# Cek System logs juga
Get-WinEvent -LogName "System" -MaxEvents 20 | 
Where-Object {$_.Message -like "*CloudWatch*" -or $_.Message -like "*Amazon*"} | 
Select-Object TimeCreated, LevelDisplayName, ProviderName, Message | 
Format-Table -AutoSize
```

## **LANGKAH 8: ALTERNATIF - CLEAN INSTALL**

```powershell
# 1. Stop and disable service
Stop-Service -Name "AmazonCloudWatchAgent" -Force -ErrorAction SilentlyContinue
Set-Service -Name "AmazonCloudWatchAgent" -StartupType Disabled -ErrorAction SilentlyContinue

# 2. Delete config files
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Force -ErrorAction SilentlyContinue
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\*" -Force -ErrorAction SilentlyContinue

# 3. Create VERY basic config
$basicConfig = @'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": []
      }
    }
  }
}
'@

$basicConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# 4. Try to configure with wizard
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent-config-wizard.exe
```

## **LANGKAH 9: QUICK FIX - USE DEFAULT CONFIG**

```powershell
# Copy default config dari installation folder
Copy-Item "C:\Program Files\Amazon\AmazonCloudWatchAgent\config.json" "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Force

# Start service
Start-Service -Name "AmazonCloudWatchAgent" -ErrorAction SilentlyContinue

# Jika masih error, coba via net command
net start AmazonCloudWatchAgent
```

## **LANGKAH 10: MANUAL START PROCESS**

```powershell
# Coba start process manual tanpa service
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Start agent langsung
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# Atau dengan control script mode console
.\amazon-cloudwatch-agent-ctl.ps1 -a run
```

## **LANGKAH 11: VERIFIKASI PATH DAN PERMISSIONS**

```powershell
# Cek permissions pada folder
Get-Acl "C:\ProgramData\Amazon\AmazonCloudWatchAgent" | Format-List

# Cek apakah ada di PATH
$env:Path -split ';' | Select-String "Amazon"

# Cek file agent
Get-ChildItem "C:\Program Files\Amazon\AmazonCloudWatchAgent" | Select-Object Name, Length, LastWriteTime
```

## **SOLUSI CEPAT - Run ini sebagai Administrator:**

```powershell
# SOLUSI CEPAT - Run semua command ini sebagai Administrator:

# 1. Stop dan disable service
Stop-Service AmazonCloudWatchAgent -Force -ErrorAction SilentlyContinue
sc.exe delete AmazonCloudWatchAgent

# 2. Hapus config lama
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Force -ErrorAction SilentlyContinue

# 3. Buat config baru SANGAT sederhana
@'
{
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
'@ | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# 4. Install service baru
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent-ctl.ps1 -a install

# 5. Start service
.\amazon-cloudwatch-agent-ctl.ps1 -a start

# 6. Cek status
Get-Service AmazonCloudWatchAgent
```

## **Jika MASIH ERROR, coba ini:**

```powershell
# FULL RESET
# Run sebagai Administrator

# 1. Uninstall CloudWatch Agent dari Control Panel
# 2. Atau gunakan: 
msiexec /x "C:\Program Files\Amazon\AmazonCloudWatchAgent\uninstall.exe" /quiet

# 3. Download dan install ulang
$url = "https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi"
$output = "$env:TEMP\amazon-cloudwatch-agent.msi"
Invoke-WebRequest -Uri $url -OutFile $output
msiexec /i $output /quiet /norestart

# 4. Tunggu install selesai
Start-Sleep -Seconds 30

# 5. Buat config sederhana
@'
{
  "metrics": {
    "metrics_collected": {
      "Processor": {
        "measurement": ["% Processor Time"]
      }
    }
  }
}
'@ | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# 6. Start service
Start-Service -Name "AmazonCloudWatchAgent"
```

**Coba step by step dari atas, dan beritahu saya error di step mana!**
