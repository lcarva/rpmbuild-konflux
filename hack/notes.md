# Notes on using mock locally

## Buildsys Build

When creating a chroot, it is possible to instruct mock to install certain packages. This is done
via the `chroot_setup_cmd` option in the config. (Most mock configs seem to use this approach.) For
Fedora Rawhide, this option looks like this:

```python
config_opts['chroot_setup_cmd'] = 'install @buildsys-build'
```

(Actually, there's an expression in there to handle i686 but that's not important for this.)

`@buildsys-build` is an RPM group.

Let's use Hermeto to create a local repo with those packages. The first challenge is that Hermeto
doesn't understand RPM groups, so we expand the group to their individual RPMs:

```python
config_opts['chroot_setup_cmd'] = 'install bash bzip2 coreutils cpio diffutils fedora-release-common findutils gawk glibc-minimal-langpack grep gzip info patch redhat-rpm-config rpm-build sed shadow-utils tar unzip util-linux which xz'
```

It's not pretty but it gets the job done. You can use `dnf group info buildsys-build` to get the
list.

We can then use Hermeto to download those RPM packages:

```bash
rm -rf repo-buildsys-build-local && mkdir repo-buildsys-build-local && \
hermeto fetch-deps --output repo-buildsys-build-local '{"type": "rpm", "path": "./buildsys-build"}'
```

And create the required repo metadata and repo file:

```bash
hermeto inject-files ./repo-buildsys-build-local --for-output-dir=/repo-buildsys-build-local
```

Now `repo-buildsys-build-local` can be used as a local repo that contains all the RPM packages
required to install the RPM packages from the `buildsys-build` group.

## Buildroot

This is the more complicated part. We need to create a repo that can be used as the buildroot when
we build the actual RPM we want. This can be achieved with mock's `--calculate-build-dependencies`
parameter.

Here's how we would build it with packages from Fedora Rawhide:

```bash
rm -rf buildroot-results && \
mkdir --mode=777 buildroot-results && \
podman run -u mockbuilder \
    -v `pwd`/buildroot-results:/results:Z \
    -v `realpath ..`:/source:Z \
    --privileged --rm -it \
    quay.io/redhat-user-workloads/rpm-build-pipeline-tenant/environment:latest@sha256:5f96bc9fff7e084dc62f077edeafef5c7ff105a51b1a53ce00d8e1ce04660d3e \
    mock -r fedora-rawhide-x86_64 \
    --spec /source/hello.spec \
    --sources /source \
    --resultdir /results \
    --calculate-build-dependencies
```

This creates a few files in the `buildroot-results` directory. We really care about the
`buildroot_lock.json` file. This contains a list of all the RPM packages required for building our
RPM. Each RPM reference is pinned to a version and a digest.

What if we want to use a local repository? It should be possible. However! That means any RPM that
is a build dependency, must also be available for download. If we're using a local repo, those RPMs
must be available locally as well. We don't know upfront which RPM packages will be required.

Here's what I tried (and failed). Download the `repodata` files from one of Fedora Rawhide's mirrors.
Creating something like this:

```bash
repo-local/
└── repodata
    ├── 0e36e76126656f96550728d41d0e6af4f008a2fd842902d3da416521649e62db-primary.xml.zck
    ├── 3868a5b6bf87511fd0cc66ad981bfe0bef7f05f4a3b9d1aea51cd7f032e5db60-filelists.xml.zst
    ├── 5378931815144310e972be7e4552c65a1ae91cd5a9bd2d7ea30f1ad6338f5825-other.xml.zck
    ├── 56c91c93104138cfca3eae9dcd84c294a340bcb519c83b8c3f09250ed5b73b4b-comps-Everything.x86_64.xml.zst
    ├── 97e13e1c91bdcf7511f31b693b6d427869883b05c8569a2ee46142318193482a-filelists.xml.zck
    ├── a9cd7dd1969da27a0c77cd98564e642648b166e649649981b63744688979f672-primary.xml.zst
    ├── b615b362790cb38e300afb255751bf30b7d847f96ce7008ca110875db9a666e9-comps-Everything.x86_64.xml.zck
    ├── c9972c1fe3a2afe529742c38b4fc32287e7da017a9f97803fb3c40ed05c0eaea-other.xml.zst
    ├── d4ee8fc716f618122562fbdd11b775e5ee70f95198188b7816c22567a05d3ebb-filelists.sqlite.zst
    ├── d991edfd174319d08b8059b7656f4a0ec1c7c5ca6b16039bff72200d96d81a19-other.sqlite.zst
    ├── eaec0e9081582a17ddef3d6d1afe21cf79e878fe13f09bb1fe06067f618dde7b-primary.sqlite.zst
    └── repomd.xml

2 directories, 12 files
```

Notice how `repo-local` doesn't actually contain any packages. It only contains metadata about them.

`buildroot-local.cfg` is our mock config. It has some interesting properties:

