# OMV-USB-Automount
Automount of USB storage device on OMV-powered NAS.
Built with ChatGPT-4.1
Originally shared here : https://github.com/openmediavault/openmediavault/issues/1913

# Step-by- step

**A. Create a script for the automatic mounting of USB storage devices to the /media/usbdevices/ directory upon connection**

`sudo nano /usr/local/bin/mount_usb.sh
`

```
#!/bin/bash

# Define full paths for commands to work properly with udev
BLKID="/usr/sbin/blkid"
MOUNT="/bin/mount"
MKDIR="/bin/mkdir"
LOG_FILE="/var/log/usb_mount.log"

# Directory where USB devices will be mounted
MOUNT_DIR="/media/usbdevices"

# Create the mount directory if it doesn't exist
$MKDIR -p "$MOUNT_DIR"

# Function to log messages
log_message() {
    echo "$(date): $1" >> "$LOG_FILE"
}

# Function to mount USB devices
mount_usb() {
    DEVNAME=$1
    log_message "Attempting to mount $DEVNAME"
    LABEL=$($BLKID -o value -s LABEL "$DEVNAME")
    if [ -z "$LABEL" ]; then
        LABEL=$($BLKID -o value -s UUID "$DEVNAME")
    fi
    MOUNT_POINT="$MOUNT_DIR/$LABEL"
    $MKDIR -p "$MOUNT_POINT"
    /usr/bin/systemd-mount --no-block --options=rw "$DEVNAME" "$MOUNT_POINT"
    if [ $? -eq 0 ]; then
        log_message "Successfully mounted $DEVNAME at $MOUNT_POINT"
    else
        log_message "Failed to mount $DEVNAME"
    fi
}

# Check if DEVNAME is passed as an argument
if [ -z "$1" ]; then
    log_message "Error: No device name provided"
    exit 1
fi

mount_usb "$1"
```

**B. Create a script for the automatic unmounting of USB storage devices**

`sudo nano /usr/local/bin/unmount_usb.sh
`

```
#!/bin/bash

LOG_FILE="/var/log/usb_mount.log"

# Function to log messages
log_message() {
    echo "$(date): $1" >> "$LOG_FILE"
}

# Function to unmount USB devices
unmount_usb() {
    DEVNAME=$1
    log_message "Attempting to unmount $DEVNAME"

    # Get the mount point using `findmnt`
    MOUNT_POINT=$(findmnt -n -o TARGET --source "$DEVNAME")

    if [ -n "$MOUNT_POINT" ]; then
        systemd-umount "$MOUNT_POINT"
        sleep 2  # Give time for unmount to complete

        # Check if the directory is empty, then remove it
        if [ -d "$MOUNT_POINT" ]; then
            rm -rf "$MOUNT_POINT"
            log_message "Successfully unmounted and removed $MOUNT_POINT"
        else
            log_message "Unmounted $DEVNAME, but $MOUNT_POINT was already removed"
        fi
    else
        log_message "No mount point found for $DEVNAME"
    fi
}

unmount_usb "$1"
```


**C. Make both scripts executable**

`sudo chmod +x /usr/local/bin/mount_usb.sh
`
`sudo chmod +x /usr/local/bin/unmount_usb.sh
`


III. Create a udev rule to call the mounting and unmounting scripts when the USB drive is manually plugged in or unplugged

```
ACTION=="add", SUBSYSTEM=="block", KERNEL=="sd[a-z][0-9]", ENV{ID_BUS}=="usb", RUN+="/bin/bash -c 'sleep 2; /usr/local/bin/mount_usb.sh \"/dev/%k\" &'"
ACTION=="remove", SUBSYSTEM=="block", KERNEL=="sd[a-z][0-9]", ENV{ID_BUS}=="usb", RUN+="/bin/bash -c '/usr/local/bin/unmount_usb.sh \"/dev/%k\" &'"
```

**D. Load the udev rules**

`sudo udevadm control --reload-rules
`
