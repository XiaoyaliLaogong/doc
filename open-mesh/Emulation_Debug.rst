.. SPDX-License-Identifier: GPL-2.0

Emulation Debug
===============

Create an Image
---------------

Download the fstab and rc.local file attached to this wiki page into
your directory before getting started.

::

    qemu-img create debian.img 8G
    sudo mkfs.ext4 -O '^has_journal' -F debian.img
    sudo mkdir debian
    sudo mount -o loop debian.img debian
    sudo debootstrap stretch debian
    sudo chroot debian apt update
    sudo chroot debian apt install build-essential vim openssh-server \
     pkg-config libnl-3-dev libnl-genl-3-dev libcap-dev
    sudo mkdir debian/root/.ssh/
    ssh-add -L | sudo tee debian/root/.ssh/authorized_keys
    sudo mkdir debian/host
    sudo cp fstab debian/etc/fstab
    sudo cp rc.local debian/etc/rc.local
    sudo chmod a+x debian/etc/rc.local
    sudo sed -i 's/^root:[^:]*:/root::/' debian/etc/shadow

    ## optionally: allow ssh logins without passwords
    # sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' debian/etc/ssh/sshd_config
    # sudo sed -i 's/^#PermitEmptyPasswords.*/PermitEmptyPasswords yes/' debian/etc/ssh/sshd_config
    # sudo sed -i 's/^UsePAM.*/UsePAM no/' debian/etc/ssh/sshd_config

    ## optionally: enable autologin for user root
    #sudo mkdir debian/etc/systemd/system/serial-getty@ttyS0.service.d/
    #sudo sh -c 'cat > debian/etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf  << EOF
    #[Service]
    #ExecStart=
    #ExecStart=-/sbin/agetty --autologin root -s %I 115200,38400,9600 vt102
    #EOF'

    sudo sh -c 'echo '\''PATH="$PATH":/host/batctl/'\'' >> debian/etc/profile'
    sudo umount debian


    ## optionally: convert image to qcow2
    #sudo qemu-img convert -f raw -O qcow2 debian.img debian.qcow2
    #sudo mv debian.qcow2 debian.img

Kernel compile
--------------

Any recent kernel can be used for the setup. The build result
``arch/x86/boot/bzImage`` has to be copied to the same directory as the
debian.img

