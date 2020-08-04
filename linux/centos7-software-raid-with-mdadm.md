# Manage Linux Software RAID with mdadm

https://www.thegeekdiary.com/redhat-centos-managing-software-raid-with-mdadm/

> **MD** stands for **M**ultiple **D**evices

**Install mdadm**

```shell
yum install -y mdadm
```

**Create RAID array**

> ```
> # mdadm --create --help
>         -C | --create /dev/mdn
>         -l | --level  linear|0|1|4|5 
>         -n | --raid-devices device [..]
>         -x | --spare-devices device [..]
> ```

```
# create RAID0 array /dev/md0 with two disks /dev/sda and /dev/sdb
mdadm -C --verbose /dev/md0 -l 0 -n 2 /dev/sda /dev/sdb

# RAID1 array with one spare disk
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc --spare-devices=/dev/sdd

# RAID5
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd --spare-devices=/dev/sde
```

**Verify Configuration**

1. Get all RAID arrays

   ```shell
   # cat /proc/mdstat
   Personalities : [raid0]
   md0 : active raid0 sdb[1] sda[0]
         937438208 blocks super 1.2 512k chunks
   
   
   unused devices: <none>
   ```

2. Check RAID array detail

   ```shell
   # mdadm --detail /dev/md0
   /dev/md0:
              Version : 1.2
        Creation Time : Sat Feb  8 16:38:12 2020
           Raid Level : raid0
           Array Size : 937438208 (894.01 GiB 959.94 GB)
         Raid Devices : 2
        Total Devices : 2
          Persistence : Superblock is persistent
   
   
          Update Time : Sat Feb  8 16:38:12 2020
                State : clean
       Active Devices : 2
      Working Devices : 2
       Failed Devices : 0
        Spare Devices : 0
   
   
           Chunk Size : 512K
   
   
   Consistency Policy : none
   
   
                 Name : desktop.homelab:0  (local to host desktop.homelab)
                 UUID : eafe1f61:de102310:e597f55a:8894899c
               Events : 0
   
   
       Number   Major   Minor   RaidDevice State
          0       8        0        0      active sync   /dev/sda
          1       8       16        1      active sync   /dev/sdb
   ```

3. Check member disk status

   ```
   # mdadm --examine /dev/sda
   /dev/sda:
             Magic : a92b4efc
           Version : 1.2
       Feature Map : 0x0
        Array UUID : eafe1f61:de102310:e597f55a:8894899c
              Name : desktop.homelab:0  (local to host desktop.homelab)
     Creation Time : Sat Feb  8 16:38:12 2020
        Raid Level : raid0
      Raid Devices : 2
   
   
    Avail Dev Size : 937438896 sectors (447.01 GiB 479.97 GB)
       Data Offset : 264192 sectors
      Super Offset : 8 sectors
      Unused Space : before=264112 sectors, after=0 sectors
             State : clean
       Device UUID : 564c8005:cc93738b:bdfa4607:8fc8eae8
   
   
       Update Time : Sat Feb  8 16:38:12 2020
     Bad Block Log : 512 entries available at offset 8 sectors
          Checksum : c130a24b - correct
            Events : 0
   
   
        Chunk Size : 512K
   
   
      Device Role : Active device 0
      Array State : AA ('A' == active, '.' == missing, 'R' == replacing)
   ```

**add extra disk to array**

https://zackreed.me/adding-an-extra-disk-to-an-mdadm-array/

```shell
# add disk as host spare
# you can check its status with 'cat /proc/mdstat'
mdadm --add /dev/md0 /dev/sdd

# grow array to new disk
# raid-devices = <total_number_of_disks_including_new>
mdadm --grow /dev/md0 --raid-devices=4

# (optional) check filesystem
xfs_repair /dev/md0

# expand filesystesm


```


**Save mdadm configuration**

The /etc/mdadm.conf file is used to identify which devices are RAID devices and to which array a specific device belongs. This is required to auto-build your RAID devices at boot.

```
mdadm --verbose --detail -scan > /etc/mdadm.conf
```



**Start, Stop and Remove RAID array**

```
mdadm --stop /dev/md0
mdadm --start /dev/md0
mdadm --remove /dev/md0
```



**Fail, Remove and Add a drive**

```
mdadm --manage /dev/md0 -f /dev/sda
mdadm --manage /dev/md0 -r /dev/sda
mdadm --manage /dev/md0 -a /dev/sda
```

