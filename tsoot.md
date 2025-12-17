Ada error di JSON config! Ada karakter yang tidak valid. Mari kita perbaiki:

## **MASALAH:** JSON config corrupt atau ada karakter yang tidak valid

## **LANGKAH 1: HAPUS DAN BUAT ULANG CONFIG FILE**

```powershell
# Hapus config file yang corrupt
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Force

# Hapus juga file config lainnya
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*.toml" -Force -ErrorAction SilentlyContinue
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*.json" -Force -ErrorAction SilentlyContinue

# Buat folder Logs jika belum ada
New-Item -ItemType Directory -Path "C:\ProgramData\Amazon\AmazonCloudWatchAgent\Logs" -Force
```

## **LANGKAH 2: BUAT CONFIG YANG BENAR-BENAR VALID**

```powershell
# Buat config JSON yang benar
$validConfig = @'
{
  "metrics": {
    "metrics_collected": {
      "Processor": {
        "measurement": [
          "% Processor Time"
        ],
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

# Save dengan encoding yang benar
$validConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

## **LANGKAH 3: VALIDASI JSON KEMBALI**

```powershell
# Test membaca JSON
$testConfig = Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Raw
Write-Host "Config length: $($testConfig.Length) characters"

# Test parse JSON
try {
    $parsed = $testConfig | ConvertFrom-Json
    Write-Host "✓ JSON valid!" -ForegroundColor Green
    $parsed.metrics.metrics_collected
} catch {
    Write-Host "✗ JSON invalid: $_" -ForegroundColor Red
    
    # Tampilkan karakter masalah
    for ($i = 0; $i -lt [math]::Min($testConfig.Length, 100); $i++) {
        $char = $testConfig[$i]
        $ascii = [int][char]$char
        Write-Host "Position $i : '$char' (ASCII: $ascii)"
    }
}
```

## **LANGKAH 4: GUNAKAN CONFIG YANG LEBIH SEDERHANA**

```powershell
# Config yang sangat minimal
$minimalConfig = @'
{}
'@

$minimalConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json" -Encoding UTF8 -Force
```

## **LANGKAH 5: COBA START DENGAN CONFIG MINIMAL**

```powershell
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"

# Coba start dengan config kosong
.\start-amazon-cloudwatch-agent.exe

# Atau langsung jalankan agent
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"
```

## **LANGKAH 6: BUAT CONFIG MENGGUNAKAN WIZARD**

```powershell
# Hapus semua config
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Force -ErrorAction SilentlyContinue

# Jalankan wizard config
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent-config-wizard.exe
```

**Jika wizard tidak ada, coba ini:**

## **LANGKAH 7: MANUAL FIX - EDIT FILE LANGSUNG**

```powershell
# Buka Notepad sebagai Administrator
notepad "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# Hapus SEMUA isi file, lalu copy-paste ini:
```

```json
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
```

**Save file dan tutup Notepad.**

## **LANGKAH 8: TEST DENGAN POWERSHELL COMMAND LANGSUNG**

```powershell
# Test config file
$configPath = "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# Baca file sebagai teks biasa
$content = Get-Content $configPath -Raw
Write-Host "=== FILE CONTENT ==="
$content
Write-Host "=== END CONTENT ==="

# Cek karakter tersembunyi
$bytes = [System.Text.Encoding]::UTF8.GetBytes($content)
Write-Host "`nFirst 50 bytes:"
$bytes[0..49] | ForEach-Object { Write-Host "$_ : [$(if($_ -eq 10){'LF'}elseif($_ -eq 13){'CR'}elseif($_ -lt 32){'CTRL'}else{[char]$_})]" }
```

## **LANGKAH 9: BUAT FILE BARU DENGAN POWERSHELL**

```powershell
# Hapus file lama
Remove-Item $configPath -Force

# Buat dengan method yang lebih clean
[System.IO.File]::WriteAllText($configPath, '{}', [System.Text.Encoding]::UTF8)

# Test
.\start-amazon-cloudwatch-agent.exe
```

## **LANGKAH 10: ONE-LINE FIX**

```powershell
# Run sebagai Administrator
[System.IO.File]::WriteAllText("C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json", '{"metrics":{"metrics_collected":{"cpu":{"measurement":["cpu_usage_idle"]}}}}', [System.Text.Encoding]::UTF8); cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"; .\start-amazon-cloudwatch-agent.exe
```

## **LANGKAH 11: JIKA MASIH ERROR, GUNAKAN CONFIG TOML**

```powershell
# Agent juga support TOML format
$tomlConfig = @'
[agent]
  interval = "60s"

[[outputs.cloudwatch]]
  region = "us-east-1"
  namespace = "GPU/Monitoring"

[[inputs.cpu]]
  percpu = false
  totalcpu = true
'@

$tomlConfig | Out-File "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.toml" -Encoding UTF8 -Force

# Coba start dengan TOML
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\amazon-cloudwatch-agent.exe --config "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.toml"
```

## **SOLUSI CEPAT - COPY PASTE INI DI POWERSHELL:**

```powershell
# 1. Hapus semua config
Remove-Item "C:\ProgramData\Amazon\AmazonCloudWatchAgent\*" -Force

# 2. Buat config baru dengan cara yang benar
$configJson = '{"metrics":{"metrics_collected":{"cpu":{"measurement":["cpu_usage_idle"],"metrics_collection_interval":60}}}}'
[System.IO.File]::WriteAllText("C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json", $configJson, [System.Text.Encoding]::UTF8)

# 3. Verifikasi
Get-Content "C:\ProgramData\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json"

# 4. Start agent
cd "C:\Program Files\Amazon\AmazonCloudWatchAgent"
.\start-amazon-cloudwatch-agent.exe
```

**Coba langkah 11 (SOLUSI CEPAT) dulu!** Error sebelumnya karena ada karakter tidak valid di JSON file.
