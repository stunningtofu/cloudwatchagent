# Test langsung di PowerShell
$instance = Invoke-RestMethod -Uri "http://169.254.169.254/latest/meta-data/instance-id"
$gpu = nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits

Write-Host "Instance: $instance"
Write-Host "GPU Usage: $gpu%"

# Send test
aws cloudwatch put-metric-data `
    --namespace "GPU-Test" `
    --metric-name "TestMetric" `
    --value $gpu `
    --unit "Percent" `
    --dimensions "Instance=$instance"
