# Fault Tolerant Umbrel Home & Pi 5 RAID - phase 2 - ZFS Setup

# ZFS

```
sudo apt install raspberrypi-kernel-headers
sudo apt install zfs-dkms zfsutils-linux
# so we can hex encode messages
sudo apt install xxd
```

NOTE: over SSH, the license warning will stop you. You will have to hit <tab> to get to the <OK> button and then hit <enter> to accept the license.

```
# verify
dmesg | grep ZFS
```

should output something like this:

```
[60034.431590] ZFS: Loaded module v2.1.11-1, ZFS pool version 5000, ZFS filesystem version 5
```

## Prepare disks

```
lsblk
```

outputs:

```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
└─sda1        8:1    0 238.5G  0 part /media/umbrel/Transcend
```

Now sticker your drive with a label `A`

Add the second drive, then run it again:

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
└─sda1        8:1    0 238.5G  0 part /media/umbrel/Transcend
sdb           8:16   0 238.5G  0 disk
└─sdb1        8:17   0 238.5G  0 part /media/umbrel/Transcend1
```

and sticker the next drive with `B`

repeast for drive 3 (`C`), 4 (`D`), and 5 (`E`).

## Format the drives

The drives have partitions, but we don't want those, so format them:

```
sudo umount /dev/sda?; sudo wipefs --all --force /dev/sda?; sudo wipefs --all --force /dev/sda
sudo umount /dev/sdb?; sudo wipefs --all --force /dev/sdb?; sudo wipefs --all --force /dev/sdb
sudo umount /dev/sdc?; sudo wipefs --all --force /dev/sdc?; sudo wipefs --all --force /dev/sdc
sudo umount /dev/sdd?; sudo wipefs --all --force /dev/sdd?; sudo wipefs --all --force /dev/sdd
sudo umount /dev/sde?; sudo wipefs --all --force /dev/sde?; sudo wipefs --all --force /dev/sde
```

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
└─sda1        8:1    0 238.5G  0 part
sdb           8:16   0 238.5G  0 disk
└─sdb1        8:17   0 238.5G  0 part
sdc           8:32   0 238.5G  0 disk
sdd           8:33   0 238.5G  0 disk
sde           8:36   0 238.5G  0 disk
```

If you see that there are still partitions, run `wipefs` again and make sure they are gone:

```
$ sudo wipefs --all --force /dev/sda
$ sudo wipefs --all --force /dev/sdb
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
sdb           8:16   0 238.5G  0 disk
sdc           8:32   0 238.5G  0 disk
sdd           8:34   0 238.5G  0 disk
sde           8:36   0 238.5G  0 disk
```

# Verify Drives are NOT Dead on Arrival

Since drives can arrive already dead or with unwritable areas, running `badblocks` will write to every block on the drive and verify that it can be written to and read from. This will take a long time, but it's worth it to make sure you don't have a dead drive in your array.

```
sudo badblocks -c 2048 -sw /dev/sda
```

```
Testing with pattern 0xaa:   4.82% done, 0:31 elapsed. (0/0/0 errors)
```

then it will show:

```
$ sudo badblocks -c 2048 -sw /dev/sda
Testing with pattern 0xaa: done
Reading and comparing:  58.07% done, 51:41 elapsed. (0/0/0 errors)
```

then

```
 $ sudo badblocks -c 2048 -sw /dev/sda
Testing with pattern 0xaa: done
Reading and comparing: done
Testing with pattern 0x55:  32.94% done, 1:12:22 elapsed. (0/0/0 errors)
```

it will go through 4 patters, then show:

```
Testing with pattern 0xaa: done
Reading and comparing: done
Testing with pattern 0x55: done
Reading and comparing: done
Testing with pattern 0xff: done
Reading and comparing: done
Testing with pattern 0x00: done
Reading and comparing: done
```

This took just over 4 hours on the first drive.

> NOTE: this drive gets very hot if you write a ton of data very fast, but it's fine :)

Do this for all the drives. You can create multiple ssh/terminal sessions on the node and run them in parallel.

## Disable 5th Drive

Push the power button to disconnect the last drive `sde` -- this drive is a prepared backup drive.
You should buy more of these and have them on hand. If you have a drive failure, you can swap out the bad drive with this one and rebuild the array.

## Create the Pool

