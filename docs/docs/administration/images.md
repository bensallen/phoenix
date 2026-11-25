# Building Images
Images are built from recipes and saved as artifacts.

## Recipe Definitions

The following parameters can be used to define a recipe:

| Parameter      | Description                                             |
| :--------------| :-------------------------------------------------------|
| vars           | A map of variable names to values made available to Jinja templating in the recipe. See below |
| architecture   | The target architecture for the image. Defaults to the same architecture where the command is running, but it's useful for cross-building. |
| distro         | Mostly informational. If the `packagemanager` option is unset, the `distro` will be parsed to attempt to guess the correct package manager. |
| packagemanager | Specifies what command should be called to install packages to the image. Supported options are currently `zypper`, `yum`, and `dnf`. If unset, this is assumed from the `distro` setting. |
| initfrom       | The base image that `buildah` should use to build the recipe. The special value `scratch` means that an empty image will be used (and `initpackages` should be specified). Otherwise specify an image that `buildah` can access, such as `ubi` or a custom image that has been pushed to a configured registry. |
| initpackages   | A list of packages to install **outside** of the chroot, useful when creating an image from `scratch`. |
| repos          | A map of repo entries to enable in the image. See below for the supported forms |
| steps          | An ordered list of actions to take to build the image. See below for supportes steps types |
| artifacts      | An ordered list of artifacts to capture from the image. See below for supported artifact types |

`architecture`, `imagetype`, `distro`, `packagemanager`, and `initfrom`
describe the image as a whole rather than accumulating, so any recipe may set
them and the first definition wins. This lets settings shared by several
images live in a common base recipe instead of being repeated in each of them.
A recipe may set one of them again to the same value, so that it can still be
built on its own as well as merged, but setting it to a different value is an
error.

`initpackages`, `repos`, `steps`, and `artifacts` collect the contributions of
every recipe that is merged.

### Variables

Recipes are rendered with Jinja before they are parsed, so `{{name}}` style
references may be used anywhere in the file, including in keys, values, and
`{% raw %}{% if %}{% endraw %}` blocks.

The `vars` map defines variables in the recipe itself:

<!-- {% raw %} -->
```yaml
vars:
  rocmver: "6.4.1"
steps:
  - command: ls /opt/rocm-{{rocmver}}
```
<!-- {% endraw %} -->

These values describing the recipe and the build are also available:

| Variable       | Description                                        |
| :--------------| :--------------------------------------------------|
| tag            | The build tag. See below                           |
| name           | The recipe name                                    |
| architecture   | The target architecture, defaulting to the host    |
| imagetype      | The `imagetype` setting, if set                    |
| distro         | The `distro` setting, if set                       |
| packagemanager | The package manager, either set or guessed from `distro` |
| initfrom       | The `initfrom` setting, if set                     |

A recipe is rendered before the recipes it merges are read, so it only sees
the metadata that it or an earlier recipe set. Referencing one that nothing
has set yet is an undefined name error; use `{{distro|default('')}}` or
`{% raw %}{% if distro is defined %}{% endraw %}` where that is expected.

The build tag is resolved before the recipe is rendered, so `{{tag}}` can be
used anywhere in a recipe, including artifact filenames and push tags. It
defaults to `<datetime>[-<githash>[-dirty]]` and can be set with `--tag`.

<!-- {% raw %} -->
```yaml
artifacts:
  - squashfs:
      output: rootdir-{{tag}}.squashfs
```
<!-- {% endraw %} -->

A variable given with `--define` overrides the `vars` entry of the same name,
and also overrides `tag` from `--tag`, so a recipe can define defaults that are
overridden on the command line.

A recipe merged with the `recipe` step sees the variables of the recipe that
merged it, which take precedence over the ones it defines itself. This lets a
shared recipe define defaults that the recipes using it can override. Merged
recipes do not see each other's variables.

Because `vars` and the values in the table above are read before the recipe is
rendered, they must be literals and cannot themselves be templated. Referencing
a variable that is not defined anywhere is an error.

Variable names follow the same rules as Python identifiers: letters, digits,
and underscores, not starting with a digit. A hyphen is the subtraction
operator in Jinja, so `{{ my-var }}` reads as `my - var` rather than a name;
use `my_var` instead. Names are checked when the recipe is read.

Variables are ordinary Jinja values, so the usual filters and methods work on
them, including on values defined in the same recipe:

<!-- {% raw %} -->
```yaml
vars:
  groups:
    - systems
    - users
steps:
  - file:
      dst: /etc/security/access.conf
      content: |
        {%- for group in groups %}
        + : {{group}} : ALL
        {%- endfor %}
```
<!-- {% endraw %} -->

### Repos

Each entry in the `repos` map is keyed by the repo name and is either a bare url
string or a mapping with a `url` key plus any number of options.

Default `enabled = 1` and `gpgcheck = 0` unless the entry overrides them.

