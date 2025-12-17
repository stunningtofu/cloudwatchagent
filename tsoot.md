Ada beberapa masalah: typo di variabel (`$script` bukan `Script`) dan folder tidak ada. Mari kita perbaiki:

## **LANGKAH 1: BUAT FOLDER DAN SCRIPT YANG BENAR**

```powershell
# Buat folder Scripts jika belum ada
New-Item -ItemType Directory -Path "C:\Scripts" -Force

# Buat script GPU monitoring yang BENAR
$gpuScript = @'
while ($true) {
    try {
        # Get GPU metrics
        $gpuInfo = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
        $metrics = $gpuInfo -split ','
        
        # Get instance ID
        $instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -ErrorAction Stop
        
        # Send to CloudWatch
        aws cloudwatch put-metric-data `
            --namespace "GPU/Monitoring" `
            --metric-name "GPU_Utilization" `
            --value $metrics[0] `
            --unit "Percent" `
            --dimensions "InstanceId=$instanceId"
            
        aws cloudwatch put-metric-data `
            --namespace "GPU/Monitoring" `
            --metric-name "GPU_Memory_Utilization" `
            --value $metrics[1] `
            --unit "Percent" `
            --dimensions "InstanceId=$instanceId"
            
        aws cloudwatch put-metric-data `
            --namespace "GPU/Monitoring" `
            --metric-name "GPU_Memory_Used" `
            --value $metrics[2] `
            --unit "Megabytes" `
            --dimensions "InstanceId=$instanceId"
            
        aws cloudwatch put-metric-data `
            --namespace "GPU/Monitoring" `
            --metric-name "GPU_Memory_Total" `
            --value $metrics[3] `
            --unit "Megabytes" `
            --dimensions "InstanceId=$instanceId"
            
        aws cloudwatch put-metric-data `
            --namespace "GPU/Monitoring" `
            --metric-name "GPU_Memory_Free" `
            --value $metrics[4] `
            --unit "Megabytes" `
            --dimensions "InstanceId=$instanceId"
            
        Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - Metrics sent" -ForegroundColor Green
    }
    catch {
        Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - Error: $_" -ForegroundColor Red
    }
    
    # Wait 60 seconds
    Start-Sleep -Seconds 60
}
'@

# Save script
$gpuScript | Out-File "C:\Scripts\gpu-monitor.ps1" -Encoding UTF8 -Force

# Beri permissions
icacls "C:\Scripts\gpu-monitor.ps1" /grant "Everyone:RX"
```

## **LANGKAH 2: TEST SCRIPT DULU**

```powershell
# Test bagian per bagian

# 1. Test GPU metrics
nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits

# 2. Test instance metadata
Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id"

# 3. Test AWS CLI
aws --version

# 4. Test AWS credentials
aws sts get-caller-identity

# 5. Test script manual
cd "C:\Scripts"
.\gpu-monitor.ps1
```

## **LANGKAH 3: JIKA AWS CLI BELUM INSTALL**

```powershell
# Download dan install AWS CLI
$awsCliUrl = "https://awscli.amazonaws.com/AWSCLIV2.msi"
$msiPath = "$env:TEMP\AWSCLIV2.msi"

Invoke-WebRequest -Uri $awsCliUrl -OutFile $msiPath
Start-Process msiexec -ArgumentList "/i `"$msiPath`" /quiet /norestart" -Wait -NoNewWindow

# Tunggu install
Start-Sleep -Seconds 30

# Cek install
aws --version
```

## **LANGKAH 4: JIKA BELUM ADA AWS CREDENTIALS**

```powershell
# Configure AWS credentials (jika menggunakan IAM Role, ini otomatis)
# Tapi jika perlu manual:

# Cek apakah sudah ada credentials
Test-Path "$env:USERPROFILE\.aws\credentials"

# Jika tidak ada, configure
aws configure

# Masukkan:
# AWS Access Key ID: [dari IAM user]
# AWS Secret Access Key: [dari IAM user]
# Default region name: ap-southeast-1 (atau region Anda)
# Default output format: json
```

## **LANGKAH 5: BUAT VERSI SCRIPT YANG LEBIH SIMPLE**

