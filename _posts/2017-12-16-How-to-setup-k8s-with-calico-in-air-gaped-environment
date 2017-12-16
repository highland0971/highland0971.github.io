 1. Setup the kvm-qemu virtual machine with virt-install
 
 virt-install -n master --memory 3072 --vcpus 2 \
  -l /tmp/CentOS-7-x86_64-Minimal-1708.iso \
  -w network=default --graphics=none \
  --disk path=/mnt//slow_storage/master.qcow2 \
  --os-type=linux \
  --extra-args='console=tty0 console=ttyS0,115200n8 serial'
  
  
