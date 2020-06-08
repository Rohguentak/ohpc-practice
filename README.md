# ohpc_practice(xcat+pbspro[stateless])
vm에서 ohpc이용 해서 centos7.7 provisioning 하기
------------------------------------------------------------------------------------------------------------------------------------
      -os: CentOS-7-x86_64-DVD-1908.iso(minimal 버전 사용시 raminitfs 생성 시 에러 발생

      -pbspro 버전: pbspro-19.1.3

      -구성 : master 서버 1대, 계산 노드 2대 
      
      -vm에서의 설정 : cluster내부 통신은 host-only 네트워크로 설정, 계산노드는 OS provisioning을 위해 2GB이상의 메모리로 설정
      
      -주소체계 : master(172.28.128.10), cn01(172.28.128.11, mac=08:00:27:97:41:E9), cn02(172.28.128.12 mac=08:00:27:71:F1:0B)
      
   
   ohpc를 위한 초기 
   ------------------------------------------------------------------------------------------------------------------------------------


      [root@master ~]# ip addr show
      1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
          valid_lft forever preferred_lft forever
      2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:86:8d:49 brd ff:ff:ff:ff:ff:ff
          inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic enp0s3
       valid_lft 86332sec preferred_lft 86332sec
          inet6 fe80::6f22:b9f0:93a2:3a11/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
      3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:e7:86:93 brd ff:ff:ff:ff:ff:ff
          inet 172.28.128.10/24 brd 172.28.128.255 scope global noprefixroute enp0s8
        valid_lft forever preferred_lft forever
          inet6 fe80::23d4:fcee:90ec:c64e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
      

      [root@master ~]# cat /etc/hosts
      127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
      ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
      172.28.128.10 master master.test.com

      [root@master ~]# hostname
      master
      [root@master ~]# domainname
      test.com
      [root@master ~]# systemctl status firewalld
      ● firewalld.service - firewalld - dynamic firewall daemon
         Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
         Active: inactive (dead)
           Docs: man:firewalld(1)
ohpc 및 xcat 설치
   ------------------------------------------------------------------------------------------------------------------------------------

      [root@master ~]# yum install -y http://build.openhpc.community/OpenHPC:/1.3/CentOS_7/x86_64/ohpc-release-1.3-1.el7.x86_64.rpm 
      [root@master ~]# yum install -y yum-utils
      [root@master ~]# yum install -y ohpc-base
      [root@master ~]# wget -P /etc/yum.repos.d https://xcat.org/files/xcat/repos/yum/latest/xcat-core/xcat-core.repo
      [root@master ~]# wget -P /etc/yum.repos.d https://xcat.org/files/xcat/repos/yum/xcat-dep/rh7/x86_64/xcat-dep.repo
      
      [root@master ~]# yum install -y xCAT
      [root@master ~]# . /etc/profile.d/xcat.sh
 클러스터내 시간 sync맞추기 위한 ntp server설정 & 스케줄러 설치
 -----------------------------------------
      [root@master ~]# systemctl enable ntpd.service
      [root@master ~]# echo "server 127.127.0.1" >> /etc/ntp.conf   //서버의 시계로 sync맞추기 위해 서버 local 서버로 설정
      [root@master ~]# systemctl restart ntpd
      
      [root@master ~]# yum -y install pbspro-server-ohpc //pbspro 서버 설치
      
      [root@master ~]# ifconfig enp0s8 172.28.128.10 netmask 255.255.255.0 up
      [root@master ~]# chdef -t site dhcpinterfaces="xcatmn|enp0s8" 
      1 object definitions have been created or modified.

      [root@master ~]# tabdump site
      #key,value,comments,disable
      "blademaxp","64",,
      "fsptimeout","0",,
      "installdir","/install",,
      "ipmimaxp","64",,
      "ipmiretries","3",,
      "ipmitimeout","2",,
      "consoleondemand","no",,
      "master","172.28.128.10",,
      "forwarders","192.168.21.1",,
      "nameservers","172.28.128.10",,
      "maxssh","8",,
      "ppcmaxp","64",,
      "ppcretry","3",,
      "ppctimeout","0",,
      "powerinterval","0",,
      "syspowerinterval","0",,
      "sharedtftp","1",,
      "SNsyncfiledir","/var/xcat/syncfiles",,
      "nodesyncfiledir","/var/xcat/node/syncfiles",,
      "tftpdir","/tftpboot",,
      "xcatdport","3001",,
      "xcatiport","3002",,
      "xcatconfdir","/etc/xcat",,
      "timezone","Asia/Seoul",,
      "useNmapfromMN","no",,
      "enableASMI","no",,
      "db2installloc","/mntdb2",,
      "databaseloc","/var/lib",,
      "sshbetweennodes","ALLGROUPS",,
      "dnshandler","ddns",,
      "vsftp","n",,
      "cleanupxcatpost","no",,
      "dhcplease","43200",,
      "auditnosyslog","0",,
      "auditskipcmds","ALL",,
      "dhcpinterfaces","xcatmn|enp0s8",,      //chdef로 변경한 내용 반영됨
배포할 이미지 생성
------------------
      [root@master ~]# copycds CentOS-7-x86_64-DVD-1908.iso 
      Copying media to /install/centos7.7/x86_64
      Media copy operation successful

      [root@master ~]# lsdef -t osimage
      centos7.7-x86_64-install-compute  (osimage)
      centos7.7-x86_64-netboot-compute  (osimage)
      centos7.7-x86_64-statelite-compute  (osimage)

      [root@master ~]# export CHROOT=/install/netboot/centos7.7/x86_64/compute/rootimg/
      [root@master ~]# echo $CHROOT
      /install/netboot/centos7.7/x86_64/compute/rootimg/

      [root@master ~]# genimage centos7.7-x86_64-netboot-compute       //CHROOT에 이미지 생성
      Generating image:                                                    
      Complete!
      Enter the dracut mode. Dracut version: 033. Dracut directory: dracut_033.
      Try to load drivers: tg3 bnx2 bnx2x e1000 e1000e igb mlx4_en virtio_net be2net ext3 ext4 to initrd.
      chroot /install/netboot/centos7.7/x86_64/compute/rootimg dracut  -f /tmp/initrd.7029.gz 3.10.0-1062.el7.x86_64
      No '/dev/log' or 'logger' included for syslog logging
      Turning off host-only mode: '/sys' is not mounted!
      Turning off host-only mode: '/proc' is not mounted!
      Turning off host-only mode: '/run' is not mounted!
      Turning off host-only mode: '/dev' is not mounted!
      grep: /etc/udev/rules.d/*: No such file or directory
      the initial ramdisk for stateless is generated successfully.
      Try to load drivers: tg3 bnx2 bnx2x e1000 e1000e igb mlx4_en virtio_net be2net ext3 ext4 to initrd.
      chroot /install/netboot/centos7.7/x86_64/compute/rootimg dracut  -f /tmp/initrd.7029.gz 3.10.0-1062.el7.x86_64
      No '/dev/log' or 'logger' included for syslog logging
      Turning off host-only mode: '/sys' is not mounted!
      Turning off host-only mode: '/proc' is not mounted!
      Turning off host-only mode: '/run' is not mounted!
      Turning off host-only mode: '/dev' is not mounted!
      grep: /etc/udev/rules.d/*: No such file or directory
      the initial ramdisk for statelite is generated successfully.    //해당 문구가 떠야 올바른 것(minimal이미지의 경우 여기서 error )

      [root@master ~]# yum-config-manager --installroot=$CHROOT --enable base
      Loaded plugins: fastestmirror, langpacks
      ===================================== repo: base ======================================
      [base]
      async = True
      bandwidth = 0
      base_persistdir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/lib/yum/repos/x86_64/7
      baseurl = 
      cache = 0
      cachedir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/cache/yum/x86_64/7/base
      check_config_file_age = True
      compare_providers_priority = 80
      cost = 1000
      deltarpm_metadata_percentage = 100
      deltarpm_percentage = 
      enabled = 1
      enablegroups = True
      exclude = 
      failovermethod = priority
      ftp_disable_epsv = False
      gpgcadir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/lib/yum/repos/x86_64/7/base/gpgcadir
      gpgcakey = 
      gpgcheck = True
      gpgdir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/lib/yum/repos/x86_64/7/base/gpgdir
      gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
      hdrdir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/cache/yum/x86_64/7/base/headers
      http_caching = all
      includepkgs = 
      ip_resolve = 
      keepalive = True
      keepcache = False
      mddownloadpolicy = sqlite
      mdpolicy = group:small
      mediaid = 
      metadata_expire = 21600
      metadata_expire_filter = read-only:present
      metalink = 
      minrate = 0
      mirrorlist = http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=stock
      mirrorlist_expire = 86400
      name = CentOS-7 - Base
      old_base_cache_dir = var/cache/yum/x86_64/7
      password = 
      persistdir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/lib/yum/repos/x86_64/7/base
      pkgdir = /install/netboot/centos7.7/x86_64/compute/rootimg/var/cache/yum/x86_64/7/base/packages
      proxy = False
      proxy_dict = 
      proxy_password = 
      proxy_username = 
      repo_gpgcheck = False
      retries = 10
      skip_if_unavailable = False
      ssl_check_cert_permissions = True
      sslcacert = 
      sslclientcert = 
      sslclientkey = 
      sslverify = True
      throttle = 0
      timeout = 30.0
      ui_id = base/7/x86_64
      ui_repoid_vars = releasever,
         basearch
      username = 

      [root@master ~]# cp /etc/yum.repos.d/OpenHPC.repo $CHROOT/etc/yum.repos.d
      [root@master ~]# cp /etc/yum.repos.d/OpenHPC.repo $CHROOT/etc/yum.repos.d
      
      [root@master ~]# yum -y --installroot=$CHROOT install ohpc-base-compute
      [root@master ~]# yum -y --installroot=$CHROOT install pbspro-execution-ohpc     //provisioning 할 os에 필요한 패키지 설치
      [root@master ~]# yum -y --installroot=$CHROOT install ntp
      [root@master ~]# yum -y --installroot=$CHROOT install kernel
      [root@master ~]# yum -y --installroot=$CHROOT install lmod-ohpc

      [root@master ~]# echo "172.28.128.10:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $CHROOT/etc/fstab
      [root@master ~]# echo "172.28.128.10:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $CHRROOT/etc/fstab //계산노드에서 마운트할 디렉토리 설정
      [root@master ~]# cat $CHROOT/etc/fstab
      proc            /proc    proc   rw 0 0
      sysfs           /sys     sysfs  rw 0 0
      devpts          /dev/pts devpts rw,gid=5,mode=620 0 0
      172.28.128.10:/home /home nfs nfsvers=3,nodev,nosuid 0 0
      172.28.128.10:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0
      
      [root@master ~]# echo "/home *(rw,no_subtree_check,fsid=10,no_root_squash)" >> /etc/exports
      [root@master ~]# echo "/opt/ohpc/pub *(ro,no_subtree_check,fsid=11)" >> /etc/exports        //nfs를 위한 설정
      [root@master ~]# exportfs -a
      [root@master ~]# exporfs
      /tftpboot     	<world>
      /install      	<world>
      /home         	<world>
      /opt/ohpc/pub 	<world>

      [root@master ~]# systemctl restart nfs-server
      [root@master ~]# systemctl enable nfs-server
      
      [root@master ~]# chroot $CHROOT systemctl enable ntpd       //부팅시 ntp실행되도록 설정
      Created symlink /etc/systemd/system/multi-user.target.wants/ntpd.service, pointing to /usr/lib/systemd/system/ntpd.service.

      [root@master ~]# echo "server 172.28.128.10" >> $CHROOT/etc/ntp.conf //ntp로 시간 맞출 서버 설정
      [root@master ~]# vi $CHROOT/etc/pbs.conf    //PBS_SERVER의 값을 pbs server 역할하는 서버의 hostname으로 설정

      PBS_SERVER=master
      PBS_START_SERVER=0
      PBS_START_SCHED=0
      PBS_START_COMM=0
      PBS_START_MOM=1
      PBS_HOME=/var/spool/pbs
      PBS_CORE_LIMIT=unlimited
      PBS_SCP=/bin/scp

      [root@master ~]# vi /var/spool/pbs/mom_priv/config     //pbs server 역할하는 hostname으로 수정

      [root@master ~]# cat $CHROOT/var/spool/pbs/mom_priv/config 
      $clienthost master
서버와 계산노드가 공유할 파일을 설정 
-------------------------------------
      [root@master ~]# mkdir -p /install/custom/netboot       
      [root@master ~]# chdef -t osimage -o centos7.7-x86_64-netboot-compute synclists="/install/custom/netboot/compute.synclist
      1 object definitions have been created or modified.
      [root@master ~]# vi /install/custom/netboot/compute.synclist

      [root@master ~]# cat /install/custom/netboot/compute.synclist 
      /etc/passwd -> /etc/passwd                //계정정보를 공유
      /etc/group -> /etc/group
      /etc/shadow -> /etc/shadow
배포할 이미지 압축 및 노드 object생성
--------------------------------------
      [root@master ~]# packimage centos7.7-x86_64-netboot-compute   //설정한 이미지 압축
      Packing contents of /install/netboot/centos7.7/x86_64/compute/rootimg
      archive method:cpio
      compress method:gzip

      [root@master ~]# mkdef -t node cn01 groups=compute,all ip=172.28.128.11 mac=08:00:27:97:41:E9 netboot=xnba arch=x86_64 mgt=kvm\
      1 object definitions have been created or modified.
      [root@master ~]# mkdef -t node cn02 groups=compute,all ip=172.28.128.12 mac=08:00:27:71:F1:0B netboot=xnba arch=x86_64 mgt=kvm
      1 object definitions have been created or modified.
                                                                            //cn01과 cn02 추가
      [root@master ~]# chdef -t site domain=test.com
      1 object definitions have been created or modified.

      [root@master ~]# makehosts      //host파일에 노드들 자동 추가
      [root@master ~]# cat /etc/hosts
      127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
      ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
      172.28.128.10 master master.test.com
      172.28.128.11 cn01 cn01.test.com 
      172.28.128.12 cn02 cn02.test.com 
      [root@master ~]# makenetworks
      Warning: [master]: The network entry '10_0_2_0-255_255_255_0' already exists in xCAT networks table. Cannot create a definition for '10_0_2_0-255_255_255_0'
      Warning: [master]: The network entry '172_28_128_0-255_255_255_0' already exists in xCAT networks table. Cannot create a definition for '172_28_128_0-255_255_255_0'
      Warning: [master]: The network entry '192_168_122_0-255_255_255_0' already exists in xCAT networks table. Cannot create a definition for '192_168_122_0-255_255_255_0'

      [root@master ~]# makedhcp -n
      Renamed existing dhcp configuration file to  /etc/dhcp/dhcpd.conf.xcatbak

      The dhcp server must be restarted for OMAPI function to work
      Warning: [master]: No dynamic range specified for 10.0.2.0. If hardware discovery is being used, a dynamic range is required.
      Warning: [master]: No dynamic range specified for 172.28.128.0. If hardware discovery is being used, a dynamic range is required.
      Warning: [master]: No dynamic range specified for 192.168.122.0. If hardware discovery is being used, a dynamic range is required.

      [root@master ~]# makedns -n
      Warning: SELINUX is not disabled. The makedns command will not be able to generate a complete DNS setup. Disable SELINUX and run the command again.
      Handling localhost in /etc/hosts.
      Handling cn01 in /etc/hosts.
      Handling localhost in /etc/hosts.
      Handling cn02 in /etc/hosts.
      Handling master in /etc/hosts.
      Getting reverse zones, this may take several minutes for a large cluster.
      Completed getting reverse zones.
      Updating zones.
      Completed updating zones.
      Restarting named
      Restarting named complete
      Updating DNS records, this may take several minutes for a large cluster.
      Completed updating DNS records.
      DNS setup is completed

      [root@master ~]# nodeset compute osimage=centos7.7-x86_64-netboot-compute
      cn01: netboot centos7.7-x86_64-compute
      cn02: netboot centos7.7-x86_64-compute
계산노드에 OS provisioning
--------------------------
      vm에서 생성한 노드 설정 : 해당 가상머신[설정] -> [시스템] 에서 부팅순서 네트워크 체크 후 나머지 체크 해제, 메모리 2GB이상,네트워크에 host-only 네트워크 추가
      
pbspro 확인
-----------
      기존의 pbspro 테스트 방식과 동일(예시: https://github.com/Rohguentak/pbs_pro_exercise)