```
sudo zpool create tank raidz2 sda sdb sdc sdd
```

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
├─sda1        8:1    0 238.5G  0 part
└─sda9        8:9    0     8M  0 part
sdb           8:16   0 238.5G  0 disk
├─sdb1        8:17   0 238.5G  0 part
└─sdb9        8:25   0     8M  0 part
sdc           8:32   0 238.5G  0 disk
├─sdc1        8:33   0 238.5G  0 part
└─sdc9        8:41   0     8M  0 part
sdd           8:48   0 238.5G  0 disk
├─sdd1        8:49   0 238.5G  0 part
└─sdd9        8:57   0     8M  0 part
```

```
$ zfs list
NAME   USED  AVAIL     REFER  MOUNTPOINT
tank   152K   459G     32.9K  /tank
```

```
$ zpool status
  pool: tank
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	tank        ONLINE       0     0     0
	  raidz2-0  ONLINE       0     0     0
	    sda     ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sdd     ONLINE       0     0     0

errors: No known data errors
```

### And is is mounted

We can verify that our zfs RAIDZ2 drive is mounted as /tank

```
$ df -H /tank
Filesystem      Size  Used Avail Use% Mounted on
tank            494G  132k  494G   1% /tank
```

## Configure ZFS

First, let's see what configuration was automatically created for the pool:

```
$ zfs get all tank
NAME  PROPERTY              VALUE                  SOURCE
tank  type                  filesystem             -
tank  creation              Wed Jan 24  8:09 2024  -
tank  used                  152K                   -
tank  available             459G                   -
tank  referenced            32.9K                  -
tank  compressratio         1.00x                  -
tank  mounted               yes                    -
tank  quota                 none                   default
tank  reservation           none                   default
tank  recordsize            128K                   default
tank  mountpoint            /tank                  default
tank  sharenfs              off                    default
tank  checksum              on                     default
tank  compression           off                    default
tank  atime                 on                     default
tank  devices               on                     default
tank  exec                  on                     default
tank  setuid                on                     default
tank  readonly              off                    default
tank  zoned                 off                    default
tank  snapdir               hidden                 default
tank  aclmode               discard                default
tank  aclinherit            restricted             default
tank  createtxg             1                      -
tank  canmount              on                     default
tank  xattr                 on                     default
tank  copies                1                      default
tank  version               5                      -
tank  utf8only              off                    -
tank  normalization         none                   -
tank  casesensitivity       sensitive              -
tank  vscan                 off                    default
tank  nbmand                off                    default
tank  sharesmb              off                    default
tank  refquota              none                   default
tank  refreservation        none                   default
tank  guid                  18157519002953900807   -
tank  primarycache          all                    default
tank  secondarycache        all                    default
tank  usedbysnapshots       0B                     -
tank  usedbydataset         32.9K                  -
tank  usedbychildren        119K                   -
tank  usedbyrefreservation  0B                     -
tank  logbias               latency                default
tank  objsetid              54                     -
tank  dedup                 off                    default
tank  mlslabel              none                   default
tank  sync                  standard               default
tank  dnodesize             legacy                 default
tank  refcompressratio      1.00x                  -
tank  written               32.9K                  -
tank  logicalused           42K                    -
tank  logicalreferenced     12K                    -
tank  volmode               default                default
tank  filesystem_limit      none                   default
tank  snapshot_limit        none                   default
tank  filesystem_count      none                   default
tank  snapshot_count        none                   default
tank  snapdev               hidden                 default
tank  acltype               off                    default
tank  context               none                   default
tank  fscontext             none                   default
tank  defcontext            none                   default
tank  rootcontext           none                   default
tank  relatime              off                    default
tank  redundant_metadata    all                    default
tank  overlay               on                     default
tank  encryption            off                    default
tank  keylocation           none                   default
tank  keyformat             none                   default
tank  pbkdf2iters           0                      default
tank  special_small_blocks  0                      default
```

Per the tuning guide: https://www.high-availability.com/docs/ZFS-Tuning-Guide/

```
# disable recording of access time, only write create/modify time
sudo zfs set atime=off tank
# we are only going to write large files, so set record size to 1M
sudo zfs set recordsize=1M tank
# enable compression (this makes zfs pools faster!)
sudo zfs set compression=lz4 tank
# use System Attributes (inode instead of hidden directory)
sudo zfs set xattr=sa tank
# enable user acls
sudo zfs set acltype=posixacl tank
```

> NOTE: if you plug in another USB drive directly into a port, it may load up on reboot as one of these drive slots and confuse ZFS! If this happens, you will need to remove the new drive, reboot to rescan the attached disks in the proper order, then add the drive to the end of the chain so that it shows up later.

```
 $ zpool status
  pool: tank
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-4J
config:

	NAME                      STATE     READ WRITE CKSUM
	tank                      DEGRADED     0     0     0
	  raidz2-0                DEGRADED     0     0     0
	    sda                   ONLINE       0     0     0
	    sdb                   ONLINE       0     0     0
	    2252711134268226928   UNAVAIL      0     0     0  was /dev/sdc1
	    17273444761242032304  FAULTED      0     0     0  was /dev/sdd1
