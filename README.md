**Tags:** ironic standalone baremetal
## Установка узлов в режиме Legasy Boot
### Утановка и настройка Dnsmasq
Тестирование происходило на версии dnsmasq-2.76

**CentOS**
```
yum install -y dnsmasq
```
**Ubuntu**
```
apt-get install dnsmasq
```
**Конфиг dnsmasq**
```
interface=bond0
bind-dynamic
enable-tftp
tftp-root=/tftpboot

# Disable listening for DNS
port=0

log-dhcp
dhcp-range=10.1.1.100,10.1.1.105,12h

# Disable default router(s) and DNS over provisioning network
dhcp-option=3
dhcp-option=6

# IPv4 Configuration:
dhcp-match=ipxe,175
# Client is already running iPXE; move to next stage of chainloading
dhcp-boot=tag:ipxe,http://10.1.1.25:8080/boot.ipxe

# Note: Need to test EFI booting
dhcp-match=set:efi,option:client-arch,7
dhcp-match=set:efi,option:client-arch,9
dhcp-match=set:efi,option:client-arch,11
# Client is PXE booting over EFI without iPXE ROM; send EFI version of iPXE chainloader
dhcp-boot=tag:efi,tag:!ipxe,ipxe.efi

# Client is running PXE over BIOS; send BIOS version of iPXE chainloader
dhcp-boot=/undionly.kpxe,10.1.1.25
```
### Подготавлаиваем загрузку по iPXE
Создаем все нужные директории и сокпируем фалы согластно иструкции
https://docs.openstack.org/ironic/latest/install/configure-pxe.html#ipxe-setup

### Скачиваем IPA vmzlinuz и initram
На момент тестов работаь с много томными имаджами может только ipa-centos8
**IPA:** https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/

# Файл boot.ipxe
```shell
#!ipxe

# NOTE(lucasagomes): Loop over all network devices and boot from
# the first one capable of booting. For more information see:
# https://bugs.launchpad.net/ironic/+bug/1504482
set netid:int32 -1
:loop
inc netid || chain pxelinux.cfg/${mac:hexhyp} || goto old_rom
isset ${net${netid}/mac} || goto loop_done
echo Attempting to boot from MAC ${net${netid}/mac:hexhyp}
chain pxelinux.cfg/${net${netid}/mac:hexhyp} || goto loop

:loop_done
echo PXE boot failed! No configuration found for any of the present NICs.
echo Press any key to reboot...
prompt --timeout 180
reboot

:old_rom
echo PXE boot failed! No configuration found for NIC ${mac:hexhyp}.
echo Please update your iPXE ROM and retry.
echo Press any key to reboot...
prompt --timeout 180

```
### Примерное содержимое директории /tftpboot/
```
ls -1 /tftpboot/
boot.ipxe
ipa-centos8-master.initramfs
ipa-centos8-master.kernel
ipxe.efi
ipxe.pxe
map-file
master_images
undionly.kpxe
```
### Устанавливаем Ironic
**CentOS**

https://docs.openstack.org/ironic/latest/install/install-rdo.html

```
rpm -qa | grep ironic
openstack-ironic-conductor-13.0.2-1.el7.noarch
python2-ironic-python-agent-5.0.1-1.el7.noarch
openstack-ironic-python-agent-5.0.1-1.el7.noarch
python2-ironic-lib-2.21.0-1.el7.noarch
openstack-ironic-common-13.0.2-1.el7.noarch
openstack-ironic-api-13.0.2-1.el7.noarch
openstack-ironic-staging-drivers-0.12.0-1.el7.noarch
openstack-ironic-python-agent-builder-1.0.0-1.el7.noarch
python2-ironicclient-3.1.1-1.el7.noarch
python2-ironic-inspector-client-3.7.0-1.el7.noarch
diskimage-builder-2.27.2-1.el7.noarch
```

