# Creating Proxmox Templates with cloud-init

Collection of example cloud-init config files and shell scripts to download cloud-images and create
[Proxmox templates].

## Shell Scripts

- [`build-template`](/scripts/build-template): Create a proxmox template.
- [`image-update`](/scripts/image-update): Check for an updated image and
  download the latest or missing image.
  - Works with `centos`, `debian` and `ubuntu` cloud images.
  - Limited support for `fedora`, as the image value is hardcoded to the latest
    version.
  - **Requires** `curl`
  - **Note**: This script will change the file extension to `*.img` for visibility in the Proxmox GUI, `qm disk import`
    will automatically convert disk image.

## Use

- Copy the [scripts](/scripts/) into `/usr/local/bin` on your Proxmox node(s).
- Change the scripts ownership and permissions:

  ```bash
  chown root:root /usr/local/bin/{build-template,image-update}
  chmod +x /usr/local/bin/{build-template,image-update}
  ```

- Confirm `/usr/local/bin` is added to `PATH`:

  ```bash
  root@pve:~$ echo $PATH
  /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

  # test using tab-completion
  image[TAB] -> image-update
  ```

### Image Download / Update Script

```bash
# SSH into your Proxmox host
user@desktop:~$ ssh root@pve

# usage
root@pve:~$ image-update -d <DISTRO_NAME> -r <RELEASE_NAME> [ARGS]
```

<details>
  <summary>Code Examples</summary>

```bash
image-update -d centos -r 9
```

```bash
image-update -d debian -r 10
```

```bash
image-update -d ubuntu -r 20
```

</details>

<details>
  <summary>Alternative Usage</summary>

```bash
# usage
image-update <DISTRO_NAME>-<RELEASE_NAME> [ARGS]
# example
image-update ubuntu-20 --remove
```

</details>

| CLI Flags         | Description                                                               | Default                    | Required |
| ----------------- | ------------------------------------------------------------------------- | -------------------------- | -------- |
| `--distro`, `-d`  | Specify the distribution name, e.g. `ubuntu`.                             |                            | **Yes**  |
| `--release`, `-r` | Specify the release name, e.g. `focal` or `20`.                           |                            | **Yes**  |
| `--backup`, `-b`  | Backup the existing image before updating. Not compatible with `--remove` | `false`                    | No       |
| `--date`          | Append the date to the image name, e.g. `ubuntu*-$DATE.img`               | `false`                    | No       |
| `--remove`        | Remove image and backups before updating. Not compatible with `--backup`  | `false`                    | No       |
| `--storage`, `-s` | Specify the image storage path                                            | `/var/lib/vz/template/iso` | No       |

Adding the `--backup` flag will backup existing images for the set **distribution and release** before downloading the
latest image, using the naming scheme: `IMAGE_NAME.backup.img`.

Adding the `--date` flag will append the release date to name in `YYYY-MM-DD` or `YYYY-MMM-DD` format, e.g.
`ubuntu-20.04-server-cloudimg-amd64-YYYY-MM-DD.img`. The format is not configurable as it is pulled from the
distribution's release page.

### Proxmox Template Script

[`build-template`](/scripts/build-template): A wrapper script around `qm` to build Proxmox templates. This script
assumes the use of cloud-init vendor data file located at `local:snippets/vendor-data.yaml`. Detailed usage available on
the blog post: [Golden Images and Proxmox Templates with cloud-init], and example file:
[`cloud-init/vendor-data-minimal.yaml`](/cloud-init/vendor-data-minimal.yaml).

```bash
# usage
build-template -i <vm_id> -n <vm_name> --img <vm_image> [ARGS]
```

Example:

```bash
build-template -i 9000 -n ubuntu20 --img /var/lib/vz/template/iso/ubuntu-20.04-server-cloudimg-amd64.img
```

| CLI Flags          | Description                                                       | Default            | Required |
| ------------------ | ----------------------------------------------------------------- | ------------------ | -------- |
| `--id`, `-i`       | Specify the VM ID (1-999999999)                                   | `9000`             | **Yes**  |
| `--name`, `-n`     | Specify the VM name                                               | `template`         | **Yes**  |
| `--image`, `--img` | Specify the VM image (path), ex: `/PATH/TO/image.{img,iso,qcow2}` |                    | **Yes**  |
| `--bios`, `-b`     | Specify the VM bios, `seabios` or `ovmf`                          | `seabios`          | No       |
| `--machine`        | Specify the VM machine type, `q35` or `pc` for i440               | `q35`              | No       |
| `--memory`, `-m`   | Specify the VM memory                                             | `1024`             | No       |
| `--net-bridge`     | Specify the VM network bridge                                     | `vmbr0`            | No       |
| `--net-type`       | Specify the VM network type                                       | `virtio`           | No       |
| `--net-vlan`       | Specify the VM network vlan tag                                   | `1`                | No       |
| `--os`             | Specify the VM OS                                                 | `l26`              | No       |
| `--resize`         | Increase the VM boot disk size                                    | `1G`               | No       |
| `--storage`, `-s`  | Specify the VM storage                                            | `local-lvm`        | No       |
| `--scsihw`         | Specify the VM storage controller                                 | `virtio-scsi-pci`  | No       |
| `--vendor-file`    | Specify the cloud-init vendor file                                | `vendor-data.yaml` | No       |

## Maintainers & License

Taylor Fore (https://github.com/trfore)

See [LICENSE](LICENSE) File

## References

Linux Cloud Images:

- [CentOS Cloud Images]
  - Default User: `centos`
  - Uses: `CentOS-Stream-GenericCloud-X-latest.x86_64.qcow2`
- [Debian Cloud Images]
  - Default User: `debian`
  - Uses `debian-1x-generic-amd64.qcow2`
- [Ubuntu Cloud Images]
  - Default User: `ubuntu`
  - Uses `ubuntu-2x.04-server-cloudimg-amd64.img`

Proxmox:

- [Proxmox]
- [Proxmox templates]

[CentOS Cloud Images]: https://cloud.centos.org/
[Debian Cloud Images]: https://cloud.debian.org/images/cloud/
[Ubuntu Cloud Images]: https://cloud-images.ubuntu.com/releases/
[Proxmox]: https://www.proxmox.com/
[Proxmox templates]: https://pve.proxmox.com/wiki/VM_Templates_and_Clones
