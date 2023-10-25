# About this repo

This repository is modified based on the release version of OpenZFS 2.1. This is part of MLEC (Multi-level Erasure Coding) project that aims to implement MLEC in a real system. In the end, we hope to build HDFS on top of OpenZFS, with MLEC implemented.

# Initial Installation

First, reserve a node from Chameleon following this [guide](https://chameleoncloud.readthedocs.io/en/latest/technical/reservations.html), and launch an [instances](https://chameleoncloud.readthedocs.io/en/latest/technical/baremetal.html). The installation generally works for Ubuntu 20.04 and 22.04.

Then, clone the repository, and switch to the branch with MLEC implementation. 
```
git clone https://github.com/zhynwng/zfs-MLEC.git
cd zfs-MLEC/
git checkout zfs-2.1-myraid
```

Run the initialization script, which will install all necessary packages and set up the environment. Press Y for all options during package installation.
```
./zfs_init.sh
```

Then reboot the machine. Make sure all changes are saved before you reboot. 
```
sudo reboot
```

After the reboot, you can run the full installation script. This script will set up system configuration to build ZFS from source.
```
./zfs_full_install.sh
```

You can see whether zfs is successfully installed by trying the command "zfs list". This the expected result:
```
cc@node2:~/zfs-MLEC$ zfs list
no datasets available
```

# Later install and uninstall

After the first installation, you can using the more light-weight installation script. This script omits some of the steps to make the build process faster
```
./zfs_install.sh
```

You can remove ZFS from the system by the following script
```
./zfs_uninstall.sh
```

# Failure and Repair Test 

Here is a hypothetical experiment to munual create and detect failures in ZFS. To conduct, you will need the [pv](https://linux.die.net/man/1/pv) linux tool. 

First, create a test disk images for the ZFS pools. In ZFS, any file can be treated as a disk, with roughly the same functionality, so we can conduct the experiment without using actual disks. 
```
sudo mkdir /scratch/
for i in {1..3}; do sudo truncate -s 4G /scratch/$i.img; done sudo 
```

Then, we create a 2+1 zpool named “test”. It should be mounted the to directory /test/
```
zpool create zpool raidz /scratch/1.img /scratch/2.img /scratch/3.img -f 
```

Then, we manually zero out 1 disks in the zpool. For test purpose, we will first write random bytes to the test zpool until it is full, and then partially zero out 1 disk to see the effect on ZFS data:
```
pv < /dev/urandom > /test/urandom.bin
```

Run this command for 2 second, and then press “^C” to terminate it. 
```
pv < /dev/zero > /scratch/1.img
```

At this point, there should already be some data errors in ZFS. We can conduct a full read by the following command, which will give you some error.
```
pv < /test/urandom.bin > /dev/nul
zpool status
```

Since we have a 2+1 zpool, that means the data failure is tolerable. The repair would be easy. It would be the following command:
```
sudo zpool scrub test
zpool status
```
You should be able to see that data is repaired in the output. 

Finally, destroy experimental zpool. 
```
sudo zpool destroty test
```


There are also testing scripts to test the general functionality of ZFS erasure coding systems (one is for general repair test, the other is to test the new easyscrub method). Feel free to explore and change these two scripts
```
./zfs_test.sh
./scrub_tesh.sh
```