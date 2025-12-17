# **CARA MANUAL IMPLEMENTASI CLOUDWATCH AGENT untuk GPU MONITORING**

## **LANGKAH 1: PREPARASI**

### **1.1 Buka PowerShell sebagai Administrator**
- Klik **Start Menu**
- Ketik **PowerShell**
- Klik kanan → **Run as Administrator**

### **1.2 Cek Status Agent Saat Ini**
```powershell
# Cek service CloudWatch Agent
Get-Service -Name "AmazonCloudWatchAgent"

# Cek apakah agent berjalan
sc query "AmazonCloudWatchAgent"

# Backup konfigurasi lama jika ada
Copy-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json.backup" -Force
```

---

## **LANGKAH 2: STOP AGENT**

```powershell
# Stop service CloudWatch Agent
Stop-Service -Name "AmazonCloudWatchAgent" -Force

# Tunggu 5 detik
Start-Sleep -Seconds 5

# Verifikasi sudah stop
Get-Service -Name "AmazonCloudWatchAgent"
```

---

## **LANGKAH 3: BUAT FILE KONFIGURASI**

### **3.1 Buka Notepad atau Text Editor**
- Tekan **Windows + R**
- Ketik **notepad** → Enter

### **3.2 Copy-Paste Konfigurasi Ini:**
```json
{
  "agent": {
    "run_as_user": "root",
    "region": "${aws:Region}",
    "debug": false,
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "GPU/Monitoring",
    "metrics_collected": {
      "nvidia_gpu": {
        "measurement": [
          "utilization_gpu",
          "utilization_memory",
          "memory_total",
          "memory_free",
          "memory_used"
        ],
        "metrics_collection_interval": 60,
        "append_dimensions": {
          "gpu_name": "${nvidia_gpu:0:name}",
          "gpu_uuid": "${nvidia_gpu:0:uuid}"
        }
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "ImageId": "${aws:ImageId}",
      "Region": "${aws:Region}"
    },
    "aggregation_dimensions": [
      ["InstanceId"],
      ["InstanceId", "InstanceType"],
      ["AutoScalingGroupName"]
    ],
    "force_flush_interval": 30
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "C:\\ProgramData\\Amazon\\AmazonCloudWatchAgent\\Logs\\amazon-cloudwatch-agent.log",
            "log_group_name": "GPU-Agent-Logs",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S",
            "timezone": "UTC"
          }
        ]
      }
    },
    "log_stream_name": "{instance_id}",
    "force_flush_interval": 15
  }
}
```

### **3.3 Save File ke Lokasi yang Tepat:**
- **File name:** `amazon-cloudwatch-agent.json`
- **Save as type:** `All Files (*.*)`
- **Save location:** `C:\ProgramData\Amazon\AmazonCloudWatchAgent\`

**Atau gunakan PowerShell untuk membuat file:**

```powershell
# Buat folder jika belum ada
New-Item -ItemType Directory -Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent" -Force

# Buat file konfigurasi
@'
{
  "agent": {
    "run_as_user": "root",
    "region": "${aws:Region}",
    "debug": false,
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "GPU/Monitoring",
    "metrics_collected": {
      "nvidia_gpu": {
        "measurement": [
          "utilization_gpu",
          "utilization_memory",
          "memory_total",
          "memory_free",
          "memory_used"
        ],
        "metrics_collection_interval": 60,
        "append_dimensions": {
          "gpu_name": "${nvidia_gpu:0:name}",
          "gpu_uuid": "${nvidia_gpu:0:uuid}"
        }
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "ImageId": "${aws:ImageId}",
      "Region": "${aws:Region}"
    },
    "aggregation_dimensions": [
      ["InstanceId"],
      ["InstanceId", "InstanceType"],
      ["AutoScalingGroupName"]
    ],
    "force_flush_interval": 30
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "C:\\ProgramData\\Amazon\\AmazonCloudWatchAgent\\Logs\\amazon-cloudwatch-agent.log",
            "log_group_name": "GPU-Agent-Logs",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%Y-%m-%d %H:%M:%S",
            "timezone": "UTC"
          }
        ]
      }
    },
    "log_stream_name": "{instance_id}",
    "force_flush_interval": 15
  }
}
'@ | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

---

