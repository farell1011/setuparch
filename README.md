### install arch linux systemd-boot gummiboot full disk encrypt gpt for my setup
#### dont do your data delete permanent
	head -c 1052672 /dev/urandom > /dev/sdaX; sync
	blkdiscard /dev/sda

#### Set the keyboard layout
	loadkeys us

#### Verify the boot mode
	ls /sys/firmware/efi/efivars

#### Connect to the internet
	iwctl
	device list
	station wlan0 scan
	station wlan0 get-networks
	station wlan0 connect SSID
	exit
	ping archlinux.org

#### Update the system clock
	timedatectl set-ntp true

#### Partition the disks
	gdisk /dev/sda
	#### EFI partition
	o		create gpt partition scheme
	w		write
	n		create a new partition
	1		default partition number (1)
	default	 first	First sector: leave blank
	260M		Last sector
	ef00		partition type code – EFI system

	#### Encrypted partition
	n		create a new partition
	2		default partition number (2)
	default		First sector: leave blank
	default last	Last sector
	8e00		partition type code – Linux LVM
	w		write
	q

#### setup encrypt
	cryptsetup --type luks2 --cipher aes-xts-plain64 --hash sha256 --iter-time 2000 --key-size 256 --pbkdf argon2i --sector-size 512 --use-urandom --verify-passphrase luksFormat /dev/sda2

#### setup encrypt open
	cryptsetup --allow-discards --persistent luksOpen /dev/sda2 encryptluks

#### setup encrypt physical volume
	pvcreate /dev/mapper/encryptluks

#### setup encrypt create a volume group
	vgcreate encryptgp /dev/mapper/encryptluks

#### setup encrypt Create logical volumes
	lvcreate --size 8G encryptgp --name swap
	lvcreate -l 100%FREE encryptgp --name root

#### Format the partitions
	mkfs.vfat -F32 /dev/sda1
	mkfs.ext4 /dev/mapper/encryptgp-root
	mkswap /dev/mapper/encryptgp-swap

#### Mount the file systems
	mount /dev/mapper/encryptgp-root /mnt 
	swapon /dev/mapper/encryptgp-swap 
	mkdir -p /mnt/boot
	mount /dev/sda1 /mnt/boot

#### Update the mirrors
	reflector --score 100 --fastest 25 --sort rate --save /etc/pacman.d/mirrorlist --verbos

#### Install essential packages
	pacstrap /mnt base base-devel linux linux-headers linux-firmware intel-ucode zsh git nano lvm2 networkmanager network-manager-applet lzop dnscrypt-proxy broadcom-wl terminus-font

#### Configure the system
##### fstab
	genfstab -U /mnt >> /mnt/etc/fstab

##### Chroot
	arch-chroot /mnt

##### Time zone
	ln -sf /usr/share/zoneinfo/Zone/City /etc/localtime
	hwclock --systohc

##### Localization
###### Edit /etc/locale.gen uncomment
	en_US.UTF-8 UTF-8

###### create /etc/locale.conf
	LANG=en_US.UTF-8

###### create /etc/vconsole.conf
	FONT=ter-p14b
	KEYMAP=us

###### generate locale
	locale-gen

##### Network configuration
###### create /etc/hostname
	myhostname

###### Add matching entries /etc/hosts
	127.0.0.1	localhost
	127.0.1.1	myhostname.localdomain	myhostname

###### edit /etc/resolv.conf
	nameserver 127.0.0.1

###### write-protect
	chattr +i /etc/resolv.conf

##### install boot loader
	bootctl install

###### edit /boot/loader/loader.conf
	default  arch
	timeout  4
	console-mode max
	editor   no
	auto-entries 0
	auto-firmware 0
	random-seed-mode off

###### edit /boot/loader/entries/arch.conf
	title   Arch Linux
	linux   /vmlinuz-linux
	initrd  /intel-ucode.img
	initrd  /initramfs-linux.img
	options rd.luks.name=UUIDofLuksPartition=encryptluks root=/dev/mapper/encryptgp-root rw resume=/dev/mapper/encryptgp-swap ipv6.disable=1 net.ifnames=0 biosdevname=0 audit=0

###### to get UUIDofLuksPartition
	blkid -s UUID -o value /dev/sda2 encryptluks >> /boot/loader/entries/arch.conf

##### initramfs

###### edit /etc/mkinitcpio.conf
	HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt lvm2 filesystems fsck)

	COMPRESSION="lzop"

###### generate initramfs
	mkinitcpio -P

##### root password
	passwd 

##### add user
	useradd -m -G wheel -s /usr/bin/zsh username
	passwd username

##### edit /etc/sudoers uncomment
	wheel ALL=(ALL) ALL

##### set group wheel to use sudo with no password edit /etc/pam.d/sudo
	auth           sufficient      pam_wheel.so trust use_uid

##### enable network
	systemctl enable NetworkManager.service
	systemctl enable dnscrypt-proxy.service

##### exit chroot
	exit
	umount -R /mnt
	swapoff -a
	reboot

##### login as user
##### connect wifi
	nmcli device wifi list
	nmcli device wifi connect SSID_or_BSSID password password
	nmcli device wifi connect SSID_or_BSSID password password hidden yes
	nmcli device disconnect ifname wlan0
	nmcli radio wifi off
	
##### restore keyboard, mouse conf, firmware bluethooth, icons blueman
	git clone https://github.com/farell1011/setuparch.git
	cd setuparch
	cp etc/X11/xorg.conf.d/* /etc/X11/xorg.conf.d
	cp lib/firmware/brcm/* /lib/firmware/brcm
	cp usr/share/icons/hicolor/16x16/status/* /usr/share/icons/hicolor/16x16/status

##### restore dotfile dengan chezmoi
[my_dot_config](https://github.com/farell1011/dots)
