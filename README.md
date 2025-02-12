# llama.zero

This is a fork of llama.cpp that can compile on Pi Zero or Pi 1 or on any arm1176jzf device. It does this by modifying CMake build files to not recognize armv6 as an architecture with neon support. The original repo forces the build to use unsupported instructions, making it run into inevitable failure.

Furthermore, this repo also includes instructions on how to make an AI assist USB device.

## How to set up Llama.zero on Pi Zero as USB device

### USB Module Setup
Reference can be found [here](https://gist.github.com/gbaman/50b6cca61dd1c3f88f41)

1. Add dwc2 device tree overlay
```bash
# This gets appended to /boot/config.txt
echo "dtoverlay=dwc2" | sudo tee -a /boot/config.txt
```
2. In /boot/cmdline.txt, insert **modules-load=dwc2,g_multi** after root_wait.
```bash
sudo sed -i 's/rootwait/rootwait modules-load=dwc2,g_multi/' /boot/cmdline.txt
```
4. Create image file for USB mass storage
```bash
sudo dd if=/dev/zero of=/llamazero.img bs=1M count=64
sudo mkfs.vfat /llamazero.img
```
4. Create mount directory
```bash
sudo mkdir /mnt/llamazero
sudo mount /llamazero.img /mnt/llamazero
```
5. Enable USB kernel module
```bash
sudo modprobe g_multi file=/llamazero.img cdrom=0 ro=0
```
6. Add mount command to rc.local, so system auto mount when pi starts.
```bash
sudo sed -i '/exit 0/i modprobe g_multi file=/llamazero.img cdrom=0 ro=0' /etc/rc.local
```

#### Useful commands
```bash
# Unmount fs
sudo umount /mnt/llamazero
# Mount fs
sudo mount /llamazero.img /mnt/llamazero
# Mount usb
sudo modprobe g_multi file=/llamazero.img cdrom=0 ro=0
# Unmount usb
sudo modprobe g_multi -r
```
### Behavior

To check for new files created on computer, we:

```bash
sudo umount /mnt/llamazero
sudo mount /llamazero.img /mnt/llamazero
sudo ls /mnt/llamazero
```

To show for new edits created on pi, mount and unmount usb module:
```bash
# While the file system folder is mounted
sudo modprobe -r g_multi
sudo modprobe g_multi file=/llamazero.img cdrom=0 ro=0
```

### Auto-updater script
```bash
#!/bin/bash

MOUNT_POINT=/mnt/llamazero
IMG_FILE=/llamazero.img

while true; do
    # Unmount and mount to detect new files from computer
    sudo umount $MOUNT_POINT
    sudo mount $IMG_FILE $MOUNT_POINT
    
    # Check for empty files and append "hello"
    for file in $(sudo find $MOUNT_POINT -type f -empty); do
	echo "Processing file: $(basename "$file")"
    	echo "$file" | sudo tee -a "$file" > /dev/null
	sleep 1
        # Unmount and remount USB gadget to show changes
	echo "Unmount usb"
        sudo modprobe -r g_multi
	sleep 1
	echo "Mount USB"
        sudo modprobe g_multi file=$IMG_FILE cdrom=0 ro=0
    done
    
    sleep 2  # Adjust sleep time as needed
done
```

### Memory set-up for llama.cpp
Allocate memory for llama.cpp compilation
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
### Compile llama.cpp

1. Install dependencies
```bash
sudo apt install -y git cmake ccache libpthread-stubs0-dev build-essential
```
2. Clone this repo
```bash
git clone https://github.com/pham-tuan-binh/llama.zero.git
cd llama.zero
```
3. Build
This will take a very long time. My first successful compilation took 24 hours, no cap, but it was because I had some errors. Hopefully yours wont.
```bash
cmake -B build
cmake --build build --config Release
```
4. Add to path
```bash
echo 'export PATH=$PATH:~/llama.zero/build/bin/' >> ~/.bashrc
```
5. Test run
You can test any model you want, as long as they are <512MB and is in gguf format.
```
llama-cli -m model.gguf -p "The meaning of life is" -n 16 2> /dev/null
```
### Auto fill in USB file script
Make this script into a systemctl service so that it is run whenever you plug in your pi.
```bash
#!/bin/bash

MOUNT_POINT=/mnt/llamazero
IMG_FILE=/llamazero.img

while true; do
    # Unmount and mount to detect new files from computer
    sudo umount $MOUNT_POINT
    sudo mount $IMG_FILE $MOUNT_POINT

    # Check for empty files and append "hello"
    for file in $(sudo find $MOUNT_POINT -type f -empty); do
        para=$(basename "$file")
        echo "Processing file: $file"
        llama-cli -n 16 -p "$para" -m ~/tiny-15M-Q4KM.gguf 2> /dev/null | sudo tee $file

        sleep 1
        # Unmount and remount USB gadget to show changes
        echo "Unmount usb"
        sudo modprobe -r g_multi
        sleep 1
        echo "Mount USB"
        sudo modprobe g_multi file=$IMG_FILE cdrom=0 ro=0
    done

    sleep 2  # Adjust sleep time as needed
done
```
