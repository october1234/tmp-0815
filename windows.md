Windows also lacks one clean, guaranteed HDMI history. It records related Plug-and-Play, display-driver, and monitor events, but the amount of detail depends on the GPU driver.

Check existing Windows logs

Open Event Viewer with:

eventvwr.msc

Look under:

Windows Logs → System

Filter by these event sources:

* Kernel-PnP
* UserPnp
* Display
* DisplayDriver
* DxgKrnl

Also inspect:

Applications and Services Logs
  → Microsoft
    → Windows
      → Kernel-PnP
      → UserPnp
      → DriverFrameworks-UserMode
      → DeviceSetupManager

Some of these operational logs may need to be enabled by right-clicking the log and selecting Enable Log.

A quick PowerShell search of recent relevant system events:

Get-WinEvent -LogName System -MaxEvents 3000 |
    Where-Object {
        $_.ProviderName -match 'Display|DxgKrnl|Kernel-PnP|UserPnp' -or
        $_.Message -match 'monitor|HDMI|display|EDID'
    } |
    Select-Object TimeCreated, ProviderName, Id, LevelDisplayName, Message |
    Format-List

Create a continuous monitor logger

This PowerShell script records changes detected through Windows’ monitor-management interface. It includes whether the display is active, its connection technology, monitor identity, and instance name.

Save it as C:\Scripts\MonitorStateLogger.ps1:

$LogDirectory = "$env:ProgramData\MonitorStateLog"
$LogFile = Join-Path $LogDirectory "monitor-events.jsonl"
$PollSeconds = 1
New-Item -ItemType Directory -Path $LogDirectory -Force | Out-Null
$TechnologyNames = @{
    -2 = "Uninitialized"
    -1 = "Other"
     0 = "VGA"
     1 = "S-Video"
     2 = "Composite"
     3 = "Component"
     4 = "DVI"
     5 = "HDMI"
     6 = "LVDS"
     8 = "D-Jpn"
     9 = "SDI"
    10 = "DisplayPort"
    11 = "Embedded DisplayPort"
    12 = "UDI External"
    13 = "UDI Embedded"
    14 = "SDTV Dongle"
    15 = "Miracast"
    16 = "Indirect Wired"
    17 = "Indirect Virtual"
    18 = "Internal"
}
function Convert-MonitorString {
    param([UInt16[]]$Value)
    if (-not $Value) {
        return $null
    }
    return (-join ($Value |
        Where-Object { $_ -ne 0 } |
        ForEach-Object { [char]$_ })).Trim()
}
function Get-MonitorSnapshot {
    $identities = @{}
    Get-CimInstance -Namespace root\wmi -ClassName WmiMonitorID `
        -ErrorAction SilentlyContinue |
        ForEach-Object {
            $identities[$_.InstanceName] = @{
                Manufacturer = Convert-MonitorString $_.ManufacturerName
                Model        = Convert-MonitorString $_.UserFriendlyName
                Serial       = Convert-MonitorString $_.SerialNumberID
            }
        }
    $connections = Get-CimInstance `
        -Namespace root\wmi `
        -ClassName WmiMonitorConnectionParams `
        -ErrorAction SilentlyContinue
    @($connections | ForEach-Object {
        $technology = [int]$_.VideoOutputTechnology
        $identity = $identities[$_.InstanceName]
        [ordered]@{
            InstanceName          = $_.InstanceName
            Active                = [bool]$_.Active
            VideoOutputTechnology = $technology
            Connection            = if ($TechnologyNames.ContainsKey($technology)) {
                                        $TechnologyNames[$technology]
                                    } else {
                                        "Unknown ($technology)"
                                    }
            Manufacturer          = $identity.Manufacturer
            Model                 = $identity.Model
            Serial                = $identity.Serial
        }
    } | Sort-Object InstanceName)
}
$PreviousState = $null
while ($true) {
    $Snapshot = @(Get-MonitorSnapshot)
    $State = ConvertTo-Json $Snapshot -Depth 5 -Compress
    if ($State -ne $PreviousState) {
        $Record = [ordered]@{
            Timestamp = (Get-Date).ToString("o")
            Monitors  = $Snapshot
        }
        $Record |
            ConvertTo-Json -Depth 6 -Compress |
            Add-Content -Path $LogFile -Encoding UTF8
        Write-Host "$(Get-Date -Format o) Monitor configuration changed"
        $Snapshot | Format-Table Active, Connection, Manufacturer, Model, Serial
        $PreviousState = $State
    }
    Start-Sleep -Seconds $PollSeconds
}

Run it from an elevated PowerShell window:

powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Scripts\MonitorStateLogger.ps1

Read the raw history:

Get-Content "$env:ProgramData\MonitorStateLog\monitor-events.jsonl"

Show it as PowerShell objects:

Get-Content "$env:ProgramData\MonitorStateLog\monitor-events.jsonl" |
    ForEach-Object { $_ | ConvertFrom-Json } |
    Format-List

Run it automatically

Open Task Scheduler with:

taskschd.msc

Create a task that runs at startup with:

Program:
powershell.exe
Arguments:
-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File "C:\Scripts\MonitorStateLogger.ps1"

Enable “Run whether user is logged on or not” and “Run with highest privileges.”

One caveat: an HDMI-to-DisplayPort adapter, USB-C dock, or DisplayLink device may be reported as DisplayPort, indirect wired, or another technology even though the final physical cable is HDMI. The monitor identity and active-state transitions will still help pinpoint the disconnect.