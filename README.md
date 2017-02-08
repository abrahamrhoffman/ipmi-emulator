# ipmi-emulator
Emulator for the Intelligent Platform Management Interface (IPMI) on KVM
<br><br>
<b>Background</b><br>
The Intelligent Platform Management Interface (IPMI), is a standard interface for Independant Lights Out (ILO) and the Dell Remote Access Controllers (DRAC). Although newer APIs extist for controlling physical servers out-of-band, this remains the current market leader. For those interested in the latest standard, you can read about the DMTF's RedFish standard here: https://www.dmtf.org/standards/redfish <br>
<br>
<b>Purpose</b>
Those interested in exploring the ipmi API and interface, but without access to an ILO / DRAC, or other out-of-band management device, this is for you! I found myself interested in testing a scaled down version of OpenStack Ironic (called BiFrost), but I did not want to interrupt current DHCP settings.<br>
<br>
<b>Design</b>
The script interacts directly with KVM. Reboot machines, change boot order, PXE boot and view console logs.

<b>Dependencies</b>
- qemu-kvm
- OpenIPMI
- OpenIPMI-tools

1) Install qemu-kvm. Proper versioning is critical, as we need the QEMU -> IPMI libraries.
   git clone https://github.com/cminyard/qemu/tree/stable-2.2-ipmi
   ./configure; make && make install
   
2) Download OpenIPMI libraries: http://sourceforge.net/projects/openipmi/
   Follow the documented process in 'lanserv/README.vm'
      ./configure --prefix=/opt/openipmi/usr --sysconfdir=/opt/openipmi/etc \
            --with-perlinstall=/opt/openipmi/usr/lib/perl \
            --with-pythoninstall=/opt/openipmi/usr/lib/python
      make && make install
      
If you received the following errors while compiling: 
'/usr/bin/ld: cannot find -lOpenIPMIutils'
export CC="gcc -lrt" and reconfigure

3) Configure the IPMI simulator:
- qemu-img create -f qcow2 -o size=1G /opt/KVM/disk1.qcow2
- download test kernel: http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-i386-kernel
- edit /opt/openipmi/etc/ipmi/lan.conf
- modify 'startcmd' to use the qemu compiled
- /usr/local/bin/qemu-system-x86_64 --enable-kvm -drive file=/opt/KVM/Test4sda -nographic -net nic,model=e1000,macaddr=52:54:00:12:34:59 -net user,hostfwd=tcp::5556-10.0.2.15:22 -chardev socket,id=ipmi0,host=localhost,port=9011,reconnect=10 -device isa-ipmi,chardev=ipmi0,interface=bt,irq=5 -serial mon:tcp::9012,server,telnet,nowait
- /usr/local/bin/qemu-system-x86_64 --enable-kvm -drive file=/opt/KVM/disk1.qcow2,format=qcow2 -nographic -net nic,model=e1000,macaddr=52:54:00:12:34:59 -net user,hostfwd=tcp::5555-10.0.2.15:22 -chardev socket,id=ipmi0,host=localhost,port=9002,reconnect=10 -device isa-ipmi,chardev=ipmi0,interface=bt,irq=5 -serial mon:tcp::9003,server,telnet,nowait -kernel /opt/cirros-0.3.4-i386-kernel --append 'root=/dev/hda console=ttyS0,115200'
- Serial console is configured to /dev/ttyUSB0 in the lan.conf file, Find if your serial console is configured as such (or) change.
- sol "/dev/ttyS0" 38400 history=4000 historyfru=10

4) Run IPMI-Emulator
/opt/openipmi/usr/bin/ipmi-emulator.py

5) Connect to the simulator using ipmitool as below:
 ipmitool -I lanplus -H localhost -p 9011 -U cirros power status
