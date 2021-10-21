# Ironic Agent Container

This is a container and associated tooling for using
[ironic-python-agent](https://docs.openstack.org/ironic-python-agent/latest/)
(IPA) on top of [CoreOS](https://docs.fedoraproject.org/en-US/fedora-coreos/).
This is different from standard IPA images that are built with
[diskimage-builder](https://docs.openstack.org/diskimage-builder/latest/) from
a conventional Linux distribution, such as CentOS.

This image is [published](https://quay.io/repository/metal3-io/ironic-agent)
on every commit, you can pull it with:
```
podman pull quay.io/metal3-io/ironic-agent
```

## Installation

This is intended for testing with [Metal3](http://metal3.io/), for example
using a test environment provided via [metal3-dev-env](https://github.com/metal3-io/metal3-dev-env)

Since this process is not currently fully integrated into metal3-dev-env,
some additional images and preparatory steps are required.

### Preparing Ignition config

Next we need to generate an [ignition](https://github.com/coreos/ignition/)
configuration that starts this IPA container, this can be done via the Metal3
[ironic image](https://github.com/metal3-io/ironic-image) - `IRONIC_IP` must
match the ironic-api endpoint in your environment:

```
podman run --pull=always --rm -e IRONIC_IP=172.22.0.2 -e IGNITION_FILE=/mnt/ironic-python-agent.ign -v .:/mnt quay.io/metal3-io/ironic configure-coreos-ipa
```

### Preparing deploy image - Redfish virtual media

Firstly download a recent [Fedora CoreOS](https://getfedora.org/en/coreos)
live-iso image, either manually or [using coreos-installer](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/):

```
podman run --pull=always --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest download -s stable -p metal -f iso
```

Then use `coreos-installer` to embed the ignition config into the iso image (replace filename with that downloaded as the version may differ):

```
podman run --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest iso ignition embed -i /data/ironic-python-agent.ign -f /data/fedora-coreos-34.20211004.3.1-live.x86_64.iso

# You can verify this worked e.g
podman run --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest iso ignition show /data/fedora-coreos-34.20211004.3.1-live.x86_64.iso
```

### Preparing deploy image - PXE

For PXE/iPXE boot you will need to download 3 artifacts: the kernel, the
initramfs and the root file system, for example:

```
podman run --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest download -s stable -p metal -f pxe
```

Alternatively if you already downloaded the ISO image the pxe artifacts can be extracted with recent versions of coreos-installer:

```
podman run --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest iso extract pxe /data/fedora-coreos-34.20211004.3.1-live.x86_64.iso

```

The ignition configuraiton can then be wrapped in an initrd image and appended to the PXE initrd:

```
podman run --rm -v .:/data -w /data quay.io/coreos/coreos-installer:latest pxe ignition wrap -i ironic-python-agent.ign > ironic-python-agent.img
cat ironic-python-agent.img >> fedora-coreos-34.20211016.3.0-live.x86_64-initrd.img
```


### Test environment configuration

The images prepared above must be made available via a webserver, for example if using metal3-dev-env:

```
cp *.iso *.img *-vmlinuz $WORKING_DIR/ironic/html/images/
```

They may then be provided to metal3-dev-env with a configuration similar to:

```
export DEPLOY_RAMDISK_URL=http://172.22.0.1/images/fedora-coreos-34.20211016.3.0-live.x86_64-initrd.img
export DEPLOY_KERNEL_URL=http://172.22.0.1/images/fedora-coreos-34.20211016.3.0-live.x86_64-vmlinuz
export DEPLOY_ISO_URL=http://172.22.0.1/images/fedora-coreos-34.20211016.3.0-live.x86_64.iso
FIXME - how to specify the rootfs?
```


### Testing deployment

TODO
