# Creating Proxmox Templates with cloud-init

Collection of example cloud-init config files and shell scripts to download cloud-images and create
[Proxmox templates].

## Shell Scripts

- [`build-template`](/scripts/build-template): Create a proxmox template.
- [`image-update`](/scripts/image-update): Check for an updated image and
  download the latest or missing image.
  - Works with `debian` and `ubuntu` cloud images.
  - **Requires** `curl` and `qemu`
  - **Note**: This script will convert `*.qcow2` images to a raw format,
    `*.img`.

## Use

- Copy the [scripts](/scripts/) into `/usr/local/bin` on your Proxmox node(s).
- Confirm `/usr/local/bin` is added to `PATH`

  ```bash
  admin@proxmox:~$ echo $PATH
  /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

  # test using tab-completion
  image[TAB] -> image-update
  ```

### Image Download / Update Script

```bash
# SSH into your Proxmox host
user@desktop:~$ ssh admin@proxmox
# change into the default iso storage folder
admin@proxmox:~$ cd /var/lib/vz/template/iso/
```

Debian:

```bash
image-update -d debian -r 10
```

Ubuntu:

```bash
image-update -d ubuntu -r 20
```

| CLI Flags         | Description                                                         | Required |
| ----------------- | ------------------------------------------------------------------- | -------- |
| `--distro`, `-d`  | Specify the distribution name, e.g. `ubuntu`.                       | Yes      |
| `--release`, `-r` | Specify the release name, e.g. `focal` or `20`.                     | Yes      |
| `--storage`, `-s` | Specify the image storage path, default: `/var/lib/vz/template/iso` | No       |
| `--keep`, `-k`    | Keep qcow2/raw image after converting, default: `false`             | No       |
| `--remove`        | Remove old images before updating, default: `false`                 | No       |

Images are backed up prior to downloading new images under the naming scheme:
`IMAGE_NAME.backup-$DATE.FILE_EXTENSION`. Adding the `--remove` flag will remove
the original image, converted image, and backups.

### Proxmox Template Script

```bash
build-template -i <vm_id> -n <vm_name> --img <vm_image>
```

Example:

```bash
build-template -i 9000 -n ubuntu20 --img /var/lib/vz/template/iso/ubuntu-20.04-server-cloudimg-amd64.img
```

| CLI Flags          | Default           | Description                                                                    | Required |
| ------------------ | ----------------- | ------------------------------------------------------------------------------ | -------- |
| `--id`, `-i`       | `9000`            | Specify the VM ID (1-999999999)                                                | **Yes**  |
| `--name`, `-n`     | `template`        | Specify the VM name                                                            | **Yes**  |
| `--image`, `--img` |                   | Specify the VM image (path), ex: `/PATH/TO/image.{img,iso,qcow2}`              | **Yes**  |
| `--bios`, `-b`     | `seabios`         | Specify the VM bios, `seabios` or `ovfm`                                       | No       |
| `--machine`        | `q35`             | Specify the VM machine type, `q35` or `pc` for i440                            | No       |
| `--memory`, `-m`   | `1024`            | Specify the VM memory                                                          | No       |
| `--net-bridge`     | `vmbr0`           | Specify the VM network bridge                                                  | No       |
| `--net-type`       | `virtio`          | Specify the VM network type                                                    | No       |
| `--net-vlan`       | `1`               | Specify the VM network vlan tag                                                | No       |
| `--os`             | `l26`             | Specify the VM OS                                                              | No       |
| `--resize`         | `1G`              | Increase the VM boot disk size                                                 | No       |
| `--storage`, `-s`  | `local-lvm`       | Specify the VM storage, ex: `local-lvm`, `local-zfs`                           | No       |
| `--scsihw`         | `virtio-scsi-pci` | Specify the VM storage controller, ex: `virtio-scsi-pci`, `virtio-scsi-single` | No       |

## Maintainers & License

Taylor Fore (https://github.com/trfore)

See [LICENSE](LICENSE) File

## References

Proxmox:

- [Proxmox]
- [Proxmox templates]

Linux Cloud Images:

- [Debian Cloud Images]
  - Default User: `debian`
  - Uses `debian-1x-generic-amd64.qcow2`
- [Ubuntu Cloud Images]
  - Default User: `ubuntu`
  - Uses `ubuntu-20.04-server-cloudimg-amd64.img`

[Debian Cloud Images]: https://cloud.debian.org/images/cloud/
[Ubuntu Cloud Images]: https://cloud-images.ubuntu.com/releases/
[Proxmox]: https://www.proxmox.com/
[Proxmox templates]: https://pve.proxmox.com/wiki/VM_Templates_and_Clones
