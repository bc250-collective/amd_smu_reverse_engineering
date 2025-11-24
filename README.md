# amd_smu_reverse_engineering
AMD SMU Reverse Engineering (for AMD BC-250)

## Background

### SMU on Ryzen CPUs

The System Management Unit (SMU) is a tiny microcontroller on the CPU die that handles power, voltage, frequency, and thermal management.
It enforces boost behavior, current/power limits, and dynamic voltage/frequency scaling.

### Why Tools Like ZenStates-Core or ryzen_smu Need It

ZenStates-Core or ryzen_smu talks directly to the SMU via low-level commands to change voltages, frequencies, and power limits in real time.
This allows fine-grained tuning (like undervolting or overclocking) that BIOS or standard software can’t fully control.
SMU access is required because the OS alone cannot safely modify these internal settings.

Boards like the AMD BC-250 also have a SMU. Only the interface with the graphics core are documented in the amdgpu code. Currently there is no way to do power management or overclocking on the cpu. Therefore we reverse engineer the SMU firmware to try to gain access to these functions. 

The SMU works by sending messages with ids and parameters to get information back or change the smu state thereby controlling the system. The main goal of this repo is to get a list of these messages and understand their function by means of inspecting the firmware. 

(This was only extensively tested on BC-250 firmware but should with most Xtensa based AMD SMU firmware which is everything since Zen + GPUs since ???)f

## How to extract firmware:

1) Get a bios image 
2) Use psptool https://github.com/PSPReverse/PSPTool to extract the firmware
3) First run <pre> psptool -E bios.bin </pre> to inspect the content
4) SMU firmware images are located in SMU_OFFCHIP_FW_x-entries usually there are 2 (cpu + gpu maybe)
5) Then extract them using <pre> psptool -X -d 0 -e 1 -o firmware.bin bios.bin </pre> (replace directories and entry with your specific entries)
6) You now have the firmware image 
7) To analyze it in ghidra, remove the header with size 0x100: <pre> dd bs=256 skip=1 if=firmware.bin of=firmware_trim.bin </pre> (this is important)

## To install the script "smu_message_helper.py": 
1) Launch ghidra from <pre> /support/pyghidraRun </pre> (make sure pyghidra is working)
2) Open the ghidra script manager and import the script as a pyghidra script (just create a new script and paste the code)

## How to use the script:
1) Import your firmware, the architecture is Xtensa-le and the address space starts at 0x0000_0000
2) Auto analyze the firmware - make sure to disable "Non-Returning Functions - Discovered" in ghidra - else it will get confused

3) Run the script, it will result in something like this in the ghidra console (you can double click on everything to jump to it in ghidra)
   
<pre>
table entry func addr.  symbol
0x00006368  0x0001D0F4  FUN_0001d0f4
0x00006370  0x0001D10C  FUN_0001d10c
...
0x000063D0  0x000295F0  Missing Function at 0x000295F0  </pre>


Missing Functions are a result of ghidra not finding the functions on its own. You can manually add them by going to the address and adding a function label

4) Now you can locate the queue table (look for an xref 4 bytes before the first entry, 0x6364 in this example. at this xref the queue table lives) 

5) Go to the queue table and rename the symbol to "queue_table_x" where x is the number of entries in the table

6) Rerun the script, it will now order message handlers by queue e.g.

<pre>
Start of Queue 0
msgid  table entry func addr.  symbol
0x000063D0
 0x01  0x00006368  0x0001D0F4  FUN_0001d0f4
 0x02  0x00006370  0x0001D10C  FUN_0001d10c
 0x03  0x00006378  0x0001D394  FUN_0001d394
 0x04  0x00006380  0x00000000  No Handler / Reserved </pre>
 
More importantly, it will now also recognize messages with no defined handler.
This allows us to generate the correct message ids

7) If you go to the function of first entry of a queue (especially queue 0) you should see the TestMessage signature in the decompiled code (returns the provided parameter + 1). This confirms the successful decoding 

