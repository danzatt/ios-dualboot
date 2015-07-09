Obtaining tools
============
__Note:__ *if you are going to build the tools yourself you will need to have a [Theos](http://www.iphonedevwiki.net/index.php/Theos) set up. If you are on __OS X__ you need to have Xcode installed and follow [the guide on iPhone Dev Wiki](http://www.iphonedevwiki.net/index.php/Theos/Setup#Installing_Theos). If you are on __Windows__ [coolstar has done most all the hard work](https://coolstar.org/iOSToolchainOnWindows.html). And if you are on __Linux__ you can [use Darling to run OS X binaries](http://www.eswick.com/2014/03/building-ios-arm64-binaries-on-linux/), go for a [native toolchain](https://github.com/waneck/linux-ios-toolchain) or [grab a fully setup Theos with compiled toolchain](https://twitter.com/coolstarorg/status/496362772217597952).*  

1. **GPT fdisk**  
To resize **G**UID **P**artition **T**able we will be needing GPT fdisk.  
`git clone git://git.code.sf.net/p/gptfdisk/code gptfdisk-code && cd gptfdisk-code`  
`wget https://gist.githubusercontent.com/danzatt/190544d85b2ae80cf986/raw/ba6b64e692651ec54a273cca7760ffdc2f24e7d1/gptfdisk.patch -O gptfdisk.patch`  
Since  iOS uses weird revision we must patch gpt.cc (+ Theos makefile & control)  
`patch -p1 < gptfdisk.patch`  
Link your Theos installation e.g. `ln -s /opt/theos theos` and `make package install`.

2. **hfs_resize**  
`git clone https://github.com/danzatt/hfs_resize.git && cd hfs_resize`  
Link your Theos installation e.g. `ln -s /opt/theos theos`  and `make package install`.

3. **attach-and-detach** *(optional)*  
*You need to put IOKit headers (can be obtained from e.g. https://github.com/comex/spirit/tree/master/igor/headers/IOKit) to e.g. `/opt/theos/include`.*  
`git clone https://github.com/danzatt/attach-and-detach.git && cd attach-and-detach`  
Link your Theos installation to theos e.g. `ln -s /opt/theos theos` and `make package install`.

So firstly we must decide how much space we are going to dedicate to the other iOS (~1-1.5GB for system partition + min. 2-3GB for data partiton = ~4.5GB at least). I'll use 1GB for system and 5GB for data (6GB total). _6GB = 6 * 1024 ^ 3  = 6442450944_ bytes.  
**EDIT: After writing this article I realised that 1GB system partition isn't enough even for iOS 4. You should use 1.5GB instead.**

Let's check for free space:  
`ssh root@poor-phone.local`  
`df -B1`  
```
poor-phone:~ root# df -B1
Filesystem       1B-blocks       Used   Available Use% Mounted on
/dev/disk0s1s1  1444536320 1115332608   314761216  78% /
devfs                34816      34816           0 100% /dev
/dev/disk0s1s2 14510596096 4416036864 10094559232  31% /private/var
```
`/dev/disk0s1s2` has _10094559232_ bytes available, OK. It has total of 14510596096 bytes. 14510596096 - 6442450944 = 8068145152 (the new size).  
```
poor-phone:~ root# hfs_resize /private/var/ 8068145152
Resizing /private/var/ to 8068145152 bytes.
```
Good, we have just shrunk the filesystem and made free space for the new partitions. Now we have to edit the GPT:  
`gptfdisk /dev/rdisk0s1` *(on the device, of course)*  
get info about the second partiton (`i <Enter> 2 <Enter>`):
```
Command (? for help): i
Partition number (1-2): 2
Partition GUID code: 48465300-0000-11AA-AA11-00306543ECAC (Apple HFS/HFS+)
Partition unique GUID: 1AFFD618-1C26-408E-975B-F20562CE7F9D
First sector: 176339 (at 1.3 GiB)
Last sector: 1947651 (at 14.9 GiB)
Partition size: 1771313 sectors (13.5 GiB)
Attribute flags: 0003000000000000
Partition name: 'Data'
```
write down the partition unique GUID (`1AFFD618-1C26-408E-975B-F20562CE7F9D` in my case). Now delete the 2nd partition so that we can create a smaller one.  
`d <Enter> 2 <Enter>`  
```
Command (? for help): d
Partition number (1-4): 2
```
Create new partition `n <Enter>`. Keep the first sector default `<Enter>`.  
```
Command (? for help): n
Using 2
First sector (176339-1947651, default = 176339) or {+-}size{KMGTP}: <Enter>
```
Now comes the tricky part. If your size wasn't blocksize-aligned and hfs_resize complained ("[-] Adjusting size to xyz to match next block.") you should use *xyz* in this calculation. To calculate the last sector we must divide the size we resized to (or size reported by hfs_resize) by blocksize (8192): 8068145152 / 8192 = 984881. And add it to the default first sector (176339 in my case): 984881 + 176339 = 1161220. 
```
Last sector (176339-1947651, default = 1947651) or {+-}size{KMGTP}: 1161220
Current type is 'Apple HFS/HFS+'
Hex code or GUID (L to show codes, Enter = AF00): <Enter>
Changed type of partition to 'Apple HFS/HFS+'
```
change the name to "Data":
```
Command (? for help): c
Partition number (1-2): 2
Enter name: Data
```
enter expert mode and turn on attribute 48 and 49 (`x <Enter> a <Enter> 2 <Enter> 48 <Enter> 49 <Enter> <Enter>`):
```
Command (? for help): x

Expert command (? for help): a
Partition number (1-2): 2
Known attributes are:
0: system partition
1: hide from EFI
2: legacy BIOS bootable
60: read-only
62: hidden
63: do not automount

Attribute value is 0000000000000000. Set fields are:
  No fields set

Toggle which attribute field (0-63, 64 or <Enter> to exit): 48
Have enabled the 'Undefined bit #48' attribute.
Attribute value is 0001000000000000. Set fields are:
48 (Undefined bit #48)

Toggle which attribute field (0-63, 64 or <Enter> to exit): 49
Have enabled the 'Undefined bit #49' attribute.
Attribute value is 0003000000000000. Set fields are:
48 (Undefined bit #48)
49 (Undefined bit #49)

Toggle which attribute field (0-63, 64 or <Enter> to exit): <Enter>
```
now we must restore the unique GUID we have written down (`c <Enter> 2 <Enter> <paste GUID> <Enter>`):
```
Expert command (? for help): c
Partition number (1-2): 2
Enter the partition's new unique GUID ('R' to randomize): 1AFFD618-1C26-408E-975B-F20562CE7F9D
New GUID is 1AFFD618-1C26-408E-975B-F20562CE7F9D
```

and return back to basic mode (`m <Enter>`):  
```
Expert command (? for help): m
Command (? for help):
```
Now we need to create two new partitions but firstly we must resize the partition table itself (it currently can hold 2 entries, we want 4).  
Enter expert mode `x <Enter>`, resize table to 4 `s <Enter> 4 <Enter>` and return from expert mode `m <Enter>`:
```
Command (? for help): x

Expert command (? for help): s
Current partition table size is 2.
Enter new size (64 up, default 128): 4 
Caution: The partition table size should officially be 16KB or larger,
which works out to 128 entries. In practice, smaller tables seem to
work with most OSes, but this practice is risky. I'm proceeding with
the resize, but you may want to reconsider this action and undo it.

Adjusting GPT size from 4 to 64 to fill the sector

Expert command (? for help): m
```

`gptfdisk` has rounded that it to 64 entries but since this isn't real GPT LwVM will remove all empty entries after reboot.  
Now create the actual partitions `n <Enter> <Enter> <Enter>` (keep the index and first sector default):
```
Command (? for help): n
Partition number (3-4, default 3): 
First sector (1161221-1947651, default = 1161221) or {+-}size{KMGTP}:
```
To calculate the last sector you divide the size of the second system partition (I am using 1GB **EDIT: and should have used 1.5GB instead**) by blocksize: *1073741824 (1GB) / 8192 = 131072* and add the result to the default first sector (1161221 in my case): *1161221 + 131072 = 1292293*  
```Last sector (1161221-1947651, default = 1947651) or {+-}size{KMGTP}: 1292293```  
When it asks you for "Hex code or GUID" keep the default one (so just `<Enter>`):
```
Current type is 'Apple HFS/HFS+'
Hex code or GUID (L to show codes, Enter = AF00): 
Changed type of partition to 'Apple HFS/HFS+'
```
*(You can type `p <Enter>` to print the partition map so that you can check what you've just done. If that's not what you expected... don't worry we haven't written the changes yet so you can `Ctrl + C` and do the whole process again.)*

Create fourth partition (`n <Enter> <Enter> <Enter>`):  

```
Command (? for help): n
Using 4
First sector (1292294-1947651, default = 1292294) or {+-}size{KMGTP}:
```

Theoretically you could keep the last sector default (as that's the LBA = last usable block/sector specified by the GPT itself) but a LwVM seemed to complain about it being out of rage so you better subtract ~2 blocks from the default last sector:
```
Last sector (1292294-1947651, default = 1947651) or {+-}size{KMGTP}:
```
In my case: 1947651 - 2 = 1947649 (so `1947649 <Enter>`) and again keep the default hex code/whatever (`<Enter>`):
```
Last sector (1292294-1947651, default = 1947651) or {+-}size{KMGTP}: 1947649
Current type is 'Apple HFS/HFS+'
Hex code or GUID (L to show codes, Enter = AF00): 
Changed type of partition to 'Apple HFS/HFS+'
```

*(Remember ? `p <Enter>`)*

now you can rename the partitions (`c <Enter>) 3 <Enter> System2 <Enter> c <Enter> 4 <Enter> Data2 <Enter>`):
```
Command (? for help): c
Partition number (1-4): 3
Enter name: System2

Command (? for help): c
Partition number (1-4): 4
Enter name: Data2
```
And set the flags (`x <Enter> a <Enter> 4 <Enter> 48 <Enter> 49 <Enter> <Enter>`):
```
Command (? for help): x

Expert command (? for help): a
Partition number (1-4): 4
Known attributes are:
0: system partition
1: hide from EFI
2: legacy BIOS bootable
60: read-only
62: hidden
63: do not automount

Attribute value is 0000000000000000. Set fields are:
  No fields set

Toggle which attribute field (0-63, 64 or <Enter> to exit): 48
Have enabled the 'Undefined bit #48' attribute.
Attribute value is 0001000000000000. Set fields are:
48 (Undefined bit #48)

Toggle which attribute field (0-63, 64 or <Enter> to exit): 49
Have enabled the 'Undefined bit #49' attribute.
Attribute value is 0003000000000000. Set fields are:
48 (Undefined bit #48)
49 (Undefined bit #49)

Toggle which attribute field (0-63, 64 or <Enter> to exit): 

Expert command (? for help):
```

Now we can write our changes (`w <Enter>`). **Check everything again (remember ?). LwVM does some bounds checking but it (I guess) won't prevent you from cutting the end of HFS filesystem (i.e. making GPT partition smaller than HFS on said partition).**

Now reboot so that LwVM can do the hard work for us:   
`poor-phone:~ root# reboot`

After the system boots up you can run `gptfdisk /dev/rdisk0s1` again and `p <Enter> q <Enter>` to see if our changes have been applied. Also check if new devices are there:
```
poor-phone:~ root# ls /dev/disk0s1*
/dev/disk0s1  /dev/disk0s1s1  /dev/disk0s1s2  /dev/disk0s1s3  /dev/disk0s1s4
```
If they aren't there repeat the whole process.

Create HFS on the new partitions:  
`newfs_hfs -s -b 8192 -J 8192k -v System /dev/rdisk0s1s3`  
`newfs_hfs -s -b 8192 -J 8192k -v Data /dev/rdisk0s1s4`
  
Now we can copy the rootfs to the newly created partitions. Firstly we need a copy of IPSW of the iOS we want to dualboot. You can get the link at [ipsw.me](https://ipsw.me/). I'll use 4.3.3. Go to [the iPhone Wiki](https://www.theiphonewiki.com/wiki/Firmware_Keys) and find the version you've downloaded (make sure you get the keys for the right model). Extract the rootfs (the exact filename is on the iPhone Wiki) and decrypt it using [VFDecrypt](https://www.theiphonewiki.com/wiki/VFDecrypt):  
`vfdecrypt -k 246f17ec6660672b3207ece257938704944a83601205736409b61fc3565512559abd0f82 -i 038-1423-003.dmg -o 4_3_3_rootfs.dmg`  
Now copy it to the device:  
`scp 4_3_3_rootfs.dmg root@poor-phone.local:/tmp` *(or you can copy over USB if you have AFC2 installed and your Wifi sucks)*  
\**On the device*\* : attach the DMG:  
`attach  /tmp/4_3_3_rootfs.dmg`  
mount it  
`mkdir /mnt/dmg`  
`mount_hfs -o ro /dev/disk1s3 /mnt/dmg/`  
mount the other partitions  
`mkdir /mnt/second`  
`mount_hfs /dev/disk0s1s3 /mnt/second/`  

`mkdir -p /mnt/second/private/var`  
`mount_hfs /dev/disk0s1s4 /mnt/second/private/var/`  
copy the rootfs to the new partitions  
`cp -a -v /mnt/dmg/* /mnt/second/`  
clean up  
`umount /mnt/dmg/`  
`detach disk1`  
`rm /tmp/4_3_3_rootfs.dmg` *(or reboot, lol)*  
Now we need to edit `fstab` to reflect our partition map. *(Note: IIRC newer iOS versions ignore fstab)*  
`nano /mnt/second/etc/fstab`
and change it to  
```
/dev/disk0s3 / hfs ro 0 1
/dev/disk0s4 /private/var hfs rw,nosuid,nodev 0 2
```

After that copy decrypted kernelcache (using keys from the iPhone Wiki) to `/mnt/second/System/Library/Caches/com.apple.kernelcaches/kernelcache`  

This is the furthest I managed to get. Now we need to boot from the second partition. We can do so by using `boot-partition` nvram (partitions are indexed from 0, so third partition is 2. [More info here](http://wikee.iphwn.org/s5l8900:dualboot).) and bootstrapping pwned iBEC + iBoot using `kloader`. The problem is if something goes wrong and your device reboots stock iBoot will accept this variable (tested on 7.1.2 iBoot) which will result in iBoot attempting to load kernel from third partition which (obviously) isn't signed and encrypted. After failed attempt iBoot will enter Recovery Mode. Because recovery shell doesn't allow you to set critical variables (`boot-partition` included) your device **will be stuck in recovery mode**. **Therefore I highly recomend you to not touch this variable**. To get around it we need to patch iBoot to use 2 as `boot-partiton` regardless of the actual variable. I also realised that iOS 4 kernelcache doesn't support LwVM so it probably couldn't boot anyway (and newer versions ignore fstab which probably requires further kernel/kext patching to re-enable it). `ibsspatch` does common patches for iOS 4 & 7 iBoot/iBSS/iBEC (disable signature, enable `boot-args` nvram, disable checking of IMG3 tags) but it doesn't disable/NOP out image decryption (which is required for decrypted bootchain, i.e. when using `kloader`).
