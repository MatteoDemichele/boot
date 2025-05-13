AUR packages:
- https://aur.archlinux.org/packages/clevis-pin-fido2-git
- https://aur.archlinux.org/packages/mkinitcpio-clevis-hook

Packages required for the modified clevis install hook to properly work:
- https://archlinux.org/packages/extra/x86_64/libfido2/

The patch files can be used as follow:
```sh
patch /usr/bin/clevis-decrypt-fido2 --output /usr/bin/initramfs-clevis-decrypt-fido2 < clevis-decrypt-fido2.patch
patch /etc/initcpio/install/clevis < initcpio-install-clevis.patch
patch /etc/initcpio/hooks/clevis < initcpio-hooks-clevis.patch
```

To create a loop device to test encryption with clevis without risking a real device:
```sh
truncate --size 50M /tmp/test_volume
sudo cryptsetup luksFormat /tmp/test_volume
sudo cryptsetup open /tmp/test_volume my_super_volume
sudo mkfs.ext4 /dev/mapper/my_super_volume
sudo mount /dev/mapper/my_super_volume /mnt
echo hello tpm + fido | sudo tee /mnt/file.txt > /dev/null
sudo umount /mnt
sudo cryptsetup close my_super_volume
```

Note: in the following commands, to encrypt or decrypt a volume using the TPM, `clevis` must
have access to `/dev/tpmrm*`, which means that it must either be run as root, or as a user in the
`tss` group.

To encrypt a device using a combination of TPM and FIDO2 with clevis:
```sh
clevis luks bind -d /tmp/test_volume sss '{
    "t": 2,
    "pins": {
        "tpm2": [
            {
                "pcr_ids": "7",
                "pcr_bank": "sha256"
            }
        ],
        "fido2": [
            {
                "pin": true
            }
        ]
    }
}'
```

To test if the device has been properly encrypted:
```sh
clevis luks unlock -d /tmp/test_volume -n my_super_volume
sudo mount /dev/mapper/my_super_volume /mnt
sudo cat /mnt/file.txt
sudo umount /mnt
sudo cryptsetup close my_super_volume
```

From here:
- apply the same process for a real device
- ensure the encrypted device has been configured in the kernel cmdline (for example in
`/etc/kernel/cmdline` if using UKI)
- add `clevis` in the list of mkinitcpio hooks (the file `mkinitcpio.conf` in this repo is an
  example)
- regenerate the initramfs with `mkinitcpio --allpresets`
- reboot, you will be prompted for the FIDO2 PIN code to decrypt the device
