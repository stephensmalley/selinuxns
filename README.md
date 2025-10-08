# SELinux Namespaces

This repository is merely for documentation and tracking [issues](https://github.com/stephensmalley/selinuxns/issues) associated with the SELinux namespaces support. The code all lives elsewhere in branches of other repositories as identified below. Slides from a presentation about this work can be found [here](https://static.sched.com/hosted_files/lssna2025/27/NamespacesforSELinux.pdf), and a recording is available [here](https://youtu.be/AwzGCOwxLoM). 

## Getting Started
Clone and build the working-selinuxns branch of my selinux kernel fork containing the SELinux namespace patches.

    git clone -b working-selinuxns https://github.com/stephensmalley/selinux-kernel

See the SELinux kernel wiki [Getting Started](https://github.com/SELinuxProject/selinux-kernel/wiki/Getting-Started) guide for instructions on building, booting, and testing kernels in general.

You will need to enable the SELinux namespaces support option under Security options in make menuconfig (CONFIG_SECURITY_SELINUX_NS=y).

Reviewing the patches on the branch is a good way to learn a lot about SELinux and the current state of SELinux namespaces, starting with just reading the patch descriptions and any TODO comments sprinkled in the code.

Note that earlier versions of the SELinux namespaces support added and used a /sys/fs/selinux/unshare pseudo file interface for unsharing the SELinux namespace; given upstream preference for a syscall-based LSM namespacing API and implementation of such an API, the /sys/fs/selinux/unshare API is deprecated and likely will not be included when these patches are upstreamed. Hence, all references to /sys/fs/selinux/unshare below have been rewritten to use the corresponding libselinux utility programs and wrapper APIs instead which internally use the new syscall-based interfaces.

Clone and build the selinuxns branch of my selinux userspace fork containing a modified libselinux that provides a selinux_unshare() API call for unsharing the SELinux namespace, an is_selinux_unshared() API call for detecting whether one is in an unshared SELinux namespace that has not yet been fully initialized (i.e. no policy loaded yet), an unshareselinux utility program that allows one to exercise the APIs to run a shell or command in its own SELinux namespace, and an selinuxunshared utility program to test for being in an unshared SELinux namespace that has not yet been fully initialized.

    git clone -b selinuxns https://github.com/stephensmalley/selinux
    cd selinux/libselinux
    # Caveat: This will clobber your system libselinux. If you don't want that, then set DESTDIR accordingly!
    # If installing to a private destination directory, you can do this instead:
    # make DESTDIR=/path/to/destdir install
    # However, if you do this, then you need to set LD_LIBRARY_PATH accordingly when running programs that use these APIs.
    # NB Failing to set LIBDIR and SHLIBDIR will incorrectly install to just /lib and /usr/lib
    # which are for 32-bit compatibility libraries on Linux distributions.
    # On Debian/Ubuntu, make LIBDIR=/usr/lib/x86_64-linux-gnu SHLIBDIR=/usr/lib/x86_64-linux-gnu install
    sudo make LIBDIR=/usr/lib64 SHLIBDIR=/lib64 install relabel

Once you have booted the modified kernel and installed the modified libselinux, you can unshare the SELinux namespace and load a policy into it as follows:
<details><summary>Expand commands</summary>
    
    id -Z # See your context in the current SELinux namespace
    getenforce # See enforcing status in the current SELinux namespace
    # Create a shell with unshared SELinux and mount namespaces
    sudo unshareselinux bash
    # See that you are in an unshared SELinux namespace that is not yet fully initialized
    selinuxunshared
    # Mount new selinuxfs instance referencing the child namespace
    mount -t selinuxfs none /sys/fs/selinux
    getenforce # Child namespace starts in permissive
    # Load a policy into the child SELinux namespace, parent unaffected
    load_policy
    # See that the namespace is now initialized
    selinuxunshared
    id -Z # Context is now kernel_generic_helper_t on Fedora due to a default transition in its policy
    # Switch to a suitable security context before trying to go enforcing
    runcon unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 /bin/bash
    id -Z # See that you actually are in the right context now
    # Switch child to enforcing, checking that you didn't get killed once enforcing
    echo $$
    setenforce 1
    echo $$
    # Do stuff in child, run testsuite (switch parent to permissive first to avoid denials from it), etc.
    # When finished experimenting with the child namespace, do:
    # Exit shell created by runcon
    exit
    # Exit shell created by unshareselinux
    exit
    # Child namespace should be gone and you should be back to the parent/init namespace.
</details>

## Resource Limits

Two limits are defined for SELinux namespaces, maxnsdepth (maximum depth for nesting namespaces, default 32) and maxns (maximum number of namespaces, default 65535). The default value for each limit can be overridden via kernel configuration, and the active value can be modified from the init SELinux namespace by writing to /sys/fs/selinux/maxnsdepth and /sys/fs/selinux/maxns respectively. In child namespaces, the values can only be read, not written, to prevent child namespaces from escaping the limits (alternatively, we could allow them to only be lowered).

Testing of maxnsdepth was done using the following script named doit.sh:

<details><summary>Expand doit.sh</summary>
    
    #!/bin/sh -e
    echo $1
    arg="$1"
    argplus=$((arg + 1))
    mount -t selinuxfs none /sys/fs/selinux
    unshareselinux ./doit.sh $argplus
</details>

which can be called as follows:

<details><summary>Expand commands</summary>
    
    # Unshare the SELinux and mount namespaces and invoke the recursive script; should fail upon hitting the limit
    sudo unshareselinux ./doit.sh 1
    # Expect failure when it tries to create the 33rd nested namespace
</details>

Testing of maxns was done by lowering it to a value below maxnsdepth and then running the same script.

<details><summary>Expand commands</summary>

    # Create root shell to modify maxns and unshare namespaces
    sudo bash
    cat /sys/fs/selinux/maxns
    echo 5 > /sys/fs/selinux/maxns
    unshareselinux ./doit.sh 1
    # Expect failure when it tries to create the 5th namespace (init namespace + 4 children = 5)
    # Restore original maxns value
    echo 65535 > /sys/fs/selinux/maxns
</details>

## Testing

See the SELinux kernel wiki [Getting Started](https://github.com/SELinuxProject/selinux-kernel/wiki/Getting-Started) guide for instructions on testing kernels in general. I have made some changes to the SELinux testsuite policy to improve the ability to run the testsuite within a child SELinux namespace with an enforcing parent namespace, which can be found on the selinuxns branch of my fork of the selinux-testsuite.

    git clone -b selinuxns https://github.com/stephensmalley/selinux-testsuite

Basic functional testing consists of running the SELinux testsuite in the init SELinux namespace and in a child SELinux namespace to confirm no regressions in the init namespace and correct behavior in the child namespace. Running the testsuite in the init SELinux namespace can be done as usual following the instructions in SELinux and should be checked first prior to testing child namespaces. During or after running the testsuite, check dmesg or journalctl -k output for any warnings/errors during the run (in particular look for BUG: reports and kmemleak reports).

The entire SELinux testsuite can be run within a child SELinux namespace as long as the parent namespace is permissive (to avoid denying access due to the context in the parent not being allowed the same permissions as the test domains), with only the labeled IPSEC tests failing (2 tests each for inet_socket/tcp, inet_socket/udp, and inet_socket/mptcp) due to the labeled IPSEC hooks not supporting namespaces at this time. I have disabled these tests in my selinuxns branch to avoid noise from them. To run the testsuite in a child SELinux namespace with the parent namespace in permissive, do the following:

<details><summary>Expand commands</summary>

    # Switch init SELinux namespace to permissive so it doesn't interfere with testsuite operation in the child
    sudo setenforce 0
    # Create a shell with unshared SELinux and mount namespaces
    sudo unshareselinux bash
    # Mount new selinuxfs for child SELinux namespace
    mount -t selinuxfs none /sys/fs/selinux
    # Load a policy into the child SELinux namespace, parent unaffected
    load_policy
    # Switch to a suitable security context before trying to go enforcing
    runcon unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 /bin/bash
    id -Z # See that you actually are in the right context now
    # Switch child to enforcing, checking that you didn't get killed once enforcing
    echo $$
    setenforce 1
    echo $$
    # Run testsuite
    cd selinux-testsuite
    make test
    # Exit shell created by runcon
    exit
    # Exit shell that unshared its SELinux namespace
    exit
    # Restore init SELinux namespace to enforcing
    sudo setenforce 1
    # Check for any BUG or kmemleak reports during the test
    sudo journalctl -k | grep BUG:
    sudo journalctl -k | grep kmemleak:
</details>

It is also possible to run much of the SELinux testsuite in the child namespace with the parent in enforcing mode if you load the test policy into the init/parent namespace first so that the test domains/types are defined in both namespaces and the process context in the parent namespace is allowed the necessary permissions. To run the testsuite in the child namespace with the parent namespace enforcing, do the following (this assumes that the host OS/parent has a working policy and is already enforcing initially):

<details><summary>Expand commands</summary>

    # Confirm that the init/parent namespace is enforcing; if not, fix.
    getenforce
    # If Permissive, then run setenforce 1 and recheck.
    # Load the testsuite policy in the parent so that the test domains/types are defined in both
    # and unconfined_t in the parent is allowed the necessary permissions to them.
    cd selinux-testsuite
    make -C policy load
    # Create a shell with unshared SELinux and mount namespaces
    unshareselinux bash
    # Mount new selinuxfs for child SELinux namespace
    mount -t selinuxfs none /sys/fs/selinux
    # Load a policy into the child SELinux namespace, parent unaffected
    load_policy
    # Switch to a suitable security context before trying to go enforcing
    runcon unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 /bin/bash
    id -Z # See that you actually are in the right context now
    # Switch child to enforcing, checking that you didn't get killed once enforcing
    echo $$
    setenforce 1
    echo $$ # should have same value
    # Run testsuite in the child
    make test
    # Exit shell created by runcon
    exit
    # Exit shell that originally unshared its SELinux namespace
    exit
    # Unload test policy from init/parent namespace
    make -C policy unload
    # if this triggers an error due to parent/child sharing the same /etc/selinux, try semodule -B to clean up.
    # Check for any BUG or kmemleak reports during the test
    sudo journalctl -k | grep BUG:
    sudo journalctl -k | grep kmemleak:
</details>

Selective testing has also confirmed that if the parent denies access, the operation fails, even if the child allows it or is permissive, although this has not been exhaustively checked for every permission check.

In order to enable testing of SELinux namespaces for full containers, potentially with a different policy than the host OS, corresponding userspace changes are required for SELinux namespaces; see [Userspace](#userspace) for further discussion and prerequisites. The following table shows the current set of combinations of host OS and container OS that have been tested with separate SELinux namespaces. Unless otherwise indicated, each host and container OS is used in its default configuration, i.e. if the OS distribution enables SELinux by default, then it is kept enabled, else it is left disabled. Technically the host OS must always enable SELinux in order to support containers with SELinux but the host OS can be left with no policy loaded for those cases where the distribution doesn't default enable SELinux or provide a working policy out of the box. 

|Host OS|Fedora|Rocky 9|Rocky 8|Debian|Ubuntu|Android|
|-------|------|-------|-------|------|------|-------|
Fedora  |y|y|y|
Fedora w/o policy [^1] |y|y|y|
Ubuntu [^2] |y|y|y|

[^1]: Fedora host OS with SELinux enabled but no policy loaded. Currently requires relabel from container on first boot.
[^2]: Ubuntu host OS with SELinux enabled but no policy loaded. Currently requires relabel from container on first boot.

## Userspace

The userspace support for using SELinux namespaces with Linux containers can be subdivided into changes to the host OS and changes to the container OS. 

### Host OS
On the host OS, you need a modified container runtime to launch the container with its own SELinux namespace. To avoid hardcoding the current kernel API for unsharing the SELinux namespace into the container runtime code, a modified libselinux is provided that abstracts this interface behind a general API. For prototyping purposes, we are using the systemd-nspawn container runtime since it is small, easily extended, and in C (i.e. no need for other language bindings), and because we already have to modify systemd for the container anyway.

#### Build modified libselinux
A modified version of libselinux has been created on the selinuxns branch of my selinux userspace fork that provides a selinux_unshare() API call for unsharing the SELinux namespace, an is_selinux_unshared() API call for detecting whether one is in an unshared SELinux namespace that has not yet been fully initialized (i.e. no policy loaded yet), an unshareselinux utility program that allows one to exercise the APIs to run a shell or command in its own SELinux namespace, and an selinuxunshared utility program to test for being in an unshared SELinux namespace that has not yet been fully initialized.

<details><summary>Expand commands</summary>
    
    # Download, build, and install the modified libselinux
    git clone -b selinuxns https://github.com/stephensmalley/selinux
    cd selinux/libselinux
    # Caveat: This will clobber your system libselinux. If you don't want that, then set DESTDIR accordingly!
    # If installing to a private destination directory, you can do this instead:
    # make DESTDIR=/path/to/destdir install
    # However, if you do this, then you need to set LD_LIBRARY_PATH accordingly before building and
    # running systemd-nspawn so that it uses the modified libselinux. 
    # NB Failing to set LIBDIR and SHLIBDIR will incorrectly install to just /lib and /usr/lib
    # which are for 32-bit compatibility libraries on Linux distributions.
    # On Debian/Ubuntu, make LIBDIR=/usr/lib/x86_64-linux-gnu SHLIBDIR=/usr/lib/x86_64-linux-gnu install
    sudo make LIBDIR=/usr/lib64 SHLIBDIR=/lib64 install relabel
    # Create a shell with its own SELinux and mount namespaces.
    # If you installed to a private destination directory, you'll need to run it from
    # that directory and set LD_LIBRARY_PATH=/path/to/destdir/lib64 to pick up the right libselinux.
    sudo unshareselinux bash
    # Mount a new selinuxfs private to the new namespace.
    mount -t selinuxfs none /sys/fs/selinux
    # The child namespace starts life with the kernel SID/context and permissive.
    id -Z
    getenforce
    # Now you can load a policy into the child namespace.
    load_policy
    # Now you can launch a shell in an unconfined context.
    runcon unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 /bin/bash
    # Now you should be able to safely switch the child to enforcing mode.
    # But check the PID before and after doing so to verify that the shell
    # wasn't immediately killed when it went enforcing.
    echo $$
    setenforce 1
    echo $$
    # Do stuff
</details>

#### Build modified systemd-nspawn
A modified version of systemd-nspawn to invoke this selinux_unshare() API when systemd-nspawn is passed the --selinux-namespace option can be found in the selinuxns-host branch of my fork of systemd. To build the modified systemd-nspawn:

<details><summary>Expand commands</summary>
    
    # Install systemd-nspawn first for runtime dependencies
    sudo dnf install systemd-container
    # Install systemd's build dependencies
    sudo dnf build-dep systemd
    git clone -b selinuxns-host https://github.com/stephensmalley/systemd
    cd systemd
    meson setup build/
    ninja -C build/
    # Optional - you can always just run it from the build/ directory instead.
    # Caveat: This will clobber your host OS' systemd and related files.
    meson install -C build/
</details>

Next, you need a container OS image with its own corresponding userspace support.

### Container OS

For the container OS, you need the following:
* A container image that can be successfully booted via systemd-nspawn. The systemd-nspawn man page provides multiple examples under EXAMPLES of how to create containers for different Linux distributions.
* A version of systemd modified to detect that it is being run in its own SELinux namespace, and if so, to perform the usual SELinux initialization (load policy, set enforcing mode, etc based on /etc/selinux/config in the container). https://github.com/stephensmalley/systemd/tree/selinuxns-container contains just the patch for the container's systemd.
* A working base SELinux policy, typically available in your distribution's selinux-policy-targeted (Fedora) or selinux-policy-default (Debian), and an /etc/selinux/config that specifies SELINUX=permissive (best for first boot and trying it out) or SELINUX=enforcing (the end goal).
* A correctly labeled container filesystem. This may need to be done on first boot of the container if using a non-SELinux host OS or using a significantly different policy (in particular a different set of SELinux types) than the host OS; otherwise, it can be applied from the host OS before first boot of the container. In either case, this can be done via the setfiles utility (more specific instructions below).
* A local policy module to allow permissions unique to running the container within its own SELinux namespace since checks are applied against both the container's policy and the host policy for the container processes. This can be initially generated from collected audit logs using audit2allow (more specific instructions below).

#### Create or obtain a nspawn image

For the simplest case of creating an nspawn container image of the latest Fedora on a Fedora host, you can do the following (taken from the systemd-nspawn man page, amended to also install the SELinux policy and augmented to set a root password):

<details><summary>Expand commands</summary>

    sudo dnf -y --releasever=42 --use-host-config --installroot=/var/lib/machines/f42 \
        --setopt=install_weak_deps=False install \
        passwd dnf fedora-release vim-minimal util-linux systemd systemd-networkd \
        selinux-policy-targeted
    # Set root password so you can login to the container
    # NB Might require switching to permissive mode (setenforce 0) temporarily to work around a labeling issue in the container filesystem
    sudo chroot /var/lib/machines/f42 passwd
</details>
    
Confirm that you can successfully boot this container without using SELinux namespaces:

<details><summary>Expand commands</summary>
    
    # Boot the container named f42
    # Assumes you have systemd-nspawn installed; if not, dnf install systemd-container first.
    sudo systemd-nspawn -b -M f42
    # Login to the container, look around, getenforce will show Disabled because
    # selinuxfs is mounted read-only to tell the container userspace to not try to
    # load its own policy or do any SELinux processing.
    # Use Ctrl-] three times to exit and shut down the container.
</details>

#### Build modified systemd
The next step is to build and install the modified systemd into the container OS. Note that this is NOT the same as the modified systemd-nspawn built earlier for the host OS. You need to build with the same environment and dependencies as the container OS; the simplest way is to build from within the container itself (downside: pollutes the container with the build dependencies and systemd build tests will fail) or on another VM running the same OS as the container. Depending on the container OS, you may need to cherry-pick the patch over to the base systemd version used by the container OS. If you are instead just using my systemd fork (which may or may not work on containers expecting older versions), from the build environment, do the following:

<details><summary>Expand commands</summary>

    sudo dnf build-dep systemd
    git clone -b selinuxns-container https://github.com/stephensmalley/systemd
    cd systemd
    meson setup build/
    ninja -C build/
    meson install -C build/
</details>

From the host OS, test that you can still boot the container with the modified systemd using the same instructions from the prior section.

#### Install base policy

Install a known working base policy into the container filesystem if you haven't already done so, and initially set it to permissive mode. From within the container or using the --installroot option if installing from the host OS:

<details><summary>Expand commands</summary>

    dnf install selinux-policy-targeted
    vi /etc/selinux/config
    # Change enforcing to permissive; this will get fixed later once we resolve labeling and any denials
</details>

#### Label the container filesystem
Label the container filesystem based on its policy (not the host policy!).  Use setfiles not restorecon (ask me why)! This is best done from within the container although it may be possible to do from the host if the policies are relatively similar. From within the container or chroot'd to its root directory:

    setfiles /etc/selinux/targeted/contexts/files/file_contexts /

#### Boot with its own namespace
Once the container has the modified systemd, a policy, and labeled files, you can test booting with its own SELinux namespace. From the host OS:

<details><summary>Expand commands</summary>
    
    # This is done from the host OS, not the container.
    cd systemd
    sudo ./build/systemd-nspawn --selinux-namespace -b -M f42
    # Login, check getenforce and id output to see that SELinux is enabled within the container.
    # ps -eZ will show the labels of the processes within the container, which
    # will be the expected labels (e.g. systemd running in init_t, journald running in syslogd_t).
    # If you run ps -eZ on the host, you'll see that the same processes (not the same PIDs!) are
    # viewed as all running in a single label (whichever context systemd-nspawn was
    # run in, likely unconfined_t) on the host.
    # Use Ctrl-] three times to exit.
</details>

#### Check for any residual unlabeled or mislabeled files in the container and fix if present. From the container:

    # This is done from within the container
    setfiles /etc/selinux/targeted/contexts/files/file_contexts /

#### Create a local policy module for denials

Currently some policy changes are needed within the container to allow it to boot in enforcing mode due to some tmpfs mounts and a socket created by systemd-nspawn that are inherited and used by the container at runtime. The tmpfs mounts should be labeled by systemd-nspawn with an appropriate security context to reduce the need for such policy changes in the container but this should only be done when unsharing the SELinux namespace to avoid breaking existing container setups. systemd already has helpers for labeling files it creates (e.g. label_fix) but systemd-nspawn needs to be modified to call these helpers appropriately when --selinux-namespace is specified. The socket should also be labeled with its own type but policy changes might still be needed within the container to allow the container init to sendto the socket since that is not normally something required when running outside of a container. Until/unless these issues are fully resolved, you will need a local policy module added to the container to boot in enforcing mode.

In this case, you need to collect the audit logs or journal from the host OS since audit is not currently namespaced and kernel journal messages are not exposed to the container, copy those logs to the container, and then generate a policy module from the logs within the container. Avoid adding any rules on unlabeled files since those should be fixed through relabeling or, if necessary, a kernel fix. On the host OS:

<details><summary>Expand commands</summary>
    
    # Consider changing 'today' to a more precise time to only collect messages
    # from when you were trying to boot the container.
    # Intentionally using >> (append) below so that we keep a running log of all denials encountered so far.
    # If running auditd:
    sudo ausearch -m avc -ts today -i | grep -v unlabeled_t >> denials.log
    # If not running auditd:
    sudo journalctl -k -S today | grep avc | grep -v unlabeled_t >> denials.log
    # NB you may wish to add additional filters to the pipe and/or selectors to ausearch or journalctl
    # to only select the avc denials relevant to the container in question
    # and avoid allowing unnecessary accesses.
</details>

Copy the logs (denials.log) to the container filesystem and then run audit2allow and semodule from within the container. While you could run audit2allow on the host and then copy its policy module to the container, this could yield problems if the host and container policy differ, so it is better to run it within the container using the container's policy to interpret the denials.

<details><summary>Expand commands</summary>

    # From within the container
    sudo dnf install selinux-policy-devel
    audit2allow -M selinuxns < denials.log
    # If you get any errors/warnings from audit2allow, this may indicate that some of the logged denials were not from the container.
    sudo semodule -i selinuxns.pp
</details>

#### Rinse and Repeat
Keep trying to boot the container with its own SELinux namespace and augmenting the local policy module until you get no new denials in your ausearch or journalctl output. This shouldn't be necessary if you booted the container in permissive mode originally, since all denials should have been logged, but YMMV. It is also possible that some denials may be hidden by dontaudit rules in which case you may need to unhide them via semodule -DB, but I have not encountered this for the containers to date. Note to self: audit2allow -M doesn't seem to offer any way to add to an existing policy module, so you have to always feed it all the logs when updating a module, which means you need to retain a copy of all the logs ever used for that module. Someone ought to fix that sometime. While it's easy to just append the allow rules, that doesn't address the require blocks and it doesn't merge rules on the same (type,type,class) triple.

#### Switch container to enforcing
When you believe you have addressed all labeling and policy issues for the container, edit the /etc/selinux/config within container filesystem and set it to enforcing. Then boot the container with its own SELinux namespace in enforcing mode and confirm that there are no errors during boot and that you can login successfully. If it fails, then check ausearch or journalctl output to determine whether it is a labeling problem or a policy problem, fix and retry.

If the container boots successfully in enforcing mode, you've succeeded! Congrats! If your container OS isn't already listed as a tested combination under Testing, please add it to the table there.
