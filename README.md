Xenomai 3.1.1 + linux 5.4.124 on Ubuntu 20.04 + NVIDIA driver+ CUDA

---Sunhong Kim & Minchang Sung----
06.17.2023 Update

# Install the build-tools and dependencies
```bash
sudo apt-get update
sudo apt install build-essential kernel-package libncurses5 libncurses5-dev libelf-dev openssl libssl-dev libtool libltdl-dev git distcc xenomai-system-tools rt-tests libc-dev libc6-dev pkg-config ncurses-dev stress autoconf libncurses-dev flex bison libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev dwarves fakeroot bin86 curl bc gcc
```
--> install the package maintainer's version


```bash
mkdir ~/xeno_ws
cd ~/xeno_ws
```
# Get Xenomai 3.1.1
```bash
 git config --global http.sslverify false
 git clone https://source.denx.de/Xenomai/xenomai.git
 cd xenomai 
 git checkout v3.1.1 
 cd ../
```
# Get ipipe patched Linux kernel
```bash
 git clone https://source.denx.de/Xenomai/ipipe-x86.git
 git cd ipipe-x86
 git checkout ipipe-core-5.4.124-x86-5 
 wget https://xenomai.org/downloads/ipipe/v5.x/x86/ipipe-core-5.4.124-x86-5.patch
 ../xenomai/scripts/prepare-kernel.sh --arch=x86_64 --ipipe=ipipe-core-5.4.124-x86-5.patch
```

#  Configure the kernel
```bash
 yes "" | make oldconfig
```
```bash
 cp /path/to/config .config
 make menuconfig
```
Recommended options:

* General setup
  --> Local version - append to kernel release: -xenomai-3.1.1
  --> Timers subsystem
      --> High Resolution Timer Support (Enable)
  -->  Preemption Model
      --> Preemption Kernel (Low-Latency Desktop)
      
* Pocessor type and features
  --> Numa Memory Allocation and Scheduler Support (Disable)
  --> Linux guest support (Disable) 
  --> Processor family 
      --> Core 2/newer Xeon
      --> if "`cat /proc/cpuinfo | grep family`" returns 6, set as Generic otherwise
  // Xenomai will issue a warning about CONFIG_MIGRATION, disable those in this order
  --> Multi-core scheduler support (Enable)
  --> CPU core priorities scheduler support (Disable)
  --> Timer frequency (1000 Hz)

* Power management and ACPI options
  --> CPU Frequency scaling (disable)
  --> ACPI (Advanced Configuration and Power Interface) Support 
  	--> Processor (disable)
  --> CPU Idle --> CPU idle PM support (disable)
  
* Xenomai/cobalt
  --> Sizes and static limits
    --> Number of registry slots (512 --> 4096)
    --> Size of system heap (Kb) (512 --> 4096)
    --> Size of private heap (Kb) (64 --> 256)
    --> Size of shared heap (Kb) (64 --> 256)
    --> Maximum number of POSIX timers per process (128 --> 512)
  --> Drivers
    --> Real-time IPC drivers
      --> RTIPC protocol family(Enable)
    --> RTnet
        --> RTnet, TCP/IP socket interface (enable)
        	--> Protocol stack --> TCP support (enable)
        	--> Drivers
        		//
        		--> New Intel(R) PRO/1000 PCIe (Gigabit) (Enable)
        		// If you use external NIC
        		--> Intel(R) 82575(Gigabit) (Enable)
        	--> Add-Ons
        		--> Real-Time Capturing Support (Enable)
              
* Memory Management options 
  --> Transparent Hugepage Support (Disable)
  --> Contiguous Memory Allocator (Disable)
  --> Allow for memory compaction (Disable)
      --> Page Migration (Disable)
 
* Device Drivers
  --> Staging drivers
      --> Unisys SPAR driver support (Disable)
  --> Network device support(enable)
      --> Ethernet driver support (enable)
          --> # enable ethernet driver w.r.t. each h/w system
  ???
  {--> Graphics support --> Frame buffer Devices
  --> Microsoft Hyper-V guest support
      --> < > Microsoft Hyper-V client drivers (Disable)
  }???

* Kernel hacking
  --> KGDB: kernel debugger (Disable)
  --> Debug preemtible kernel (Disable)

# Preventing the error occurs such as 
#make[2]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
#make[2]: *** Waiting for unfinished jobs....
#make[1]: *** [Makefile:1734: certs] Error 2
#make[1]: *** Waiting for unfinished jobs....
#make[1]: Leaving directory '/home/xeno/xeno_ws/linux-5.4.124'
#make: *** [debian/ruleset/targets/common.mk:301: debian/stamp/build/kernel] Error 2
$ sudo gedit .config
# CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem" --> CONFIG_SYSTEM_TRUSTED_KEYS=""
# CONFIG_SYSTEM_REVOCATION_KEYS="debian/canonical-certs.pem" --> CONFIG_SYSTEM_REVOCATION_KEYS=""

  
### to make package file of the kernel
$ sudo CONCURRENCY_LEVEL=$(nproc) make-kpkg --rootcmd fakeroot --initrd kernel_image kernel_headers

