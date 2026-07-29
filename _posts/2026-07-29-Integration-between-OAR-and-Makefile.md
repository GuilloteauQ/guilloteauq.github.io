---
layout: post
author: Quentin
---

At work I am more frequently intercting with [OAR](https://oar.imag.fr) clusters rather than SLURM ones.
One feature that is missing in OAR 2.5, is a way for the submission command (`oarsub`) to not yield until the job has been completed.
For the SLURM-speaking readers: there is no `--wait` equivalent.
(If you are a OAR user, here is the issue to have this flag in OAR3: [#108](https://github.com/oar-team/oar3/issues/108)).

I generally have a (work\|data)flow to execute with tasks on some node (maybe even on several nodes) and maybe some different clusters, etc.

What I was doing before was to run my `Makefile` on the compute nodes directly (`oarsub -l host=1 "make -j 32"`).
But this is annoying, because I have to have an estimation of the total completion time of the whole workflow to correcty specify the walltime of the submission.
In the frequent cases where I underestimate, I have to resubmit the exact same submission.
Moreover, the parallelism at the node level is inexistant :(

On SLURM machine I was submitting `sbatch` scripts with the `--wait` flag in the rules of the `Makefile`: this is roughtly what I would like to reproduce with OAR.

In this post, we will show a simple example of how to achieve this with some """tricks""".

### A `--wait` equivalent

It turns out someone was also looking for this feature some years ago and came up with a script to emultate `--wait` !

You can find the original page [here](https://oar.imag.fr/wiki:an_equivalent_to_slurm_s_srun_for_OAR), but I will also copy-paste the code below (with a bit less commands for ease of reading).

```bash
#!/usr/bin/env bash
#
# Credits to Emmanuel Thomé and Pierre Neyron
#
# Usage: oarrun.sh [[oar args]] -- [[child command]]

set -e

oar_args=()
child=()

# PARSING CLI ARGS --------------------------------------
while [ $# -gt 0 ] ; do
    if [ "$1" = "--" ] ; then shift; break; fi
    oar_args=("${oar_args[@]}" "$1")
    shift
done
if [ $# -eq 0 ] ; then
    child=("${oar_args[@]}")
    oar_args=()
else
    while [ $# -gt 0 ] ; do
        if [ "$1" = "--" ] ; then shift; break; fi
        child=("${child[@]}" "$1")
        shift
    done
fi

# SETTING UP HOOKS --------------------------------------
control=`mktemp -d /tmp/oarrun.XXXXXXXXXX`

on_exit() {
  [ -n "$OAR_JOB_ID" ] && oardel $OAR_JOB_ID
  rm -rf $control
}
trap on_exit EXIT

FIFO=$control/oarrun.fifo.$$
NOTIFY_SCRIPT=$control/oarrun.notify.$$
cat <<EOF > $NOTIFY_SCRIPT
#!/bin/bash
echo "\$@" > /dev/stderr
echo "\$@" >> /tmp/oarlog
echo \$1 > $FIFO
EOF
chmod 755 $NOTIFY_SCRIPT
mkfifo $FIFO
exec 3<>$FIFO

# MAKE RESERVATION --------------------------------------
# -O and -E can be overridden by the user in the oar_args
oarsub -O /dev/null -E /dev/null "${oar_args[@]}" "sleep 99999d" --notify "[RUNNING]exec:$NOTIFY_SCRIPT"

read -u 3 OAR_JOB_ID
export OAR_JOB_ID

LEADER=$(oarstat -fj $OAR_JOB_ID | grep assigned_hostnames | cut -d\= -f2 | cut -d\+ -f1)

# EXECUTE COMMAND --------------------------------------
oarsh -t $LEADER cd "$PWD" \; "${child[@]}"
```

The idea is the following:

1. We create a submission which only `sleep`s

2. We add some hooks

3. We execute the command in the submission via `oarsh`

4. On completion, the hooks will activate and clean things up


### Integration with `Makefile`

We will actually reuse the same trick we have seen in a [previous post]({{ site.url }}/2026/07/21/A-Per-rule-Software-Environment-with-Guix-or-Nix-and-Makefile.html).

After the copy-paste the `oarrun.sh` file above and `chmod`ed it (`chmod u+x oarrun.sh`), we can write a simple `Makefile`.
(This `Makefile` supposes we are on the Grenoble site of [Grid'5000](https://www.grid5000.fr) where there are some clusters named `dahu` and `yeti`)


```makefile
.ONESHELL:
SHELL:=./oarrun.sh
.SHELLFLAGS= -p $(CLUSTER) -l host=$(NB_HOSTS) $(EXTRA_ARGS) --

CLUSTER=dahu
NB_HOSTS=1
EXTRA_ARGS=

TARGETS:=hostname.txt uname_dahu.txt uname_yeti.txt

all: $(TARGETS)

hostname.txt:
	hostname > $@

uname_dahu.txt:
	uname -a > $@

uname_yeti.txt: CLUSTER=yeti
uname_yeti.txt: EXTRA_ARGS=-t exotic
uname_yeti.txt:
	uname -a > $@

.PHONY: clean
clean: SHELL=bash
clean: .SHELLFLAGS= -c
clean:
	rm -rf $(TARGETS)
```

As you can see, we use the fact that the `.SHELLFLAGS` variable will be re-evaluated everytime there is a rule to execute, to setup some OAR-related variables such as the cluster and the number of hosts for the submission.

In this simple example, we have a configuration by default which will excute the rules on one `dahu` node.
We can override those values at the rule level, as shown for the `uname_yeti.txt` rule where we specify that we want to execute the rule on a node of the `yeti` cluster.
Accessing the `yeti` nodes actually requires to specify that we want to access an `exotic` resource (some [Grid'5000 term](https://www.grid5000.fr/w/Exotic)), which we pass as the `EXTRA_ARGS` variable.

Note that this is a very minimal example.
We could extend the `.SHELLFLAGS` with variables for all the different configuration knobs of `oarsub` (e.g., `gpu`, `walltime`, etc.)


There you have it!
A not too bad way to execute a `Makefile` workflow on OAR clusters :)
