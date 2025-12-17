# RUN SEMUA INI SEBAGAI ADMINISTRATOR:

# 1. Hapus semua config
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Recurse -Force

# 2. Buat folder
New-Item -ItemType Directory -Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs" -Force

# 3. Buat config JSON SANGAT SIMPLE
$config = '{"metrics":{"metrics_collected":{"statsd":{"service_address":":8125"}}}}'
[System.IO.File]::WriteAllText("C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json", $config, [System.Text.Encoding]::UTF8)

# 4. Buat common-config.toml yang benar
@'
[agent]
  interval = "60s"
  logfile = "C:\\\\ProgramData\\\\Amazon\\\\AmazonCloudWatchAgent\\\\Logs\\\\amazon-cloudwatch-agent.log"
'@ | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\common-config.toml" -Encoding UTF8

# 5. Hapus service jika ada
sc.exe delete "AmazonCloudWatchAgent" 2>$null

# 6. Jalankan agent langsung (tanpa service dulu)
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
Start-Process -FilePath ".\amazon-cloudwatch-agent.exe" -ArgumentList "--config `"C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json`" --mode auto" -NoNewWindow -Wait

# 7. Cek process
Get-Process -Name "amazon-cloudwatch-agent" -ErrorAction SilentlyContinue
