# selinuxns
SELinux namespace support

## Getting Started
Clone and build the working-selinuxns branch of my tree containing the SELinux namespace patches. NB Do not use Paul Moore's working-selinuxns branch, which has not been updated since 2020.

    git clone -b working-selinuxns https://github.com/stephensmalley/selinux-kernel

You will need to enable the SELinux namespaces support option under Security options in make menuconfig (CONFIG_SECURITY_SELINUX_NS=y).

Reviewing the patches on the branch is a good way to learn a lot about SELinux and the current state of SELinux namespaces, starting with just reading the patch descriptions and any TODO comments sprinkled in the code.

Once you have booted this kernel, you can unshare the SELinux namespace and load a policy into it as follows:

    # Create root shell
    sudo bash
    id -Z # See your context in the current SELinux namespace
    getenforce # See enforcing status in the current SELinux namespace
    # Unshare SELinux namespace
    echo 1 > /sys/fs/selinux/unshare
    id -Z # Context is now "kernel" in child; ps -eZ from parent will still show original context
    getenforce # Still enforcing because you haven't yet mounted a new selinuxfs and are reading the parent's state
    setenforce 0 # Fails because you are in a child namespace and aren't allowed to modify the parent
    # Unshare mount namespace and mount new selinuxfs for child SELinux namespace
    # NB unshare(1) creates a child shell with the unshared mount namespace
    # so the commands after the unshare command actually run in their own shell.
    # If you want to script this, you'd need to put the commands after the unshare
    # in their own script and pass the script name to unshare, or use a Here document (<<EOF).
    unshare -m
    umount /sys/fs/selinux
    mount -t selinuxfs none /sys/fs/selinux
    getenforce # Child namespace starts in permissive
    # Load a policy into the child SELinux namespace, parent unaffected
    load_policy
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
    # Exit shell created by unshare
    exit
    # Exit shell that originally unshared SELinux namespace
    exit
    # Child namespace should be gone and you should be back to the parent/init namespace.
