#!/bin/bash
# WARNING: this script will destroy data on the selected disk.


# Update the system clock
timedatectl set-ntp true


pacman -Sy --noconfirm pacman-contrib dialog
action=$(dialog --stdout --menu "What to do?" 0 0 0 install "Arch Linux" repair "bootloader") || exit 1


set -euEo pipefail
trap 'echo "${BASH_SOURCE:-unknown}:${LINENO:-unknown}: $BASH_COMMAND";' ERR

# Set up logging
#exec 1> >(tee "stdout.log")
#exec 2> >(tee "stderr.log")

# REPO_URL="https://github.com/moretticb/archlinux/releases/latest/download"
readonly REPO_URL="https://github.com/moretticb/archlinux/releases/download/v0.0.1-alpha"
readonly MIRRORLIST_URL="https://archlinux.org/mirrorlist/?country=NL&protocol=http&protocol=https&use_mirror_status=on"

if [ "$action" = "install" ]; then

	echo "Updating mirror list"
	curl -s "$MIRRORLIST_URL" | \
	    sed -e 's/^#Server/Server/' -e '/^#/d' | \
	    rankmirrors -n 5 - > /etc/pacman.d/mirrorlist

	# Get infomation from user
	readonly hostname=$(dialog --stdout --inputbox "Enter hostname" 0 0) || exit 1
	clear
	: "${hostname:?'hostname cannot be empty'}"

	readonly user=$(dialog --stdout --inputbox "Enter admin username" 0 0) || exit 1
	clear
	: "${user:?'user cannot be empty'}"

	password=$(dialog --stdout --passwordbox "Enter admin password" 0 0) || exit 1
	clear
	: "${password:?'password cannot be empty'}"
	password_confirmation=$(dialog --stdout --passwordbox "Enter admin password again" 0 0) || exit 1
	clear
	[[ "$password" == "$password_confirmation" ]] || ( echo "Passwords did not match"; exit 1; )


	readonly devicelist=$(lsblk -dplnx size -o name,size | grep -Ev "boot|rpmb|loop" | tac)
	# shellcheck disable=SC2086
	readonly device=$(dialog --stdout --menu "Select installation disk" 0 0 0 ${devicelist}) || exit 1
	clear


	# steps for the partition scheme using parted:
	# - remove them all with `rm X`, where X is the number of the partition
	# - Create boot partition:
	#   mkpart
	#   primary
	#   ext2
	#   40
	#   240
	#   
	# - Flag boot partition as boot:
	#   set 1 boot on
	#   
	# - Create swap partition
	#   mkpart
	#   primary
	#   linux-swap(v1)
	#   240
	#   6240
	#   
	# - Create root partition
	#   mkpart
	#   primary
	#   ext4
	#   6240
	#   121G
	#   


	#readonly partlist=$(fdisk -l | awk '//{print $1,$5"__"$7"__"$8"__"$9}' | grep $(echo $device | cut -d"/" -f 3))
	#readonly partlist=$(lsblk | grep $(echo $device | cut -d"/" -f 3) | grep part | sed "s/[^0-9a-zA-Z .]//g" | awk '//{print $1, $4}')
	readonly partlist=$(fdisk -l | grep "^${device}" | sed -E "s/([0-9]) ([0-9])/\1  \2/g" | awk -F" {2,}" '{print $1 " \"" $5,$6 "\""}')

	#readonly ospart=$(dialog --stdout --menu "Select partition to install Arch Linux" 0 0 0 ${partlist}) || exit 1
	eval "$(echo dialog --stdout --menu "\"Select partition to install Arch Linux\"" 0 0 0 ${partlist}) > /tmp/ospart"
	readonly ospart=$(cat /tmp/ospart)

	#readonly swappart=$(dialog --stdout --menu "Select swap partition" 0 0 0 ${partlist}) || exit 1
	eval "$(echo dialog --stdout --menu "\"Select swap partition\"" 0 0 0 ${partlist}) > /tmp/swappart"
	readonly swappart=$(cat /tmp/swappart)

	#readonly efipart=$(dialog --stdout --menu "Select EFI partition" 0 0 0 ${partlist}) || exit 1
	eval "$(echo dialog --stdout --menu "\"Select EFI partition\"" 0 0 0 ${partlist}) > /tmp/efipart"
	readonly efipart=$(cat /tmp/efipart)



	# Format the partitions
	mkfs.ext4 ${ospart}
	mkfs.fat -F 32 ${efipart}


	# initializing and enabling swap
	mkswap ${swappart}


	# mounting root partition
	mount ${ospart} /mnt
	mkdir /mnt/boot
	mount ${efipart} /mnt/boot


	swapon ${swappart}


	# Install essential packages
	while pacstrap /mnt base base-devel linux linux-firmware; ret=$?; [ $ret -ne 0 ]; do
		dialog --title "unable to pacstrap" --backtitle "pacstrap error" --yesno "Do you want to try again?" 0 0
		if [ "$?" -ne "0" ]; then
			exit 1
		fi

		# sleep, so ctrl+c can break this infinite loop if no internet
		sleep 1
	done


	# Generate fstab
	genfstab -t PARTUUID /mnt >> /mnt/etc/fstab



	# Setting time zone
	readonly timezone=$(dialog --stdout --title "Set a timezone" --fselect /mnt/usr/share/zoneinfo/ 15 70) || exit 1
	clear
	ln -sf ${timezone} /mnt/etc/localtime


	# Running hwclock according to the selected timezone
	arch-chroot /mnt hwclock --systohc


	# Localization
	#locales=$(dialog --stdout --checklist "Select en_US.UTF-8 UTF-8 and other needed locales" 0 0 0 $(cat /mnt/etc/locale.gen | grep -E "^#[a-z]{2}" | cut -d"#" -f 2 | sed "s/[ ]*$/ off/g")) || exit 1
	#clear
	#
	#cat /mnt/etc/locale.gen | grep -E $(echo $locales | sed "s/ /|/g") | grep "#" | cut -d"#" -f 2 > /tmp/temp_locales.txt
	#cat /tmp/temp_locales.txt >> /mnt/etc/locale.gen
	sed 's/#pt_BR.UTF-8/pt_BR.UTF-8/' -i /mnt/etc/locale.gen
	sed 's/#en_US.UTF-8/en_US.UTF-8/' -i /mnt/etc/locale.gen

	#echo "LANG=$(cat /tmp/temp_locales.txt | head -1 | cut -d" " -f 1)" > /mnt/etc/locale.conf
	echo "LANG=en_US.UTF-8" > /mnt/etc/locale.conf


	arch-chroot /mnt locale-gen



	# Setting hostname
	echo "${hostname}" > /mnt/etc/hostname


