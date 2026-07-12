# Local-Labs
Practice Labs for cybersecurity/cloud/linux enthusiasts 

**# Setting up LVM(Linux Volume Management):(CENTOS)**

LVM is a storage management layer that sits between your physical disk and filesystem.
Instead of creating partitions directly on disk and formatting them , LVM creates flexible storage pool that you can resize , combine and manage much easily.

**3 Building Blocks :** 1) Physical Volume (PV): A storage device that LVM can use . eg. "This disk is now part of LVM".
2)Volume Group (VG): Pool of storage made from one or more PV.
3)Logical Volumes (LV): Similar to normal partitions , but more dynamic (add , remove , extend ,etc.).



<img width="860" height="531" alt="image" src="https://github.com/user-attachments/assets/a8a16344-e7e6-42de-8233-001960b465d0" />


**STEPS:**

1)Create a new disk on your VM by going into the settings and then storage

now inside the VM:

2)fdisk -l

2)fdisk /dev/sdc

3)n

4)p

5)Enter for first sector

6)Enter for last sector

7)p = print the partition table

8)t = change a partition's system id

9)L = type L to list all codes

10)8e = Partition type from Linux to Linux LVM

11)p = verify partition table

12)w

13)Create Physical Volume (PV) = pvcreate /dev/sdc1

14)Verify physical volume = pvdisplay

15)Create Volume Group (VG) = vgcreate shlok_vg /dev/sdc1

16)Verify Volume group = vgdisplay shlok_vg

17)Create Logical Volumes (LV) = lvcreate –n shlok_lv –size 1G shlok_vg

18)Verify logical volumes = lvdisplay

19)Format Logical Volumes = mkfs.xfs /dev/shlok_vg/shlok_lv
20)Create a new directory = mkdir /shlokdisk

21)Mount the new file system = mount /dev/shlok_vg/shlok_lv /shlokdisk

22)Verify = df –h
