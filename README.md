# sev-step-meta

This meta repo tracks a compatible state of all SEV STEP components and contains scripts to install everything required to setup a SEV VM.

This manual was tested on a Dell PowerEdge R6515 with a AMD EPYC 7763 CPU running Ubuntu 20.04.5 LTS.


This repo uses git submodules. Run `git submodule update --init --recursive` to ~~catch~~ fetch them all.
# Build Hypervisor Components
In this section you will install the modified SEV STEP kernel, as well as compatible stock versions of QEMU and OVMF/edk2.

1) Run `./build.sh` to build OVMF, QEMU, and the Linux kernel. If you run into any missing dependencies
try `sudo apt-get build-dep ovmf qemu-system-x86`.
All packages are only installed locally in `./local-installation/`.
2) Install the host kernel  using `dpkg -i ./kernel-packages/*.deb`
3) Create the config file `/etc/modprobe.d/kvm.conf` with content
```
# Enable SEV Support
options kvm_amd sev-snp=1 sev=1 sev-es=1

# Pagetracking code for sev step does not work if this option is set
# Context: there seems to be a transition in the page table management code.
# Our patch is for the old version that is phased out if this flag is set
options kvm tdp_mmu=0                    
```
4) Boot into the new kernel. It is named `...-sev-step-<git commit>`


The following is based on [1].

To enable SEV on your system, you might need to change some BIOS options.
Usually, there is a general "SEV Enabled" option as well as a `SEV-ES ASID space limit` option and a `SNP Memory Coverage`. `SEV-ES ASID space limit` should be greater than `1`.

If SEV is enabled, you  should get values similar to the following when running the specified commands on the host system. (Make sure you already rebooted into the new kernel)
```
# dmesg | grep -i -e rmp -e sev
SEV-SNP: RMP table physical address 0x0000000035600000 - 0x0000000075bfffff
ccp 0000:23:00.1: sev enabled
ccp 0000:23:00.1: SEV-SNP API:1.51 build:1
SEV supported: 410 ASIDs
SEV-ES and SEV-SNP supported: 99 ASIDs
# cat /sys/module/kvm_amd/parameters/sev
Y
# cat /sys/module/kvm_amd/parameters/sev_es 
Y
# cat /sys/module/kvm_amd/parameters/sev_snp 
Y
```


## Create and Run a SEV VM
In this section, we will setup a SEV SNP VM using the toolchain we just built.

1) Create a disk for the VM with `./local-installation/usr/local/bin/qemu-img  create -f qcow2 <VM_DISK.qcow2> 20G`
2) Start VM with `sudo ./launch-qemu.sh -hda <VM_DISK.qcow2> -cdrom <Ubuntu 22.10 iso> -vnc :1` where `<Ubuntu 22.10 iso>` is the path
to the regular Ubuntu installation iso. While you can use other Distros, their SEV support might vary. The most important part is, that they ship at least
kernel version 5.19, as this is the first mainline kernel version that supports running as a SEV SNP guest.
3) Connect to the VM via a VNC viewer on port `5901` and perform a standard installation
4) Once the installation is done, terminate qemu with `ctrl a+x` or use `sudo kill $(pidof qemu-system-x86)`
4) Start the VM again and connect with VNC. Install the OpenSSH server and configure it to autostart. The `./launch-qemu.sh` script already forwards VM port 22 to host port 2222 and VM port 8080 to host port 8080
5) Use `sudo ./launch-qemu.sh -hda <VM_DISK.qcow2> -sev-snp` to start the VM with SEV-SNP protection. You might want to also supply the `-allow-debug`
which enables the SEV debug API. Many **test functions** of the sev step framework require this to get access to the VM's instruction pointer or other register content.


## Use Sev Step Library
Head over to the `README.md` in the `sev-step-userland` submodule to learn how to use the sev step library.

# References
[1] https://github.com/AMDESE/AMDSEV/tree/sev-snp-devel#prepare-host