### Ironic.conf
Привожу diff
```
--- ironic.conf.default 2020-02-27 09:30:34.333185363 -0500
+++ ironic.conf 2020-03-02 04:23:01.882999831 -0500
@@ -11,6 +11,7 @@                                            
 # noauth - no authentication      
 # keystone - use the Identity service for authentication
 #auth_strategy = keystone                           
+auth_strategy = noauth                                
               
 # Return server tracebacks in the API response for any error
 # responses. WARNING: this is insecure and should not be used
@@ -35,6 +36,8 @@                                        
 # enumerating the "ironic.hardware.types" entrypoint. (list
 # value)                   
 #enabled_hardware_types = ipmi    
+enabled_hardware_types = ipmi                    
+                            
                                  
 # Specify the list of bios interfaces to load during service                                                                                                  
 # initialization. Missing bios interfaces, or bios interfaces
@@ -51,6 +54,7 @@                                        
 # enabled bios interfaces on every ironic-conductor service.
 # (list value)                   
 #enabled_bios_interfaces = no-bios                                   
+enabled_bios_interfaces = no-bios,redfish,ilo
                               
 # Default bios interface to be used for nodes that do not have
 # bios_interface field set. A complete list of bios interfaces
@@ -72,13 +76,14 @@                                      
 # every enabled hardware type will have the same set of
 # enabled boot interfaces on every ironic-conductor service.
 # (list value)                                                
-#enabled_boot_interfaces = pxe
+enabled_boot_interfaces = ipxe,pxe
                                                               
 # Default boot interface to be used for nodes that do not have
 # boot_interface field set. A complete list of boot interfaces
 # present on your system may be found by enumerating the   
 # "ironic.hardware.interfaces.boot" entrypoint. (string value)
 #default_boot_interface = <None>                            
+#default_boot_interface = ipxe                             
                                                         
 # Specify the list of console interfaces to load during
 # service initialization. Missing console interfaces, or    
@@ -165,6 +170,7 @@                                      
 # enabled management interfaces on every ironic-conductor
 # service. (list value)                                       
 #enabled_management_interfaces = ipmitool
+enabled_management_interfaces = ipmitool             
                                                   
 # Default management interface to be used for nodes that do
 # not have management_interface field set. A complete list of
@@ -188,6 +194,7 @@
 # hardware type will have the same set of enabled network
 # interfaces on every ironic-conductor service. (list value)
 #enabled_network_interfaces = flat,noop
+enabled_network_interfaces=noop
  
 # Default network interface to be used for nodes that do not
 # have network_interface field set. A complete list of network
@@ -211,6 +218,7 @@
 # type will have the same set of enabled power interfaces on
 # every ironic-conductor service. (list value)
 #enabled_power_interfaces = ipmitool
+enabled_power_interfaces = ibmc,ipmitool
  
 # Default power interface to be used for nodes that do not
 # have power_interface field set. A complete list of power
@@ -302,6 +310,7 @@
 # type will have the same set of enabled vendor interfaces on
 # every ironic-conductor service. (list value)
 #enabled_vendor_interfaces = ipmitool,no-vendor
+enabled_vendor_interfaces = ibmc,ipmitool,no-vendor

 # Default vendor interface to be used for nodes that do not
 # have vendor_interface field set. A complete list of vendor
@@ -483,6 +492,7 @@
 # oslo - use oslo.messaging transport
 # json-rpc - use JSON RPC transport
 #rpc_transport = oslo
+rpc_transport = json-rpc
  
 # Path to the rootwrap configuration file to use for running
 # commands as root. (string value)
@@ -504,6 +514,7 @@
 # instead of the default INFO level. (boolean value)
 # Note: This option can be changed without restarting.
 #debug = false
+debug = false
  
 # The name of a logging configuration file. This file is
 # appended to any existing logging configuration files. For
@@ -787,7 +798,7 @@
 # always - always collect the logs
 # on_failure - only collect logs if there is a failure
 # never - never collect logs
-#deploy_logs_collect = on_failure
+deploy_logs_collect = always
  
 # The name of the storage backend where the logs will be
 # stored. (string value)
@@ -823,6 +834,7 @@
 # http - IPA ramdisk retrieves instance image from HTTP
 # service served at conductor nodes.
 #image_download_source = swift
+image_download_source = http
  
 # Timeout (in seconds) for IPA commands. (integer value)
 #command_timeout = 60
@@ -1002,6 +1014,8 @@
 # be determined, else a default worker count of 1 is returned.
 # (integer value)
 #api_workers = <None>
+api_workers = 5
 
 # Enable the integrated stand-alone API to service requests
 # via HTTPS instead of HTTP. If there is a front-end service
@@ -1244,6 +1258,7 @@
 # instead if required to use a specific ironic api address,
 # for example in noauth mode.
 #api_url = <None>
+#api_url = http://10.1.1.25:6385/
  
 # Maximum time (in seconds) since the last check-in of a
 # conductor. A conductor is considered inactive when this time
@@ -1375,6 +1390,7 @@
 # are trusted (eg, because there is only one tenant), this
 # option could be safely disabled. (boolean value)
 #automated_clean = true
+automated_clean = false
  
 # Whether to allow nodes to enter or undergo deploy or
 # cleaning when in maintenance mode. If this option is set to
@@ -1569,6 +1585,7 @@
 # Deprecated group/name - [DATABASE]/sql_connection
 # Deprecated group/name - [sql]/connection
 #connection = <None>
+connection=mysql+pymysql://ironic:IRONIC_DBPASSWORD@127.0.0.1/ironic?charset=utf8
  
 # The SQLAlchemy connection string to use to connect to the
 # slave database. (string value)
@@ -1675,9 +1692,10 @@
 # ironic-conductor node's HTTP server URL. Example:
 # http://192.1.2.3:8080 (string value)
 #http_url = <None>
+http_url = http://10.1.1.25:8080
  
 # ironic-conductor node's HTTP root path. (string value)
-#http_root = /httpboot
+http_root = /tftpboot
 # Whether to support the use of ATA Secure Erase during the
 # cleaning process. Defaults to True. (boolean value)
@@ -1791,6 +1809,7 @@
 # DHCP provider to use. "neutron" uses Neutron, and "none"
 # uses a no-op provider. (string value)
 #dhcp_provider = neutron
+dhcp_provider = none
    
 [disk_partitioner]
@@ -2603,6 +2622,7 @@
 # Minimum value: 0
 # Maximum value: 65535
 #port = 8089
+#port = 9999
  
 # Whether to use TLS for JSON RPC (boolean value)
 #use_ssl = false
@@ -4046,7 +4066,20 @@
 #filter_error_trace = false

[pxe]
+pxe_append_params = systemd.journald.forward_to_console=yes ipa-debug=1 nofb nomodeset vga=normal
+tftp_server = 10.220.36.25
+tftp_root = /tftpboot
+tftp_master_path = /tftpboot/master_images
+pxe_bootfile_name = undionly.kpxe
+pxe_config_template = $pybasedir/drivers/modules/ipxe_config.template
+#ipxe_enabled = true
+pxe = true
+#ipxe_boot_script = /tftpboot/boot.ipxe
+instance_master_path = /tftpboot/master_images
+images_path = /tftpboot/cache
+uefi_pxe_bootfile_name=ipxe.efi
  
@@ -4230,6 +4263,7 @@
 # request a particular API version, use the `version`, `min-
 # version`, and/or `max-version` options. (string value)
 #endpoint_override = <None>