Yum/dnf options are not validated; any option the package manager understands
(`excludepkgs`,`includepkgs`, `gpgkey`, `priority`, ...) may be used.
Yaml booleans are written as `1`/`0` and yaml lists are joined with spaces, matching
the glob list syntax used by options such as `excludepkgs`.

Zypper only supports `enabled`, `gpgcheck`, and `priority`. Other options are ignored with
a warning.

```yaml
repos:
  rocky9: https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/
  epel:
    url: https://download.fedoraproject.org/pub/epel/9/Everything/x86_64/
    excludepkgs: slurm* pmix*
```

### Step Types

| Step      | Description                                             |
| :---------| :-------------------------------------------------------|
| recipe    | Merge the specified recipe into this one                |
| package   | A string or list of strings of package names to install |
| file      | Copy a file from the management node into the image. A string specifies a common source and destination. A mapping with `src` and `dst` can specify different paths |
| command   | Run a command inside the image                          |
| osrelease | Write `IMAGE_ID` (recipe name) and `IMAGE_VERSION` (build tag) to `/etc/os-release`. Accepts a boolean to enable `IMAGE_ID` and `IMAGE_VERSION`, or a dictionary of additional os-release fields. `IMAGE_ID` and `IMAGE_VERSION` is always added. |


### Artifact Types

| Artifact  | Description                                        |
| :---------| :--------------------------------------------------|
| file      | A path or list of paths to copy                    |
| initramfs | Creates a gzip'ed cpio of the image root (boolean) |
| squashfs  | Creates a squashfs of the image root. Optionally specify `output` to control the generated filename or `include` to limit what paths are included |
| uki       | Builds a Unified Kernel Image, a single signed EFI binary holding the kernel, the initrd, and the kernel command line, for booting over HTTP Boot without iPXE. See below |
| push      | Commits the image and pushes it to a container registry with `buildah push`. A string specifies the registry. A mapping accepts `registry` (required), `image` to override the repository name (defaults to the recipe name), `tag` to override the destination tag (defaults to the build tag), and `format` to select the manifest type (`oci`, `docker`, `v2s2`, or `v2s1`) |

Setting `tag` replaces the build tag rather than adding to it. It may be a
single tag or a list of tags; use `{{tag}}` in the list where the build tag is
wanted. `buildah push` accepts only one destination, so each tag is a separate
push; layers already in the repository are not resent, so the additional
pushes only upload a manifest.

Buildah must already be logged into the registry used by the `push` artifact;
Phoenix does not run `buildah login`.

#### UKI

A Unified Kernel Image is one EFI binary containing the kernel, the initrd,
the kernel command line, and `os-release`. Firmware loads it directly over
HTTP Boot, so no iPXE and no per-node bootfile is involved. With Secure Boot
on, the command line inside the signed binary is the only one used and cannot
be edited at the console. With Secure Boot off, the stub accepts a command line
passed by the firmware or boot loader instead.

The image needs a kernel, `dracut`, the dracut modules for a live network
root (`dracut-live` and `dracut-network`), and `systemd-boot-unsigned` for
the EFI stub. `systemd-ukify` must be installed on the build node.

```yaml
artifacts:
  - squashfs: true
  - uki:
      url: http://boot.example.com:8000
      kcmdline:
        - console=ttyS0,115200n8
      key: /etc/phoenix/secureboot/db.key
      cert: /etc/phoenix/secureboot/db.crt
```

| Option    | Description                                        |
| :---------| :--------------------------------------------------|
| output    | The artifact filename, defaulting to `uki.efi`. The artifact directory is already keyed by image name and build tag, so leaving this alone keeps the boot URL predictable |
| url       | Base URL the nodes fetch artifacts from. Required unless `kcmdline` supplies its own `root=` |
| squashfs  | The squashfs filename to boot. Defaults to the `output` of the recipe's `squashfs` artifact, and must be set if the recipe has none or more than one |
| kcmdline  | A string or list of additional kernel parameters |
| kver      | The kernel version to build for. Defaults to the only kernel in the image, and it is an error for an image to hold more than one without setting this |
| kernel    | The kernel image, a path or glob relative to the image root. Defaults to the first of `/lib/modules/<kver>/vmlinuz`, `/boot/vmlinuz-<kver>`, and `/boot/Image-<kver>` that exists, which covers RHEL and SLES |
| initrd    | Reuse an initrd already in the image, such as `/boot/initramfs-*.img`, rather than running dracut again. A path or glob relative to the image root |
| dracutmodules | Dracut modules to add, defaulting to `dmsquash-live` and `livenet`. Setting this replaces the defaults rather than adding to them, so include those two when booting a live network root |
| dracutargs | Extra arguments for the dracut invocation |
| stub      | The EFI stub to wrap, a path or glob relative to the image root. Defaults to `/usr/lib/systemd/boot/efi/linux<arch>.efi.stub`, where `<arch>` is `x64` on x86_64 and `aa64` on aarch64 |
| key, cert | A PEM private key and certificate to sign with. Both are required together, and the paths are on the build node, not in the image |
| signkernel | Whether to also sign the kernel inside the UKI. Defaults to true when signing |
| signtool  | The ukify signing backend, `sbsign` (the default) or `systemd-sbsign` |

