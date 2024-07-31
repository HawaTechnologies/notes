sudThese are the console specifications:

1. SBC brand & model: `Banana Pi BPI-M5`.
2. Processor: Amlogic S905X3.
3. RAM memory: 4GB.
4. Storage: 16GB eMMC.
	1. Partitions: We can list it with `lsblk`. Contents are:
    ```
    mmcblk0 (disk)
    - mmcblk0p1 (part /boot)
    - mmcblk0p2 (part /)
    - mmcblk0p3 (part /mnt/CURRENT_SAVE)
    - mmcblk0p4 (part /mnt/SAVES)
    ```
1. OS: `Linux Raspberry 4.9.241-BPI-M5 #2 SMP PREEMPT Mon Jun 21 16:29:40 HKT 2021 aarch64 GNU/Linux` (obtained with: `uname -a`).
2. Compatibility:
	1. External WIFI adapter:
		1. brand & model: `TL-WN725N Wi-Fi`.
		2. driver: `rtl8188eu`.
	2. HDMI displays:
		1. Resolution: 1920x1080 32-bit color __(There's an ongoing issue which sometimes downgrades the resolution to 800x600, but the idea is to keep it as stated)__.
	3. Sound:
		1. ALSA drivers.
3. USB Ports: A total of 4 ports are provided in this SBC.
	1. Reserved ports: 1.
		1. Intended for the External WIFI adapter.
	2. Remaining ports: 3.
4. Default users' passwords: `Admin$2024`.
	1. This password stands both for `root` and `pi`.
	2. With `sudo passwd` the password can be changed for `root`.
	3. With `passwd` the password can be changed for `pi`.
		1. Also with: Start > Preferences > Raspberry Pi Configuration.
			1. Button: `Change Password`.

Currently, our OS fork image is stored at: __TODO__.