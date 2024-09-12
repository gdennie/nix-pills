# Developing with `nix-shell`

Welcome to the 10th Nix pill. In the previous [9th pill](09-automatic-runtime-dependencies.md) we saw one of the powerful features of Nix: automatic discovery of runtime dependencies. We also finalized the GNU `hello` package.

In this pill, we will introduce the `nix-shell` tool and use it to hack on the GNU `hello` program. We will see how `nix-shell` gives us an isolated environment in which we can modify the source files of the project, similar to how `nix-build` created an isolated environment in which it build the derivation.

Finally, we will modify our builder to work more ergonomically with a `nix-shell`-focused workflow.

## What is `nix-shell`?

The [nix-shell](https://nixos.org/manual/nix/stable/command-ref/nix-shell.html) tool drops us in a shell after setting up the environment variables according to the attribute set of a derivation. It does not build the derivation; it only serves as a preparation so that we can run whatever build steps it has manually.

Recall that in a nix environment, we don't have access to libraries or programs unless they have been installed with `nix-env`. However, installing libraries with `nix-env` is not good practice. We prefer to have isolated environments for development, which `nix-shell` provides for us.

We can call `nix-shell` on any Nix expression which returns a derivation, if possible; however, the resulting `bash` shell's `PATH` as defined by the derivation attribute set may not have the utilities we need:

```console
$ nix-shell hello.nix
[nix-shell]$ make
bash: make: command not found
[nix-shell]$ echo $baseInputs
/nix/store/jff4a6zqi0yrladx3kwy4v6844s3swpc-gnutar-1.27.1 [...]
```

This shell is rather useless. It would be reasonable to expect that the GNU `hello` build inputs are available in `PATH`, including GNU `make`, but this is not the case. Only the environment variables, `name`, `builder`, args`, `buildInputs`, `src`, and `system` are defined.

However, this means we can `source` our `builder.sh`, which uses those environment variables, and it will build the derivation as expected. Note: you may get an error in the installation phase because your user account may not have write permissions to `/nix/store`:

```console
[nix-shell]$ source builder.sh
...
```

The derivation didn't install, but it did build. Note the following:

- We sourced `builder.sh` and it ran all of the build steps, including setting up the `PATH` for us.

- `nix-shell` used the current directory and not a temp directory like `nix-build`. Therefore, `hello-2.10` has been unpacked in the current directory.

We are able to `cd` into `hello-2.10` and type `make`, because `make` is now available after sourcing, `builder.sh`.

The take-away is that `nix-shell` drops us in a shell at the current directory with the same (or very similar) environment used to run the builder.

## A builder for nix-shell

The previous steps require some manual commands to be run and are not optimized for a workflow centered on `nix-shell`. We will now improve our builder to be more `nix-shell` friendly.

There are a few things that we would like to change.

First, when we `source` `builder.sh`, we are using the file that is in the current directory. Now, when we perform the `nix-shell` it will have created a copy of that file in the nix store as part of creating the derivation (`.drv`) and ensuring closure (all references and tertiary references of a derivation reside in the local store).

As such, in the general case what we really want is to use the `builder.sh` file that has been stored in the nix store, as this would also be the file that would be used by `nix-build`. Presently, the correct technique is to define a separate attribute/environment variable containing the path to the local `builder.sh` file. The derivation will update to point to its copy within nix store. (Note, the `builder` is presently defined as the bash executable rather than our `builder.sh`. Our `builder.sh` will be passed as an argument to `bash`.)

Second, we don't want to run the whole builder: we only want to setup the necessary environment for manually building the project. Thus, we can break `builder.sh` into two files: a `setup.sh` for setting up the environment, and the real `builder.sh` that `nix-build` expects. Hence we first establish the environment to invoke the `builder.sh`

During our refactoring, we will wrap the build phases in functions to give more structure to our design. Additionally, we'll move the `set -e` into the builder file instead of the setup file. The `set -e` is annoying at the `nix-shell` scope, as it would terminate the shell itself if an error is encountered during the build.

Here is our modified `autotools.nix`. Noteworthy is the `setup = ./setup.sh;` attribute in the derivation, which adds `setup.sh` to the nix store and correspondingly adds a `$setup` environment variable in the builder.

```nix
pkgs: attrs:
let
  defaultAttrs = {
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./builder.sh ];
    setup = ./setup.sh;
    baseInputs = with pkgs; [
      gnutar
      gzip
      gnumake
      gcc
      coreutils
      gawk
      gnused
      gnugrep
      binutils.bintools
      patchelf
      findutils
    ];
    buildInputs = [ ];
    system = builtins.currentSystem;
  };
in
derivation (defaultAttrs // attrs)
```

Thanks to that, we can split `builder.sh` into `setup.sh` and `builder.sh`. What `builder.sh` does is `source` `$setup` and call the `genericBuild` function. Everything else is just some changes to the bash script.

Here is the modified `builder.sh`:

```sh
set -e
source $setup
genericBuild
```

Here is the newly added `setup.sh`:

```sh
unset PATH
for p in $baseInputs $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
done

function unpackPhase() {
    tar -xzf $src

    for d in *; do
    if [ -d "$d" ]; then
        cd "$d"
        break
    fi
    done
}

function configurePhase() {
    ./configure --prefix=$out
}

function buildPhase() {
    make
}

function installPhase() {
    make install
}

function fixupPhase() {
    find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
}

function genericBuild() {
    unpackPhase
    configurePhase
    buildPhase
    installPhase
    fixupPhase
}
```

Finally, here is `hello.nix`:

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "hello";
  src = ./hello-2.12.1.tar.gz;
}
```

Now back to nix-shell:

```console
$ nix-shell hello.nix
[nix-shell]$ source $setup
[nix-shell]$
```

Now, for example, you can run `unpackPhase` which unpacks `$src` and enters the directory. And you can run commands like `./configure`, `make`, and so forth manually, or run phases with their respective functions.

The process is that straightforward. `nix-shell` builds the `.drv` file from the given `.nix` file and its input dependencies copying content into nix store, then drops into a shell by setting up the environment variables according to the attribute set values of the resulting `.drv`.

## Conclusion

With `nix-shell` we are able to drop into an isolated environment suitable for developing a project. This environment provides the necessary dependencies for the development shell, similar to how `nix-build` provides the necessary dependencies to a builder. Additionally, we can build and debug the project manually, executing step-by-step like we would in any other operating system. Note that we never installed tools such `gcc` or `make` system-wide; these tools and libraries are isolated and available per-build.

## Next pill

In the next pill, we will clean up the nix store. We have written and built derivations which add to the nix store, but until now we haven't worried about cleaning up the used space in the store.
