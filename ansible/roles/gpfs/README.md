GPFS
====

Extracts GPFS RPMs from the distributed installer package and compiles a kernel module.
Installs RPMs and kernel module on a GPFS client.

This role is designed to allow GPFS RPMs to be extracted, and the kernel module built, on a single node, and for the RPMs and kernel module to be installed on other nodes.
This means only one node needs to contain a kernel development environment, and this node does not need to be an active GPFS client node (for example a plain CentOS 7 virtual machine is sufficient).

After extracting and building the RPMs and kernel module they will be copied to the local machine (the one that's running Ansible).
The install part of this role will take the locally copied RPMs, send them to the nodes on which GPFS should be deployed, and install them.
The GPFS binaries path will be added to `PATH` in `/root/.bash_profile`.
Also note that the GPFS control servers need to be able to ssh into all servers as root.
This role will enable root access in `/etc/ssh/sshd_config`, and you should provide a list of public keys in `gpfs_public_keys`.


Requirements (Building)
-----------------------

Set `gpfs_build: True`.
Ensure all system packages are updated.
The GPFS kernel module will be compiled for the kernel currently running on the machine (so reboot if it has been updated).
This role will attempt to install the following kernel-related packages if they are missing:

- kernel-devel
- kernel-headers
- kernel-tools
- kernel-tools-libs

However if you are running an older kernel it is sometimes possible for a more recent version of these packages to be automatically installed, which will lead to failure of the kernel module build.
To avoid these problems it is highly recommended that you install these packages first and verify that their versions match the kernel version exactly.

Ensure the GPFS installer packages are in `gpfs_package_source_dir`, default location is the `files/` directory of this role.
The extracted and built RPMs will be copied into `gpfs_local_rpm_dir` which must be defined.


Requirements (Installing)
-------------------------

Set `gpfs_install: True` (this is the default).
Ensure the kernel corresponding to the built GPFS module is installed.
Ensure `gpfs_local_rpm_dir` contains the extracted and built RPMs.


Role Variables
--------------

You will need to override many of the variables in `defaults/main.yml`, depending on your GPFS version.

- `gpfs_build`: If True extract and build GPFS RPMs. You will need to run with this set on at least one node, but note the build node doesn't have to be an active GPFS client since it's purely used for extracting and building packages (default False)
- `gpfs_install`: If True (default) install GPFS using RPMs provided in a local directory, this should be True unless you are only extracting/building the GPFS RPMs
- `gpfs_configure`: If True (default same as `gpfs_install`) configure the system, including SSH access and local GPFS options.
- `gpfs_kernel_version`: Compile/install the GPFS module for this kernel version
- `gpfs_local_rpm_dir`: A local directory to which the extracted/built RPMs can be copied from the build node and subsequently deployed onto other nodes (mandatory, no default).
- `gpfs_package_source_dir`: A local path to a directory containing the GPFS packages (default `files/`)
- `gpfs_install_check_kernel_version`: If True check that the currently running kernel version matches that of the compiled GPFS kernel module. Set to False if you are upgrading the kernel and GPFS kernel module at the same time.
- `gpfs_public_keys`: A list of the SSH public keys belonging to the root users on the NSD nodes, needed so that the NSD nodes can connect to this GPFS client node (default: don't configure)
- `gpfs_node_specific_mount_options`: A dictionary of special mount options for a specific node only, for example to make one node mount GPFS as read-only: `gpfs-name: ro`, to remove custom mount options set the dictionary value to empty: `gpfs-name: `.
- `gpfs_enable_systemd_workaround`: Workaround a bug in GPFS v4.1 when used with systemd-219, (default False)

For convenience you may want to define some variables on the command line, e.g. `-e gpfs_local_rpm_dir=/data/gpfs/rpms -e gpfs_package_source_dir=/data/gpfs/src`.

Example Playbook
----------------

    # Build GPFS on one node (this must be run before installing GPFS anywhere)
    - hosts: gpfs-build-node
      roles:
      - gpfs
      vars:
      - gpfs_build: True
      - gpfs_install: False
      - gpfs_local_rpm_dir: /data/gpfs/rpms

    # Install and configure GPFS on client nodes:
    - hosts: gpfs-client-nodes
      roles:
      - gpfs
      vars:
      #(Default) gpfs_build: False
      #(Default) gpfs_install: True
      - gpfs_local_rpm_dir: /data/gpfs/rpms
      - gpfs_public_keys:
          - ssh-rsa AAAA1...
          - ssh-rsa AAAA2...

    # Configure GPFS only (assumes GPFS has already been installed):
    - hosts: gpfs-client-nodes
      roles:
      - gpfs
      vars:
      #(Default) gpfs_build: False
      gpfs_install: False
      gpfs_configure: True
    - gpfs_public_keys:
        - ssh-rsa AAAA1...
        - ssh-rsa AAAA2...


Additional Notes
----------------

- An earlier version of this role used parameters extensively.
  However, the addition of the GPFS patch package complicates things since the names of packages aren't completely consistent, so for many tasks it's easier to hardcode things.
- The GPFS patch refuses to install unless some of the original installer packages are present.
- This role contains additional internal patches for GPFS which may be fixed in the next release.
- Possible todo: Don't re-copy when building and installing on the same node
- Relevant GPFS kernel/systemd issues:
    - [GPFS filesystem mounts and unmounts on RHEL-7](https://www.ibm.com/developerworks/community/forums/html/topic?id=00104bb5-acf5-4036-93ba-29ea7b1d43b7&ps=25#176065bb-65f3-48c0-b97b-4d14094fd77e)
    - [What are the current restrictions on IBM Spectrum Scale Linux kernel support?](http://www-01.ibm.com/support/knowledgecenter/api/content/SSFKCN/com.ibm.cluster.gpfs.doc/gpfs_faqs/gpfsclustersfaq.html?locale=en&ro=kcUI#linuxrest)


Adding a node to the GPFS cluster (interactive)
-----------------------------------------------

Once this role has been applied you must manually add the node to the GPFS cluster using these instructions as a guide (this is not handled by Ansible).
First log into a root shell on one of the GPFS admin nodes.
Commands will be sent from the admin nodes to the other cluster nodes, this is why password-less ssh root access is required.

1. Check you can ssh as root without a password into the new node
2. Check the reverse IP of the new node matches it's hostname
3. Run `mmaddnode -N new.node.hostname` to add the node
4. Run `mmchlicense {client|server} -N new.node.hostname` to assign a license
5. Run `mmlscluster` and `mmlslicense` to check the cluster
6. Run `mmstartup -N new.node.hostname` to start GPFS on the new node
7. Run `mmmount filesystem-name -N new.node.hostname` to enable the mount on the new node (this will automatically add an entry to `/etc/fstab`)
8. You can check the state of all nodes by running `mmgetstate -a`

Note do not edit `/etc/fstab` directly (it is managed centrally by GPFS), instead use `gpfs_node_specific_mount_options`.


Author Information
------------------

ome-devel@lists.openmicroscopy.org.uk
