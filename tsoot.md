Terlihat control script versi lama tidak support `install` action. Mari kita perbaiki dengan cara manual.

## **LANGKAH 1: INSTALL SERVICE MANUAL**

```powershell
# Coba install service dengan command langsung
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Coba jalankan executable yang digunakan service
.\start-amazon-cloudwatch-agent.exe

# Atau coba buat service manual
sc.exe create "AmazonCloudWatchAgent" `
  binPath= "\"C:\Program Files\Amazon\AmazonCloudWatchAgent\start-amazon-cloudwatch-agent.exe\"" `
  DisplayName= "Amazon CloudWatch Agent" `
  start= "auto" `
  depend= "Tcpip/Dhcp/Dnscache"

# Set service description
sc.exe description "AmazonCloudWatchAgent" "Collects metrics and logs and sends them to Amazon CloudWatch"
```

## **LANGKAH 2: GUNAKAN CONFIG YANG LEBIH SIMPLE LAGI**

```powershell
# Buat config yang sangat simple
$simpleConfig = @'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": []
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "resources": ["*"],
        "measurement": [
          {"name": "cpu_usage_active", "unit": "Percent"}
        ],
        "totalcpu": false
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    }
  }
}
'@

$simpleConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

## **LANGKAH 3: START SERVICE**

```powershell
# Start service
sc.exe start "AmazonCloudWatchAgent"

# Atau via PowerShell
Start-Service -Name "AmazonCloudWatchAgent"

# Cek status
sc.exe query "AmazonCloudWatchAgent"
Get-Service -Name "AmazonCloudWatchAgent"
```

## **LANGKAH 4: JIKA MASIH ERROR, COBA RUN MANUAL DULU**

```powershell
# Run agent secara manual untuk debugging
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Coba run agent langsung (akan output error ke console)
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" --console
```

## **LANGKAH 5: ALTERNATIF - USE telegraf_win_perf_counters**

```powershell
# Config menggunakan telegraf plugin yang tersedia
$telegrafConfig = @'
{
  "metrics": {
    "namespace": "GPU/Monitoring",
    "metrics_collected": {
      "telegraf": {
        "telegraf_metrics_input_interval": "60s",
        "telegraf_metrics_plugin_config": [
          {
            "name": "win_perf_counters",
            "config": [
              "[[inputs.win_perf_counters]]",
              "  [[inputs.win_perf_counters.object]]",
              "    ObjectName = \"Processor\"",
              "    Instances = [\"_Total\"]",
              "    Counters = [\"% Processor Time\"]",
              "    Measurement = \"cpu\"",
              "  [[inputs.win_perf_counters.object]]",
              "    ObjectName = \"GPU Engine\"",
              "    Instances = [\"* engtype_3D\"]",
              "    Counters = [\"Utilization Percentage\"]",
              "    Measurement = \"gpu\""
            ]
          }
        ]
      }
    }
  }
}
'@

$telegrafConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

## **LANGKAH 6: FIX DENGAN MENGGUNAKAN DEFAULT CONFIG**

```powershell
# Coba gunakan config default dari agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Copy default config
Copy-Item ".\config\config.json" "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Force

# Atau generate config default
$defaultConfig = @'
{
  "agent": {
    "region": "us-east-1"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "C:\\ProgramData\\Amazon\\AmazonCloudWatchAgent\\Logs\\amazon-cloudwatch-agent.log",
            "log_group_name": "amazon-cloudwatch-agent.log",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "statsd": {}
    }
  }
}
'@

$defaultConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

## **LANGKAH 7: COBA ACTION YANG TERSEDIA DI CONTROL SCRIPT**

```powershell
# Coba action yang tersedia
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Cek status
.\amazon-cloudwatch-agent-ctl.ps1 -a status

# Start
.\amazon-cloudwatch-agent-ctl.ps1 -a start

# Fetch config
.\amazon-cloudwatch-agent-ctl.ps1 -a fetch-config --c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

## **LANGKAH 8: SOLUSI PASTI - UNINSTALL & REINSTALL**

```powershell
# 1. Uninstall agent dari Programs and Features
#    atau gunakan:
msiexec /x "{B1F0E1D7-0DF6-4B2D-B8A0-1B3D5D05D9E7}" /quiet

# 2. Download versi terbaru
$url = "https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi"
$output = "$env:TEMP\cloudwatch-agent.msi"
Invoke-WebRequest -Uri $url -OutFile $output

# 3. Install
msiexec /i $output /quiet /norestart

# 4. Tunggu
Start-Sleep -Seconds 30

# 5. Gunakan wizard config
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent-config-wizard.exe
```

## **LANGKAH 9: QUICK WORKAROUND - PowerShell Script Saja**

Jika agent tetap tidak bisa, gunakan PowerShell script saja:

```powershell
# Save sebagai: C:\Scripts\monitor-gpu.ps1
while ($true) {
    $gpuInfo = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
    $metrics = $gpuInfo -split ','
    
    # Log ke file
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - GPU: $($metrics[0])%, Mem: $($metrics[1])%, Used: $($metrics[2]) MB, Total: $($metrics[3]) MB, Free: $($metrics[4]) MB" | 
        Out-File "C:\GPU-Monitor.log" -Append
    
    # Tunggu 60 detik
    Start-Sleep -Seconds 60
}
```

## **RUN INI SEBAGAI ADMINISTRATOR:**

```powershell
# 1. Delete service yang rusak
sc.exe delete "AmazonCloudWatchAgent"

# 2. Buat config super simple
@'{"metrics":{"metrics_collected":{"cpu":{"measurement":["cpu_usage_idle"]}}}}'@ | 
    Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# 3. Coba start agent langsung (bukan sebagai service)
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
Start-Process -FilePath ".\amazon-cloudwatch-agent.exe" -ArgumentList "--config `"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json`"" -NoNewWindow

# 4. Tunggu 10 detik
Start-Sleep -Seconds 10

# 5. Cek apakah process berjalan
Get-Process -Name "amazon-cloudwatch-agent" -ErrorAction SilentlyContinue
```

**Coba langkah-langkah di atas dan beritahu saya hasilnya!**
