---
title: Unbreakable Linux - Daily Driver
description: Building a resilient, secure and reliable OS using btrfs subvolumes, snapper, and grub-btrfs that I use on my primary machine.
author: Aman Shaw
date: 2026-01-30 12:00:00 +0000
categories: [Blogging, Workflow]
tags: [linux, fedora, cosmic, btrfs, zstd, snapper, snapshots ]
---

After years of using PopOS as my primary operating system, I've now switched to Fedora, and settled on a setup that gives me the perfect balance of performance, security, usability, and peace of mind.

Following the excellent methodology from [SysGuides](https://sysguides.com/install-fedora-42-with-full-disk-encryption-snapshot-and-rollback-support), I've customised it to be performant and reliable, making system failures a thing of the past with full-disk encryption, bootable snapshots, rollbacks, and Zstd compression to save space, reducing I/O and improving performance.

In this blog I have provided customisations on top of [this](https://sysguides.com/install-fedora-42-with-full-disk-encryption-snapshot-and-rollback-support) article, which can be applied for majority of users.

## Why I switched to Fedora

Fedora is a great balance of modern and stable packages and is early adopter of modern technology like wayland, btrfs, selinux, and many more. It is known for its security and early adoption of software technologies.

Fedora's release cycle allows modern cutting-edge packages with decent stability. Every 6 months, a new release is available that is supported for 13 months, which gives an option to skip a release.

Fedora is updated with latest kernel without needing to upgrade to newer release by default. Fedora also comes with a large set of packages and external sources like Copr repos, similar to Ubuntu's PPA.

### Modern Desktop

System76 who created PopOS also created cosmic desktop which is a new rust base desktop which includes both a keyboard centric tiling features and a standard gnome like floating mode.

Currently Cosmic Desktop is also available with new PopOS but 3 months ago it was in alpha stage and was available with fedora 43 as a spin and as copr package for nightly builds.

### Smart Storage Layout

Fedora by design is simple and modern and defaults to btrfs, which comes with advanced features like subvolumes and compression support. I have used btrfs with PopOS but with fedora it's default.

The core of this setup's reliability comes from intelligent Btrfs subvolume organization. Separate subvolumes allows you to exclude data from snapshots and assign different mount options as required.

### Rollback Capabilities

Fedora uses `grub` as a bootloader whereas PopOS uses `systemd-boot`. Grub allows customizations to integrate live boot to snapshot using `grub-btrfs` and `snapper`.

This setup allows a risk-free testing and low maintainance overhead. If in case something goes wrong, you can rollback and fix the issue on your own time.

### The Game Changer: Live Boot Snapshots

This is where the setup truly shines for daily use. The integration of grub-btrfs with Snapper means I can boot directly into any previous snapshot from the GRUB menu.
When something goes wrong - whether it's a botched system update, a misconfiguration, or problematic software installation - my workflow is simple:

1. **Reboot the system**
2. **Select the working snapshot from GRUB**
3. **Continue working as if nothing happened**
No recovery media, no chroot environments, no hours of troubleshooting. Just instant recovery to a known working state.

This capability has fundamentally changed how I approach system maintenance:

- **Risk-Free Experimentation**: I can try new software, test system configurations, or attempt complex modifications without fear
- **Update Confidence**: System updates are no longer a source of anxiety - if something breaks, I'm a reboot away from recovery
- **Productivity Preservation**: When issues do occur, my workflow disruption is measured in minutes, not hours or days

## Customizations

### Using zstd compression

You can save a lot of space and improve disk performance by using zstd compression that is builtin btrfs filesystem. You just need to add an option in your fstab file for every subvolume that you want to be compressed.

Btrfs allows different types of algorithms with diffrent types of intensity, in general for faster performance we can use zstd level 1 for faster ssd's and use zstd level 3 for slower hard drives. In hard drives using compression can reduce I/O cycles improving performance.

Here is an example of fstab entry with the options.

```fstab
UUID=6283c14b-7a32-4100-a259-56fb7033aedd /             btrfs compress=zstd:1,subvol=root,x-systemd.device-timeout=0   0 0
UUID=3608dfbe-0b7a-40bf-be1a-ebd2d36dcb02 /mnt/archive  btrfs compress=zstd:3,nosuid,nodev,nofail                      0 0
```

### Excluding data from snapshots

Subvolumes allows us to separate data that should be treated in a different manner. For example in the guide you can notice that they mentioned to use different subvolume for `/var/lib/libvirt` that is used for storing KVM/QEMU virtual machines, doing so will exclude the data on the directory to be excluded from automatic snapshots by snapper.

In my install I sepparate:

- `/var/lib/flatpak` which is used for storing flatapk's installed on system wide level.
- `/var/lib/cosmic-greeter` which is used by cosmic's own display manager similar to gdm in the guide.

To implement this first create backups if the data is already present.

```sh
sudo cp -r /var/lib/flatpak /var/lib/flatpak.bak
```

Mount the root partition, where all the subvolumes were created.

> Remember to replace the device location and mount point accordingly
{: .prompt-warning }

```sh
sudo mount -o defaults,compress=zstd:1 /dev/mapper/luks-uuid /mnt
```

Create subvolumes for data that you want to exclude.

```sh
sudo btrfs subvolume create /mnt/flatpak
```

Move the original data to the new subvolume.

> Remember the permissions needs to match with original, we are using rsync for this reason.
{: .prompt-info}

```sh
sudo rsync -aP /var/lib/flatpak /mnt/flatpak
```

Delete the files present in the mount point.

```sh
rm -rf /var/lib/flatpak/* /var/lib/flatpak/.*
```

Add the mount points in the fstab file by copying existing entry and modify the subvolume name, options, and mount point. Reboot and make sure if it is mounted using `lsblk`.

If every thing is fine and working as expected, you can delete the backup files.

### Snapshot Strategy: Weekly Instead of Hourly

Here's where my approach really differs from most tutorials. Instead of the default hourly timeline snapshots, I configured weekly snapshots with intelligent cleanup:

First you need to customize the snapper timeline timers, by modifying a systemd configuration file.

```sh
sudo cp /usr/lib/systemd/system/snapper-timeline.timer /etc/systemd/system/
sudo nano /etc/systemd/system/snapper-timeline.timer
```

Now change the hourly to weekly in the file. You can also adjust the configuration file for each snapper config for the number of timeline snapshots that you want to store, you can also do this using `snapper set-config` command.

```bash
sudo snapper -c root set-config TIMELINE_LIMIT_HOURLY=0
sudo snapper -c root set-config TIMELINE_LIMIT_DAILY=0
sudo snapper -c root set-config TIMELINE_LIMIT_WEEKLY=4
sudo snapper -c root set-config TIMELINE_LIMIT_MONTHLY=2
```

**Benefits of this approach:**

- **Reduced System Overhead**: Weekly snapshots are executed less frequently.
- **Better Storage Management**: Fewer snapshots mean less disk space usage
- **Sufficient Coverage**: Combined with automatic pre/post snapshots during package operations, I still get comprehensive rollback protection
- **Faster Operations**: Snapshot creation and cleanup operations complete much quicker

## Real-World Daily Driving Benefits

After months of using this setup, I can share concrete examples of how these features have improved my daily workflow:

### The "Broken Update" Scenario

Last month, a kernel update caused compatibility issues with my graphics drivers. Instead of spending hours debugging, I:

1. Rebooted and selected the previous snapshot from GRUB
2. Continued working immediately
3. Fixed the issue on my own schedule
4. Re-applied the update when I had time to troubleshoot properly

### The "Software Experiment" Freedom

I wanted to test a new development toolchain that required significant system changes. With my setup:

1. Created a manual snapshot before installation
2. Installed and tested the toolchain
3. Decided it wasn't suitable
4. Rolled back to the previous snapshot in minutes
5. System was exactly as it was before the experiment

### The "Performance Maintenance" Advantage

The weekly snapshot schedule, combined with zstd compression optimization, means:

- Reduced disk I/O during routine maintenance
- More predictable performance characteristics
- Better storage utilization

## Advantages

One of the best aspects of this setup is how low-maintenance it is once configured:

- **Automatic Operations**: The system handles snapshot creation, cleanup, and GRUB integration automatically
- **User-Friendly Management**: Btrfs Assistant provides a graphical interface for snapshot management when I prefer GUI tools
- **Storage Efficiency**: Weekly snapshots combined with intelligent compression keep storage usage reasonable
- **System Resilience**: Even when something does go wrong, recovery is quick and painless

## Is This Setup Right For You?

This configuration is ideal for users who:

- Value system stability above all else
- Want the freedom to experiment without consequences
- Need reliable recovery from system issues
- Work with important data that requires protection
- Prefer a "set it and forget it" approach to maintenance
- Want modern desktop features without sacrificing reliability
It might be overkill if you frequently reinstall systems or enjoy troubleshooting, but for those of us who rely on our systems for daily productivity, this approach is transformative.

## Conclusion and Acknowledgments

The combination of these customizations provides a computing environment that's both powerful and resilient. This setup represents more than just technical configuration - it's a fundamentally different approach to system management where failures are not just recoverable, but essentially non-disruptive.

**Special thanks to [SysGuides](https://sysguides.com/install-fedora-42-with-full-disk-encryption-snapshot-and-rollback-support)** for their comprehensive tutorial that formed the foundation of this setup. Their detailed methodology provided the framework that I've customized with my own optimizations for performance and usability.

The investment in setting this up pays dividends every day in the form of reduced stress, increased productivity, and the confidence to tackle any computing challenge without fear of system-breaking consequences.

For anyone looking to create a truly resilient Linux setup, I can't recommend this approach enough. It's not just about having a backup plan - it's about building a system that adapts to your needs while providing the freedom to explore without limits.