## How to modify ghidra to work with some missing floating point instructions (not complete):

You need ghidra 12.1+

copy the files in the ghidra folder into
<pre> /opt/ghidra/Ghidra/Processors/Xtensa/data/languages/ </pre>

Then, in /opt/ghidra/ run this:
<pre> support/sleigh -a Ghidra/Processors/Xtensa </pre>

## Example with AMD BC-250: 

I have included bc250_smu_3_trim. 
This is the smu firmware image from BIOS 3.0 of the AMD BC-250. If you extract the data from the bios you will notice that there is only 1 firmware that has significant data in it (indicating that the BC-250 only has one SMU for both gpu and cpu?)

If you get to step 3 you will get an output like this:
(notice that i have already added the matching function names from linux/drivers/gpu/drm/amd/pm/swsmu/inc/pmfw_if/smu_v11_8_ppsmc.h, on your dissasembly they will have generic FUN... names)

<pre>
table entry func addr.  symbol
0x00007070  0x0001B3A8  PPSMC_MSG_TestMessage
0x00007078  0x0001B3C0  PPSMC_MSG_GetSmuVersion
0x00007080  0x0001B94C  PPSMC_MSG_GetDriverIfVersion
0x00007088  0x0001B998  PPSMC_MSG_SetDriverTableDramAddrHigh
0x00007090  0x0001B9B4  PPSMC_MSG_SetDriverTableDramAddrLow
....
</pre>

This output will not contain messages that do not have a defined handler function (i.e. the func addr. is 0000_0000), to get this we need to parse the queue table
(locate it according to step 4)

If you rerun the script you now get:
Notice that the msgids perfectly match with the ids in the amdgpu driver.
You can now start to assign the corresponding the function names

<pre>
Start of Queue 0
msgid  table entry func addr.  symbol
 0x01  0x00007070  0x0001B3A8  PPSMC_MSG_TestMessage
 0x02  0x00007078  0x0001B3C0  PPSMC_MSG_GetSmuVersion
 0x03  0x00007080  0x0001B94C  PPSMC_MSG_GetDriverIfVersion
 0x04  0x00007088  0x0001B998  PPSMC_MSG_SetDriverTableDramAddrHigh
 0x05  0x00007090  0x0001B9B4  PPSMC_MSG_SetDriverTableDramAddrLow
 0x06  0x00007098  0x0001BA5C  PPSMC_MSG_TransferTableSmu2Dram
 0x07  0x000070A0  0x0001BAFC  PPSMC_MSG_TransferTableDram2Smu
 0x08  0x000070A8  0x00000000  No Handler / Reserved
 0x09  0x000070B0  0x00000000  No Handler / Reserved
 0x0A  0x000070B8  0x00000000  No Handler / Reserved
 0x0B  0x000070C0  0x000229DC  PPSMC_MSG_RequestCorePstate
 0x0C  0x000070C8  0x00022AB4  PPSMC_MSG_QueryCorePstate
....
</pre>
There are multiple more queues which also implement PPSMC_MSG_TestMessage / PPSMC_MSG_GetSmuVersion so they are probably also valid (cpu control?)

To check if your results are correct go to PPSMC_MSG_TestMessage (the message with id 1) 
The disassembly should look like this (with generic function names)

<pre>
void PPSMC_MSG_TestMessage(undefined4 param_1)
{
  int iVar1;
  iVar1 = pmfw_queue_read_head(param_1);
  pmfw_queue_store_word(param_1,iVar1 + 1);
  pmfw_queue_write_status(param_1,1);
  return;
}
</pre>
  
It basically returns the parameter + 1 which is intended behaviour (see https://github.com/irusanov/ZenStates-Core/blob/5986e1c380896803d3478ce4eb45b983d60770fa/SMUCommands/SendTestMessage.cs#L4 for example) and returns status ok