```

```
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 238.5G  0 disk
├─sda1        8:1    0 238.5G  0 part
└─sda9        8:9    0     8M  0 part
sdb           8:16   0 238.5G  0 disk
├─sdb1        8:17   0 238.5G  0 part
└─sdb9        8:25   0     8M  0 part
sdc           8:32   0   1.8T  0 disk
└─sdc1        8:33   0   1.8T  0 part /media/zap/Samsung_T2
sdd           8:48   0 238.5G  0 disk
├─sdd1        8:49   0 238.5G  0 part
└─sdd9        8:57   0     8M  0 part
sde           8:64   0 238.5G  0 disk
├─sde1        8:65   0 238.5G  0 part
└─sde9        8:73   0     8M  0 part
mmcblk0     179:0    0 119.4G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 118.9G  0 part /
```

```
umount /media/zap/Samsung_T2
# remove extra drive
sudo reboot
```

```
$ zpool status
  pool: tank
 state: ONLINE
  scan: resilvered 352K in 00:00:00 with 0 errors on Thu Jan 25 08:42:59 2024
config:

	NAME        STATE     READ WRITE CKSUM
	tank        ONLINE       0     0     0
	  raidz2-0  ONLINE       0     0     0
	    sda     ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sdd     ONLINE       0     0     0

errors: No known data errors
```

## Setup Testnet Lightning Nodes

1. install bitcoin and set to Testnet
2. install Lightning Node app from Umbrel Store
3. install Core Lightning app from Umbrel Store
4. deposit Testnet funds via https://alt.signetfaucet.com/ into the Lightning Node (LND)
5. batch open a bunch of channels with push enabled (signet sats are free so we can just gift them to the other side to start with balanced channels)

```
# peer connect
cat addresses.txt | xargs -I {} sudo docker exec lightning_lnd_1 lncli --network testnet connect {}

# batch open channels and push half the sats
cat addresses.txt | while IFS= read -r line; do pubkey=$(echo "$line" | cut -d "@" -f 1); echo "{\"node_pubkey\":\"$pubkey\",\"local_funding_amount\":500000,\"push_sat\":250000}"; done | jq -c -r -s '.' > input.txt


sudo docker exec lightning_lnd_1 lncli --network testnet batchopenchannel --sat_per_vbyte=1 "$(<input.txt)"
```

```
sudo docker exec lightning_lnd_1 lncli --network testnet connect 023bbcc47e1542848e01ccb03e36ea628bdea851dc49bdca203726726e6158d200@181.191.1.242:39735
sudo docker exec lightning_lnd_1 lncli --network testnet openchannel 023bbcc47e1542848e01ccb03e36ea628bdea851dc49bdca203726726e6158d200 500000 250000
```

```
sudo docker exec lightning_lnd_1 lncli --network testnet connect 02312627fdf07fbdd7e5ddb136611bdde9b00d26821d14d94891395452f67af248@23.237.77.12:9735
sudo docker exec lightning_lnd_1 lncli --network testnet openchannel 02312627fdf07fbdd7e5ddb136611bdde9b00d26821d14d94891395452f67af248 500000 250000
```

```
cat addresses.txt | while IFS= read -r line; do pubkey=$(echo "$line" | cut -d "@" -f 1); sudo docker exec lightning_lnd_1 lncli openchannel $pubkey 500000 250000; done
```

```
sudo docker exec bitcoin_bitcoind_1 bitcoin-cli -datadir=/data/.bitcoin/testnet getnewaddress
```

## install bos

```
cd
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
echo `export BOS_DEFAULT_SAVED_NODE=lnd` >> ~/.bashrc
source ~/.bashrc
nvm install stable
npm install -g balanceofsatoshis
mkdir -p ~/.bos/lnd
echo '{"cert_path":"/home/umbrel/umbrel/app-data/lightning/data/lnd/tls.cert","macaroon_path":"/home/umbrel/umbrel/app-data/lightning/data/lnd/data/chain/bitcoin/testnet/admin.macaroon","socket":"127.0.0.1:10009"}' > ~/.bos/lnd/credentials.json
# test it out:
bos utxos
```

## send a payment

```
sudo docker exec lightning_lnd_1 lncli --network testnet sendpayment --dest 02ca56a6e8c10d58fbf0348ad7bba8b3c43a1fb6398ac21115053ac57f2a16e63a --amt 10 --amp
```

---

1. Use multipass to setup multiple testnet hosts

https://github.com/canonical/multipass

```
sudo apt update
sudo apt install snapd
sudo apt install lxd
sudo reboot
```

then

```
sudo lxd -d -v init
sudo snap install core
#sudo snap install lxd
sudo snap install multipass
multipass launch mantic
```
