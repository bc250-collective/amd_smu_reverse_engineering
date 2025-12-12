## amdgpu_full_metrics_table.patch
Patch the amdgpu driver in the linux kernel to show the full metrics table (current values, no averages) from the smu. Also sets the max gpu clock to 2230 MHz (hardware limit)
Once patched into the kernel, access them via 
```
cat /sys/devices/pci0000:00/0000:00:08.1/0000:01:00.0/pp_dpm_socclk
```
Obviously this overwrites the default behaviour of pp_dpm_socclk, so don't use it if you rely on it. 
Tested on Kernel 6.18. Should work on others.