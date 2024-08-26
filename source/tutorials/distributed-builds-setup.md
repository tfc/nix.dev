---
myst:
  html_meta:
    "description lang=en": "Setting up distributed builds"
    "keywords": "Nix, builds, distribution, scaling"
---

(distributed-build-setup)=
# Setting up distributed builds

Nix can automatically distribute builds over multiple machines to accelerate builds with parallel execution.

## Introduction

### What will you learn?

You'll learn how to
- Create a new user for remote build access from a local machine to the remote builder
- test remote builder connectivity and authentication
- configure the local machine to automatically distribute builds
- Configure remote builders with a sustainable setup

### What do you need

- The *local machine* (Hostname `localmachine`): The central machine that distributes builds among remote builders.
- The *remote machine* (hostname `remotemachine`): One (of possibly many) machines that accept build jobs from the local machine.

Both machines should already be running NixOS.

The local machine can later be configured to distribute among multiple remote builders.

### How long will it take?

<!-- TODO -->

## Create SSH key pair and prepare local machine

On the *local machine*, run the following command as `root` to create an SSH key pair:

```shell-session
ssh-keygen -f /root/.ssh/remotebuild
```

The local machine's Nix daemon runs as the `root` user and will need the private key file to authenticate itself to remote machines.
The remote builder configuration will store the public key to recognize the local machine.

:::{note}
The name and location of the key pair files can be freely chosen.
:::

## Set up remote builder

In the configuration folder for the *remote machine*, create the file `remote-builder.nix`:

```{code-block} nix
{
  users.users.remotebuild = {
    isNormalUser = true;
    createHome = false;
    group = "remotebuild";

    openssh.authorizedKeys.keyFiles = [ ./remotebuild.pub ];
  };

  users.groups.remotebuild = {};

  nix.settings.trusted-users = [ "remotebuild" ];
}
```

This configuration module creates a new user `remotebuild` with no home directory.
The `root` user will be able to log into the remote builder via SSH because the configuration module installs the earlier generated key in this user account.

Copy the file `remotebuild.pub` into the same folder.

Add this NixOS configuration module to your existing `/etc/nixos/configuration.nix` import list:

```{code-block} nix
{
  imports = [
    ./remote-builder.nix
  ];

  # ...
}
```

Activate the new configuration as root:

```shell-session
# nixos-rebuild switch
```

### Test authentication

Make sure that the SSH connection and authentication work.
On the *local machine*, run as `root`:

```shell-session
# ssh remotebuild@remotebuilder -i /root/.ssh/remotebuild "echo hello"
Could not chdir to home directory /home/remotebuild: No such file or directory
hello
```
The `Could not chdir to ...` message can be ignored.

It also adds the host key of the remote builder to the `/root/.ssh/known_hosts` file of the local machine.
Future logins will not be interrupted by host key checks.

## Set up distributed builds

### On NixOS

On the *local machine*, add the file `/etc/nixos/distributed-builds.nix`:

```{code-block} nix
{
  nix.buildMachines = [
    {
      hostName = "remotebuilder";
      sshUser = "remotebuild";
      sshKey = "/root/.ssh/remotebuild";
      system = "x86_64-linux";
      maxJobs = 1;
      speedFactor = 2;
      supportedFeatures = [ "nixos-test" "benchmark" "big-parallel" "kvm" ];
    }
  ];

  nix.distributedBuilds = true;
  nix.extraOptions = ''
    builders-use-substitutes = true
  '';
}
```

This configuration module enables distributed builds and adds the remote builder.
Replace `system` with the correct architecture if yours differs.

Add this NixOS configuration module to `/etc/nixos/configuration.nix`:

```{code-block} nix
{
  imports = [
    ./distributed-builds.nix
  ];

  # ...
}
```

Run as root:

```shell-session
# nixos-rebuild switch
```

### On Non-NixOS systems

```
builders = ssh-ng://remotebuild@remotebuilder x86_64-linux /root/.ssh/remotebuilder 1 - - -
builders-use-substitutes = true
```

Add the following lines to `/root/.ssh/config`:

```
Host remotebuilder
  User remotebuild
  IdentityFile /root/.ssh/remotebuild
```

## Test distributed builds

Run this command on the *local machine*:

```shell-session
$ nix-build -E "(import <nixpkgs> {}).writeText \"test\" \"$(date)\"" -j0
this derivation will be built:
  /nix/store/9csjdxv6ir8ccnjl6ijs36izswjgchn0-test.drv
building '/nix/store/9csjdxv6ir8ccnjl6ijs36izswjgchn0-test.drv' on 'ssh://remotebuilder'...
Could not chdir to home directory /home/remotebuild: No such file or directory
copying 0 paths...
copying 1 paths...
copying path '/nix/store/hvj5vyg4723nly1qh5a8daifbi1yisb3-test' from 'ssh://remotebuilder'...
/nix/store/hvj5vyg4723nly1qh5a8daifbi1yisb3-test
```

This command builds a minimal uncacheable example derivation.
The `-j0` command line argument forces nix to build it on the remote builder.

The last line contains the output path and indicates that build distribution works as expected.

## Optimize remote builder configuration

To optimize memory and disk space, add the following lines to your `/etc/nixos/remote-builder.nix` configuration module:

<!-- Adding "nix" to this code block produces the following error:

    /.../tutorials/distributed-builds-setup.md:181:Could not lex literal_block as "nix". Highlighting skipped.

-->

```{code-block}
{
  nix = {
    nrBuildUsers = 64;
    settings = {
      min-free = 10 * 1024 * 1024;
      max-free = 200 * 1024 * 1024;

      max-jobs = "auto";
      cores = 0;
    };
  };

  systemd.services.nix-daemon.serviceConfig = {
    MemoryAccounting = true;
    MemoryMax = "90%";
    OOMScoreAdjust = 500;
  };
}
```


## Outlook

To set up multiple builders, repeat the "Set Up Remote Builder" step for each remote builder.
Add the new remote builders to the `nix.buildMachines` list on the local machine.

Remote builders can have different performance characteristics.
For each `nix.buildMachines` item, set the `maxJobs`, `speedFactor`, and `supportedFeatures` attributes correctly for each different remote builder (refer to the [attribute reference][build-machines-reference]).
This helps nix on the local machine distributing builds the best way.

You can also set the `nix.buildMachines.*.publicHostKey` field with each remote builder's public host key to increase security.

## Alternatives

- [nixbuild.net](https://nixbuild.net) - Nix remote builders as a Service
- [hercules CI](https://hercules-ci.com/) - CI with automatic build distribution

## References

- [`nix.buildMachines` reference][build-machines-reference]
- [Settings for Distributed Builds in the Nix Manual][distributed-builds-nix]

[nix-serve-options]: https://search.nixos.org/options?query=services.nix-serve
[distributed-builds-nix]: https://nix.dev/manual/nix/2.24/command-ref/conf-file#conf-builders
[build-machines-reference]: https://search.nixos.org/options?query=nix.buildMachines.
