# Nextcloud on OpenBSD

Summary
---

![puffy.png](images/Puffy.png)

The most secure cloud server on the most secure operating system.

### Goal

* OpenBSD 6.6 with full disk encryption
* Nextcloud 18.0.4
* PostgreSQL
* PHP 7.3 and PHP-FPM
* OpenBSD httpd
* Caching and file-locking through Redis

Preparation
---

1. Assemble your hardware

	I used an [APU2E4](https://www.varia-store.com/en/produkt/36818-pc-engines-apu2e4-system-board-3x-lan-4-gb-ram.html) with 500 GB 860 EVO SATA III mSATA SSD. Don't forget to setup cooling on the [APU2E4](https://www.pcengines.ch/apucool.htm), otherwise it will randomly shut down every couple minutes when it overheats. You will also need a USB to Serial (9-Pin) Converter Cable and Modem Serial RS232 Cable to connect over serial port for OS installation.

2. Connect over serial


	The APU serial connection uses 115200 baud rate, 8N1 (8 data bits, no parity, 1 stop bit).

	On OpenBSD, use [cu](https://man.openbsd.org/cu):

	```console
	$ doas cu -r -s 115200 -l cuaU0
	```

	On Linux, use [screen](https://www.gnu.org/software/screen/manual/screen.html):

	```console
	$ screen /dev/ttyUSB0 115200 8N1
	```

	Or use [minicom](https://linux.die.net/man/1/minicom):

	```console
	$ sudo minicom -D /dev/ttyUSB0
	```

	Power on the APU and make note of the firmware version displayed briefly during boot.

3. Update firmware

	Follow [this section](https://github.com/drduh/PC-Engines-APU-Router-Guide#updating-firmware) of the excellent drduh guide.

	Note that the APU2E4 takes files with the `apu2` prefix, not the `apu4` prefix drduh uses.

4. Prepare OpenBSD installer

	Use another computer to prepare an installer for OpenBSD 6.6. This example uses Debian 10.

	Download the installation image - [`amd64/install66.fs`](https://cdn.openbsd.org/pub/OpenBSD/6.6/amd64/install66.fs)  and [`SHA256.sig`](https://cdn.openbsd.org/pub/OpenBSD/6.6/amd64/SHA256.sig) files.

	Verify the signatures file and hash of the installation image from Debian 10 (you might have to manually add openbsd-66-base.pub):

	```console
	$ sudo apt-get install signify-openbsd signify-openbsd-keys

	$ cat /usr/share/signify-openbsd-keys/openbsd-66-base.pub
	untrusted comment: openbsd 6.6 base public key
	RWSvK/c+cFe24BIalifKnqoqdvLlXfeZ9MIj3MINndNeKgyYw5PpcWGn

	$ signify-openbsd -C -p /usr/share/signify-openbsd-keys/openbsd-66-base.pub -x SHA256.sig install66.fs
	Signature Verified
	install66.fs: OK
	```

	Insert a USB disk. Run `dmesg` to identify its label. Then copy the installation file to the USB disk:

	```console
	$ sudo dd if=install66.fs of=/dev/sdd bs=1M
	```

Install OpenBSD 6.6
---

1. Boot the APU and set serial console parameters

	Press `F10` at boot and select the USB disk.

	Set the serial console parameters:

	```console
	Booting from Hard Disk...
	Using drive 0, partition 3.
	Loading......
	probing: pc0 com0 com1 com2 com3 mem[639K 3325M 752M a20=on]
	disk: hd0+ hd1+*
	>> OpenBSD/amd64 BOOT 3.45
	boot> stty com0 115200
	boot> set tty com0
	switching console to com>> OpenBSD/amd64 BOOT 3.45
	boot> [Press Enter]
	```

2. Enable full disk encryption

	Select the shell option:

	```console
	Welcome to the OpenBSD/amd64 6.6 installation program.
	(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? S
	```

	Since the installer does not have many device nodes by default, make sure the /dev/sd0 device exists:

	```console
	# cd /dev && sh MAKEDEV sd0
	```

	You may want to write random data to the drive first with something like the following. This will take a while:

	```console
	# dd if=/dev/urandom of=/dev/rsd0c bs=1m
	```

	If you're booting from MBR (which we are on the APU), do:

	```console
	# fdisk -iy sd0
	Writing MBR at offset 0.
	```

	Next, create the partition layout:

	```console
	# disklabel -E sd0
	Label editor (enter '?' for help at any prompt)
	sd0> a a
	offset: [64]
	size: [39825135] *
	FS type: [4.2BSD] RAID
	sd0> w
	sd0> q
	No label changes.
	```

	We'll use the entire disk, but note that the encrypted device can be split up into multiple partitions as if it were a regular hard drive.

	Now we can build the encrypted device on our "a" partition.

	```console
	# bioctl -c C -l sd0a softraid0
	New passphrase: 
	Re-type passphrase: 
	sd2 at scsibus2 targ 1 lun 0: <OPENBSD, SR CRYPTO, 006>
	sd2: 476937MB, 512 bytes/sector, 976767473 sectors
	softraid0: CRYPTO volume attached as sd2
	```

	Make sure the /dev/sd2 device is accounted for:

	```console
	# cd /dev && sh MAKEDEV sd2
	```

	As in the previous example, we'll overwrite the first megabyte of our new pseudo-device.

	```console
	# dd if=/dev/zero of=/dev/rsd2c bs=1m count=1
	```

	Type `exit` to return to the main installer.

3. Install OpenBSD 6.6

	Select the install option:

	```console
	Welcome to the OpenBSD/amd64 6.6 installation program.
	(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? I
	```

	Perform your install (em0 is the ethernet port closest to the serial port):

	```console
	Terminal type? [vt220] 
	System hostname? (short form, e.g. 'foo') nextcloud

	Available network interfaces are: em0 em1 em2 vlan0.
	Which network interface do you wish to configure? (or 'done') [em0] 
	IPv4 address for em0? (or 'dhcp' or 'none') [dhcp] 
	em0: * lease accepted from * (*)
	IPv6 address for em0? (or 'autoconf' or 'none') [none]
	Available network interfaces are: em0 em1 em2 vlan0.
	Which network interface do you wish to configure? (or 'done') [done] 
	Using DNS domainname *
	Using DNS nameservers at *

	Password for root account? (will not echo)
	Password for root account? (again)
	Start sshd(8) by default? [yes]
	Change the default console to com0? [yes]
	Available speeds are: 9600 19200 38400 57600 115200.
	Which speed should com0 use? (or 'done') [115200] 
	Setup a user? (enter a lower-case loginname, or 'no') [no] admin
	Full name for user admin? [admin] 
	Password for user admin? (will not echo) 
	Password for user admin? (again) 
	WARNING: root is targeted by password guessing attacks, pubkeys are safer.
	Allow root ssh login? (yes, no, prohibit-password) [no] 
	What timezone are you in? ('?' for list) [*] UTC

	Available disks are: sd0 sd1 sd2.
	Which disk is the root disk? ('?' for details) [sd0] ?
	sd0: ATA, Samsung SSD 860, RVT4 naa.5002538e40e58a29 (465.8G)
	sd1: SanDisk, Ultra USB 3.0, 1.00 serial.07815591270227120553 (14.3G)
	sd2: OPENBSD, SR CRYPTO, 006  (465.8G)
	Available disks are: sd0 sd1 sd2.
	Which disk is the root disk? ('?' for details) [sd0] sd2
	No valid MBR or GPT.
	Use (W)hole disk MBR, whole disk (G)PT or (E)dit? [whole] 
	Setting OpenBSD MBR partition to whole sd2...done.
	```

	We need to enlarge the /var partition because this is where our nextcloud data directory is located. We do this by swapping the sizes of /home and /var (i.e. make /home 11.9G and /var 300.0G:

	```console
	The auto-allocated layout for sd2 is:
	#                size           offset  fstype [fsize bsize   cpg]
	  a:             1.0G               64  4.2BSD   2048 16384     1 # /
	  b:             4.2G          2097216    swap                    
	  c:           465.8G                0  unused                    
	  d:             4.0G         10941664  4.2BSD   2048 16384     1 # /tmp
	  e:            11.9G         19330240  4.2BSD   2048 16384     1 # /var
	  f:             3.0G         44359136  4.2BSD   2048 16384     1 # /usr
	  g:             1.0G         50650592  4.2BSD   2048 16384     1 # /usr/X11R6
	  h:            20.0G         52747744  4.2BSD   2048 16384     1 # /usr/local
	  i:             2.0G         94690784  4.2BSD   2048 16384     1 # /usr/src
	  j:             6.0G         98885088  4.2BSD   2048 16384     1 # /usr/obj
	  k:           300.0G        111468032  4.2BSD   4096 32768     1 # /home
	Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout? [a] e
	Label editor (enter '?' for help at any prompt)
	sd2> p
	OpenBSD area: 64-976752000; size: 976751936; free: 236138472
	#                size           offset  fstype [fsize bsize   cpg]
	  a:          2097152               64  4.2BSD   2048 16384     1 # /
	  b:          8844440          2097216    swap                    
	  c:        976767473                0  unused                    
	  d:          8388576         10941664  4.2BSD   2048 16384     1 # /tmp
	  e:         25028896         19330240  4.2BSD   2048 16384     1 # /var
	  f:          6291456         44359136  4.2BSD   2048 16384     1 # /usr
	  g:          2097152         50650592  4.2BSD   2048 16384     1 # /usr/X11R6
	  h:         41943040         52747744  4.2BSD   2048 16384     1 # /usr/local
	  i:          4194304         94690784  4.2BSD   2048 16384     1 # /usr/src
	  j:         12582912         98885088  4.2BSD   2048 16384     1 # /usr/obj
	  k:        629145536        111468032  4.2BSD   4096 32768     1 # /home
	sd2> c k
	Partition k is currently 629145536 sectors in size, and can have a maximum
	size of 865283968 sectors.
	size: [629145536] 11.9G
	sd2*> R e
	[+|-]new size (with unit): [25028896] 300.0G
	sd2*> p
	OpenBSD area: 64-976752000; size: 976751936; free: 236195291
	#                size           offset  fstype [fsize bsize   cpg]
	  a:          2097152               64  4.2BSD   2048 16384     1 # /
	  b:          8844440          2097216    swap                    
	  c:        976767473                0  unused                    
	  d:          8388576         10941664  4.2BSD   2048 16384     1 # /tmp
	  e:        629145600         19330240  4.2BSD   4096 32768     1 # /var
	  f:          6291456        648475840  4.2BSD   2048 16384     1 # /usr
	  g:          2097152        654767296  4.2BSD   2048 16384     1 # /usr/X11R6
	  h:         41943040        656864448  4.2BSD   2048 16384     1 # /usr/local
	  i:          4194304        698807488  4.2BSD   2048 16384     1 # /usr/src
	  j:         12582912        703001792  4.2BSD   2048 16384     1 # /usr/obj
	  k:         24972013        715584704  4.2BSD   4096 32768     1 # /home
	sd2*> w
	sd2> q
	No label changes.
	/dev/rsd2a: 1024.0MB in 2097152 sectors of 512 bytes
	6 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2k: 12193.4MB in 24972008 sectors of 512 bytes
	15 cylinder groups of 814.44MB, 26062 blocks, 52224 inodes each
	/dev/rsd2d: 4096.0MB in 8388576 sectors of 512 bytes
	21 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2f: 3072.0MB in 6291456 sectors of 512 bytes
	16 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2g: 1024.0MB in 2097152 sectors of 512 bytes
	6 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2h: 20480.0MB in 41943040 sectors of 512 bytes
	102 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2j: 6144.0MB in 12582912 sectors of 512 bytes
	31 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2i: 2048.0MB in 4194304 sectors of 512 bytes
	11 cylinder groups of 202.47MB, 12958 blocks, 25984 inodes each
	/dev/rsd2e: 307200.0MB in 629145600 sectors of 512 bytes
	378 cylinder groups of 814.44MB, 26062 blocks, 52224 inodes each
	Available disks are: sd0 sd1.
	Which disk do you wish to initialize? (or 'done') [done] 
	/dev/sd2a (e1a56df125cd094b.a) on /mnt type ffs (rw, asynchronous, local)
	/dev/sd2k (e1a56df125cd094b.k) on /mnt/home type ffs (rw, asynchronous, local, nodev, nosuid)
	/dev/sd2d (e1a56df125cd094b.d) on /mnt/tmp type ffs (rw, asynchronous, local, nodev, nosuid)
	/dev/sd2f (e1a56df125cd094b.f) on /mnt/usr type ffs (rw, asynchronous, local, nodev)
	/dev/sd2g (e1a56df125cd094b.g) on /mnt/usr/X11R6 type ffs (rw, asynchronous, local, nodev)
	/dev/sd2h (e1a56df125cd094b.h) on /mnt/usr/local type ffs (rw, asynchronous, local, nodev)
	/dev/sd2j (e1a56df125cd094b.j) on /mnt/usr/obj type ffs (rw, asynchronous, local, nodev, nosuid)
	/dev/sd2i (e1a56df125cd094b.i) on /mnt/usr/src type ffs (rw, asynchronous, local, nodev, nosuid)
	/dev/sd2e (e1a56df125cd094b.e) on /mnt/var type ffs (rw, asynchronous, local, nodev, nosuid)

	Let's install the sets!
	Location of sets? (disk http nfs or 'done') [http] 
	HTTP proxy URL? (e.g. 'http://proxy:8080', or 'none') [none] 
	HTTP Server? (hostname, list#, 'done' or '?') cdn.openbsd.org
	Server directory? [pub/OpenBSD/6.6/amd64]

	Select sets by entering a set name, a file name pattern or 'all'. De-select
	sets by prepending a '-', e.g.: '-game*'. Selected sets are labelled '[X]'.
	    [X] bsd           [X] base66.tgz    [X] game66.tgz    [X] xfont66.tgz
	    [X] bsd.mp        [X] comp66.tgz    [X] xbase66.tgz   [X] xserv66.tgz
	    [X] bsd.rd        [X] man66.tgz     [X] xshare66.tgz
	Set name(s)? (or 'abort' or 'done') [done] 
	Get/Verify SHA256.sig   100% |**************************|  2141       00:00    
	Signature Verified
	Get/Verify bsd          100% |**************************| 18250 KB    00:02    
	Get/Verify bsd.mp       100% |**************************| 18336 KB    00:02    
	Get/Verify bsd.rd       100% |**************************| 10058 KB    00:01    
	Get/Verify base66.tgz   100% |**************************|   236 MB    00:34    
	Get/Verify comp66.tgz   100% |**************************| 72109 KB    00:10    
	Get/Verify man66.tgz    100% |**************************|  7418 KB    00:01    
	Get/Verify game66.tgz   100% |**************************|  2745 KB    00:00    
	Get/Verify xbase66.tgz  100% |**************************| 22092 KB    00:03    
	Get/Verify xshare66.tgz 100% |**************************|  4482 KB    00:00    
	Get/Verify xfont66.tgz  100% |**************************| 39342 KB    00:06    
	Get/Verify xserv66.tgz  100% |**************************| 15757 KB    00:02    
	Installing bsd          100% |**************************| 18250 KB    00:00    
	Installing bsd.mp       100% |**************************| 18336 KB    00:00    
	Installing bsd.rd       100% |**************************| 10058 KB    00:00    
	Installing base66.tgz   100% |**************************|   236 MB    00:41    
	Extracting etc.tgz      100% |**************************|   260 KB    00:00    
	Installing comp66.tgz   100% |**************************| 72109 KB    00:19    
	Installing man66.tgz    100% |**************************|  7418 KB    00:02    
	Installing game66.tgz   100% |**************************|  2745 KB    00:00    
	Installing xbase66.tgz  100% |**************************| 22092 KB    00:05    
	Extracting xetc.tgz     100% |**************************|  7017       00:00    
	Installing xshare66.tgz 100% |**************************|  4482 KB    00:02    
	Installing xfont66.tgz  100% |**************************| 39342 KB    00:07    
	Installing xserv66.tgz  100% |**************************| 15757 KB    00:03    
	Location of sets? (disk http nfs or 'done') [done]
	Saving configuration files... done.
	Making all device nodes... done.
	Multiprocessor machine; using bsd.mp instead of bsd.
	Relinking to create unique kernel... done.

	CONGRATULATIONS! Your OpenBSD install has been successfully completed!

	When you login to your new system the first time, please read your mail
	using the 'mail' command.

	Exit to (S)hell, (H)alt or (R)eboot? [reboot]
	```

4. Configure your system

	Don't forget to remove your USB stick, or you will continue to boot into the installer. With full disk encryption [if your password starts with an "n"](https://poolp.org/posts/2018-01-29/install-openbsd-on-dedibox-with-full-disk-encryption/) you must first enter the serial console parameters before you can decrypt the disk. Press enter twice to skip the password prompt:

	```console
	Booting from Hard Disk...
	Using drive 0, partition 3.
	Loading......
	probing: pc0 com0 com1 com2 com3 mem[639K 3325M 752M a20=on] 
	disk: hd0+ sr0*
	>> OpenBSD/amd64 BOOT 3.45
	Passphrase: 
	aborting...
	Passphrase: 
	aborting...
	open(sr0a:/etc/boot.conf): Operation not permitted
	boot> stty com0 115200
	boot> set tty com0
	switching console to com>> OpenBSD/amd64 BOOT 3.45
	boot> 
	Passphrase:
	```

	Log in as admin

	Enable doas:

	```console
	$ su
	# cp /etc/examples/doas.conf /etc
	```

	Make it look like this:

	```
	# $OpenBSD: doas.conf,v 1.1 2016/09/03 11:58:32 pirofti Exp $
	# Configuration sample file for doas(1).
	# See doas.conf(5) for syntax and examples.

	# Non-exhaustive list of variables needed to build release(8) and ports(7)
	#permit nopass setenv { \
	#    FTPMODE PKG_CACHE PKG_PATH SM_PATH SSH_AUTH_SOCK \
	#    DESTDIR DISTDIR FETCH_CMD FLAVOR GROUP MAKE MAKECONF \
	#    MULTI_PACKAGES NOMAN OKAY_FILES OWNER PKG_DBDIR \
	#    PKG_DESTDIR PKG_TMPDIR PORTSDIR RELEASEDIR SHARED_ONLY \
	#    SUBPACKAGE WRKOBJDIR SUDO_PORT_V1 } :wsrc

	# Allow wheel by default
	permit persist keepenv :wheel
	```

	Test doas:

	```console
	# exit
	$ doas su
	doas (admin@nextcloud.*) password:
	# exit
	```

	Enable ntpd. Edit `/etc/rc.conf.local` and append:

	```
	ntpd_flags=-s
	```

	Disable root account:

	```console
	$ doas usermod -p'*' root
	```

	Update your system:

	```console
	$ doas syspatch
	$ doas fw_update
	```

	[Add your ssh key](https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/) to `~/.ssh/authorized_keys`

	Edit `/etc/ssh/sshd_config` and set these values unless you explicitly need them:

	```
	PasswordAuthentication no
	PermitRootLogin no
	MaxAuthTries 2
	MaxSessions 2
	AllowAgentForwarding no
	AllowTcpForwarding no
	TCPKeepAlive no
	Compression no
	ClientAliveInterval 2
	ClientAliveCountMax 2
	```

	Paste this to /etc/pf.conf:

	```
	#! ##################################### !#
	#! ## pf.conf for NextCloud instances ## !#
	#! ##             ---                 ## !#
	#! ## Last edit: 20200330 by admin    ## !#
	#! ##################################### !#


	#####################################
	###    MACRO AND TABLE SECTION    ###
	#####################################

	ext_if = "em0"							# External interface
	int_if = "lo0"							# Internal/loopback interface
	public_ip = "{" $ext_if "}"					# Grabs the public IPv4 and IPv6 of this machine
	local_ip = $int_if:0						# Grabs the local/loopback IP

	in_tcp_services_restricted = "{ ssh }" 				# Allow SSH incoming - restrictive
	in_tcp_services = "{ http, https }" 				# Allow HTTP and HTTPS incoming
	out_tcp_services = "{ ssh, http, https, domain, ntp, 67 }" 	# Allow SSH, HTTP, HTTPS, DNS, NTP, and DHCP outgoing
	out_udp_services = "{ domain, ntp, 67 }" 			# Allow DNS, NTP, and DHCP outgoing
	lo_tcp_services = "{ postgresql, 9000, 6379}"			# Loopback rule for PSQL and Redis

	icmp_types = "{ echoreq, unreach }"
	icmp6_types = "{ echoreq, routersol, routeradv, neighbradv, neighbrsol }" 

	table <bruteforce> persist				# Table for SSH bruteforcers

	match in all scrub (no-df)

	#####################################
	####        FIREWALL RULES        ###
	#####################################

	block log all

	antispoof log quick for $ext_if
	antispoof log for $int_if

	block in log quick from {no-route urpf-failed} to any

	block log quick from <bruteforce>
	block log quick from <fail2ban>

	pass in on $ext_if inet proto tcp from any to $public_ip port $in_tcp_services_restricted flags S/SA keep state (max-src-conn 8, max-src-conn-rate 4/30, overload <bruteforce> flush global)
	#pass in on $ext_if inet6 proto tcp from any to $public_ip port $in_tcp_services_restricted flags S/SA keep state (max-src-conn 8, max-src-conn-rate 4/30, overload <bruteforce> flush global)

	pass in on $ext_if inet proto tcp from any to $public_ip port $in_tcp_services keep state
	#pass in on $ext_if inet6 proto tcp from any to $public_ip port $in_tcp_services keep state

	pass out on $ext_if inet proto tcp from $public_ip to any port $out_tcp_services keep state
	#pass out on $ext_if inet6 proto tcp from $public_ip to any port $out_tcp_services keep state
	pass out on $ext_if inet proto udp from $public_ip to any port $out_udp_services keep state
	#pass out on $ext_if inet6 proto udp from $public_ip to any port $out_udp_services keep state

	pass on $int_if inet proto tcp from $local_ip to $local_ip port $lo_tcp_services
	#pass on $int_if inet6 proto tcp from $local_ip to $local_ip port $lo_tcp_services


	pass on $ext_if inet proto icmp all icmp-type $icmp_types keep state
	#pass on $ext_if inet6 proto icmp6 all icmp6-type $icmp6_types keep state

	pass out on $ext_if inet proto udp from $public_ip to any port 33433 >< 33626 keep state
	#pass out on $ext_if inet6 proto udp from $public_ip to any port 33433 >< 33626 keep state
	```

	Turn PF off and back on again:

	```console
	$ doas pfctl -d ; doas pfctl -e -f /etc/pf.conf
	pf disabled
	pf enabled 
	```

	Reboot so you boot on a brand new up-to-date system with latest stable kernel:

	```console
	$ doas reboot
	```

Download/install prerequisites
---

1. Install PostgreSQL server:

	```console
	$ doas pkg_add postgresql-server
	```

2. Install PHP and some modules. If you are asked which PHP-version to install, answer PHP 7.3 each time.

	```console
	$ doas pkg_add php php-curl php-gd php-intl php-pdo_pgsql php-xml php-zip redis pecl73-redis
	```

3. Enable the PHP modules

	```console
	$ doas cp /etc/php-7.3.sample/* /etc/php-7.3
	```

4. Automatically start these services. Do not start these services yet. 

	```console
	$ doas rcctl enable postgresql php73_fpm redis
	```

5. Install a file so the webserver can resolve DNS queries from inside its chroot and another file so NextCloud can verify HTTPS certificates:

	```console
	$ doas mkdir -p /var/www/etc/ssl
	$ doas cp /etc/resolv.conf /var/www/etc/resolv.conf
	$ doas cp /etc/ssl/cert.pem /var/www/etc/ssl/cert.pem
	$ doas cp /etc/ssl/openssl.cnf /var/www/etc/ssl/openssl.cnf
	$ doas chown -R www:www /var/www/etc
	```

Dynamic DNS
---

1. If you're setting up the box at home and you have a dynamic IP address I recommend getting a [dynamic DNS host](https://nsupdate.info).

2. Install ddclient

	```console
	$ doas pkg_add ddclient
	```

3. Append dynamic DNS host's ddclient config to `/etc/ddclient/ddclient.conf`

4. Inital sync

	```console
	$ doas ddclient
	```

5. Enable ddclient service

	```console
	$ doas rcctl enable ddclient
	```

6. Apply correct ownership to `/var/db/client/ddclient.cache`

	```console
	$ doas chown -R _ddclient:wheel /var/db/ddclient
	```

Enable and configure httpd
---

1. Use this configuration as a starting point for your `/etc/httpd.conf`, replacing `<domain.tld>` with the dynamic DNS hostname you just created:

	```
	server "<domain.tld>" {
		listen on * port 80
		root "/nextcloud"
		location "/.well-known/acme-challenge/*" {
		        root { "/acme" }
		        request strip 2
		}
	}
	```

	Check whether the configuration is deemed valid with `doas httpd -n`. 

2. Prepare acme-client for the SSL certificates, courtesy of Let’s Encrypt. Edit `/etc/acme-client`, replacing `<domain.tld>` with your dynamic DNS hostname:

	```
	authority letsencrypt {
		api url "https://acme-v02.api.letsencrypt.org/directory"
		account key "/etc/acme/letsencrypt-privkey.pem"
	}
	authority letsencrypt-staging {
		api url "https://acme-staging.api.letsencrypt.org/directory"
		account key "/etc/acme/letsencrypt-staging-privkey.pem"
	}
	domain <domain.tld> {
		domain key "/etc/ssl/private/<domain.tld>.key"
		domain certificate "/etc/ssl/<domain.tld>.crt"
		domain full chain certificate "/etc/ssl/<domain.tld>.pem"
		sign with letsencrypt
	}
	```

3. Create the corresponding directories:

	```console
	$ doas mkdir -p -m 700 /etc/acme
	$ doas mkdir -p -m 700 /etc/ssl/acme/private
	$ doas mkdir -p -m 755 /var/www/acme
	```

4. Time to fetch the certificates

	```console
	$ doas rcctl enable httpd && doas rcctl restart httpd && doas acme-client -v <domain.tld>
	```

5. If that went successful, grab the OCSP stapling file:

	```console
	$ doas ocspcheck -N -o /etc/ssl/<domain.tld>.ocsp.pem /etc/ssl/<domain.tld>.pem
	```

6. Edit the crontab to automatically renew the certificates and stapling file. Append the following in the crontab (`doas crontab -e`):

	```
	0 0 * * * acme-client <domain.tld> && rcctl reload httpd
	0 * * * * ocspcheck -N -o /etc/ssl/<domain.tld>.ocsp.pem /etc/ssl/<domain.tld>.pem && rcctl reload httpd
	```

Configure httpd further
---

1. Edit /etc/httpd.conf with these values. _Do not restart httpd afterwards!_

	```
	server "<domain.tld>" {
		listen on * tls port 443
		hsts {
		        preload
		        subdomains
		}
		root "/nextcloud"
		directory index "index.php"
		tls {
		        certificate "/etc/ssl/<domain.tld>.pem"
		        key "/etc/ssl/private/<domain.tld>.key"
		        ocsp "/etc/ssl/<domain.tld>.ocsp.pem"
		        ciphers "ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA"
		        dhe "auto"
		        ecdhe "P-384"
		}
		connection max request body 537919488
		        
		location "/.well-known/acme-challenge/*" {
		        root { "/acme" }
		        request strip 2
		}

		# First deny access to the specified files
		location "/db_structure.xml" { block }
		location "/.ht*"             { block }
		location "/README"           { block }
		location "/data*"            { block }
		location "/config*"          { block }
		        location "/build*"              { block }
		        location "/tests*"              { block }
		        location "/config*"             { block }
		        location "/lib*"                { block }
		        location "/3rdparty*"           { block }
		        location "/templates*"          { block }
		        location "/data*"               { block }
		        location "/.ht*"                { block }
		        location "/.user*"              { block }
		        location "/autotest*"           { block }
		        location "/occ*"                { block }
		        location "/issue*"              { block }
		        location "/indie*"              { block }
		        location "/db_*"                { block }
		        location "/console*"            { block }

		location "/*.php*" {
		        fastcgi socket "/run/php-fpm.sock"
		}

		location "/.well-known/host-meta" {
			block return 301 "/public.php?service=host-meta"
	   	}
		    location "/.well-known/host-meta.json" {
			block return 301 "/public.php?service=host-meta-json"
	   	}
	    	location "/.well-known/webfinger" {
			block return 301 "/public.php?service=webfinger"
	    	}
	    	location "/.well-known/carddav" {
			block return 301 "/remote.php/dav/"
	    	}
	    	location "/.well-known/caldav" {
			block return 301 "/remote.php/dav/"
	    	}
	}

	server "<domain.tld>" {
		listen on * port 80
		block return 301 "https://<domain.tld>$REQUEST_URI"
	}
	```

PHP and PostgreSQL
---

1. Since this is the first time we’re using PostgreSQL on this machine, we’ll need to initialize the database.

	```console
	$ doas su - _postgresql
	$ mkdir /var/postgresql/data
	$ initdb -D /var/postgresql/data -U postgres -A md5 -W
	```

2. Switch back to our regular user by typing `exit` and confirming with `whoami`. Now, let’s start the database by issuing a single command.

	```console
	$ doas rcctl enable postgresql && doas rcctl start postgresql
	```

PHP Configuration
---

1. Now we have to change the PHP configuration with some higher limits as the default only allows uploading of files that are two MB at max. Open `/etc/php-7.3.ini` and edit these values:

	```
	memory_limit = -1
	max_input_time = 180
	upload_max_filesize = 512M
	post_max_size = 32M
	opcache.enable=1
	opcache.enable_cli=1
	opcache.memory_consumption=128
	opcache.interned_strings_buffer=8
	opcache.max_accelerated_files=10000
	opcache.revalidate_freq=1
	opcache.save_comments = 1
	```

Installing Nextcloud
---

1. We’ve come quite a long way. Fortunately, we’re almost there! Grab and verify the most current version of Nextcloud and extract it to `/var/www/nextcloud`:

	```console
	$ ftp https://download.nextcloud.com/server/releases/nextcloud-18.0.4.zip
	$ ftp https://download.nextcloud.com/server/releases/nextcloud-18.0.4.zip.asc
	$ doas pkg_add gnupg unzip
	$ gpg2 --fetch-keys https://nextcloud.com/nextcloud.asc
	$ gpg2 --verify nextcloud-18.0.4.zip.asc
	gpg: assuming signed data in 'nextcloud-18.0.4.zip'
	gpg: Signature made Wed Apr 22 19:35:41 2020 UTC
	gpg:                using RSA key D75899B9A724937A
	gpg: Good signature from "Nextcloud Security <security@nextcloud.com>" [unknown]
	gpg: WARNING: This key is not certified with a trusted signature!
	gpg:          There is no indication that the signature belongs to the owner.
	Primary key fingerprint: 2880 6A87 8AE4 23A2 8372  792E D758 99B9 A724 937A
	$ doas unzip -d /var/www nextcloud-18.0.4.zip
	$ doas chown -R www:www /var/www/nextcloud
	```

2. Before we can use Nextcloud, we need to create a database to store the data. Replace `secret-password` with a strong passphrase of your liking:

	```console
	$ doas su - _postgresql
	$ psql -d template1 -U postgres
	template1=# CREATE USER nextcloud WITH PASSWORD 'secret-password';
	template1=# CREATE DATABASE nextcloud;
	template1=# GRANT ALL PRIVILEGES ON DATABASE nextcloud to nextcloud;
	template1=# \q
	```

3. Check whether you are still running as the postgresql user with `whoami`. If so, just give it an exit to switch back to admin. Start the required services. We’ve set these services earlier to enabled, so with a restart, the services will also start on boot time:

	```console
	doas reboot
	```
4. PHP must append the chroot path `/var/www` to any command. Create and edit `/var/www/nextcloud/config/custom.config.php` to look like this:

	```
	<?php
	$CONFIG = array (
	'datadirectory' => ((php_sapi_name() == 'cli') ? '/var/www' : '') . '/nextcloud/data',
	);
	```

5. Fire up your browser and head to your subdomain. You should be greeted there with the installation wizard. 

6. Fill out the fields

	```
	Username: Choose a username
	Password: Enter a strong password
	Datadirectory: `nextcloud/data`
	Database type: PostgreSQL
	Database user: nextcloud
	Database password: 'secret-password' from before
	Database name: nextcloud
	Database host: localhost
	```

7. After the installation, edit `/var/www/nextcloud/config/config.php` and add these lines before the closing `);`.

	```
	  'memcache.local' => '\OC\Memcache\Redis',
	  'memcache.locking' => '\OC\Memcache\Redis',
	  'redis' => array(
	    'host' => '127.0.0.1',
	    'port' => 6379,
	  ),
	```

8. And last but not least, a cronjob for some regular housekeeping and indexing. By default, it runs one task with each pageload. The preferred way here is to set this via a cronjob under the `www` user (`doas -u www crontab -e`).

	```
	*/5  *  *  *  * php -f /var/www/nextcloud/cron.php
	```

9. Customize Nextcloud to your liking and definitely add 2FA for the admin account.

Fix Warnings
---

1. If you see the warning `PHP does not seem to be setup properly to query system environment variables. The test with getenv("PATH") only returns an empty response. Please check the installation documentation ↗ for PHP configuration notes and the PHP configuration of your server, especially when using php-fpm.`

	Edit `/etc/php-fpm.conf` and uncomment these lines

	```
	;env[HOSTNAME] = $HOSTNAME
	;env[PATH] = /usr/local/bin:/usr/bin:/bin
	;env[TMP] = /tmp
	;env[TMPDIR] = /tmp
	;env[TEMP] = /tmp
	```

	Restart services to apply changes 

	```console
	$ doas rcctl restart httpd redis php73_fpm postgresql
	```

2. If you see the warning `The database is missing some indexes. Due to the fact that adding indexes on big tables could take some time they were not added automatically. By running "occ db:add-missing-indices" those missing indexes could be added manually while the instance keeps running. Once the indexes are added queries to those tables are usually much faster.`

	Run this command

	```console
	$ doas -u www php /var/www/nextcloud/occ db:add-missing-indices
	```

	Restart services to apply changes

	```console
	$ doas rcctl restart httpd redis php73_fpm postgresql
	```

3. If you see the warning `This instance is missing some recommended PHP modules. For improved performance and better compatibility it is highly recommended to install them.`

	Run this command

	```console
	$ doas pkg_add pecl73-imagick 
	$ doas cp /etc/php-7.3.sample/imagick.ini /etc/php-7.3
	```

	Restart services to apply changes

	```console
	$ doas rcctl restart httpd redis php73_fpm postgresql
	```

4. If you see the warning `Some columns in the database are missing a conversion to big int. Due to the fact that changing column types on big tables could take some time they were not changed automatically. By running 'occ db:convert-filecache-bigint' those pending changes could be applied manually. This operation needs to be made while the instance is offline. For further details read the documentation page about this.

	Run this command

	```console
	$ doas -u www php /var/www/nextcloud/occ db:convert-filecache-bigint
	```

	Restart services to apply changes

	```console
	$ doas rcctl restart httpd redis php73_fpm postgresql
	```

Tests
---

1. Go to [scan.nextcloud.com](https://scan.nextcloud.com/) and enter your dynamic DNS hostname. 

2. Go to [ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/) and enter your dynamic DNS hostname.

Backups
---

1. Database

	You might have to play around with permissions on /var/www/nextcloud and your mountpoint.

	GPG encrypted backup to USB drive

	```console
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --on
	$ PGPASSWORD="<secret-password>" pg_dump nextcloud -U nextcloud | gpg2 -e -r <your_key_id> -o /<path_to_your_usbdrive>/nextcloud-sqlbkp_`date +"%Y%m%d"`.bak.gpg
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --off
	```

	Unencrypted backup to USB drive
	```console
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --on
	$ PGPASSWORD="<secret-password>" pg_dump nextcloud -U nextcloud -f /<path_to_your_usbdrive>/nextcloud-sqlbkp_`date +"%Y%m%d"`.bak
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --off
	```

2. Full server backup with duplicity

	Duplicity is an excellent backup software. It creates a full backup at first and then performs an incremental backup (i.e., only backing up any files that are new or changed since the last backup run), until the last full backup is older than one month.

	```console
	$ doas pkg_add duplicity
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --on
	$ duplicity --encrypt-key <your_key_id>  --exclude /<path_to_your_usbdrive> --exclude /tmp --exclude /sys --exclude /dev /  file:///<path_to_your_usbdrive>
	$ doas -u www /usr/local/bin/php /var/www/nextcloud/occ maintenance:mode --off
	```

Resources
---

* [drduh PC-Engines-APU-Router-Guide](https://github.com/drduh/PC-Engines-APU-Router-Guide)
* [OpenBSD Full Disk Encryption](https://www.openbsd.org/faq/faq14.html#softraidFDE)
* [h3artbl33d NextCloud on OpenBSD](https://www.h3artbl33d.nl/communication/2018/10/07/nextcloud-openbsd/)
