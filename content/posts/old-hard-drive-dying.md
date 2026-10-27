---
title: "When an Old Hard Drive Starts Dying: A Troubleshooting Walkthrough"
date: 2026-10-27
draft: false
tags: ["troubleshooting", "windows", "storage", "hard-drive", "legacy", "corrupted-partitions"]
summary: "Bad block errors, two corrupted partitions and a repair popup I almost clicked. How I worked out what was wrong and fixed it without losing my files."
ShowToc: true
---


This whole thing started because I decided my laptop deserved some much-needed attention. It's more than a decade old, and I'd been ignoring it the way people ignore a car that still starts every morning. So I set aside a weekend for maintenance. 
I started by clearing out about 6 GB of temp files, then finally updated a graphics driver that was six years out of date, and spent the evening switching off every background task that was quietly slowing my system down.

By the end of it I was feeling pretty pleased with myself. Then I opened an old project folder to check on a notebook I hadn't touched in ages, and Windows popped up a notification I'd never seen before: "Restart to repair drive errors." Clicking it would have been the easy option. I'm glad I didn't, because what came next was a day of reading logs, learning what half of chkdsk's parameters actually do, and realising how little I knew about the drive holding most of my files.

The full write-up below is everything I did, in order, with the commands. If your drive is doing something similar right now, skip to [The step I skipped](#the-step-i-skipped-please-dont) and copy your important files off before you do anything else.

## The notification I almost clicked

I nearly clicked that popup to restart my system. Windows offered to fix things, so why not let it? But ten minutes later my D: drive wouldn't open, and copying anything off G: failed with error 0x80004005. I was hoping it was just a glitch.

For some context, my laptop runs Windows 10 from an SSD, and its original 1 TB hard drive holds most of my data, split into D:, E:, F: and G:. 

## Which drive was actually sick?

The popup doesn't tell you directly, but Windows does keep track of it. When it finds file system errors on a volume, it marks that volume as "dirty" so that it gets checked at the next boot. 
You can check it yourself too using the following command:

```
fsutil dirty query <Drive Letter>:
```

I ran it for every drive letter, and my D: and G: drives were marked dirty. Everything else was fine.


The next stop was Event Viewer (`eventvwr.msc` > Windows Logs > System, filtered by the sources disk and Ntfs). Three event IDs told me most of the story:

| Event ID | Source | Meaning |
| --- | --- | --- |
| 7 | disk | The physical disk hit a bad block |
| 55 | Ntfs | File system corruption, usually naming the file or folder |
| 98 | Ntfs | Volume health, including "requires an Online Scan" |

The event 7 messages all said `\Device\Harddisk1\DR1, has a bad block`. Harddisk1 is Disk 1 in Disk Management, which was the old HDD. Thankfully the SSD was fine. 

After that I counted the errors that were generated on that day:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='disk'; Id=7; StartTime=(Get-Date).Date} | Select TimeCreated, Message
```

There were fourteen errors in one day, and six of them were in the last ninety minutes. That's when it stopped feeling like a glitch.

## The step I skipped (please don't)

The 'repair' Windows wanted to run at boot was `chkdsk`. When chkdsk can't read a damaged record, it can fix the problem by deleting the record and the file it pointed to. On a disk that's getting worse, you'd want your files copied off before that happens.

So the right move was to stop poking the drive. Don't keep hitting Try Again on failed copies, don't format anything Windows offers to format, and don't run `chkdsk /r`, which reads every sector and is the heaviest job you can give a dying disk. If you need to restart before you're ready, you can tell Windows to skip the boot-time check on specific drives with `chkntfs /x D: G:` (undo it later with `chkntfs /d`).

Then copy your important stuff off with the robocopy command. File Explorer usually gives up after trying to copy the first unreadable file; robocopy skips it and moves on:

```
robocopy "G:\SomeFolder" "H:\Rescue\G\SomeFolder" /E /R:0 /W:0 /LOG+:H:\Rescue\G_log.txt
```

`/R:0 /W:0` means no retries and no waiting on a bad file, and the log lists everything that failed. Do it one folder at a time, most important first.

Unfortunately I couldn't run it. My laptop forced my hand before I got there, but I got lucky. Next time, I'm doing these first.

## The restart I didn't get to choose

My laptop suddenly froze and I had to restart. The restart also installed a pending Windows update on the way, and chkdsk ran on the D: drive whether I liked it or not.

Once the system was back up, I wanted to know what exactly chkdsk had done. Boot-time chkdsk reports land in the Application log:

```powershell
Get-WinEvent -LogName Application -MaxEvents 1000 | Where-Object { $_.ProviderName -match 'Wininit|Chkdsk' } | Select-Object -First 3 | Format-List TimeCreated, ProviderName, Message
```

The news was decent. It had deleted three corrupt file records, removed two small `.ini` settings files belonging to some app, and moved 12 orphaned files into `D:\found.001`. The D: drive was no longer marked dirty and opened again. But the G: drive was still waiting its turn.

Those `found.00X` folders are where chkdsk drops files it rescued but couldn't put back. They're hidden, but typing the path into File Explorer's address bar usually opens them.

The bad block errors kept coming even after the restart, though. Fixing the file system fixes the map of where your files live, but the disk underneath was still the same disk.

## Asking the drive how it feels

Event logs will tell you what Windows noticed. But the drive also keeps its own health record (S.M.A.R.T. - Self-Monitoring, Analysis, and Reporting Technology), and that's what tells you if it's dying. PowerShell's `Get-PhysicalDisk` just threw "Invalid property" at me, so I used CrystalDiskInfo, a free tool that only reads that data.

The result surprised me:

| Attribute | Value | What it counts |
| --- | --- | --- |
| 05 Reallocated Sectors | 0 | Sectors permanently swapped for spares |
| C5 Current Pending Sectors | 0 | Sectors waiting because they couldn't be read |
| C6 Uncorrectable Sectors | 0 | Data the drive couldn't recover |
| C7 UltraDMA CRC Errors | 0 | Cable or connection errors |

All zeros and a "Good" status, after a whole day of bad block errors. My best guess is that some weak sectors were hard to read and got refreshed when chkdsk rewrote the damaged records, so nothing ever needed remapping.

But the drive is still old and tired. It has about 22,000 power-on hours, around 443,000 head parking cycles (drives like it are typically rated for about 600,000), a logged temperature peak above its safe limit, and a few thousand recorded shocks. It isn't dying today, but trusting it with the only copy of anything is a huge risk.

CrystalDiskInfo now sits in my system tray (Function > Keep Running in System Tray, Startup, Auto Refresh) and will alert me the moment any of those first three numbers moves off zero.

## Round two: fixing G: 

There was another popup, "Scan drive for errors," and this time the Event ID 98 entries named the drive exactly: "G: requires an Online Scan." An online scan checks the drive while Windows keeps running:

```
chkdsk G: /scan
```

First attempt: it found a corrupted folder, then gave up with "was not able to send command for self-healing due to lack of memory." My RAM was already over 70% used because of browser tabs that I had running in the background. I closed the browser and a few other apps and tried again.

Second attempt went much better. It repaired seven corrupt records and six broken folder links online, recovered 43 orphaned files into `G:\found.002` to `found.007`, and queued one damaged folder for an offline fix. 
For that last bit it suggested spotfix, which only repairs the queued problems and, on a data drive, just dismounts it for a moment:

```
chkdsk G: /spotfix
```

I typed Y to dismount. 0.2 seconds later: "Windows has fixed all previously identified issues with this drive." The four Jupyter notebooks inside that folder even went back where they belonged.

While digging through the found folders, I listed the hidden contents with `dir -Force G:\found.004`. Inside I found an old practice Jupyter notebook of mine, and in another folder a directory chkdsk had renamed to `dir_00000001.chk` because it couldn't recover the original name.

## What I lost, and what I learned

The only real damage was inside two Python virtual environments, a plotly library folder in one and some time zone files in the other. I can't repair those; I had to delete the environments and rebuild them from `requirements.txt`. 

I also lost time chasing scary-looking Event Viewer errors that had nothing to do with the disk: a failed Secure Boot SBAT update (0x800700c1, an old-firmware thing) and a GameBar DCOM timeout. Even a healthy PC's logs are filled with errors and warnings. Only experience teaches you which warnings and errors need to be acted on immediately and which ones can be ignored. 

If I had to do it again, I'd:

1. Copy important files off the disk before letting any repair run.
2. Keep two copies of anything that matters, on an external drive or in the cloud.
3. Check S.M.A.R.T. early, because a dying drive and a damaged file system need very different levels of caution.
4. Use the targeted chkdsk options (`/scan`, `/spotfix`, `/f`) and save `/r` for when your data is already safe.

Unless you're willing to shell out loads of cash for replacement parts, especially now with the prices of computers and accessories going up in recent times due to the AI boom, it's become really important to take care of your system and hard drives.

Back up all your important files. Today, if you can.

### Cheat sheet

| Goal | Command |
| --- | --- |
| Is a volume flagged for repair? | `fsutil dirty query D:` |
| Today's bad block errors | `Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='disk'; Id=7; StartTime=(Get-Date).Date}` |
| Last boot-time chkdsk report | `Get-WinEvent -LogName Application -MaxEvents 1000 \| Where-Object { $_.ProviderName -match 'Wininit\|Chkdsk' }` |
| Skip boot-time checks on some drives | `chkntfs /x D: G:` (undo with `chkntfs /d`) |
| Copy while skipping unreadable files | `robocopy "SRC" "DEST" /E /R:0 /W:0 /LOG+:log.txt` |
| Scan while Windows runs | `chkdsk G: /scan` |
| Fix only the queued problems | `chkdsk G: /spotfix` |
| Fix file system errors, no surface scan | `chkdsk G: /f` |
| List hidden items in PowerShell | `dir -Force G:\found.000` |
