$script = @'
while ($true) {
    try {
        # Get GPU metrics
        $gpuInfo = nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,memory.free --format=csv,noheader,nounits
        $metrics = $gpuInfo -split ','
        
        # Get instance ID
        $instanceId = (Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id" -ErrorAction Stop).Trim()
        
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
$script | Out-File "C:\Scripts\gpu-monitor.ps1" -Encoding UTF8 -Force

# Run as background job
Start-Job -FilePath "C:\Scripts\gpu-monitor.ps1" -Name "GPU-Monitor"
