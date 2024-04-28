# Creating Proxmox Templates with cloud-init

Collection of example cloud-init config files and shell scripts to download cloud-images and create
[Proxmox templates].

Blog post: [Golden Images and Proxmox Templates with cloud-init]

## Cloud-init Files

- [`vendor-data-minimal.yaml`](/cloud-init/vendor-data-minimal.yaml): A minimal
  cloud-init file for running a VM on Proxmox.

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

## Systemd Templates

- [image-update@.service](/systemd-timer/image-update@.service) and
  [image-update@.timer](/systemd-timer/image-update@.timer)
  - Details below: [Automatically update images using systemd timers](#automatically-check-for-updates-using-systemd-timers)
  - **Note**: Change the day and time this script runs on, `OnCalendar=*-*-10 02:00:00`.

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
image-update -d debian -r 12
```

```bash
image-update -d fedora -r 40
```

```bash
image-update -d ubuntu -r 22
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

#### Automatically Check for Updates Using Cron

The majority of images are updated every 1 to 2 months, so be considerate and don't check more than once a month for
new cloud images. **Note**: Consider staggering cron jobs to different minute (`0`), hour (`2`), and day (`10`) across
distributions and releases.

```bash
0 2 10 * * /usr/local/bin/image-update -d <distro_name> -r <release_name>
```

Example:

```bash
# create/edit crontab
crontab -e

# create a job
10 3 15 * * /usr/local/bin/image-update -d ubuntu -r focal
```

#### Automatically Check for Updates Using Systemd Timers

The `image-update` script can also parse a single argument for distribution and release, `image-update ubuntu-20`, which
is useful in creating systemd timers. You can still pass the `--remove` and `--storage` flags as well, however,
`--distro` and `--release` are overridden when using this approach.

To use systemd timers, add the [template files](/systemd-timer/) to the `/etc/systemd/system` directory and start at
step 3. Or create a service and timer template, as follows:

1. Create a service, `image-update@.service`, in the `/etc/systemd/system` directory. Add additional script flags to
   `ExecStart=` after `%i`.

   ```xml
   [Unit]
   After=network-online.target
   Description= Check for updated cloud image %i
   Documentation=https://github.com/trfore/proxmox-template-scripts

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/image-update %i --remove
   ```

2. Create a timer, `image-update@.timer`, in the `/etc/systemd/system` directory. **Note**: Change the day and time this
   script runs on, `OnCalendar=*-*-10 02:00:00`. To prevent multiple timers from triggering simultaneously,
   `RandomizedDelaySec=3h` will randomly select a time within a 3-hour window and `AccuracySec=1us` will prevent systemd
   from grouping the timers.

   ```xml
   [Unit]
   Description=Check for updated cloud image %i
   Documentation=https://github.com/trfore/proxmox-template-scripts

   [Timer]
   OnCalendar=*-*-10 02:00:00
   AccuracySec=1us
   RandomizedDelaySec=3h
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```

3. Reload the systemd configuration

   ```bash
   systemctl daemon-reload
   ```

4. Create a Timer, using the format `image-update@<DISTRO>-<RELEASE>.timer`

   ```bash
   # enable and start a timer
   systemctl enable image-update@ubuntu-20.timer --now
   ```

Additional commands:

```bash
# view all timers
systemctl list-timers --all

# view timer status
systemctl status image-update@ubuntu-20.timer

# disable a timer
systemctl disable image-update@ubuntu-20.timer

# view script update logs
systemctl status image-update@ubuntu-20.service
```

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

Blog Post:

- [Golden Images and Proxmox Templates with cloud-init]

Linux Cloud Images:

- [OpenStack: Cloud Images], collection of image links.
- [CentOS Cloud Images]
  - Default User: `centos`
  - Uses: `CentOS-Stream-GenericCloud-X-latest.x86_64.qcow2`
- [Debian Cloud Images]
  - Default User: `debian`
  - Uses `debian-1x-generic-amd64.qcow2`
- [Fedora Cloud Images]
  - Default User: `fedora`
  - Uses `Fedora-Cloud-Base-Generic.x86_64-XX-XX.qcow2`
- [Ubuntu Cloud Images]
  - Default User: `ubuntu`
  - Uses `ubuntu-2x.04-server-cloudimg-amd64.img`

Proxmox:

- [Proxmox]
- [Proxmox templates]

[CentOS Cloud Images]: https://cloud.centos.org/
[Debian Cloud Images]: https://cloud.debian.org/images/cloud/
[Fedora Cloud Images]: https://fedoraproject.org/cloud/download
[Ubuntu Cloud Images]: https://cloud-images.ubuntu.com/releases/
[OpenStack: Cloud Images]: https://docs.openstack.org/image-guide/obtain-images.html
[Golden Images and Proxmox Templates with cloud-init]: https://www.trfore.com/posts/golden-images-and-proxmox-templates-using-cloud-init/
[Proxmox]: https://www.proxmox.com/
[Proxmox templates]: https://pve.proxmox.com/wiki/VM_Templates_and_Clones
