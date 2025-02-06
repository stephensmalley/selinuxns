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

# Resource Limits

Two limits are defined for SELinux namespaces, maxnsdepth (maximum depth for nesting namespaces, default 32) and maxns (maximum number of namespaces, default 65535). The default value for each limit can be overridden via kernel configuration, and the active value can be modified from the init SELinux namespace by writing to /sys/fs/selinux/maxnsdepth and /sys/fs/selinux/maxns respectively. In child namespaces, the values can only be read, not written, to prevent child namespaces from escaping the limits (alternatively, we could allow them to only be lowered).

Testing of maxnsdepth was done using the following script named doit.sh:

    #!/bin/sh -e
    echo $1
    arg="$1"
    argplus=$((arg + 1))
    umount /sys/fs/selinux
    mount -t selinuxfs none /sys/fs/selinux
    echo 1 > /sys/fs/selinux/unshare
    unshare -m ./doit.sh $argplus

which can be called as follows:

    # Create root shell to unshare namespaces
    sudo bash
    # Unshare the mount namespace and invoke the recursive script; should fail upon hitting the limit
    unshare -m ./doit.sh 1
    # Expect failure when it tries to create the 33rd nested namespace

Testing of maxns was done by lowering it to a value below maxnsdepth and then running the same script.

    # Create root shell to modify maxns and unshare namespaces
    sudo bash
    cat /sys/fs/selinux/maxns
    echo 5 > /sys/fs/selinux/maxns
    unshare -m ./doit.sh 1
    # Expect failure when it tries to create the 5th namespace (init namespace + 4 children = 5)
    # Restore original maxns value
    echo 65535 > /sys/fs/selinux/maxns