$ cd ..
$ sudo dpkg -i linux-headers-5.4.124-xenomai-3.1.1+_5.4.124-xenomai-3.1.1+-10.00.Custom_amd64.deb linux-image-5.4.124-xenomai-3.1.1+_5.4.124-xenomai-3.1.1+-10.00.Custom_amd64.deb


######  Allow non-root user

$ sudo addgroup xenomai --gid 1234
if 'addgroup: The group 'xenomai' already exists.' error occurs,
$ sudo groupmod -g 1234 xenomai
$ sudo addgroup root xenomai
$ sudo usermod -a -G xenomai $USER

Configure GRUB and reboot
Editing the grub config:

$ sudo gedit /etc/default/grub

GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.4.124-xenomai-3.1.1+"
#GRUB_DEFAULT=saved
#GRUB_SAVEDEFAULT=true
# Comment the following lines
#GRUB_HIDDEN_TIMEOUT=0
#GRUB_HIDDEN_TIMEOUT_QUIET=true
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash xenomai.allowed_group=1234"
GRUB_CMDLINE_LINUX=""
If you have an Intel HD Graphics integrated GPU (any type) :
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_rc6=0 i915.enable_dc=0 noapic xenomai.allowed_group=1234"
# This removes powersavings from the graphics, that creates disturbing interruptions.
If you have an Intel Skylake (2015 processors), you need to add nosmap to fix the latency hang

GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_rc6=0 i915.enable_dc=0 xenomai.allowed=1234 nosmap"

#####  Update grub and reboot
$ sudo update-initramfs -c -k 5.4.124-xenomai-3.1.1+
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
$ sudo update-grub
$ sudo reboot


######  Installing Xenomai 3.1.1 User space libraries
First, make sure that you are running the cobalt kernel :

$ uname -a
# Should return Linux ${YOUR_NAME} 5.4.124-xenomai-3.1.1 #2 SMP Wed Sep 20 16:00:12 CEST 2017 x86_64 x86_64 x86_64 GNU/Linux
$ dmesg | grep Xenomai
# [    1.417024] [Xenomai] scheduling class idle registered.
# [    1.417025] [Xenomai] scheduling class rt registered.
# [    1.417045] [Xenomai] disabling automatic C1E state promotion on Intel processor
# [    1.417055] [Xenomai] SMI-enabled chipset found, but SMI workaround disabled
# [    1.417088] I-pipe: head domain Xenomai registered.
# [    1.417704] [Xenomai] allowing access to group 1234
# [    1.417726] [Xenomai] Cobalt v3.0.9 (Sisyphus's Boulder) [DEBUG]

$ cd ~/xeno_ws/xenomai
$ automake --add-missing
$ autoreconf -i
$ ./configure --with-pic --with-core=cobalt --enable-smp --disable-tls --enable-dlopen-libs --disable-clock-monotonic-raw
$ sudo make -j`nproc`
$ sudo make install
# --disable-clock-monotonic-raw : http://xenomai.org/pipermail/xenomai/2017-September/037695.html

Update your bashrc

$ echo '
### Xenomai
export XENOMAI_ROOT_DIR=/usr/xenomai
export XENOMAI_PATH=/usr/xenomai
export PATH=$PATH:$XENOMAI_PATH/bin:$XENOMAI_PATH/sbin
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$XENOMAI_PATH/lib/pkgconfig
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$XENOMAI_PATH/lib
export OROCOS_TARGET=xenomai
' >> ~/.bashrc

$ source ~/.bashrc

$ sudo chmod -R 777 /dev/rtdm/memdev-private
$ sudo chmod -R 777 /dev/rtdm/memdev-shared

# If you have a problem that sudo does not recognize the SHARED LIBRARY PATH, type the following command.

$ cd /etc/ld.so.conf.d/
$ sudo gedit xenomai.conf
'
### Multiarch support
/usr/xenomai/lib
'
$ sudo ldconfig -v


######  Test your installation
$ cd /usr/xenomai/bin
$ sudo ./latency
This loop will allow you to monitor a xenomai latency. Here’s the output for an i7 4Ghz :

