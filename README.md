# sev-step-meta

This meta repo tracks a compatible state of all SEV STEP components and contains scripts to install everything required to setup a SEV VM.

This manual was tested on a Dell PowerEdge R6515 with a AMD EPYC 7763 CPU running Ubuntu 20.04.5 LTS.


This repo uses git submodules. Run `git submodule update --init --recursive` to ~~catch~~ fetch them all.
# Build Hypervisor Components
In this section you will install the modified SEV STEP kernel, as well as compatible stock versions of QEMU and OVMF/edk2.

1) Run `./build/build.sh` to build OVMF, QEMU, and the Linux kernel. If you run into any missing dependencies
try `sudo apt-get build-dep ovmf qemu-system-x86`.
All packages are only installed locallay in `./local-installation/` e.g. global installation of qemu wont be overwritten.
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
5) Boot into the new kernel. It is named `-sev-step-<git commit>`


The following is based on [1].

To enable SEV on your system, you might need to change some BIOS options. Look for options regarding
Usually, there is a general "SEV Enabled" option as well as a `SEV-ES ASID space limit` option and a`SNP Memory Coverage`.
`SEV-ES ASID space limit` should be greater than 1.

If SEV is enabled, you  should get values similar to the following when running the specified commands on the host system
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

1) Create VM disk with `./local-installation/usr/local/bin/qemu-img  create -f qcow2 <IMAGE_NAME>.qcow2 20G`
2) Start VM with `./launch-qemu.sh -hda <IMAGE_NAME>.qcow2 -cdrom <Ubuntu 22.04.1 iso> -vnc 1` where `<Ubuntu 22.04.1 iso>` is the path
to the regular Ubuntu installation iso. While you can use other Distros, their SEV support might vary. The most important part is, that they ship at least
kernel version 5.19.
3) Connect to the VM via a VNC viewer on port `5901` and perform a standard installation
4) Once the installation is done, terminate qemu with `ctrl a+x` or use `sudo kill $(pidof qemu-system-x86)`
4) Start the VM again and connect with VNC. Install the OpenSSH server and configure it to autostart. The `./launch-qemu.sh` script already forwards VM port 22 to host port 2222 and VM port 8080 to host port 8080
5) Use `./launch-qemu.sh -hda ../test-vm.qcow2 -console serial -sev-snp` to start the VM with SEV-SNP protection. You might want to also supply the `-allow-debug`
which enables the SEV debug API. Many **test functions** of the sev step framework require this to get access to the VM's instruction pointer or other register content.


## Use Sev Step Library
Head over to the readme in [./sev-step-userland/](sev-step-userland/README.md) to learn how to use the sev step library.

# References
[1] https://github.com/AMDESE/AMDSEV/tree/sev-snp-devel#prepare-host