# SteamOS `evdi` and SMI USB display driver

This repo contains `PKGBUILD` files for installing `evdi` and SMI USB display
drivers on the Steam Deck (steamOS). Status is "works on my machine".

I use this to use a second display with
[this HyperDrive USB-C dock](https://www.hypershop.com/products/hyperdrive-dual-4k-hdmi-10-in-1-usb-c-hub),
since the provided "SiliconMotion InstantView" driver only works on macOS and
Windows. There is also a semi-supported
[Ubuntu driver](https://hypershop.zendesk.com/hc/en-us/articles/10344404131988-Linux-Driver-Download),
which is what this package is based on (reworked for Arch / SteamOS).

## Build

The easiest way to build everything is in a container, I recommend
[`distrobox`](https://github.com/89luca89/distrobox) along with the
[`SteamDeckHomebrew/holo-base`](https://github.com/SteamDeckHomebrew/holo-docker)
container images.

```sh
# Make sure this image matches your host OS version, I think as long as the kernel
# packages match it should work, even if the exact OS version is different.
# It should be possible to do with plain docker/podman too, I just prefer distrobox
# for easy filesystem synchronization.
distrobox ephemeral --image ghcr.io/steamdeckhomebrew/holo-base:latest

# ========================
# Inside the container
# ========================

# Install valve upstream kernel headers:
sudo pacman -Syu --needed linux-neptune-headers
# OR on SteamOS 3.5 or higher (kernel upgraded to 6.1):
sudo pacman -Syu --needed linux-neptune-61-headers

# Build evdi
cd evdi-steamos-dkms
makepkg -si

# Build dkms module. You may need to adjust version numbers here to match your system
sudo dkms build evdi/1.14.1 -k "$(uname -r)"

# Export dkms module somewhere the host can access it
sudo dkms mktarball evdi/1.14.1 -k "$(uname -r)"
cp /var/lib/dkms/evdi/1.14.1/tarball/*.tar.gz ~/Downloads/

# Build the SMI usb driver itself
cd ../smiusbdisplay-driver-bin
makepkg -s
```

## Install

Now that the kernel module and driver packages are built, install them on the host.
This will need to be done again every time SteamOS is upgraded (but the rebuild
should only be needed if the kernel was upgraded).

```sh
# Disable read-only protection
sudo steamos-readonly disable

# Load and install the previously built dkms archive
sudo dkms ldtarball ~/Downloads/evdi-*.dkms.tar.gz
# Adjust version numbers as needed:
sudo dkms install evdi/1.14.1 $(uname -r)

# Install the built SMI usb driver
cd smiusbdisplay-driver-bin
sudo pacman -U smiusbdisplay-driver-bin-2.12.0.0-1-x86_64.pkg.tar.zst

# Re-enable readonly protection (optional)
sudo steamos-readonly enable

# Reboot for the installation to take effect
systemctl reboot
```
