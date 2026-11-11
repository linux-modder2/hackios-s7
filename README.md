# hackios-s7
# How to do it yourself
You need a Redmi Note 10 Pro (sweet, global or Indian variant, SM7150) with an unlocked bootloader, a Linux PC, and a GitHub account for the kernel build. This is the exact path used for the screenshots above; other variants or firmware may need changes. Everything on userdata is lost. Bootloaders, recovery, modem and persist partitions are never touched.

0. Back up the phone
Boot OrangeFox (or any recovery with adb root) temporarily and pull at least persist, modemst1, modemst2, fsg, fsc and boot, dtbo, vbmeta. They contain per-device calibration data. Keep them off git; .gitignore already excludes backup/, dtb/ and recovery/.

The shim needs the stock DTB. Unpack your phone's stock boot.img:

python3 tools/unpack_bootv2.py backup/boot.img dtb/boot_unpacked   # -> dtb/boot_unpacked/dtb
1. Build the kernel
The kernel builds on GitHub's macOS runners with blacktop/darwin-xnu-build. The workflow fetches xnu-8796, applies xnu/patches/ in order, and also stages the pinned open-source libpthread, HFS and BearSSL sources it needs. Fork the repo (public repos get free macOS minutes) and run:

gh workflow run xnu.yml -R YOU/ios-sweet
gh run download RUN_ID -R YOU/ios-sweet -n xnu-kernel -D xnu/out
To check a patch locally before spending a CI run, tools/xnu-cc.py replays the compile commands of a previous CI log for single files.

2. Build the iOS root image
Download the iPhone X (iPhone10,3) iOS 16.7.16 (20H392) IPSW from Apple. Then, on Linux, with apfs-fuse, hfsprogs and sudo:

# OS, SystemOS and AppOS disk images from the IPSW (see BuildManifest.plist for their names)
tools/mkiosroot.sh OS.dmg SystemOS.dmg AppOS.dmg ios16-root.img 16G
sudo fsck.hfsplus -fy ios16-root.img
sudo mount -t hfsplus -o loop,force ios16-root.img /mnt/iosroot
sudo python3 tools/sweet-root-tweaks.py /mnt/iosroot
sudo umount /mnt/iosroot
img2simg ios16-root.img ios16-root.simg
mkiosroot.sh copies the OS volume and both cryptexes into one case-sensitive journaled HFS+ volume. It adds the dyld cache links, the plain system keybag and the log directories the daemons expect. sweet-root-tweaks.py applies the bring-up changes: it skips Setup, seeds the activation state, removes the baseband daemon, disables the watchdogs that kill slow apps and the crash-looping media paths, and so on. The script's header explains each change.

The device tree builder copies /product from the real iPhone X device tree, which MobileGestalt needs. Extract it once, without downloading the whole IPSW:

python3 tools/remotezip.py IPSW_URL DeviceTree.d22ap.im4p research/ipsw-x/dt
ipsw img4 im4p extract research/ipsw-x/dt/Firmware/all_flash/DeviceTree.d22ap.im4p -o research/ipsw-x/d22ap.adt
3. Build the boot image
Needs aarch64-linux-gnu-gcc and mkbootimg.

python3 tools/mkadt.py shim/out/sweet.adt --highmem
python3 tools/mkpayload.py xnu/out/kernel.development.sweet shim/out/sweet.adt shim/out/payload.bin \
    --cmdline 'debug=0xc serial=3 cpus=6 keepsyms=1 -enable_kprintf_spam -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_noconsole=1 sweetfb_hz=30'
shim/build.sh ../shim/out/payload.bin      # -> shim/out/shim-boot.img
What the less obvious boot-args do:

Boot-arg	Effect
rd=ufsroot0 ufsrw=1 rootrw=1	Root on the userdata partition, writable
cpus=6	Start the six A55 cores; cpus=1 is the safe fallback
launchd_unsecure_cache=1	launchd accepts its job cache without AMFI
wdt=-1 wdt_test=no_user_checkin	No hardware watchdog
sweetfb_noconsole=1	Stop drawing kernel text over the GUI. It still goes to the persistent log; panics are drawn
sweetfb_hz=30	Panel refresh rate of the CPU copy
sweettouch=0, sweetkeys=0	Disable the touch or button driver
4. Flash the root and boot
fastboot -S 512M flash userdata ios16-root.simg   # ~7 minutes; do not unplug
# Xiaomi's fastboot often hangs after a large flash: re-enter fastboot with Power + Volume down
fastboot boot shim/out/shim-boot.img
The first boot takes about 4–5 minutes on the Apple logo, then the lock screen appears. Swipe up to unlock. To get back to fastboot, hold Power + Volume down.

