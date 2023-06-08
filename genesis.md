# A basic CLI tool to build Ubuntu images

This is just a blog post to present a tool I made on my free time. It's a CLI project written in Python. It can build Ubuntu images from scratch.

The tool is named `genesis` (because you start from nothing). And is available as a python package: https://github.com/gjolly/genesis (it's packaged as a deb in [a PPA](https://launchpad.net/~gjolly/+archive/ubuntu/genesis).

## How does it work?

### Creating a base image

First you want to start by bootstrapping a basic filesystem:

```bash
sudo genesis debootstrap \
    --series lunar \
    --mirror 'http://archive.ubuntu.com/ubuntu' \
    --output chroot-lunar
```

Then, with this filesystem, you can create a disk-image:

```bash
sudo genesis create-disk \
    --rootfs-dir ./chroot-lunar\
    --disk-image lunar.img
```

Once this is done, you need to update your system (`debootstrap` only uses the release pocket). While doing this stage, you can install some extra packages. Here we are going to build a very minimalist image of Ubuntu using only

```bash
sudo genesis update-system \
    --disk-image lunar.img \
    --mirror 'http://archive.ubuntu.com/ubuntu' \
    --series lunar \
    --extra-package openssh-server --extra-package ca-certificates --extra-package linux-kvm
```

We still need to install a boot loader and this operation requires its own command:

```bash
sudo genesis install-grub --disk-image lunar.img
```

### Final customizations

The image is almost ready but we can (and here we need) customize it by adding extra files directly on the filesystem. This is done with the `copy-files` command.

#### Configuring networking

Because we did not install cloud-init in our image, we need to pre-configure it with everything it needs. Here we assume that this image will be run with `qemu` and a virtual network card attached. We configure netplan accordingly:

```bash
sudo genesis copy-files \
    --disk-image lunar.img \
    --file $PWD/netplan.yaml:/etc/netplan/image-default.yaml
```

with `netplan.yaml` being the following:

```yaml
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
            match:
                driver: virtio_net
            set-name: eth0
```

### Configuring the sources

We want to define the Debian packages source. For now there is just a default source file pointing to `http://archive.ubuntu.com` in the image. Maybe, since I live in France, I want my image to be configured with a local mirror:

```
deb https://fr.archive.ubuntu.com/ubuntu lunar main universe restricted
deb https://fr.archive.ubuntu.com/ubuntu lunar-updates main universe restricted
deb https://fr.archive.ubuntu.com/ubuntu lunar-security main universe restricted
```

This is what my `sources.list` would look like and I can now install it on the live image:

```bash
sudo genesis copy-files \
    --disk-image lunar.img \
    --file $PWD/sources.list:/etc/apt/sources.list
```

### User config

Finally, I need to configure a user. Here I create a user with `create-user` and I copy my public ssh key in `.ssh/authorized_keys` directory for this user.

```bash
sudo genesis create-user \
    --disk-image lunar.img \
    --username ubuntu --sudo
sudo genesis copy-files \
    --disk-image lunar.img \
    --file $HOME/.ssh/id_rsa.pub:/home/ubuntu/.ssh/authorized_keys --mod 600 --owner ubuntu
```

Note that if `.ssh` does not exist under `/home/ubuntu`, it will be automatically created by `copy-files`.

## Is it usable for building production-ready images of Ubuntu?

**No**
