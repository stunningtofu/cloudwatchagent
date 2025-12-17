Masih ada masalah dengan config file. Mari kita coba approach yang berbeda:

## **MASALAH:** Agent mencoba parse JSON sebagai TOML

## **SOLUSI 1: INSTALL ULANG AGENT DENGAN VERSI TERBARU**

```powershell
# 1. Uninstall agent yang ada
$uninstallCommand = @"
Start-Process msiexec -ArgumentList '/x {B1F0E1D7-0DF6-4B2D-B8A0-1B3D5D05D9E7} /quiet /norestart' -Wait -NoNewWindow
"@
Invoke-Expression $uninstallCommand

# Tunggu
Start-Sleep -Seconds 10

# 2. Download versi terbaru
$url = "https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi"
$output = "C:\Windows\Temp\cloudwatch-agent.msi"
Invoke-WebRequest -Uri $url -OutFile $output

# 3. Install
Start-Process msiexec -ArgumentList "/i `"$output`" /quiet /norestart" -Wait -NoNewWindow

# 4. Tunggu install selesai
Start-Sleep -Seconds 30
```

## **SOLUSI 2: GUNAKAN CONFIG TOML (BUKAN JSON)**

```powershell
# Hapus semua config yang ada
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Force -ErrorAction SilentlyContinue

# Buat config TOML
$tomlConfig = @'
[agent]
  interval = "60s"
  flush_interval = "60s"
  omit_hostname = false

[[outputs.cloudwatch]]
  region = "us-east-1"
  namespace = "CWAgent"
  tagexclude = ["metric*"]

[[inputs.cpu]]
  percpu = false
  totalcpu = true
  collect_cpu_time = false
  report_active = true

[[inputs.mem]]
'@

$tomlConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.toml" -Encoding UTF8 -Force

# Jalankan dengan config TOML
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.toml"
```

## **SOLUSI 3: BUAT SEMUA FILE YANG DIBUTUHKAN**

```powershell
# Buat semua file yang diperlukan
$configDir = "C:\ProgramData\Amazon\AmazonCloudWatchAgent"

# 1. Buat common-config.toml
$commonConfig = @'
[agent]
  interval = "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "60s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log"
  hostname = ""
  omit_hostname = false
'@

$commonConfig | Out-File "$configDir\common-config.toml" -Encoding UTF8 -Force

# 2. Buat env-config.json (kosong)
'{}' | Out-File "$configDir\env-config.json" -Encoding UTF8 -Force

# 3. Buat config JSON yang valid
$jsonConfig = @'
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "CWAgent",
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
'@

$jsonConfig | Out-File "$configDir\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# 4. Jalankan agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\start-amazon-cloudwatch-agent.exe
```

## **SOLUSI 4: GUNAKAN WINDOWS BUILT-IN PERF COUNTERS**

```powershell
# Buat config untuk Windows Performance Counters
$perfConfig = @'
{
  "metrics": {
    "metrics_collected": {
      "LogicalDisk": {
        "measurement": [
          "% Free Space"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "Memory": {
        "measurement": [
          "% Committed Bytes In Use"
        ],
        "metrics_collection_interval": 60
      },
      "Processor": {
        "measurement": [
          "% Processor Time"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "_Total"
        ]
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    }
  }
}
'@

$perfConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force

# Start service
Start-Service -Name "AmazonCloudWatchAgent" -ErrorAction SilentlyContinue
```

## **SOLUSI 5: COBA DENGAN FLAG --MODE ec2**

```powershell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Coba dengan mode EC2
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" --mode ec2

# Atau
.\amazon-cloudwatch-agent-ctl.ps1 -a start -m ec2
```

## **SOLUSI 6: RESET COMPLETE**

```powershell
# Run sebagai Administrator

# 1. Stop dan hapus service
Stop-Service -Name "AmazonCloudWatchAgent" -Force -ErrorAction SilentlyContinue
sc.exe delete "AmazonCloudWatchAgent"

# 2. Hapus semua file config
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Recurse -Force -ErrorAction SilentlyContinue

# 3. Buat struktur folder
New-Item -ItemType Directory -Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs" -Force
New-Item -ItemType Directory -Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Configs" -Force

# 4. Buat config yang benar-benar clean
$cleanConfig = '{}'
[System.IO.File]::WriteAllText("C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json", $cleanConfig, [System.Text.Encoding]::UTF8)

# 5. Buat service manual
sc.exe create "AmazonCloudWatchAgent" `
  binPath= "\"C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.exe\" --config \"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json\"" `
  DisplayName= "Amazon CloudWatch Agent" `
  start= "auto"

# 6. Start service
sc.exe start "AmazonCloudWatchAgent"
```

## **SOLUSI 7: GANTI KE VERSI TELEGRAF + CLOUDWATCH OUTPUT**

```powershell
# Install Telegraf terpisah
$telegrafUrl = "https://dl.influxdata.com/telegraf/releases/telegraf-1.30.0_windows_amd64.zip"
$telegrafPath = "$env:TEMP\telegraf.zip"
Invoke-WebRequest -Uri $telegrafUrl -OutFile $telegrafPath

# Ekstrak
Expand-Archive -Path $telegrafPath -DestinationPath "C:\Program Files\Telegraf" -Force

# Buat config Telegraf untuk GPU
$telegrafConfig = @'
[agent]
  interval = "60s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  hostname = ""
  omit_hostname = false

[[outputs.cloudwatch]]
  region = "us-east-1"
  namespace = "Telegraf/GPU"
  tagexclude = ["host"]

[[inputs.exec]]
  commands = [
    "powershell -Command \"nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits\""
  ]
  timeout = "5s"
  data_format = "csv"
  csv_header_row_count = 0
  csv_column_names = ["utilization_gpu","utilization_memory","memory_used","memory_total","memory_free"]
  csv_tag_columns = []
  csv_timestamp_column = ""
  csv_timestamp_format = ""

[[inputs.win_perf_counters]]
  [[inputs.win_perf_counters.object]]
    ObjectName = "Processor"
    Instances = ["_Total"]
    Counters = ["% Processor Time"]
    Measurement = "cpu"

[[inputs.win_perf_counters]]
  [[inputs.win_perf_counters.object]]
    ObjectName = "Memory"
    Counters = ["% Committed Bytes In Use"]
    Measurement = "memory"
'@

$telegrafConfig | Out-File "C:\Program Files\Telegraf\telegraf.conf" -Encoding UTF8 -Force

# Jalankan Telegraf sebagai service
cd "C:\Program Files\Telegraf"
.\telegraf.exe --config "telegraf.conf" --service install
Start-Service -Name "telegraf"
```

## **SOLUSI CEPAT - COBA INI:**

```powershell
# 1. Hapus semua
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Recurse -Force

# 2. Buat config minimal VALID
$config = '{"metrics":{"metrics_collected":{"statsd":{}}}}'
[System.IO.File]::WriteAllText("C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json", $config, [System.Text.Encoding]::UTF8)

# 3. Jalankan dengan console output untuk debug
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" --console
```

## **REKOMENDASI:**

Coba **SOLUSI 3** dulu (buat semua file yang dibutuhkan). Jika masih error, coba **SOLUSI 7** (gunakan Telegraf sebagai alternatif).

**Yang paling penting:** Pastikan IAM Role sudah attached ke EC2 instance dengan permission CloudWatch!