1. It contains two repo definitions. One to `repo-local` and another to `fedora-rawhide`. (The
   metadata for these two repos are the same but that's ok for this experiment.) `repo-local` is
   enabled by default, `fedora-rawhide` is not.
1. The `chroot_setup_cmd` is modified to disable `repo-local` and enable `fedora-rawhide` when
   preparing the chroot. We can't get these packages from `repo-local` because well... that repo
   doesn't have any packages, just their metadata.

The following fails:

```bash
rm -rf buildroot-results && \
mkdir --mode=777 buildroot-results && \
podman run -u mockbuilder \
    -v `pwd`/repo-local:/repo-local:Z \
    -v `pwd`/buildroot-results:/results:Z \
    -v `realpath ..`:/source:Z \
    --privileged --rm -it \
    quay.io/redhat-user-workloads/rpm-build-pipeline-tenant/environment:latest@sha256:5f96bc9fff7e084dc62f077edeafef5c7ff105a51b1a53ce00d8e1ce04660d3e \
    mock -r /source/hack/buildroot-local.cfg \
    --spec /source/hello.spec \
    --sources /source \
    --resultdir /results \
    --calculate-build-dependencies
```

It configures the chroot fine, but when it gets to the part where it is resolving the dependencies
for our RPM spec, it fails:

```text
Failed to download packages
 Librepo error: Cannot download Packages/g/gcc-15.2.1-1.fc44.1.x86_64.rpm: All mirrors were tried
```

Mock really wants to install the build dependencies even when it's just resolving dependencies. It
cannot work just on the metadata. I don't see a way of bypassing it.

### Buildsys-build with Hermeto

As an extra, here's how we would calculate the build dependencies while using a repo created by
Hermeto to satisfy the buildsys-build requirements:

```bash
rm -rf build-results && \
mkdir --mode=777 build-results && \
podman run -u mockbuilder \
    -v /tmp/scratch/jolly-rubin:/repo:Z \
    -v `pwd`/repo-buildsys-build-local/deps/rpm/x86_64/rawhide:/repo-buildsys-build-local:Z \
    -v `pwd`/build-results:/results:Z \
    -v `realpath ..`:/source:Z \
    --privileged --rm -it \
    quay.io/redhat-user-workloads/rpm-build-pipeline-tenant/environment:latest@sha256:5f96bc9fff7e084dc62f077edeafef5c7ff105a51b1a53ce00d8e1ce04660d3e \
    mock -r /source/hack/local.cfg \
    --spec /source/hello.spec \
    --sources /source \
    --resultdir /results \
    --calculate-build-dependencies
```

(Well kind of. This needs a bit of work.)

## Building RPM

Build the RPM is the easy part.

Once the `calculate-deps` Task completes, it creates a Trusted Artifact that contains the RPM
packages required for building our RPMs, as well as a lockfile pinning each one of those RPMs.
Here's an existing one:

```bash
mkdir calculate-deps && \
cd calculate-deps && \
oras pull quay.io/redhat-user-workloads/rhel-primitives-tenant/rpmbuild-konflux:684926ffd414187e47ed206d225027f74ed55529.calculation-x86_64 && \
tar -xvf calculation-artifact && \
cd ..
```

Let's build our RPM by providing the buildroot and the buildroot lock from `calculate-deps`. Notice
that mock runs in hermetic mode and does not require a mock config.

```bash
rm -rf build-results && \
mkdir --mode=777 build-results && \
podman run -u mockbuilder \
    -v `pwd`/calculate-deps/x86_64/results:/buildroot:Z \
    -v `pwd`/build-results:/results:Z \
    -v `realpath ..`:/source:Z \
    --privileged --rm -it \
    quay.io/redhat-user-workloads/rpm-build-pipeline-tenant/environment:latest@sha256:5f96bc9fff7e084dc62f077edeafef5c7ff105a51b1a53ce00d8e1ce04660d3e \
    mock \
    --hermetic-build \
    /buildroot/buildroot_lock.json \
    /buildroot/buildroot_repo \
    --spec /source/hello.spec \
    --sources /source \
    --resultdir /results
```

It is also possible
The following works and it uses an existing buildroot (created with Hermeto):

```bash
rm -rf build-results && \
mkdir --mode=777 build-results && \
podman run -u mockbuilder \
    -v /tmp/scratch/jolly-rubin:/repo:Z \
    -v `pwd`/repo-buildsys-build-local/deps/rpm/x86_64/rawhide:/repo-buildsys-build-local:Z \
    -v `pwd`/build-results:/results:Z \
    -v `realpath ..`:/source:Z \
    --privileged --rm -it \
    quay.io/redhat-user-workloads/rpm-build-pipeline-tenant/environment:latest@sha256:5f96bc9fff7e084dc62f077edeafef5c7ff105a51b1a53ce00d8e1ce04660d3e \
    mock -r /source/hack/local.cfg \
    --spec /source/hello.spec \
    --sources /source \
    --resultdir /results
```

However, it actually uses packages from Fedora Rawhide to setup the chroot.