::

    git clone git://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
    cd linux-next

    # small configuration

    make allnoconfig
    cat >> .config << EOF
    CONFIG_SMP=y
    CONFIG_EMBEDDED=n
    CONFIG_EXPERT=n
    CONFIG_MODULES=y
    CONFIG_MODULE_UNLOAD=y
    CONFIG_MODVERSIONS=y
    CONFIG_MODULE_SRCVERSION_ALL=y
    CONFIG_64BIT=y
    CONFIG_X86_VSYSCALL_EMULATION=n
    CONFIG_IA32_EMULATION=n
    CONFIG_HW_RANDOM_VIRTIO=y
    CONFIG_NET_9P_VIRTIO=y
    CONFIG_SCSI_VIRTIO=y
    CONFIG_VIRTIO_BALLOON=y
    CONFIG_VIRTIO_BLK=y
    CONFIG_VIRTIO_CONSOLE=y
    CONFIG_VIRTIO_INPUT=y
    CONFIG_VIRTIO_MMIO=y
    CONFIG_VIRTIO_MMIO_CMDLINE_DEVICES=y
    CONFIG_VIRTIO_NET=y
    CONFIG_VIRTIO_PCI=y
    CONFIG_VIRTIO_PCI_LEGACY=y
    CONFIG_CRC16=y
    CONFIG_LIBCRC32C=y
    CONFIG_CRYPTO_SHA512=y
    CONFIG_NET=y
    CONFIG_INET=y
    CONFIG_DEBUG_FS=y
    CONFIG_IPV6=y
    CONFIG_BRIDGE=y
    CONFIG_VLAN_8021Q=y
    CONFIG_WIRELESS=n
    CONFIG_NET_9P=y
    CONFIG_NETWORK_FILESYSTEMS=y
    CONFIG_9P_FS=y
    CONFIG_9P_FS_POSIX_ACL=y
    CONFIG_9P_FS_SECURITY=y
    CONFIG_BLOCK=y
    CONFIG_BLK_DEV=y
    CONFIG_EXT4_FS=y
    CONFIG_EXT4_USE_FOR_EXT23=y
    CONFIG_TTY=y
    CONFIG_SERIAL_8250=y
    CONFIG_SERIAL_8250_CONSOLE=y
    CONFIG_HW_RANDOM=y
    CONFIG_VHOST_RING=y
    CONFIG_GENERIC_ALLOCATOR=y
    CONFIG_SCSI_LOWLEVEL=y
    CONFIG_SCSI=y
    CONFIG_NETDEVICES=y
    CONFIG_NET_CORE=y
    CONFIG_DEVTMPFS=y
    CONFIG_HYPERVISOR_GUEST=y
    CONFIG_PARAVIRT=y
    CONFIG_KVM_GUEST=y
    CONFIG_BINFMT_ELF=y
    CONFIG_BINFMT_SCRIPT=y
    CONFIG_BINFMT_MISC=y
    CONFIG_PCI=y
    CONFIG_SYSVIPC=y
    CONFIG_POSIX_MQUEUE=y
    CONFIG_CROSS_MEMORY_ATTACH=y
    CONFIG_UNIX=y
    CONFIG_TMPFS=y
    CONFIG_CGROUPS=y
    CONFIG_BLK_CGROUP=y
    CONFIG_CGROUP_CPUACCT=y
    CONFIG_CGROUP_DEVICE=y
    CONFIG_CGROUP_FREEZER=y
    CONFIG_CGROUP_HUGETLB=y
    CONFIG_CGROUP_NET_CLASSID=y
    CONFIG_CGROUP_NET_PRIO=y
    CONFIG_CGROUP_PERF=y
    CONFIG_CGROUP_SCHED=y
    CONFIG_DEVPTS_MULTIPLE_INSTANCES=y
    CONFIG_INOTIFY_USER=y
    CONFIG_FHANDLE=y
    CONFIG_E1000=y
    CONFIG_CPU_FREQ=y
    CONFIG_CONFIG_X86_ACPI_CPUFREQ=y
    CONFIG_CPU_FREQ_GOV_ONDEMAND=y
    CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND=y
    CONFIG_CFG80211=y
    CONFIG_PARAVIRT_SPINLOCKS=y
    CONFIG_DUMMY=y
    CONFIG_PACKET=y
    CONFIG_VETH=y
    CONFIG_IP_MULTICAST=y
    CONFIG_NET_IPGRE_DEMUX=y
    CONFIG_NET_IP_TUNNEL=y
    CONFIG_NET_IPGRE=y
    CONFIG_NET_IPGRE_BROADCAST=y
    EOF

    #debug stuff
    # make sure that libelf-dev is installed or module build will fail with something like "No rule to make target 'net/batman-adv/bat_algo.o'"

    cat >> .config << EOF
    CONFIG_CC_STACKPROTECTOR_STRONG=y
    CONFIG_LOCKUP_DETECTOR=y
    CONFIG_DETECT_HUNG_TASK=y
    CONFIG_SCHED_STACK_END_CHECK=y
    CONFIG_DEBUG_RT_MUTEXES=y
    CONFIG_DEBUG_SPINLOCK=y
    CONFIG_DEBUG_MUTEXES=y
    CONFIG_PROVE_LOCKING=y
    CONFIG_LOCK_STAT=y
    CONFIG_DEBUG_LOCKDEP=y
    CONFIG_DEBUG_ATOMIC_SLEEP=y
    CONFIG_DEBUG_LIST=y
    CONFIG_DEBUG_PI_LIST=y
    CONFIG_DEBUG_SG=y
    CONFIG_DEBUG_NOTIFIERS=y
    CONFIG_PROVE_RCU_REPEATEDLY=y
    CONFIG_SPARSE_RCU_POINTER=y
    CONFIG_DEBUG_STRICT_USER_COPY_CHECKS=y
    CONFIG_X86_VERBOSE_BOOTUP=y
    CONFIG_DEBUG_RODATA=y
    CONFIG_DEBUG_RODATA_TEST=n
    CONFIG_DEBUG_SET_MODULE_RONX=y
    CONFIG_PAGE_EXTENSION=y
    CONFIG_DEBUG_PAGEALLOC=y
    CONFIG_DEBUG_OBJECTS=y
    CONFIG_DEBUG_OBJECTS_FREE=y
    CONFIG_DEBUG_OBJECTS_TIMERS=y
    CONFIG_DEBUG_OBJECTS_WORK=y
    CONFIG_DEBUG_OBJECTS_RCU_HEAD=y
    CONFIG_DEBUG_OBJECTS_PERCPU_COUNTER=y
    CONFIG_DEBUG_KMEMLEAK=y
    CONFIG_DEBUG_STACK_USAGE=y
    CONFIG_DEBUG_STACKOVERFLOW=y
    CONFIG_DEBUG_INFO=y
    CONFIG_DEBUG_INFO_DWARF4=y
    CONFIG_GDB_SCRIPTS=y
    CONFIG_READABLE_ASM=y
    CONFIG_STACK_VALIDATION=y
    CONFIG_WQ_WATCHDOG=y
    CONFIG_DEBUG_KOBJECT_RELEASE=y
    CONFIG_DEBUG_WQ_FORCE_RR_CPU=y
    CONFIG_OPTIMIZE_INLINING=y
    CONFIG_ENABLE_MUST_CHECK=y
    CONFIG_ENABLE_WARN_DEPRECATED=y
    EOF

    # for GCC 5+
    cat >> .config << EOF
    CONFIG_KASAN=y
    CONFIG_KASAN_INLINE=y
    CONFIG_UBSAN_SANITIZE_ALL=y
    CONFIG_UBSAN=y
    EOF

    make olddefconfig
    make all -j$(nproc || echo 1)

Start of the simple environment
-------------------------------

The two node environment must be started inside a screen session. The
hub (bridge with eth0 + 2 tap devices) has to be started first to have a
simple network. A more complex network setup can be on the page
:doc:`Emulation <Emulation>`

The ``ETH`` in hub.sh has to be changed to the real interface which
provides internet-connectivity
The ``SHARED_PATH`` in run.sh has to be changed to a valid path which
is used to share the precompiled batman-adv.ko and other tools

::

    screen
    ./hub.sh
    ./run.sh

Building the batman-adv module
------------------------------

The kernel module can be build outside the virtual environment and
shared over the 9p mount. The path to the kernel sources have to be
provided to the make process

::

    make KERNELPATH=/home/batman/linux-next

The kernel module can also be compiled for better readability for the
calltraces:

::

    make EXTRA_CFLAGS="-fno-inline -O1 -fno-optimize-sibling-calls" KERNELPATH=/home/sven/tmp/linux-next V=1

Resources
---------

* :download:`fstab`
* :download:`hub.sh`
* :download:`rc.local`
* :download:`run.sh`