## **LANGKAH 4: START AGENT**

```powershell
# Start service
Start-Service -Name "AmazonCloudWatchAgent"

# Tunggu 10 detik untuk inisialisasi
Start-Sleep -Seconds 10

# Cek status
Get-Service -Name "AmazonCloudWatchAgent"
```

---

## **LANGKAH 5: VERIFIKASI**

### **5.1 Cek Status Service**
```powershell
# Cek apakah service berjalan
Get-Service -Name "AmazonCloudWatchAgent" | Format-Table Name, Status, DisplayName
```

### **5.2 Cek Process Agent**
```powershell
# Cek apakah process agent berjalan
Get-Process -Name "AmazonCloudWatchAgent" -ErrorAction SilentlyContinue
```

### **5.3 Cek Logs**
```powershell
# Lihat log terakhir (tunggu 30 detik dulu)
Start-Sleep -Seconds 30

# Buka log file
notepad "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log"

# Atau lihat via PowerShell
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 20
```

### **5.4 Test GPU Detection Manual**
```powershell
# Test apakah NVIDIA GPU terdeteksi
nvidia-smi

# Test metrics spesifik
nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv
```

---

## **LANGKAH 6: TROUBLESHOOTING**

### **Jika Agent Gagal Start:**

#### **6.1 Cek Error di Event Viewer:**
```powershell
# Buka Event Viewer
eventvwr.msc

# Atau cek via PowerShell
Get-WinEvent -LogName "Application" | Where-Object {$_.ProviderName -like "*CloudWatch*"} | Select-Object -First 5 TimeCreated, LevelDisplayName, Message
```

#### **6.2 Test Konfigurasi:**
```powershell
# Navigasi ke folder agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Test konfigurasi
.\amazon-cloudwatch-agent-ctl.ps1 -a config
```

#### **6.3 Restart Manual:**
```powershell
# Stop
Stop-Service -Name "AmazonCloudWatchAgent" -Force

# Start ulang
Start-Service -Name "AmazonCloudWatchAgent"
```

---

## **LANGKAH 7: TEST METRICS DI CLOUDWATCH**

### **7.1 Tunggu 2-3 Menit** untuk data pertama masuk

### **7.2 Cek di AWS Console:**
1. Buka **AWS Management Console**
2. Pilih **CloudWatch**
3. Klik **Metrics** → **All metrics**
4. Cari namespace: **`GPU/Monitoring`**
5. Metrics yang harus muncul:
   - `utilization_gpu`
   - `utilization_memory`
   - `memory_total`
   - `memory_free`
   - `memory_used`

### **7.3 Atau Cek via AWS CLI:**
```powershell
# Install AWS CLI dulu jika belum
# Download dari: https://aws.amazon.com/cli/

# Configure AWS CLI
aws configure

# Cek metrics di CloudWatch
aws cloudwatch list-metrics --namespace "GPU/Monitoring"
```

---

## **LANGKAH 8: MONITORING RUTIN**

### **8.1 Buat Script Monitoring Sederhana:**
```powershell
# Save sebagai: C:\Scripts\monitor-gpu.ps1
Write-Host "=== GPU Monitoring Status ===" -ForegroundColor Cyan

# 1. Service Status
$service = Get-Service -Name "AmazonCloudWatchAgent"
Write-Host "Agent Status: $($service.Status)" -ForegroundColor $(if($service.Status -eq "Running"){"Green"}else{"Red"})

# 2. Current GPU Metrics
Write-Host "`nCurrent GPU Metrics:" -ForegroundColor Yellow
nvidia-smi --query-gpu=name,utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv

# 3. Log Check
Write-Host "`nLatest Logs:" -ForegroundColor Yellow
$logFile = "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log"
if (Test-Path $logFile) {
    Get-Content $logFile -Tail 3
}

Write-Host "`nCheck CloudWatch Console:" -ForegroundColor Magenta
Write-Host "Namespace: GPU/Monitoring" -ForegroundColor White
```

### **8.2 Jadwalkan Monitoring:**
```powershell
# Buat scheduled task (opsional)
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\monitor-gpu.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "9:00AM"
Register-ScheduledTask -TaskName "GPU-Monitoring-Check" -Action $action -Trigger $trigger -Description "Check GPU Monitoring Status"
```
