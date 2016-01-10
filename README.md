# laughing-parakeet

I am trying to build my own Linux derivative to run on an TI-AR7 board. I took the board from an old Telekom Speedport W 501V router. To understand how firmware is flashed onto the device I have downloaded the most recent [official firmware](https://www.telekom.de/hilfe/geraete-zubehoer/router/weitere-router/speedport-w-5xx-serie/speedport-w-501v?samChecked=true). Using the Linux ```file``` command I determined the image is a tar archive, which can be extracted easily.

```
ubuntu@ip-172-31-23-210:~/reverse$ ls
fw_speedport_w501v_v_28.04.38.image
ubuntu@ip-172-31-23-210:~/reverse$ file fw*
fw_speedport_w501v_v_28.04.38.image: POSIX tar archive (GNU)
ubuntu@ip-172-31-23-210:~/reverse$ tar -xvf fw*
./var/
./var/tmp/
./var/tmp/kernel.image
./var/tmp/filesystem.image
./var/flash_update.ko
./var/flash_update.o
./var/info.txt
./var/install
./var/chksum
./var/regelex
./var/signature
ubuntu@ip-172-31-23-210:~/reverse$
```

According to a [wiki (Firmware-Image)](http://www.wehavemorefun.de/fritzbox/Firmware-Image) that I have found, ```./var/tmp/kernel.image``` contains the actual firmware. During the update process this image is written to the ```mtd1``` device. As stated in the [wiki (LZMA-Kernel)](http://www.wehavemorefun.de/fritzbox/LZMA-Kernel) the lzma compressed kernel starts with the magic number ```0xfeed1281```. A hexdump of  ```kernel.image``` contains that number at its beginning.

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ hexdump -n 4 kernel.image
0000000 1281 feed
0000004
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$
```

The following script given on the last wiki entry should decompress the kernel.

```perl
#! /usr/bin/perl

use Compress::unLZMA;
use Archive::Zip;

open INPUT, "<$ARGV[0]" or die "can't open $ARGV[0]: $!";

read INPUT, $buf, 4;
$magic = unpack("V", $buf);
if ($magic != 0xfeed1281) {
  die "bad magic";
}

read INPUT, $buf, 4;
$len = unpack("V", $buf);

read INPUT, $buf, 4*2; # address, unknown

read INPUT, $buf, 4;
$clen = unpack("V", $buf);
read INPUT, $buf, 4;
$dlen = unpack("V", $buf);
read INPUT, $buf, 4;
$cksum = unpack("V", $buf);
printf "Archive checksum: 0x%08x\n", $cksum;

read INPUT, $buf, 1+4; # properties, dictionary size
read INPUT, $dummy, 3; # alignment
$buf .= pack('VV', $dlen, 0); # 8 bytes of real size
#$buf .= pack('VV', -1, -1); # 8 bytes of real size
read INPUT, $buf2, $clen;

$crc = Archive::Zip::computeCRC32($buf2);
printf "Input CRC32: 0x%08x\n", $crc;
if ($cksum != $crc) {
  die "wrong checksum";
}
$buf .= $buf2;

$data = Compress::unLZMA::uncompress($buf);
unless (defined $data) {
  die "uncompress: $@";
}

open OUTPUT, ">$ARGV[1]" or die "can't write $ARGV[1]";
print OUTPUT $data;
#truncate OUTPUT, $dlen;
```

To use the script you may need to install [Compress::unLZMA](http://search.cpan.org/~ferreira/Compress-unLZMA-0.04/lib/Compress/unLZMA.pm) and [Archive::Zip](http://search.cpan.org/~phred/Archive-Zip-1.56/lib/Archive/Zip.pm) perl modules.

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ tar -xvf Compress*
Compress-unLZMA-0.04/
Compress-unLZMA-0.04/Makefile.PL
Compress-unLZMA-0.04/ppport.h
Compress-unLZMA-0.04/Changes
Compress-unLZMA-0.04/lzma_sdk/
[...]
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ cd Compress*
ubuntu@ip-172-31-23-210:~/reverse/var/tmp/Compress-unLZMA-0.04$ perl Makefile.PL
Checking if your kit is complete...
Looks good
Writing Makefile for Compress::unLZMA
Writing MYMETA.yml and MYMETA.json
ubuntu@ip-172-31-23-210:~/reverse/var/tmp/Compress-unLZMA-0.04$ make
cp lib/Compress/unLZMA.pm blib/lib/Compress/unLZMA.pm
/usr/bin/perl /usr/share/perl/5.18/ExtUtils/xsubpp  -typemap /usr/share/perl/5.18/ExtUtils/typemap  unLZMA.xs > unLZMA.xsc && mv unLZMA.xsc unLZMA.c
cc -c  -I. -Ilzma_sdk/Source -D_REENTRANT -D_GNU_SOURCE
[...]
ubuntu@ip-172-31-23-210:~/reverse/var/tmp/Compress-unLZMA-0.04$ sudo make install
Files found in blib/arch: installing files in blib/lib into architecture dependent library tree
Installing /usr/local/lib/perl/5.18.2/auto/Compress/unLZMA/unLZMA.bs
Installing /usr/local/lib/perl/5.18.2/auto/Compress/unLZMA/unLZMA.so
Installing /usr/local/lib/perl/5.18.2/Compress/unLZMA.pm
Installing /usr/local/man/man3/Compress::unLZMA.3pm
Appending installation info to /usr/local/lib/perl/5.18.2/perllocal.pod
ubuntu@ip-172-31-23-210:~/reverse/var/tmp/Compress-unLZMA-0.04$ # same for Archive::Zip module
```

After installing these dependencies the script decompressed the kernel successfully.

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ ./decompress.pl kernel.image kernel.decompressed
Archive checksum: 0x29176e12
Input CRC32: 0x29176e12
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$
```

But what kind of file is ```kernel.decompressed``` and how do I generate a similar file from my Linux kernel source? I continued analyzing it using ```file``` and [binwalk](https://github.com/devttys0/binwalk).

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ file kernel.decompressed
kernel.decompressed: data
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ binwalk kernel.decompressed

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
1509632       0x170900        Linux kernel version "2.6.13.1-ohio (686) (gcc version 3.4.6) #9 Wed Apr 4 13:48:08 CEST 2007"
1516240       0x1722D0        CRC32 polynomial table, little endian
1517535       0x1727DF        Copyright string: "Copyright 1995-1998 Mark Adler "
1549488       0x17A4B0        Unix path: /usr/gnemul/irix/
1550920       0x17AA48        Unix path: /usr/lib/libc.so.1
1618031       0x18B06F        Neighborly text, "neighbor %.2x%.2x.%.2x:%.2x:%.2x:%.2x:%.2x:%.2x lost on port %d(%s)(%s)"
1966080       0x1E0000        gzip compressed data, maximum compression, from Unix, last modified: 2007-04-04 11:45:13

ubuntu@ip-172-31-23-210:~/reverse/var/tmp$
```

So the Linux kernel starts at ```1509632``` and ends at ```1516240```. What kind of data is stored in front the Linux kernel (```0``` to ```1509632```)? I extracted the kernel and that piece of unknown data using ```dd```.

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ dd if=kernel.decompressed of=unknown.data bs=1 count=1509632
1509632+0 records in
1509632+0 records out
1509632 bytes (1.5 MB) copied, 1.62137 s, 931 kB/s
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ dd if=kernel.decompressed of=kernel bs=1 skip=1509632 count=6608
6608+0 records in
6608+0 records out
6608 bytes (6.6 kB) copied, 0.0072771 s, 908 kB/s
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$
```

I need to ask again: What kind of file is ```kernel``` and how do I generate a similar file from my Linux kernel source? I used ```xxd``` and ```strings``` to look at the file more closely.

```
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ xxd -l 100 kernel
0000000: 4c69 6e75 7820 7665 7273 696f 6e20 322e  Linux version 2.
0000010: 362e 3133 2e31 2d6f 6869 6f20 2836 3836  6.13.1-ohio (686
0000020: 2920 2867 6363 2076 6572 7369 6f6e 2033  ) (gcc version 3
0000030: 2e34 2e36 2920 2339 2057 6564 2041 7072  .4.6) #9 Wed Apr
0000040: 2034 2031 333a 3438 3a30 3820 4345 5354   4 13:48:08 CEST
0000050: 2032 3030 370a 0000 0000 0000 0000 0000   2007...........
0000060: 0000 0000                                ....
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$ strings kernel
Linux version 2.6.13.1-ohio (686) (gcc version 3.4.6) #9 Wed Apr 4 13:48:08 CEST 2007
do_be
do_bp
do_tr
do_ri
do_cpu
nmi_exception_handler
do_ade
emulate_load_store_insn
do_page_fault
context_switch
__put_task_struct
do_exit
local_bh_enable
run_workqueue
2.6.13.1-ohio gcc-3.4
enable_irq
__free_pages_ok
free_hot_cold_page
prep_new_page
kmem_cache_destroy
kmem_cache_create
pageout
vunmap_pte_range
vmap_pte_range
__vunmap
__brelse
sync_dirty_buffer
bio_endio
queue_kicked_iocb
proc_get_inode
remove_proc_entry
sysfs_get
sysfs_fill_super
kref_get
kref_put
0123456789abcdefghijklmnopqrstuvwxyz
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ
vsnprintf
{zt^f
pw0Gm
0cIZ-
68BG+
QC]S%
v,;Zk
ubuntu@ip-172-31-23-210:~/reverse/var/tmp$
```