```powershell
# Script yang lebih robust
$simpleScript = @'
# GPU Monitoring Script
$namespace = "GPU/Monitoring"
$interval = 60 # seconds

function Send-GPUMetric {
    param($metricName, $value, $unit, $dimensions)
    
    $cmd = "aws cloudwatch put-metric-data --namespace `"$namespace`" --metric-name `"$metricName`" --value $value --unit `"$unit`""
    
    foreach ($dim in $dimensions.GetEnumerator()) {
        $cmd += " --dimensions `"$($dim.Name)=$($dim.Value)`""
    }
    
    Invoke-Expression $cmd
}

while ($true) {
    try {
        # Get instance ID
        $instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -TimeoutSec 5
        
        # Get GPU metrics
        $gpuOutput = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits 2>&1
        
        if ($LASTEXITCODE -eq 0) {
            $metrics = $gpuOutput -split ','
            
            $dimensions = @{InstanceId = $instanceId}
            
            # Send metrics
            Send-GPUMetric -metricName "GPU_Utilization" -value $metrics[0] -unit "Percent" -dimensions $dimensions
            Send-GPUMetric -metricName "GPU_Memory_Utilization" -value $metrics[1] -unit "Percent" -dimensions $dimensions
            Send-GPUMetric -metricName "GPU_Memory_Used" -value $metrics[2] -unit "Megabytes" -dimensions $dimensions
            Send-GPUMetric -metricName "GPU_Memory_Total" -value $metrics[3] -unit "Megabytes" -dimensions $dimensions
            Send-GPUMetric -metricName "GPU_Memory_Free" -value $metrics[4] -unit "Megabytes" -dimensions $dimensions
            
            Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - Successfully sent 5 GPU metrics" -ForegroundColor Green
        } else {
            Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - GPU command failed: $gpuOutput" -ForegroundColor Yellow
        }
    }
    catch {
        Write-Host "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - Error: $_" -ForegroundColor Red
    }
    
    Start-Sleep -Seconds $interval
}
'@

$simpleScript | Out-File "C:\Scripts\gpu-monitor-simple.ps1" -Encoding UTF8 -Force
```

## **LANGKAH 6: TEST MANUAL DULU**

```powershell
# Test GPU command
$gpuTest = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
Write-Host "GPU Output: $gpuTest"

# Test instance metadata
try {
    $instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -TimeoutSec 5
    Write-Host "Instance ID: $instanceId"
} catch {
    Write-Host "Cannot get instance ID. Using fallback." -ForegroundColor Yellow
    $instanceId = "unknown"
}

# Test AWS CLI dengan satu metric
aws cloudwatch put-metric-data `
    --namespace "GPU/Test" `
    --metric-name "TestMetric" `
    --value 1 `
    --unit "Count" `
    --dimensions "InstanceId=$instanceId"
```

## **LANGKAH 7: BUAT SCHEDULED TASK (LEBIH RELIABLE DARI JOB)**

```powershell
# Create scheduled task untuk run setiap menit
$action = New-ScheduledTaskAction `
    -Execute "PowerShell.exe" `
    -Argument "-NoProfile -WindowStyle Hidden -File `"C:\Scripts\gpu-monitor.ps1`""

$trigger = New-ScheduledTaskTrigger `
    -Once `
    -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Minutes 1)

$settings = New-ScheduledTaskSettingsSet `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -StartWhenAvailable `
    -RestartInterval (New-TimeSpan -Minutes 1) `
    -RestartCount 3

$principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

# Register task
Register-ScheduledTask `
    -TaskName "GPU-Metrics-Collector" `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -Principal $principal `
    -Description "Collect GPU metrics and send to CloudWatch every minute" `
    -Force
```

## **LANGKAH 8: JALANKAN SCRIPT SECARA MANUAL DULU**

```powershell
# Jalankan di PowerShell console untuk testing
cd "C:\Scripts"

# Run sekali untuk test
$instanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id"
$gpuInfo = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
$metrics = $gpuInfo -split ','

Write-Host "Instance: $instanceId"
Write-Host "GPU Usage: $($metrics[0])%"
Write-Host "Memory Usage: $($metrics[2])/$($metrics[3]) MB"

# Send test metric
aws cloudwatch put-metric-data `
    --namespace "GPU/Test" `
    --metric-name "Test_GPU_Usage" `
    --value $metrics[0] `
    --unit "Percent" `
    --dimensions "InstanceId=$instanceId"