+endpoint_override = http://10.1.1.25:6385/
  
 # Verify HTTPS connections. (boolean value)
 #insecure = false
```
### Создаине образа 
#### Добавлем катстомные хуки
```
├── elements
    └── ubuntu-custom
        └── finalise.d
            └── 60-grub
```
```shell 
cat elements/ubuntu-custom/finalise.d/60-grub 
#!/bin/bash

if [ ${DIB_DEBUG_TRACE:-1} -gt 0 ]; then
    set -x
fi
set -eu
set -o pipefail

GRUB="/etc/default/grub"
GRUB_D="/etc/default/grub.d/*"

#cat ${GRUB}
perl -pi -e 's/^(GRUB_CMDLINE_LINUX.*)console=.+?\ (.*)/$1$2/g' ${GRUB}
perl -pi -e 's/^(GRUB_CMDLINE_LINUX.*)console=.+?\ (.*)/$1$2/g' ${GRUB}
perl -pi -e 's/^(GRUB_CMDLINE_LINUX.*)console=.+?\ (.*)/$1$2/g' ${GRUB_D}
perl -pi -e 's/^(GRUB_CMDLINE_LINUX.*)console=.+?\ (.*)/$1$2/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_TERMINAL.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_TERMINAL.*)/#$1/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_SERIAL_COMMAND.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_SERIAL_COMMAND.*)/#$1/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_CMDLINE_LINUX_DEFAULT.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_CMDLINE_LINUX_DEFAULT.*)/#$1/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_DISABLE_LINUX_UUID.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_DISABLE_LINUX_UUID.*)/#$1/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_TIMEOUT_STYLE.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_TIMEOUT_STYLE.*)/#$1/g' ${GRUB_D}

perl -pi -e 's/^(^GRUB_TIMEOUT.*)/#$1/g' ${GRUB}
perl -pi -e 's/^(^GRUB_TIMEOUT.*)/#$1/g' ${GRUB_D}

