---
layout: post
author: Quentin
---

When you have a master `Makefile` and try to have a precise software environment per rule, it can become quite verbose.
In this post, we are looking at a way to integrate Guix and Nix into a `Makefile` in a not (so) verbose way.


## Scenario

We will consider the following scenario: a workflow with two rules each using their own captured environment: one for Python, one for R.

One way of implementing this in Guix would be to have one `channels.scm` and one manifest file for each of the rule (`manifest-py.scm` and `manifest-r.scm`).
Then, in the rule command, we would call `guix time-machine -- shell` to use the correct channels and manifest.

This might look something like this:
```makefile
all: foo bar

CHANNELS:=channels.scm

foo: $(CHANNELS) manifest-py.scm
    guix time-machine -C $(CHANNELS) -- shell -m manifest-py.scm -- python3 --version > $@

bar: $(CHANNELS) manifest-r.scm
    guix time-machine -C $(CHANNELS) -- shell -m manifest-r.scm -- Rscript --version > $@
```

As you can see..... quite verbose.

We also need to add the channel and manifest files into the prerequisite list for `make` to rebuild the rule if the software environment changes.
However, these `.scm` file kind of get in the way of the "actual" prerequisites of the rules (which we might want to use/capture with `@<` and `@^`).


## Main Idea

The main idea is to use the fact that some variables in `Makefile` are evaluated at "start time" (`:=`) and others at "use time" (`=`).
Thus, we set per-rule variable that will populate the global shell config which is evaluated a use time.

## Guix

The Guix version might look something like this:

```makefile
# Makefile
SHELL= guix
.SHELLFLAGS= time-machine -C $(CHANNELS) -- shell -m $(MANIFEST) -- bash -eu -o pipefail -c

CHANNELS:=channels.scm

all: foo bar

foo: MANIFEST=manifest-py.scm
foo: EXTRA_PREREQS=manifest-py.scm $(CHANNELS)
foo:
    python3 --version > $@

bar: MANIFEST=manifest-R.scm
bar: EXTRA_PREREQS=manifest-R.scm $(CHANNELS)
bar:
    Rscript --version > $@
```

So, the values of `CHANNELS` and `MANIFEST` only evaluated when calling the `SHELL` for running the rule.
We define those values either globally (like for `CHANNELS`), or at the rule level (like `MANIFEST`).

We also make sure `make` captures the dependencies between those Guix files and the rule.
To not "polute" the usual prerequisite list (i.e., those will not be in `@^` and `@<`), we put them in the `EXTRA_PREREQS` variable.

The manifests are simply:

```scheme
;; manifest-py.scm 
(specifications->manifest (list "python" "bash"))
```

and

``` scheme
;; manifest-R.scm
(specifications->manifest (list "r" "bash"))
```
and `channels.scm` is the output from `guix describe -f channels`

## Nix

The Nix version looks pretty similar.
Not that we are not using Flakes here.
(This is because of what we have [discussed previously]({{ site.url}}/2026/07/20/A-Case-Against-Nix-Flakes-in-Experimental-Computer-Science.html) and because making the `SHELL` and `.SHELLFLAGS` usable with the `nix develop` command is a pain.)

```makefile
# Makefile
SHELL= nix-shell
.SHELLFLAGS= --pure --arg channels $(CHANNELS) $(NIX_SHELL) --command 

CHANNELS:=./channels.nix

all: foo bar

foo: NIX_SHELL=pyshell.nix
foo: EXTRA_PREREQS=pyshell.nix $(CHANNELS)
foo:
    python3 --version > $@

bar: NIX_SHELL=rshell.nix
bar: EXTRA_PREREQS=rshell.nix $(CHANNELS)
bar:
    Rscript --version > $@
```

where:

```nix
# channels.nix
import (builtins.fetchTarball {
    url = "https://github.com/nixos/nixpkgs/archive/25.11.tar.gz";
    sha256 = "sha256:1zn1lsafn62sz6azx6j735fh4vwwghj8cc9x91g5sx2nrg23ap9k";
})
```

```nix
# pyshell.nix
{ channels, pkgs ? import channels {} }:
pkgs.mkShell {
    packages = with pkgs; [
        python3
    ];
}
```

```nix
# rshell.nix
{ channels, pkgs ? import channels {} }:
pkgs.mkShell {
    packages = with pkgs; [
        R
    ];
}
```

Note that in the case of Nix, `bash` is already in the shell, so no need to explicit it.
But, I was not able to pass the `-eu -o pipefail` flags to the Nix bash in the `.SHELLFLAGS` as it seems we would need to capture the commands in the rule between quotes...