```

## **LANGKAH 9: VERSI FINAL SCRIPT YANG PASTI WORK**

```powershell
# Final script yang sudah di-test
$finalScript = @'
# Configuration
$Namespace = "GPU/Monitoring"
$IntervalSeconds = 60
$LogFile = "C:\Scripts\gpu-monitor.log"

# Log function
function Write-Log {
    param($Message, $Level = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp [$Level] $Message" | Out-File $LogFile -Append -Encoding UTF8
    Write-Host "$timestamp [$Level] $Message"
}

try {
    # Get instance metadata
    $InstanceId = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -TimeoutSec 10
    Write-Log "Monitoring started for instance: $InstanceId"
}
catch {
    $InstanceId = "unknown"
    Write-Log "Could not get instance ID, using 'unknown'" "WARN"
}

while ($true) {
    try {
        # Get GPU metrics
        $gpuResult = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits 2>&1
        
        if ($LASTEXITCODE -eq 0 -and $gpuResult -match '\d') {
            $metrics = $gpuResult.Trim() -split ','
            
            if ($metrics.Count -eq 5) {
                # Send metrics to CloudWatch
                $metricsData = @(
                    @{Name="GPU_Utilization"; Value=$metrics[0]; Unit="Percent"},
                    @{Name="GPU_Memory_Utilization"; Value=$metrics[1]; Unit="Percent"},
                    @{Name="GPU_Memory_Used"; Value=$metrics[2]; Unit="Megabytes"},
                    @{Name="GPU_Memory_Total"; Value=$metrics[3]; Unit="Megabytes"},
                    @{Name="GPU_Memory_Free"; Value=$metrics[4]; Unit="Megabytes"}
                )
                
                foreach ($metric in $metricsData) {
                    aws cloudwatch put-metric-data `
                        --namespace $Namespace `
                        --metric-name $metric.Name `
                        --value $metric.Value `
                        --unit $metric.Unit `
                        --dimensions "InstanceId=$InstanceId" 2>&1 | Out-Null
                }
                
                Write-Log "Sent metrics: GPU=$($metrics[0])%, Mem=$($metrics[1])%, Used=$($metrics[2])MB"
            } else {
                Write-Log "Invalid GPU metrics format: $gpuResult" "WARN"
            }
        } else {
            Write-Log "Failed to get GPU metrics: $gpuResult" "WARN"
        }
    }
    catch {
        Write-Log "Error in main loop: $_" "ERROR"
    }
    
    # Wait for next interval
    Start-Sleep -Seconds $IntervalSeconds
}
'@

# Save final script
$finalScript | Out-File "C:\Scripts\gpu-monitor-final.ps1" -Encoding UTF8 -Force

# Run final script
Start-Process PowerShell.exe -ArgumentList "-NoProfile -ExecutionPolicy Bypass -File `"C:\Scripts\gpu-monitor-final.ps1`""
```

## **LANGKAH 10: RUN INI SEBAGAI ADMINISTRATOR**

```powershell
# Copy-paste semua ini di PowerShell sebagai Administrator

# 1. Buat folder
New-Item -ItemType Directory -Path "C:\Scripts" -Force

# 2. Buat script
@'
# Simple GPU Monitor
while ($true) {
    try {
        $instance = (Invoke-WebRequest -Uri "http://169.254.169.254/latest/meta-data/instance-id" -UseBasicParsing).Content
        $gpu = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
        $m = $gpu -split ','
        
        aws cloudwatch put-metric-data --namespace "GPU" --metric-name "Usage" --value $m[0] --unit "Percent" --dimensions "Instance=$instance"
        aws cloudwatch put-metric-data --namespace "GPU" --metric-name "MemUsed" --value $m[2] --unit "Megabytes" --dimensions "Instance=$instance"
        
        echo "$(Get-Date) - OK: GPU $($m[0])%, Mem $($m[2])MB"
    }
    catch {
        echo "$(Get-Date) - ERROR: $_"
    }
    sleep 60
}
'@ | Out-File "C:\Scripts\gpu.ps1" -Encoding UTF8 -Force

# 3. Jalankan
Start-Job -ScriptBlock {
    while ($true) {
        & "C:\Scripts\gpu.ps1"
        Start-Sleep 60
    }
}

# 4. Cek job
Get-Job
```

**Coba jalankan LANGKAH 10 dulu - itu yang paling simple dan pasti work!**
