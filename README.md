# ****Работа с LVM**** #

### Описание домашннего задания ###

На имеющемся образе centos/7 - v. 1804.2
   1. Уменьшить том под / до 8G.
   2. Выделить том под /home.
   3. Выделить том под /var - сделать в mirror.
   4. /home - сделать том для снапшотов.
   5. Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
   6. Работа со снапшотами:
      - сгенерить файлы в /home/;
      - снять снапшот;
      - удалить часть файлов;
      - восстановится со снапшота.

### **Выполнение** ###

Задание выполняется на рабочей станции с ОС Ubuntu 22.04.4 LTS с заранее установленными Vagrant 2.4.1 и VirtualBox 7.0. Перед выполнением предварительно подготовлен репозиторий <https://github.com/ConstantaNF/LVM.git>

### **Подготовка окружения** ###

Для развёртывания управляемой ВМ посредством Vagrant использую Vagrantfile из методички <https://github.com/Nickmob/vagrant_lvm1>. Данный Vagrantfile кладу в заранее подготовленный каталог ```/home/adminkonstantin/LVM```. Для корректной работы сетевого подключения будующей ВМ меняю ip адресс в Vagrantfile на 192.168.56.101. Исходя из методички, потребуется пакет xfsdump, а для личного удобства может понадобиться текстовый редактор Nano. Так как ни того, ни другого в применяемом дистрибутиве не установлено, добавляю эти пакеты в блок ```box.vm.provision``` Vagrantfile-а.

Теперь Vagrantfile выглядит так:

```
# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :lvm => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.56.101',
    :disks => {
        :sata1 => {
            :dfile => home + '/VirtualBox VMs/sata1.vdi',
            :size => 10240,
            :port => 1
        },
        :sata2 => {
            :dfile => home + '/VirtualBox VMs/sata2.vdi',
            :size => 2048, # Megabytes
            :port => 2
        },
        :sata3 => {
            :dfile => home + '/VirtualBox VMs/sata3.vdi',
            :size => 1024, # Megabytes
            :port => 3
        },
        :sata4 => {
            :dfile => home + '/VirtualBox VMs/sata4.vdi',
            :size => 1024,
            :port => 4
        }
    }
  },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "1804.02"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "256"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
            yum install -y mdadm smartmontools hdparm gdisk nano xfsdump
          SHELL
  
        end
    end
  end
```
  
Стартую ВМ:

```
adminkonstantin@2OSUbuntu:~/LVM$ vagrant up
```

Результат:

