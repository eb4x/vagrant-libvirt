---
title: Installation
nav_order: 2
toc: true
---

## Requirements

* [Libvirt](https://libvirt.org) and [QEMU](https://www.qemu.org)
* [Vagrant](https://developer.hashicorp.com/vagrant/install)
* GCC, Make and the libvirt development headers, to build the
  [ruby-libvirt](https://rubygems.org/gems/ruby-libvirt) native extension

{: .warning }
Before you start using vagrant-libvirt, please make sure your Libvirt
and QEMU installation is working correctly and you are able to create QEMU or
KVM type virtual machines with `virsh` or `virt-manager`.

{% assign repo = site.github.public_repositories | where: "name", site.github.repository_name %}
Check the [unit tests](https://github.com/vagrant-libvirt/vagrant-libvirt/blob/{{ repo.first.default_branch }}/.github/workflows/unit-tests.yml)
for the tested Vagrant versions.

## Guides

### Docker / Podman

Due to the number of issues encountered around compatibility between the ruby runtime environment
that is part of the upstream vagrant installation and the library dependencies of libvirt that
this project requires to communicate with libvirt, there is a docker image built and published.

This should allow users to execute vagrant with vagrant-libvirt without needing to deal with
the compatibility issues, though you may need to extend the image for your own needs should
you make use of additional plugins.

{: .info }
The default image contains the full toolchain required to build and install vagrant-libvirt
and it's dependencies. There is also a smaller image published with the `-slim` suffix if you
just need vagrant-libvirt and don't need to install any additional plugins for your environment.

If you are connecting to a remote system libvirt, you may omit the
`-v /var/run/libvirt/:/var/run/libvirt/` mount bind. Some distributions patch the local
vagrant environment to ensure vagrant-libvirt uses `qemu:///session`, which means you
may need to set the environment variable `LIBVIRT_DEFAULT_URI` to the same value if
looking to use this in place of your distribution provided installation.

#### Using Docker

To get the image with the most recent release:
```bash
docker pull vagrantlibvirt/vagrant-libvirt:latest
```

<div class="info">If you want the very latest code you can use the <code class="language-plaintext highlighter-rouge">edge</code> tag instead.
<div class="language-bash highlighter-rouge" style="margin-top: 1em; margin-bottom: 0;"><div class="highlight"><pre class="highlight">
<code>docker pull vagrantlibvirt/vagrant-libvirt:edge</code>
</pre></div></div>
</div>

Running the image:
```bash
docker run -it --rm \
  -e LIBVIRT_DEFAULT_URI \
  -v /var/run/libvirt/:/var/run/libvirt/ \
  -v ~/.vagrant.d:/.vagrant.d \
  -v $(realpath "${PWD}"):${PWD} \
  -w "${PWD}" \
  --network host \
  vagrantlibvirt/vagrant-libvirt:latest \
    vagrant status
```

It's possible to define a function in `~/.bashrc`, for example:
```bash
vagrant(){
  docker run -it --rm \
    -e LIBVIRT_DEFAULT_URI \
    -v /var/run/libvirt/:/var/run/libvirt/ \
    -v ~/.vagrant.d:/.vagrant.d \
    -v $(realpath "${PWD}"):${PWD} \
    -w "${PWD}" \
    --network host \
    vagrantlibvirt/vagrant-libvirt:latest \
      vagrant $@
}

```

#### Using Podman

To run with Podman you need to include

```bash
  --entrypoint /bin/bash \
  --security-opt label=disable \
```

for example:

```bash
vagrant(){
  podman run -it --rm \
    -e LIBVIRT_DEFAULT_URI \
    -v /var/run/libvirt/:/var/run/libvirt/ \
    -v ~/.vagrant.d:/.vagrant.d \
    -v $(realpath "${PWD}"):${PWD} \
    -w "${PWD}" \
    --network host \
    --entrypoint /bin/bash \
    --security-opt label=disable \
    docker.io/vagrantlibvirt/vagrant-libvirt:latest \
      vagrant $@
}
```

Running Podman in rootless mode maps the root user inside the container to your host user so we need to bypass [entrypoint.sh](https://github.com/vagrant-libvirt/vagrant-libvirt/blob/main/entrypoint.sh).

#### Extending the container image with additional vagrant plugins

By default the image published and used contains the entire tool chain required
to install the vagrant-libvirt plugin and it's dependencies. This allows any plugin
that requires native extensions to be installed and should be possible to use a
simple `FROM` statement and ask vagrant to install additional plugins.

```
FROM vagrantlibvirt/vagrant-libvirt:latest

RUN vagrant plugin install <plugin>
```

Recently the image has now moved to bundling the plugin with the vagrant system plugins
it should no longer attempt to reinstall each time. Eventually this will become
the default so additional plugin installs will need to install any dependencies needed
by them.

### Distributions

These guides install upstream Vagrant from HashiCorp, and are the distributions
the [vagrant-libvirt-qa](https://github.com/vagrant-libvirt/vagrant-libvirt-qa)
harness tests by bringing up a VM: Ubuntu 22.04, 24.04 and 26.04, Debian 12 and 13,
Fedora 43 and 44, CentOS Stream 9 and 10, openSUSE Leap 16.0 and Arch Linux, all
x86_64. Other releases and distribution-packaged Vagrant may work, but are untested.

They install the latest Vagrant version, looked up with:
```shell
version="$(curl -fsSL https://checkpoint-api.hashicorp.com/v1/check/vagrant | \
    tr ',' '\n' | grep current_version | cut -d: -f2 | tr -d '"')"
```

After installing, add your user to the `libvirt` group and log in again, to use
`qemu:///system` without a password:
```shell
sudo usermod -aG libvirt $USER
```

#### Ubuntu / Debian

```shell
# enable deb-src, for apt-get build-dep
sudo sed -i 's/^# deb-src/deb-src/' /etc/apt/sources.list                         # one-line format
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/*.sources  # deb822 format
sudo apt-get update
sudo apt-get build-dep -y ruby-libvirt
sudo apt-get install -y libvirt-daemon-system qemu-system-x86 qemu-utils
curl -fLO https://releases.hashicorp.com/vagrant/${version}/vagrant_${version}-1_amd64.deb
sudo dpkg -i vagrant_${version}-1_amd64.deb
vagrant plugin install vagrant-libvirt
```

#### Fedora

```shell
sudo dnf install -y @virtualization gcc libvirt-devel make
curl -fLO https://releases.hashicorp.com/vagrant/${version}/vagrant-${version}-1.x86_64.rpm
sudo rpm -Uh vagrant-${version}-1.x86_64.rpm
vagrant plugin install vagrant-libvirt
```

#### CentOS Stream

```shell
sudo dnf config-manager --set-enabled crb
sudo dnf install -y @virtualization-host-environment gcc libvirt-devel make ruby-devel
curl -fLO https://releases.hashicorp.com/vagrant/${version}/vagrant-${version}-1.x86_64.rpm
sudo rpm -Uh vagrant-${version}-1.x86_64.rpm
vagrant plugin install vagrant-libvirt
```

#### openSUSE Leap

```shell
sudo zypper install --no-confirm gcc make libvirt libvirt-devel qemu-kvm polkit ruby-devel
curl -fLO https://releases.hashicorp.com/vagrant/${version}/vagrant-${version}-1.x86_64.rpm
sudo zypper install --allow-unsigned-rpm --no-confirm vagrant-${version}-1.x86_64.rpm
sudo rm -f /opt/vagrant/embedded/lib/libreadline.so*
vagrant plugin install vagrant-libvirt
```

Removing the embedded libreadline is needed, see
[Conflicts with Vagrant's embedded libraries](#conflicts-with-vagrants-embedded-libraries).

#### Arch

Arch no longer packages Vagrant, so install HashiCorp's package:
```shell
sudo pacman -Syu --needed dnsmasq gcc libvirt make nftables openbsd-netcat pkgconf qemu-base
sudo systemctl enable --now libvirtd
curl -fLO https://releases.hashicorp.com/vagrant/${version}/vagrant-${version}-1-x86_64.pkg.tar.zst
sudo pacman -U vagrant-${version}-1-x86_64.pkg.tar.zst
sudo rm -f /opt/vagrant/embedded/lib/lib{readline,curl}.so*
vagrant plugin install vagrant-libvirt
```

Removing the embedded libreadline and libcurl is needed, see
[Conflicts with Vagrant's embedded libraries](#conflicts-with-vagrants-embedded-libraries).

## Issues and Known Solutions

### Failure to find Libvirt for Native Extensions

Ensuring `pkg-config` or `pkgconf` is installed should be sufficient in most cases.

In some cases, you will need to specify `CONFIGURE_ARGS` variable before running running `vagrant plugin install`, e.g.:
```shell
export CONFIGURE_ARGS="with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib64"
vagrant plugin install vagrant-libvirt
```

If you have issues building ruby-libvirt, try the following (replace `lib` with `lib64` as needed):
```shell
CONFIGURE_ARGS='with-ldflags=-L/opt/vagrant/embedded/lib with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib' \
    GEM_HOME=~/.vagrant.d/gems \
    GEM_PATH=$GEM_HOME:/opt/vagrant/embedded/gems \
    PATH=/opt/vagrant/embedded/bin:$PATH \
        vagrant plugin install vagrant-libvirt
```

### Failure to Link

If have problem with installation - check your linker. It should be `ld.gold`:

```shell
sudo alternatives --set ld /usr/bin/ld.gold
# OR
sudo ln -fs /usr/bin/ld.gold /usr/bin/ld
```

### Conflicts with Vagrant's embedded libraries

Vagrant puts the libraries it bundles in `/opt/vagrant/embedded/lib` on
`LD_LIBRARY_PATH` while building ruby-libvirt, so they are used in place of the
system ones. Where they are incompatible, installing the plugin fails with e.g.:

* `symbol lookup error: ... libreadline.so.8: undefined symbol: UP` from `/bin/sh`
  (openSUSE Leap 16, Arch)
* undefined `curl_*@CURL_OPENSSL_4` symbols when linking against libvirt (Arch)

Remove the conflicting library so the system one is used instead, e.g.
`sudo rm -f /opt/vagrant/embedded/lib/libreadline.so*`, and reinstall the plugin.
Upgrading Vagrant restores the removed libraries.
