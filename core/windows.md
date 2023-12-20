# Windows
Download [Windows 11][win] English (United States) and create a memory stick with
[Rufus][ruf] on Windows or `woeusb --device "Windows 11.iso" /dev/sda` on Linux.

## Installation
Disconnect the system from all networks and boot the installation media.

```
Language to install: English (United States)
Time and currency format: English (World)
Keyboard or input method: US
```

If the minimum system requirements aren't met, adjust the registry.

1. Press SHIFT+F10 to open the Command Prompt.
2. Type `regedit` and press ENTER.
3. Create the subkey `HKLM\SYSTEM\Setup\LabConfig`.
4. Create DWORD (32-bit) value `BypassTPMCheck` and set it to `1`.
5. Create DWORD (32-bit) value `BypassSecureBootCheck` and set it to `1`.
6. Close all windows and press the "Back" button in the top left.

When the installer is stuck on "Let's connect you to a network", bypass it.

1. Press SHIFT+F10 to open the Command Prompt.
2. Type `oobe\BypassNRO` and press ENTER.
3. Wait for the system to reboot and resume.
4. Select "I don't have internet" to continue.

Choose a single word username starting with a capital letter to keep the
`%UserProfile%` path consistent and free from spaces.

Verify Windows flavor and version with `Start > "winver"`.

Switch the system clock from local time to UTC.

```reg
[HKLM\SYSTEM\CurrentControlSet\Control\imeZoneInformation]
"RealTimeIsUniversal"=dword:00000001
```

[win]: https://www.microsoft.com/en-us/software-download/windows11
[ruf]: https://rufus.ie/en/