5. Read the kernel log
fastboot boot recovery/recovery.img        # OrangeFox, temporary
tools/sweet-log-pull.sh logs/boot.txt      # 4 MiB console ring from the end of userdata
adb reboot bootloader
The log contains crash reports with backtraces (CRASH name ...), IOKit traces of backboardd and SpringBoard, and the drivers' SWEET ... lines. tools/sweet-crash.py symbolicates them against the shared cache.

QEMU
tools/qemu-sweet.py boots the same kernel on QEMU virt with a test RAM-disk root. It is useful for kernel-only changes; see NOTES.md.

Repository layout
Path	What
shim/	Bare-metal boot shim (C + asm) packed as an Android boot.img v2
xnu/patches/	The XNU patch series (applied in order by CI), plus staging scripts for libpthread, HFS and BearSSL
.github/workflows/xnu.yml	Kernel CI build
tools/mkadt.py	Generates the Apple Device Tree from sweet's MMIO layout and the iPhone X identity
tools/mkpayload.py	Packs kernel + ADT + boot-args into the shim payload
tools/mkiosroot.sh	Builds the HFS+ iOS root from the IPSW images
tools/sweet-root-tweaks.py	Bring-up changes to the root (Setup skip, activation, daemons)
tools/sweet-log-pull.sh, sweet-log.py	Read the persistent kernel log from recovery
tools/sweet-crash.py	Symbolicate console crash reports
tools/xnu-cc.py	Local single-file compile check from a CI log
tools/qemu-sweet.py	QEMU test boots with framebuffer text decoding
tools/adt.py, remotezip.py, unpack_bootv2.py, sparsediff.py	Helpers
NOTES.md	Detailed engineering log: every boot, crash and decision, plus hardware facts
Not in git: partition backups, live device trees, recovery images, Apple firmware and root images, and downloaded or generated build trees. They contain device serials and calibration data, or Apple's copyrighted files.

Kernel patches
#	What
0001–0009	Toolchain compatibility, SWEET machine config, non-PAC arm64, stubs for Apple-internal pieces
0010–0018	Early framebuffer console, physmap fix, PRNG without corecrypto, device-tree platform expert, timer PPI, boot CPU driver
0019–0020	Apple open-source libpthread kernel provider; BearSSL AES
0021–0027	RAM-disk root, mockfs, SHA-256, minimal AMFI, Apple open-source HFS+, MD5
0028–0034	UFS host driver and root device, read/write root, DMA above 4 GiB
0035–0045	Entitlements, bindfs, AppleKeyStore stub, IOWatchdog, console crash reports
0046	IOSurface (IOCoreSurfaceRoot) and IOMobileFramebuffer (AppleCLCD) services
0047–0055	Diagnostics: persistent log, /dev/null tee, IOKit trace, heap strings and user stack sampler on crash; permissive Sandbox policy; HFS rename-swap; AppleKeyStore selectors
0056–0057	Throttled panel copy; no external identity resolver
0058	HID event services. Apple's IOHIDFamily-1915.140.3 user client and event queue, matching the iOS 16.7 userland
0059	iPhone X identity (hw.product / hw.target from the device tree)
0060	Power and volume keys over SPMI; touch over GENI I²C
0061	Present the display as 1125×2436
0062	RPMh regulator vote for the touch rail; battery power source
0063	GENI FIFO packing fix and the Goodix GT9896 protocol
0064	Grant PRIV_WORK_INTERVAL (UIKit's CoreAnimation work interval)
0065	SMP: secondary A55 cores via PSCI CPU_ON
0066–0067	Fault-region crash info; console mute
0068	IOSurface lookup semantics for already-mapped surfaces
0069	Batched persistent-log flushes