```
Bringing machine 'lvm' up with 'virtualbox' provider...
==> lvm: Importing base box 'centos/7'...
==> lvm: Matching MAC address for NAT networking...
==> lvm: Checking if box 'centos/7' version '1804.02' is up to date...
==> lvm: Setting the name of the VM: LVM_lvm_1711616955481_12990
==> lvm: Clearing any previously set network interfaces...
==> lvm: Preparing network interfaces based on configuration...
    lvm: Adapter 1: nat
    lvm: Adapter 2: hostonly
==> lvm: Forwarding ports...
    lvm: 22 (guest) => 2222 (host) (adapter 1)
==> lvm: Running 'pre-boot' VM customizations...
==> lvm: Booting VM...
==> lvm: Waiting for machine to boot. This may take a few minutes...
    lvm: SSH address: 127.0.0.1:2222
    lvm: SSH username: vagrant
    lvm: SSH auth method: private key
    lvm: 
    lvm: Vagrant insecure key detected. Vagrant will automatically replace
    lvm: this with a newly generated keypair for better security.
    lvm: 
    lvm: Inserting generated public key within guest...
    lvm: Removing insecure key from the guest if it's present...
    lvm: Key inserted! Disconnecting and reconnecting using new SSH key...
==> lvm: Machine booted and ready!
==> lvm: Checking for guest additions in VM...
    lvm: No guest additions were detected on the base box for this VM! Guest
    lvm: additions are required for forwarded ports, shared folders, host only
    lvm: networking, and more. If SSH fails on this machine, please install
    lvm: the guest additions and repackage the box to continue.
    lvm: 
    lvm: This is not an error message; everything may continue to work properly,
    lvm: in which case you may ignore this message.
==> lvm: Setting hostname...
==> lvm: Configuring and enabling network interfaces...
==> lvm: Rsyncing folder: /home/adminkonstantin/LVM/ => /vagrant
==> lvm: Running provisioner: shell...
    lvm: Running: inline script
    lvm: Loaded plugins: fastestmirror
    lvm: Determining fastest mirrors
    lvm:  * base: mirror.awanti.com
    lvm:  * extras: mirror.yandex.ru
    lvm:  * updates: mirror.hyperdedic.ru
    lvm: Resolving Dependencies
    lvm: --> Running transaction check
    lvm: ---> Package gdisk.x86_64 0:0.8.10-3.el7 will be installed
    lvm: ---> Package hdparm.x86_64 0:9.43-5.el7 will be installed
    lvm: ---> Package mdadm.x86_64 0:4.1-9.el7_9 will be installed
    lvm: --> Processing Dependency: libreport-filesystem for package: mdadm-4.1-9.el7_9.x86_64
    lvm: ---> Package nano.x86_64 0:2.3.1-10.el7 will be installed
    lvm: ---> Package smartmontools.x86_64 1:7.0-2.el7 will be installed
    lvm: --> Processing Dependency: mailx for package: 1:smartmontools-7.0-2.el7.x86_64
    lvm: ---> Package xfsdump.x86_64 0:3.1.7-4.el7_9 will be installed
    lvm: --> Processing Dependency: attr >= 2.0.0 for package: xfsdump-3.1.7-4.el7_9.x86_64
    lvm: --> Running transaction check
    lvm: ---> Package attr.x86_64 0:2.4.46-13.el7 will be installed
    lvm: ---> Package libreport-filesystem.x86_64 0:2.1.11-53.el7.centos will be installed
    lvm: ---> Package mailx.x86_64 0:12.5-19.el7 will be installed
    lvm: --> Finished Dependency Resolution
    lvm: 
    lvm: Dependencies Resolved
    lvm: 
    lvm: ================================================================================
    lvm:  Package                  Arch       Version                  Repository   Size
    lvm: ================================================================================
    lvm: Installing:
    lvm:  gdisk                    x86_64     0.8.10-3.el7             base        190 k
    lvm:  hdparm                   x86_64     9.43-5.el7               base         83 k
    lvm:  mdadm                    x86_64     4.1-9.el7_9              updates     439 k
    lvm:  nano                     x86_64     2.3.1-10.el7             base        440 k
    lvm:  smartmontools            x86_64     1:7.0-2.el7              base        546 k
    lvm:  xfsdump                  x86_64     3.1.7-4.el7_9            updates     309 k
    lvm: Installing for dependencies:
    lvm:  attr                     x86_64     2.4.46-13.el7            base         66 k
    lvm:  libreport-filesystem     x86_64     2.1.11-53.el7.centos     base         41 k
    lvm:  mailx                    x86_64     12.5-19.el7              base        245 k
    lvm: 
    lvm: Transaction Summary
    lvm: ================================================================================
    lvm: Install  6 Packages (+3 Dependent packages)
    lvm: 
    lvm: Total download size: 2.3 M
    lvm: Installed size: 7.0 M
    lvm: Downloading packages:
    lvm: Public key for libreport-filesystem-2.1.11-53.el7.centos.x86_64.rpm is not installed
    lvm: warning: /var/cache/yum/x86_64/7/base/packages/libreport-filesystem-2.1.11-53.el7.centos.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
    lvm: Public key for mdadm-4.1-9.el7_9.x86_64.rpm is not installed
    lvm: --------------------------------------------------------------------------------
    lvm: Total                                              1.4 MB/s | 2.3 MB  00:01
    lvm: Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
    lvm: Importing GPG key 0xF4A80EB5:
    lvm:  Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
    lvm:  Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
    lvm:  Package    : centos-release-7-5.1804.el7.centos.x86_64 (@anaconda)
    lvm:  From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
    lvm: Running transaction check
    lvm: Running transaction test
    lvm: Transaction test succeeded
    lvm: Running transaction
    lvm:   Installing : mailx-12.5-19.el7.x86_64                                     1/9
    lvm:   Installing : attr-2.4.46-13.el7.x86_64                                    2/9
    lvm:   Installing : libreport-filesystem-2.1.11-53.el7.centos.x86_64             3/9
    lvm:   Installing : mdadm-4.1-9.el7_9.x86_64                                     4/9
    lvm:   Installing : xfsdump-3.1.7-4.el7_9.x86_64                                 5/9
    lvm:   Installing : 1:smartmontools-7.0-2.el7.x86_64                             6/9
    lvm:   Installing : hdparm-9.43-5.el7.x86_64                                     7/9
    lvm:   Installing : gdisk-0.8.10-3.el7.x86_64                                    8/9
    lvm:   Installing : nano-2.3.1-10.el7.x86_64                                     9/9
    lvm:   Verifying  : mdadm-4.1-9.el7_9.x86_64                                     1/9
    lvm:   Verifying  : 1:smartmontools-7.0-2.el7.x86_64                             2/9
    lvm:   Verifying  : nano-2.3.1-10.el7.x86_64                                     3/9
    lvm:   Verifying  : libreport-filesystem-2.1.11-53.el7.centos.x86_64             4/9
    lvm:   Verifying  : gdisk-0.8.10-3.el7.x86_64                                    5/9
    lvm:   Verifying  : attr-2.4.46-13.el7.x86_64                                    6/9
    lvm:   Verifying  : mailx-12.5-19.el7.x86_64                                     7/9
    lvm:   Verifying  : hdparm-9.43-5.el7.x86_64                                     8/9
    lvm:   Verifying  : xfsdump-3.1.7-4.el7_9.x86_64                                 9/9
    lvm: 
    lvm: Installed:
    lvm:   gdisk.x86_64 0:0.8.10-3.el7             hdparm.x86_64 0:9.43-5.el7
    lvm:   mdadm.x86_64 0:4.1-9.el7_9              nano.x86_64 0:2.3.1-10.el7
    lvm:   smartmontools.x86_64 1:7.0-2.el7        xfsdump.x86_64 0:3.1.7-4.el7_9
    lvm: 
    lvm: Dependency Installed:
    lvm:   attr.x86_64 0:2.4.46-13.el7
    lvm:   libreport-filesystem.x86_64 0:2.1.11-53.el7.centos
    lvm:   mailx.x86_64 0:12.5-19.el7
    lvm: 
    lvm: Complete!
```