== Sampling period: 100 us
== Test mode: periodic user-mode task
== All results in microseconds
warming up...
RTT|  00:00:01  (periodic user-mode task, 100 us period, priority 99)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|      0.174|      0.464|      1.780|       0|     0|      0.174|      1.780
RTD|      0.088|      0.464|      1.357|       0|     0|      0.088|      1.780
RTD|      0.336|      0.464|      1.822|       0|     0|      0.088|      1.822
RTD|      0.342|      0.464|      1.360|       0|     0|      0.088|      1.822
RTD|      0.327|      0.462|      2.297|       0|     0|      0.088|      2.297
RTD|      0.347|      0.463|      1.313|       0|     0|      0.088|      2.297
RTD|      0.314|      0.464|      1.465|       0|     0|      0.088|      2.297
RTD|      0.190|      0.464|      1.311|       0|     0|      0.088|      2.297

######  Fix negative latency issues
You need to be in root sudo -s, then you can set values to the latency calibration variable in nanoseconds:

$ echo 0 > /proc/xenomai/latency
# Now run the latency test

# If the minimum latency value is positive,
# then get the lowest value from the latency test (ex: 0.088 us)
# and write it to the calibration file ( here you have to write 88 ns) :
$ echo ${my_super_value_in_ns} > /proc/xenomai/latency


######  References
[1] https://www.programmersought.com/article/14375437246/
[2] 실시간 EtherCAT 마스터 구현에 관한 연구




$ cd /usr/src
$ sudo -s

$ git clone https://github.com/SHKim-HYU/ethercat.git
# This repository is forked from 'e1000e-5.4' branch for e1000e (https://gitlab.com/etherlab.org/ethercat.git) and from 'stable/vectioneer' branch for igb (https://git.vectioneer.com/pub/etherlab/-/tree/stable/vectioneer/)
# In this repository, there are some update w.r.t. xenomai migration from 2 to 3
# Please refer to ribalda's repository (https://github.com/ribalda/ethercat)

$ cd ethercat

### 1) e1000e-5.4
$ ./bootstrap
$ ./configure --with-module-dir=/lib/modules/5.4.124-xenomai-3.1.1+ --enable-rtdm --with-xenomai-dir=/usr/xenomai --disable-8139too --enable-e1000e --enable-generic --prefix=/opt/etherlab

### 2) igb-5.4
$ ./bootstrap
$ ./configure --with-module-dir=/lib/modules/5.4.124-xenomai-3.1.1+ --enable-rtdm --with-xenomai-dir=/usr/xenomai --disable-8139too --enable-igb --enable-generic --prefix=/opt/etherlab

$ make modules -j$(nproc)
$ make -j$(nproc)
$ make install
$ make modules_install

# Now the Etherlab master is installed in ”/opt/etherlab” and has to be configured at application level. Therefore:

$ cd /opt/etherlab/
$ mkdir /etc/sysconfig
$ cp etc/sysconfig/ethercat /etc/sysconfig/
$ ln -s /opt/etherlab/etc/init.d/ethercat /etc/init.d/ethercat

# following command makes the system call the ethercat driver when it starts
# If your system is Ubuntu 16.04, then

$ /usr/lib/insserv/insserv /etc/init.d/ethercat

# else if the system Ubuntu 18.04. Open the terminal

$ systemctl enable ethercat

# Open "gedit /etc/sysconfig/ethercat" (for instance with nano) and edit it with the previously noted information:

$ gedit /etc/sysconfig/ethercat

# fill in the information following the lshw -class network

...
MASTER0_DEVICE="$(YOUR_PCI_SERIAL)"
...
DEVICE_MODULES="$(YOUR_PCI_DRIVER)"

# Now we are good to go and we can test if the master can work properly with:

$ ./etc/init.d/ethercat start

# You should see the following line printed on the console:

# Starting EtherCAT master 1.5.2 done

# Additional command line tools can be installed by creating the following link:

$ ln -s /opt/etherlab/bin/ethercat /usr/local/bin/ethercat



# NVIDIA DRIVER install
Software & Updates -> Additional Drivers -> Using NVIDIAdriver(open kernel) metapackage from nvidia-driver-530(proprietary) check
->reboot
# CUDA 11.2 install
```bash
  wget https://developer.download.nvidia.com/compute/cuda/11.2.0/local_installers/cuda_11.2.0_460.27.04_linux.run
  sudo chmod 777 cuda_11.2.0_460.27.04_linux.run
  init 3
```
```bash
  login : user 
  passwd : passwd

  ./cuda_11.2.0_460.27.04_linux.run
  -> continue
  -> accept
  -> Don't check "INSTALL NVIDIA DRIVER"
  -> INSTALL
  -> reboot
```







![Screenshot from 2023-06-17 19-07-00](https://github.com/MinchangSung0223/Ubuntu20.04_xenomai_3_1_1_etherlab_nvidia_CUDA/assets/53217819/24ba7331-308c-4ca1-8f9f-8814dfffb748)





