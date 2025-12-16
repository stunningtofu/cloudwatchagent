### STEP 1: Install CloudWatch Agent

1. Download CloudWatch Agent MSI
```shell
$msiUrl = "https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi"
$installerPath = "$env:TEMP\amazon-cloudwatch-agent.msi"
```

2. Download installer
```shell
Invoke-WebRequest -Uri $msiUrl -OutFile $installerPath
```

3. Install MSI
```shell
msiexec.exe /i $installerPath /quiet
```

4. Verify installation
```shell
Get-Service AmazonCloudWatchAgent
```

### STEP 2: Install NVIDIA Plugin untuk GPU Monitoring

CloudWatch Agent untuk Windows sudah include NVIDIA plugin, tidak perlu install terpisah. Tapi pastikan:

<!-- Cek apakah plugin tersedia -->
```shell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
dir .\plugins\
```
<!-- Harus ada file seperti nvidia_gpu.dll -->

### STEP 3: Buat Konfigurasi File

Buat file `C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json:`

```json
{
    "metrics": {
        "metrics_collected": {
            "nvidia_gpu": {
                "measurement": [
                    "utilization_gpu",
                    "utilization_memory",
                    "memory_used"
                ],
                "metrics_collection_interval": ,
                    "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
                    "ImageId": "${aws:ImageId}",
                    "InstanceId": "${aws:InstanceId}",
                    "InstanceType": "${aws:InstanceType}"
                }
            }
        },
        "append_dimensions": {
            "InstanceId": "${aws:InstanceId}",
            "InstanceType": "${aws:InstanceType}"
        },
        "aggregation_dimensions": [["InstanceId"]],
        "force_flush_interval": 60
    },
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": []
            }
        }
    }
}
```

### STEP 4: Konfigurasi dan Start Agent

Run as Administrator

1. Navigate ke agent directory
```shell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
```

2. Apply konfigurasi
```shell
.\amazon-cloudwatch-agent-ctl.ps1 -a fetch-config -m ec2 -s -c 
file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json"
```

3. Start service
```shell
Start-Service AmazonCloudWatchAgent
```

4. Cek status
```shell
.\amazon-cloudwatch-agent-ctl.ps1 -a status
```

### STEP 5: Verifikasi Instalasi

1. Cek service status
```shell
Get-Service AmazonCloudWatchAgent | Select-Object Status, StartType
```

2. Cek logs (jika ada issue)
```shell
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs\amazon-cloudwatch-agent.log" -Tail 20
```

3. Cek proses berjalan
```shell
Get-Process amazon-cloudwatch-agent
```

4. Test config (validasi syntax)
```shell
.\amazon-cloudwatch-agent-ctl.ps1 -a config -m ec2 -c file:"C:\ProgramData\Amazon\AmazonCloudWatchAgent\config.json" -s
```