`dracutmodules` and `dracutargs` are ignored when `initrd` is set. An
unrecognized option is warned about rather than silently ignored. The artifact
may also be given as just the URL, taking every other default:

```yaml
  - uki: http://boot.example.com:8000
```

Signing is optional, so an image can be built and booted on a machine with
Secure Boot off before any keys exist. The certificate must be enrolled in
the firmware db for a signed image to boot with Secure Boot on.

The command line is assembled from the image being built, so the UKI and the
squashfs it boots are always from the same build:

```
root=live:<url>/images/<recipe>/<tag>/rootdir.squashfs rd.live.image ip=dhcp rd.neednet=1 <kcmdline>
```

The `root=` and `rd.live.image` defaults are omitted if `kcmdline` already
sets a `root=`, and the `ip=` and `rd.neednet=` defaults are omitted if
`kcmdline` contains any of `ip=`, `bond=`, `vlan=`, `bridge=`, `team=`,
`bootdev=`, `BOOTIF=`, or `rd.neednet=`. Only the
start of a parameter counts, so one of those appearing inside the value of
another parameter does not count as setting it.

`kcmdline` given as a string is split into parameters the way the kernel
splits its own command line, on whitespace outside of double quotes, and
each parameter is embedded exactly as written. A value containing spaces
keeps them:

```yaml
      kcmdline: nvidia.NVreg_RegistryDwords="RMNvLinkSpeedControl=1 Foo=2"
```

`dracutargs` is an argument vector rather than a kernel command line, so a
string is split the way a shell would and the quotes are consumed. These are
equivalent:

```yaml
      dracutargs: --add-driver "squashfs loop overlay"

      dracutargs:
        - --add-driver
        - squashfs loop overlay
```

A UKI is built once per image rather than once per node, so the default
network configuration is a generic `ip=dhcp` across every interface. Where
the interfaces that nodes boot on are known, naming them is better:

```yaml
  - uki:
      url: http://boot.example.com:8000
      kcmdline: ip=eno1:dhcp ip=eno2:dhcp rd.neednet=1 console=ttyS0,115200n8
```

The initrd is built inside the container, so the image's own kernel modules
and dracut modules are the ones dracut finds. It is written to a scratch
directory on the build node rather than into the image root, and `ukify` also
runs on the build node, so neither the generated initrd nor the signing key
ends up where a `squashfs` or `push` artifact ordered after the UKI would
capture it.

The UKI's `.osrel` section is taken from the image's `/etc/os-release`, or
`/usr/lib/os-release` if that is missing.

### Example Recipe

<!-- {% raw %} -->
```yaml
vars:
  version: "6.4.1"
architecture: x86_64
distro: rhel9
packagemanager: dnf
initfrom: scratch
initpackages:
- dnf
- bash
repos:
  rocky9: https://download.rockylinux.org/pub/rocky/9/BaseOS/x86_64/os/
  epel:
    url: https://download.fedoraproject.org/pub/epel/9/Everything/x86_64/
    excludepkgs: slurm* pmix*
steps:
  - recipe: rocky-base
  - package:
      - screen
      - golang
  - file: /etc/passwd
  - file:
      src: /root/hosts.mycluster
      dst: /etc/hosts
  - command: systemd-firstboot --timezone=America/New_York --locale=en_US.UTF-8 --locale-messages=en_US.UTF-8
  - osrelease: true

artifacts:
  - squashfs:
      output: {{version}}.squashfs
      include:
        - /opt/rocm-{{version}}
        - /etc/OpenCL/vendors
  - push: registry.example.com/myorg
  - push:
      registry: registry.example.com/myorg
      image: rocky9-compute
      tag:
        - "{{tag}}"
        - latest
      format: oci
```
<!-- {% endraw %} -->

## Listing recipes
The `pxrecipe list` command shows all detected recipes.

```yaml
# pxrecipe list
compute
login
rocky-9-base
rocky-9-bootable
```

## Showing a Recipe
The `pxrecipe show recipe_name` command will show the parsed contents of a recipe. If the recipe (or any of its sub-recipe steps) makes use of variables, they must be specified on the command line with `--define variable_name variable_value`. Pass `--tag` to see how a specific build tag renders, otherwise the default tag for the current time is used.

## Building a Recipe
The `pxrecipe build [options] recipe_name` command builds a recipe from its steps and generates the requested artifacts. If the recipe (or any of its sub-recipe steps) makes use of variables, they must be specified on the command line with `--define variable_name variable_value`. An optional `tag` can be specified with `--tag`, otherwise a tag of `<datetime>[-<githash>[-dirty]]` is used. By default, the `buildah` image is removed after the artifacts are successfully generated, but it can be retained with the `--keep` flag for additional debugging. Failed builds and builds retained with `--keep` need to be cleaned up from Buildah manually. A `push` artifact also commits the working container to a local `<recipe>:<tag>` image, which is left in Buildah storage.
