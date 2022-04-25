# Windows
Installation and configuration instructions for Windows 10 21H2.

<!--
Fix drive previously used for unix partitions with `diskpart`.

```cmd
DISKPART> list disk
DISKPART> select disk 1
DISKPART> clean
DISKPART> create partition primary
DISKPART> active
DISKPART> format fs=Fat32 quick
```
-->

## Preparations
Download the latest [Windows 10](https://www.microsoft.com/en-us/software-download/windows10) image
and create the installation media using the Media Creation Tool or [Rufus](https://rufus.ie/).

Create the file `\sources\ei.cfg` on the installation media.

```ini
[EditionID]
Professional
[Channel]
Retail
[VL]
0
```

Create the file `\sources\pid.txt` on the installation media if `Channel` is not `OEM`.

```ini
[PID]
Value={windows key}
```

Copy this repo and the latest graphics and network drivers to the installation media.<br/>
Set the BIOS date and time to the current local time.

**Keep the system disconnected from the network.**

## Installation
Boot the installation media.

```
Language to install: English (United States)
Time and currency format: {Current Time Zone Country}
Keyboard or input method: {Current Hardware Keyboard}
```

Choose a single word username starting with a capital letter to keep the `%UserProfile%` path
consistent and free from spaces.

Verify Windows flavor and version with `Start > "winver"`.

## Setup
Modify and execute [setup.ps1](setup.ps1) using the "Run with PowerShell" context menu
repeatedly until it stops rebooting the system.