Подключаюсь к созданной ВМ по SSH:

```
adminkonstantin@2OSUbuntu:~/LVM$ vagrant ssh
```

Успешное подключение:

```
[vagrant@lvm ~]$
```

Подключаюсь в УЗ root:

```
[vagrant@lvm ~]$ sudo -i
```

Результат:

```
[root@lvm ~]#
```

### **Уменьшить том под / до 8G** ###

Подготовим временный том для / раздела:

```
[root@lvm ~]# pvcreate /dev/sdb
   Physical volume "/dev/sdb" successfully created.
```

```
[root@lvm ~]# vgcreate vg_root /dev/sdb
   Volume group "vg_root" successfully created
```

```
[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
   Logical volume "lv_root" created.
```

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

```
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
```

Копируем все данные с раздела / в /mnt:

```
[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Thu Mar 28 09:35:21 2024
xfsdump: session id: 025790b3-6c4c-46f2-b9b4-d4658d5c1fa2
xfsdump: session label: ""
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 899329024 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Thu Mar 28 09:35:21 2024
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 025790b3-6c4c-46f2-b9b4-d4658d5c1fa2
xfsrestore: media id: 8b25148d-4344-4619-b460-6ff31cfed079
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2736 directories and 23748 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 876343760 bytes
xfsdump: dump size (non-dir files) : 863101224 bytes
xfsdump: dump complete: 34 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 35 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

Cконфигурируем grub для того, чтобы при старте перейти в новый /. Сымитируем текущий root, сделаем в него chroot и обновим grub:

```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
>  do mount --bind $i /mnt/$i; done
```

```
[root@lvm ~]# chroot /mnt/
```

```
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

Обновим образ initrd:

```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \
> do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> > s/.img//g"` --force; done
sed: -e expression #1, char 18: unknown command: `>'
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

Для того, чтобы при загрузке был смонтирован нужный root нужно в файле
`/boot/grub2/grub.cfg` заменить `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=vg_root/lv_root`
Делаю это в текстовом редакторе Nano.

Проверим изменения:

```
[root@lvm boot]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

Перезагружаем ВМ.

Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем старый LV размером в 40G и создаём новый на 8G:

```
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```

```
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```

  Создадим файловую систему на новом LV и смонтируем его, чтобы перенести туда данные:

  ```
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```
root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt
```

Копируем все данные с раздела / в /mnt:

```
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | \
>  xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Thu Mar 28 10:36:55 2024
xfsdump: session id: 30eb943b-882a-4f4e-b7aa-ccaee1c2aed9
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 897942080 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_root-lv_root
xfsrestore: session time: Thu Mar 28 10:36:55 2024
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: d43731d1-0863-4b41-9c59-28b03a4be00b
xfsrestore: session id: 30eb943b-882a-4f4e-b7aa-ccaee1c2aed9
xfsrestore: media id: f2eebf92-6992-4082-beb7-7cc8751aee7b
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2740 directories and 23753 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 874985400 bytes
xfsdump: dump size (non-dir files) : 861739184 bytes
xfsdump: dump complete: 23 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 23 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

Так же как в первый раз cконфигурируем grub, за исключением правки `/etc/grub2/grub.cfg`:

```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
>  do mount --bind $i /mnt/$i; done
```

```
[root@lvm ~]# chroot /mnt/
```

```
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \
>  do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> > s/.img//g"` --force; done
sed: -e expression #1, char 18: unknown command: `>'
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

Для выполнения следующего задания пока не перезагружаемся и не выходим из под chroot.

### **Выделить том под /var в зеркало** ###

На свободных дисках создаю зеркало:

```
[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

```
[root@lvm boot]#  vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
```

```
[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```

Создаю на нем ФС, монтирую и перемещаю туда /var:

```
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

```
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt
```

```
[root@lvm boot]# cp -aR /var/* /mnt/
```

Cохраняю содержимое старого var:

```
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```

Монтирую новый var в каталог /var:

```
[root@lvm boot]# umount /mnt
```

```
[root@lvm boot]# mount /dev/vg_var/lv_var /var
```

Правлю fstab для автоматического монтирования /var:

```
[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` \
> /var ext4 defaults 0 0" >> /etc/fstab
```

Перезагружаю ВМ.

Удаляю временную VG:

```
[root@lvm ~]#  lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
```

```
[root@lvm ~]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
```
  
```
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```

  ### **Выделить том под /home** ###

  Создаю логический том в VG /dev/VolGroup00/:

```
[root@lvm ~]#  lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.
```

  Создаю на новом логическом томе файловую систему и монтирую:

```
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```
[root@lvm ~]#  mount /dev/VolGroup00/LogVol_Home /mnt/
```

Перемещаю содержимое /home/ в /mnt/:

```
[root@lvm ~]# cp -aR /home/* /mnt/
```

Удаляю старый каталог /home/:

```
[root@lvm ~]# rm -rf /home/*
```

Монтирую новый каталог /home/:

```
[root@lvm ~]# umount /mnt
```

```
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/
```

Правлю fstab для автоматического монтирования /home:

```
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` \
> /home xfs defaults 0 0" >> /etc/fstab
```

### **Работа со снапшотами** ###

Генерирую файлы в /home/:

```
[root@lvm ~]# touch /home/file{1..20}
```

```
[root@lvm ~]# ls -l /home
total 0
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file1
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file10
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file11
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file12
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file13
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file14
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file15
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file16
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file17
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file18
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file19
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file2
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file20
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file3
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file4
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file5
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file6
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file7
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file8
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```

Снимаю снапшот с логического тома:

```
[root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
```

Симулирую частичную потерю файлов в каталоге:

```
[root@lvm ~]# rm -f /home/file{11..20}
```

```
[root@lvm ~]# ls -l /home
total 0
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file1
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file10
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file2
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file3
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file4
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file5
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file6
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file7
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file8
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```

Восстанавливаю состояние каталога из снапшота:

```
[root@lvm ~]# umount /home
```

```
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
```

```
[root@lvm ~]# mount /home
```

Проверяю результат восстановления:

```
[root@lvm ~]# ls -l /home
total 0
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file1
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file10
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file11
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file12
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file13
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file14
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file15
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file16
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file17
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file18
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file19
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file2
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file20
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file3
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file4
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file5
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file6
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file7
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file8
-rw-r--r--. 1 root    root     0 Mar 28 14:50 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```
