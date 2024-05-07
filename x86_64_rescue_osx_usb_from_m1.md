# Prep a rescue bootable x86_64 OSX usb device in an M1 machine

Extracted from https://apple.stackexchange.com/questions/418100/create-an-el-capitan-rescue-usb-using-a-modern-m1-mac

Start by downloading the El Capitan dmg from this link: https://support.apple.com/en-us/HT211683

Then open the dmg and copy `InstallMacOSX.pkg` on the Desktop.

Then, from the terminal (Applications/Utilities) un-compact the `InstallMacOSX.pkg` file in a directory (Installer for example) which will be created by the following `pkgutil` command:

```bash
pkgutil --expand ~/Desktop/InstallMacOSX.pkg ~/Desktop/Installer
```

Then position yourself in the `InstallMacOSX.pkg` "package" created by the `pkgutil` command:

```bash
cd ~/Desktop/Installer/InstallMacOSX.pkg
```

Then un-compact the structure using the `tar` command:

```bash
tar -xvf Payload
```

Finally move the `InstallESD.dmg` file created by the `tar` command above to the Desktop:

```bash
mv InstallESD.dmg ~/Desktop
```

You must then format a GUID partition scheme USB key of sufficient size (8 GB for El Capitan) named `KEY` in the example and execute the following instructions:

```bash
hdiutil attach ~/Desktop/InstallESD.dmg -noverify -nobrowse -mountpoint /Volumes/install_app
hdiutil convert /Volumes/install_app/BaseSystem.dmg -format UDSP -o /tmp/Installer
hdiutil resize -size 8g /tmp/Installer.sparseimage
hdiutil attach /tmp/Installer.sparseimage -noverify -nobrowse -mountpoint /Volumes/install_build
rm -r /Volumes/install_build/System/Installation/Packages
cp -av /Volumes/install_app/Packages /Volumes/install_build/System/Installation/
cp -av /Volumes/install_app/BaseSystem.chunklist /Volumes/install_build
cp -av /Volumes/install_app/BaseSystem.dmg /Volumes/install_build
hdiutil detach /Volumes/install_app
hdiutil detach /Volumes/install_build
hdiutil resize -size `hdiutil resize -limits /tmp/Installer.sparseimage | tail -n 1 | awk '{print $ 1}' `b /tmp/Installer.sparseimage
hdiutil convert /tmp/Installer.sparseimage -format UDZO -o /tmp/Installer
mv /tmp/Installer.dmg ~/Desktop
```

Here you have to plug the USB key named `KEY`, then:

```bash
sudo asr restore --source ~/Desktop/Installer.dmg --target /Volumes/KEY --noprompt --noverify --erase
```

Test the key and if ok, delete the working directories and files of this operation from the Desktop.
