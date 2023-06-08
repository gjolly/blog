# A basic CLI tool to build Ubuntu images

This is just a blog post to present a tool I made on my free time. It's a CLI project written in Python. It can build Ubuntu images from scratch.

The tool is named `genesis` (because you start from nothing). And is available as a python package: https://github.com/gjolly/genesis (it's packaged as a deb in [a PPA](https://launchpad.net/~gjolly/+archive/ubuntu/genesis).

## How does it work?

First you want to start by bootstrapping a basic filesystem:

```bash
sudo genesis debootstrap \
    --series lunar \
    --mirror 'http://127.0.0.1:3142/aws.archive.ubuntu.com/ubuntu' \
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
    --mirror 'http://127.0.0.1:3142/aws.archive.ubuntu.com/ubuntu' \
    --series lunar \
    --extra-package openssh-server --extra-package ca-certificates --extra-package linux-kvm
```

```bash
sudo genesis install-grub --disk-image lunar.img
```

```bash
sudo genesis copy-files \
    --disk-image lunar.img \
    --file $PWD/netplan.yaml:/etc/netplan/image-default.yaml
```


```bash
sudo genesis copy-files \
    --disk-image lunar.img \
    --file $PWD/sources.list:/etc/apt/sources.list
```

```bash
sudo genesis create-user \
    --disk-image lunar.img \
    --username ubuntu --sudo
sudo genesis copy-files \
    --disk-image lunar.img \
    --file /home/gauthier/.ssh/canonical.pub:/home/ubuntu/.ssh/authorized_keys --mod 600 --owner ubuntu
```

## Is it usable for building production-ready images of Ubuntu?

**No**
