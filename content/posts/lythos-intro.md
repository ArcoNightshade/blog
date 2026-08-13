+++
title = "Information on Lythos"
date = "2026-08-11T21:54:57-04:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Pureshade Labs"
authorTwitter = "" #do not include @
showFullContent = true
readingTime = true
hideComments = false
+++

# A tad bit of information about PsOS (and it's kernel)


## Lythos (the kernel)
Lythos is a minimalist, high-performance microkernel. All drivers and services run in userspace. The kernel exposes a minimal syscall surface across four categories: memory management, IPC primitives, capability operations, and scheduling.

## PureshadeOS
PureshadeOS (Or *PsOS* for short) is the actual Operating System and userspace built above Lythos. As of writing this, PsOS has basic terminal utils, a very very unfinished package manager, and a custom FS that I built because porting BTRFS would be too annoying. The OS itself is **very** inspired by NixOS but I'd say is a bit better because of me being able to learn from the mistakes of NixOS. one of the core design ideals of PsOS is stability, the more stable it is, the better for everyone, that's why we have rollbacks.

## Shade
Shade is PureshadeOS' package manager, closely modeled after Nix but also faster and entirely written in Rust, it is the successor to rpkg, which was basically a cargo wrapper.

## Prisms
Prisms are the PureshadeOS version of Nix's flakes, they are the gold standard compared to Nix which still marks them as an experimental feature even though it's been many years and a large majority of users use them. Prisms are written in the Shade DSL (not to be confused with the Shade CLI/package manager, and if you can't tell it is *also* inspired by Nix (the language) and Nix (the package manager), amazing!).

## The FileSystem Hierarchy

```
/                                   (root, RFS filesystem)
├── /lth/                           (system namespace — kernel and core daemons)
│   ├── /lth/system/                (@core subvolume, read-only)
│   │   ├── /lth/system/boot/       (Lythos kernel binary, UEFI stub)
│   │   ├── /lth/system/lib/        (core system libraries — musl, Lythos stdlib)
│   │   └── /lth/system/init        (lythd binary — PID 1)
│   └── /lth/bin → /shade/gen/system/current/profile/bin
│                                   (single symlink into the active shade
│                                    system generation — docs/shade-pkg/02-store.md §6)
│
├── /shade/                              (reserved OS-wide for shade store services —
│   │                                canonical layout: docs/shade-pkg/02-store.md §1)
│   ├── /shade/store/                   (input-addressed store: <digest>-<name>-<version>)
│   ├── /shade/db/                      (store metadata: valid set, references)
│   ├── /shade/gen/                     (generation lines: docs/shade-pkg/02-store.md §5)
│   │   ├── /shade/gen/system/          (system line: N/ + `current` flip; boot activates this)
│   │   └── /shade/gen/users/<user>/    (per-user line: N/ + `current`, one per user, independent)
│   ├── /shade/roots/                   (GC roots)
│   ├── /shade/cache/                   (fetch cache, supersedes /var/cache/shade)
│   ├── /shade/build/                   (transient build directories)
│   └── /shade/log/                     (build logs)
│
├── /cfg/                            (@cfg subvolume, read-write)
│   ├── /cfg/lythos/                (CASK kernel configuration)
│   │   ├── /cfg/lythos/rollback    (rollback flag file: set by shade, cleared by lythd)
│   │   └── /cfg/lythos/boot        (kernel command-line and boot parameters)
│   ├── /cfg/services/              (service definitions: TOML; OS-init config, see note)
│   │   ├── lythd.toml
│   │   ├── lythdist.toml
│   │   ├── lythmsg.toml
│   │   ├── lynet.toml
│   │   ├── lygpu.toml
│   │   └── (other service defs)
│   ├── /cfg/webwm/                 (webWM frontend configuration)
│   │   ├── config.toml             (keybinds, gaps, layout rules, app assignments)
│   │   └── theme.css               (visual theming via CSS custom properties)
│   ├── /cfg/shade/                 (system prism authoring + activation pointer —
│   │   │                            canonical: docs/shade-pkg/10-system-prism.md)
│   │   ├── prism.shade             (bootstrap default system prism — only enough
│   │   │                            to build the user's prism; renamed to
│   │   │                            prism.shade.bak on first `shade os rebuild`)
│   │   ├── prism.shade.bak         (retired default, kept as recovery fallback)
│   │   ├── current.pointer         (active system prism ref: <path>#<selector>,
│   │   │                            e.g. /user/lyon/.prism#workstation)
│   │   └── docs/                   (prism-authoring reference; not evaluated)
│   └── /cfg/shell/                 (shell configuration)
│       └── .shellrc                (lysh shell initialization)
│
├── /user/                           (@home subvolume, read-write)
│   ├── /user/home/                 (user home directories)
│   │   ├── /user/home/alice/       (per-user home)
│   │   │   ├── .local/
│   │   │   │   ├── /user/home/alice/.local/share/   (user data)
│   │   │   │   └── /user/home/alice/.local/state/   (user state)
│   │   │   ├── Documents/
│   │   │   ├── Downloads/
│   │   │   ├── Desktop/
│   │   │   ├── .prism/             (per-user prism profile dir; entry prism.shade —
│   │   │   │                        HM-style user config, docs/shade-pkg/10-system-prism.md §5)
│   │   │   └── .config/            (user per-app configuration)
│   │   └── /user/home/bob/         (additional users)
│   └── /user/root/                 (root home)
│
├── /bin/                            (symlinks to /lth/bin — for POSIX compatibility)
│   └── (populated at boot by lythd, points to active tools)
│
├── /sbin/                           (symlinks to /lth/bin — system binaries)
│   └── (populated at boot by lythd)
│
├── /lib/                            (symlinks to /lth/system/lib — for POSIX compat)
│   └── (populated at boot, core libraries accessible at standard path)
│
├── /var/                            (runtime and volatile state — tmpfs or small RFS)
│   ├── /var/run/                   (PID files, sockets)
│   │   ├── /var/run/lythmsg.sock   (lythmsg IPC socket)
│   │   ├── /var/run/lythd.pid
│   │   └── (other daemon sockets)
│   ├── /var/log/                   (system logs)
│   │   ├── /var/log/lythd.log
│   │   ├── /var/log/lythmsg.log
│   │   ├── /var/log/kernel.log
│   │   └── (service logs)
│   ├── /var/cache/                 (transient caches — shade caches live under
│   │                                /shade/cache/, not here)
│   └── /var/tmp/                   (temporary files)
│
├── /tmp/                            (user temporary files — tmpfs, world-writable)
│   └── (ephemeral, cleared on reboot)
│
├── /etc/                            (legacy POSIX config — minimal, mostly empty)
│   ├── /etc/passwd                 (generated from /user/home, read-only at runtime)
│   ├── /etc/group
│   ├── /etc/hostname
│   └── /etc/fstab                  (RFS subvolume mount configuration)
│
├── /root/                           (symlink to /user/root for POSIX compat)
│
├── /home/                           (symlink to /user/home for POSIX compat)
│
├── /proc/                           (kernel proc filesystem — optional, minimal)
│   ├── /proc/cmdline               (kernel command line)
│   ├── /proc/cpuinfo               (CPU info)
│   ├── /proc/meminfo               (memory info)
│   └── (minimal — Lythos kernel exposes this)
│
├── /sys/                            (kernel sysfs — optional, minimal)
│   ├── /sys/class/                 (device classes)
│   └── /sys/devices/               (device tree)
│
└── /dev/                            (device nodes — devtmpfs)
    ├── /dev/null, /dev/zero, /dev/full
    ├── /dev/urandom, /dev/random
    ├── /dev/tty, /dev/pts/         (terminal devices)
    ├── /dev/sd*, /dev/nvme*        (block devices)
    └── (managed by devtmpfs or udev equivalent)
```

Yes, that is the entire FHS, yes it comes directly from the design docs I wrote, yes I created it all.

RFS is the custom FS I mentioned, short for Raptor File System.

Oh, and did I mention it's all written in Rust, specifically no_std Rust? Because it is.

Honestly, I hope it becomes more stable for real hardware one day because it would be amazing.

This also marks the death of the "graig's word of the day" thing on every post, but I'll try to do it on every Friday instead.