echo "GRUB_TIMEOUT=${DIB_GRUB_TIMEOUT:-5}" >> ${GRUB}

update-grub
```
#### Скрипт созадния образа
DIB_BLOCK_DEVICE_CONFIG с разметкой дисков

uuid не обязательны
```shell
export DIB_BLOCK_DEVICE_CONFIG='''
- local_loop:
    name: image0
    size: 70GiB
- partitioning:
    base: image0
    label: mbr
    partitions:
      - name: boot
        size: 512MiB
        flags: [ boot,primary ]
      - name: lvm_data
        size: 100%
        flags: [ primary ]
- lvm:
    name: lvm
    base: [ lvm_data ]
    pvs:
        - name: pv
          base: lvm_data
          options: [ "-ff", "-y" ]
    vgs:
        - name: vg
          base: [ "pv" ]
          options: [ "--force" ]
    lvs:
        - name: root
          base: vg
          size: 20GiB
        - name: var_log
          base: vg
          size: 20GiB
- mkfs:
    name: fs_root
    base: root
    type: ext4
    label: "root"
    uuid: b733f302-0336-49c0-85f2-38ca109e8bda
    mount:
      mount_point: /
      fstab:
        options: "defaults"
- mkfs:
    name: fs_var_log
    base: var_log
    type: ext4
    label: "var_log"
    uuid: b733f302-0336-49c0-85f2-38ca109e8bdc      
    mount:
      mount_point: /var/log
      fstab:
        options: "defaults"
- mkfs:
    name: fs_boot
    base: boot
    type: ext4
    label: "boot"
    uuid: b733f302-0336-49c0-85f2-38ca109e8bdc
    mount:
      mount_point: /boot
      fstab:
        options: "defaults"
'''
```
```
cat test-test.sh 
#!/bin/sh

export ELEMENTS_PATH="./elements/:/usr/share/diskimage-builder/elements/"
DIB_DEV_USER_USERNAME=test_user \
DIB_DEV_USER_PWDLESS_SUDO=true \
DIB_DEV_USER_PASSWORD=test12345 \
DIB_DEV_USER_SHELL="/bin/bash" \
DIB_BOOTLOADER_DEFAULT_CMDLINE='nomodeset vga=normal' \
DIB_GRUB_TIMEOUT="5" \
DIB_BLOCK_DEVICE=mbr \
DIB_BOOTLOADER_SERIAL_CONSOLE="" \
disk-image-create ubuntu devuser baremetal cloud-init-nocloud ubuntu-custom bootloader  -o ubuntu.qcow2 -x
```
**!!ВАЖНО!!**
В конце вы должны получить строку 
```
Build completed successfully
```
Если не так то анализируйте лог и доставляйте нужное
Если сборщик не может отмонтировать директорию используйте
```
dmsetup ls 
dmsetup remove ...
losetud 
losetup -D
unmount /tmp_dib*
```

Считаем md5sum и копируем образ в /tftpboot
```
cp -af ubuntu.* /tftpboot/
export MD5SUM=`md5sum ubuntu.qcow2  | cut -d\  -f1| tr \\n ' '`
```

### Добавляем узел
  ```shell
  openstack baremetal node create --driver ipmi \
      --driver-info ipmi_address=10.1.2.67 \
      --driver-info ipmi_username=admin \
      --driver-info ipmi_password=admin \
      --driver-info deploy_kernel=http://10.1.1.25:8080/ipa-centos8-master.initramfs \
      --driver-info deploy_ramdisk=http://10.1.1.25:8080/ipa-centos8-master.kernel
  ```
### Добавляем порт(ы) ноды. С этих маков будет разрешена загрузка по iPXE
```shell
openstack baremetal port create XX:XX:XX:XX:AE:6E --node $UUID
openstack baremetal port create XX:XX:XX:XX:AE:6F --node $UUID
```
### Настариваем узел
```
openstack baremetal node set --instance-info image_source=http://10.1.1.25:8080/ubuntu.qcow2 \
    --boot-interface ipxe \
    --instance-info image_checksum=$MD5SUM \
    --instance-info capabilities='{"boot_option": "local"}' \
    --instance-info root_gb=70 $UUID
 ```
 ### Депоим узел
 ```
 openstack baremetal node deploy $UUID --config-drive '{"meta_data": {"hostname": "server1.cluster"}}'
 ```
 
