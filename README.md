## Some additional notes

### Partitions

Here are some sample partitions:
| name | size | file system     | flags          |
|------|------|-----------------|----------------|
| boot | 500M | fat32           | boot, msftdata |
| swap | 8G   | linux-swap (v1) | swap           |
| root | *    | ext4            |                |

Using `parted` you can get close to these while maintaining alignment with the following:

```shell
parted --align=opt --fix --script <target-device> \
  "print free" \
  "mklabel gpt" \ # creates the partition label/table
  "mkpart boot fat32 0% 500MiB" "set 1 boot on" \ # creates the boot partitions and sets the boot flag
  "mkpart swap linux-swap 500MiB 8GiB" \ # creates the swap partition, you can use `linux-swap(new)` if you want to
  "mkpart root ext4 8GiB 100%" \ # creates the partition that will be used to install arch on
  "print free"
```

> Note the use of the `MiB` and `GiB` units above instead of `M` and `G` to specify exact blocks instead of letting parted mess around with unpredictable auto-alignment attempts. If you want more precision, calculate exactly what you want and specify it in `s` (sectors).

### Encryption

Once you've encrypted the device and created teh filesystem you'll need to tell the boot loader how to decrypt it before booting arch. This requires you to modify some configs. We'll assume you're using `grub` and `mkinitcpio` for the below examples.

First, `mkinitcpio` hooks (modify before running any mkinitcpio commands):
```shell
# /etc/mkinitcpio.conf

# You probably want
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

Next, we need to configure `grub` kernel params (modify before running any mkinitcpio commands):
```shell
# /etc/default/grub

# Ensure we tell grub that we have crypto disks
GRUB_ENABLE_CRYPTODISK=y

# Tell grub which disks to decrypt and where to mount them
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<device-uuid>:root root=<mapped-device-uuid>"
```

- `<device-uuid>`: The UUID of the device, for instance `blkid /dev/sda`
- `<mapped-device-uuid>`: The UUID of the decrypted & mapped device, for instance `blkid /dev/mapper/root`

> To regenerate `grub` config for boot you can run `grub-mkconfig -o /boot/grub/grub.cfg`

Extra notes:
- If you are on a dynamically growing virtual disk, don't pump it full of bytes...
- I usually dont encrypt swap, but if you want to check the [arch wiki](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system).

> For more information on encryption check out the [arch wiki](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system).