cat >>/mnt/etc/pacman.conf <<EOF

[multilib-testing]
Include = /etc/pacman.d/mirrorlist

[multilib]
Include = /etc/pacman.d/mirrorlist
EOF

	# refreshing urls from multilib and multilib-testing
	arch-chroot /mnt pacman -Syy



	#arch-chroot /mnt bootctl install
	#
	#cat <<EOF > /mnt/boot/loader/loader.conf
	#default arch
	#EOF
	#
	#cat <<EOF > /mnt/boot/loader/entries/arch.conf
	#title    Arch Linux
	#linux    /vmlinuz-linux
	#initrd   /intel-ucode.img
	#initrd   /initramfs-linux.img
	#options  root=PARTUUID=$(blkid -s PARTUUID -o value "$part_root") rw
	#EOF


	arch-chroot /mnt useradd -mU -s /bin/bash -G wheel "$user"

	sed -Ei "/^#[ ]*%wheel[^NOPSWD]*$/s/^#[ ]*//g" /mnt/etc/sudoers
	#cat /mnt/etc/sudoers | sed -E "s/^#[ ](%wheel ALL=(ALL) ALL)$/\1/g" > /tmp/tmp_sudoers
	#cat /tmp/tmp_sudoers > /mnt/etc/sudoers

	arch-chroot /mnt chsh -s /bin/bash root
	arch-chroot /mnt chsh -s /bin/bash "$user"

	echo "$user:$password" | chpasswd --root /mnt
	echo "root:$password" | chpasswd --root /mnt




	# XPS13 9343 DRIVERS

	# wifi driver (assuming linux-headers has already been installed)
	arch-chroot /mnt pacman -S --noconfirm broadcom-wl networkmanager bluez bluez-tools bluez-utils grub efibootmgr

	# End of XPS13 9343 DRIVERS



	arch-chroot /mnt systemctl enable NetworkManager.service


	# add bluetooth config here I removed before



	# downloading post install script, leaving at user home directory
	curl -sL "https://raw.githubusercontent.com/$(echo ${REPO_URL} | cut -d'/' -f 4,5)/develop/post_install" > /mnt/home/$user/post_install
	arch-chroot /mnt chmod +x /home/$user/post_install

fi


if [ "$action" = "repair" ]; then

	readonly devicelist=$(lsblk -dplnx size -o name,size | grep -Ev "boot|rpmb|loop" | tac)
	readonly device=$(dialog --stdout --menu "Select installation disk" 0 0 0 ${devicelist}) || exit 1
	clear

	readonly partlist=$(fdisk -l | grep "^${device}" | sed -E "s/([0-9]) ([0-9])/\1  \2/g" | awk -F" {2,}" '{print $1 " \"" $5,$6 "\""}')

	eval "$(echo dialog --stdout --menu "\"Select Arch Linux partition\"" 0 0 0 ${partlist}) > /tmp/ospart"
	readonly ospart=$(cat /tmp/ospart)

	eval "$(echo dialog --stdout --menu "\"Select EFI partition\"" 0 0 0 ${partlist}) > /tmp/efipart"
	readonly efipart=$(cat /tmp/efipart)

	# mounting root partition
	mount ${ospart} /mnt

fi

arch-chroot /mnt grub-install $device --recheck

arch-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg



echo "Installation finished."

echo "IMPORTANT: Have a bootable MacOS device, boot into it and open the terminal. Follow these steps:"
echo "1. run `diskutil list` and check for the device referring to the boot partition (e.g., `/dev/disk0s1`)"
echo "2. run `bless --device /dev/DEVICE_GOES_HERE --setBoot --legacy --verbose`"
echo "3. reboot and grub must appear normally"
