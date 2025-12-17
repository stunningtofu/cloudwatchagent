Error masih terjadi karena **versi agent lama** tidak support `nvidia_gpu` receiver. Agent Anda versi `1.300062.0b1394` tidak punya plugin NVIDIA. Mari kita perbaiki.

## **SOLUSI: Gunakan Konfigurasi yang Support Versi Agent Lama**

### **LANGKAH 1: Stop Agent**
```powershell
Stop-Service -Name "AmazonCloudWatchAgent" -Force
Start-Sleep -Seconds 3
```

### **LANGKAH 2: Buat Konfigurasi Sederhana (Tanpa NVIDIA GPU Dulu)**
```powershell
# Buat config minimal dulu untuk test agent
$simpleConfig = @'
{
  "metrics": {
    "namespace": "GPU/Monitoring",
    "metrics_collected": {
      "Processor": {
        "measurement": ["% Processor Time"],
        "metrics_collection_interval": 60
      }
    }
  }
}
'@

$simpleConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

### **LANGKAH 3: Start Agent dengan Config Sederhana**
```powershell
Start-Service -Name "AmazonCloudWatchAgent"
Start-Sleep -Seconds 5

# Cek status
Get-Service -Name "AmazonCloudWatchAgent"
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 5
```

### **LANGKAH 4: Jika Berhasil, Update ke Versi Terbaru Agent**

#### **Download Agent Terbaru:**
```powershell
# Download versi terbaru
$url = "https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi"
$output = "$env:TEMP\amazon-cloudwatch-agent.msi"

Invoke-WebRequest -Uri $url -OutFile $output

# Install versi terbaru
msiexec /i $output /quiet /norestart

# Tunggu install selesai
Start-Sleep -Seconds 30
```

### **LANGKAH 5: Gunakan Konfigurasi yang BENAR untuk Versi Lama**

Jika tidak bisa update, gunakan config ini yang kompatibel dengan versi lama:

```powershell
# Konfigurasi untuk agent versi lama
$compatibleConfig = @'
{
  "agent": {
    "metrics_collection_interval": 60,
    "debug": false
  },
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
              "    ObjectName = \"GPU Engine\"",
              "    Counters = [\"Utilization Percentage\"]",
              "    Instances = [\"* engtype_3D\"]",
              "    Measurement = \"gpu_utilization\""
            ]
          }
        ]
      },
      "Processor": {
        "measurement": ["% Processor Time"],
        "metrics_collection_interval": 60
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    }
  }
}
'@

$compatibleConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

### **LANGKAH 6: OPSI ALTERNATIF - Gunakan Custom Script + CloudWatch CLI**

#### **6.1 Install AWS CLI:**
```powershell
# Download AWS CLI
$msiUrl = "https://awscli.amazonaws.com/AWSCLIV2.msi"
$msiPath = "$env:TEMP\AWSCLIV2.msi"
Invoke-WebRequest -Uri $msiUrl -OutFile $msiPath

# Install
msiexec /i $msiPath /quiet /norestart
```

#### **6.2 Buat PowerShell Script untuk Monitor GPU:**
```powershell
# Save sebagai: C:\Scripts\Send-GPU-Metrics.ps1
$gpuInfo = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
$metrics = $gpuInfo -split ','

# Parse metrics
$gpu_util = [float]$metrics[0]
$mem_util = [float]$metrics[1]
$mem_used = [float]$metrics[2]
$mem_total = [float]$metrics[3]
$mem_free = [float]$metrics[4]

# Get instance metadata
$instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -ErrorAction SilentlyContinue
if (-not $instanceId) { $instanceId = "unknown" }

# Send to CloudWatch
aws cloudwatch put-metric-data `
  --namespace "GPU/Monitoring" `
  --metric-name "GPU_Utilization" `
  --value $gpu_util `
  --unit "Percent" `
  --dimensions "InstanceId=$instanceId"

aws cloudwatch put-metric-data `
  --namespace "GPU/Monitoring" `
  --metric-name "GPU_Memory_Utilization" `
  --value $mem_util `
  --unit "Percent" `
  --dimensions "InstanceId=$instanceId"

aws cloudwatch put-metric-data `
  --namespace "GPU/Monitoring" `
  --metric-name "GPU_Memory_Used" `
  --value $mem_used `
  --unit "Megabytes" `
  --dimensions "InstanceId=$instanceId"

aws cloudwatch put-metric-data `
  --namespace "GPU/Monitoring" `
  --metric-name "GPU_Memory_Total" `
  --value $mem_total `
  --unit "Megabytes" `
  --dimensions "InstanceId=$instanceId"

aws cloudwatch put-metric-data `
  --namespace "GPU/Monitoring" `
  --metric-name "GPU_Memory_Free" `
  --value $mem_free `
  --unit "Megabytes" `
  --dimensions "InstanceId=$instanceId"

Write-Host "Metrics sent: GPU=$gpu_util%, Memory=$mem_util%, Used=$mem_used MB, Total=$mem_total MB, Free=$mem_free MB"
```

#### **6.3 Buat Scheduled Task:**
```powershell
# Create scheduled task untuk run setiap menit
$action = New-ScheduledTaskAction `
  -Execute "PowerShell.exe" `
  -Argument "-NoProfile -WindowStyle Hidden -File C:\Scripts\Send-GPU-Metrics.ps1"

$trigger = New-ScheduledTaskTrigger `
  -Once `
  -At (Get-Date) `
  -RepetitionInterval (New-TimeSpan -Minutes 1)

$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest

Register-ScheduledTask `
  -TaskName "GPU-Metrics-Collector" `
  -Action $action `
  -Trigger $trigger `
  -Principal $principal `
  -Description "Collect and send GPU metrics to CloudWatch"
```

### **LANGKAH 7: SOLUSI PALING SIMPLE - IAM Role + Simple Config**

#### **7.1 Attach IAM Role ke EC2 (PENTING!):**
1. Di AWS Console, **stop EC2 instance** dulu
2. **Attach IAM Role** dengan policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "cloudwatch:PutMetricData",
           "cloudwatch:GetMetricData",
           "cloudwatch:ListMetrics",
           "logs:CreateLogGroup",
           "logs:CreateLogStream",
           "logs:PutLogEvents"
         ],
         "Resource": "*"
       }
     ]
   }
   ```
3. Start instance kembali

#### **7.2 Gunakan Config yang PASTI WORK:**
```powershell
# Config minimal tanpa NVIDIA plugin
$workingConfig = @'
{
  "metrics": {
    "namespace": "GPU/Monitoring",
    "metrics_collected": {
      "Processor": {
        "measurement": ["% Processor Time"],
        "metrics_collection_interval": 60
      },
      "Memory": {
        "measurement": ["% Committed Bytes In Use"],
        "metrics_collection_interval": 60
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    }
  }
}
'@

$workingConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# Restart agent
Restart-Service -Name "AmazonCloudWatchAgent" -Force
```

### **LANGKAH 8: VERIFIKASI**

```powershell
# Cek agent status
Get-Service -Name "AmazonCloudWatchAgent"

# Cek logs
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 10

# Test GPU manual
nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv
```

## **REKOMENDASI:**

**Pilihan 1 (TERBAIK):** Update agent ke versi terbaru, lalu gunakan config `nvidia_gpu`

**Pilihan 2:** Gunakan CloudWatch CLI dengan PowerShell script (jamin work)

**Pilihan 3:** Attach IAM Role dan gunakan config sederhana dulu

**Untuk sekarang, coba Pilihan 3 dulu** untuk pastikan agent bisa berjalan, baru kemudian upgrade atau tambah custom script.
