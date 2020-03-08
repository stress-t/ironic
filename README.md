**Tags:** ironic standalone baremetal
<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-generate-toc again -->
**Table of Contents**

- [IRONIC STANDALONE](#ironic-standalone)
    - [Общие дествия](#общие-дествия)
        - [Утановка и настройка nginx](#утановка-и-настройка-nginx)
        - [Утановка и настройка Dnsmasq](#утановка-и-настройка-dnsmasq)
    - [Установка mariadb](#установка-mariadb)
    - [Устанавливаем Ironic](#устанавливаем-ironic)
        - [Ironic.conf](#ironicconf)
        - [Подготавлаиваем загрузку по iPXE](#подготавлаиваем-загрузку-по-ipxe)
            - [Скачиваем IPA vmzlinuz и initram](#скачиваем-ipa-vmzlinuz-и-initram)
            - [Файл boot.ipxe](#файл-bootipxe)
            - [Примерное содержимое директории /tftpboot/](#примерное-содержимое-директории-tftpboot)
    - [Установка пакетов дополнительных пакетов](#установка-пакетов-дополнительных-пакетов)
    - [Установка узлов в режиме Legacy Boot](#установка-узлов-в-режиме-legacy-boot)
        - [Создаине образа](#создаине-образа)
            - [Добавлем катстомные хуки](#добавлем-катстомные-хуки)
        - [Скрипт созадния образа](#скрипт-созадния-образа)
        - [Добавляем узел](#добавляем-узел)
        - [Добавляем порт(ы) ноды. С этих маков будет разрешена загрузка по iPXE](#добавляем-порты-ноды-с-этих-маков-будет-разрешена-загрузка-по-ipxe)
        - [Настариваем узел](#настариваем-узел)
        - [Деплоим узел](#деплоим-узел)
    - [Установка узлов в режиме UEFI](#установка-узлов-в-режиме-uefi)
        - [Создаине образа](#создаине-образа)
            - [Добавлем катстомные хуки](#добавлем-катстомные-хуки)
        - [Скрипт для создиня образа диска](#скрипт-для-создиня-образа-диска)
        - [Добавляем узел](#добавляем-узел)
        - [Добавляем порт(ы) ноды. С этих маков будет разрешена загрузка по iPXE](#добавляем-порты-ноды-с-этих-маков-будет-разрешена-загрузка-по-ipxe)
        - [Настариваем узел](#настариваем-узел)
        - [Деплоим узел](#деплоим-узел)

<!-- markdown-toc end -->


# IRONIC STANDALONE

## Общие дествия
Да грязно, но пока так
```
setenforce 0
systemctl stop firewalld
systemctl disable firewalld

```

```
mkdir /tftpboot
```
### Утановка и настройка nginx
**CentOS**
```
yum install -y epel-release
yum install -y nginx
```
**Ubuntu**
```
apt-get install nginx
```
**Конфиг nginx**
```
/etc/nginx/nginx.conf
http {
    ...
    server {
        ...
        listen       8080 default_server;
        listen       [::]:8080 default_server;
        ...
        root         /tftpboot;
        ...
        }
    ...
}
```
```
systemctl enable nginx
systemctl restart nginx
```
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
interface=eno1
bind-dynamic
enable-tftp
tftp-root=/tftpboot

# Disable listening for DNS
port=0

log-dhcp
dhcp-range=10.1.1.104,10.1.1.105,12h #Что-бы никому не мешать
dhcp-host=44:a1:91:fe:ae:6e,10.1.1.104 # Что-бы никому не мешать
dhcp-host=44:a1:91:fe:ae:6f,10.1.1.105 # Что-бы никому не мешать

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
## Установка mariadb
```
yum install mariadb-server mariadb -y
systemctl enable mariadb
systemctl restart mariadb
```
```
# mysql -u root -p
mysql> CREATE DATABASE ironic CHARACTER SET utf8;
mysql> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost' \
       IDENTIFIED BY 'IRONIC_DBPASSWORD';
mysql> GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' \
       IDENTIFIED BY 'IRONIC_DBPASSWORD';
```



## Устанавливаем Ironic
**CentOS**
**!!ВАЖНО!!**
Мне пришлось взять https://trunk.rdoproject.org/ (https://trunk.rdoproject.org/centos7-train/current/) потому что правки для установки uefi были только в свежих версиях Train
```
curl -o /etc/yum.repos.d/delorean.repo https://trunk.rdoproject.org/centos7-train/current/delorean.repo
yum-config-manager --enable delorean
```
https://docs.openstack.org/ironic/latest/install/install-rdo.html

```
yum install -y openstack-ironic-api openstack-ironic-conductor diskimage-builder bash-completion python-openstackclient bash-completion-extras
openstack complete | sudo tee /etc/bash_completion.d/osc.bash_completion > /dev/null
source /etc/profile.d/bash_completion.sh
```


```
rpm -qa |grep ironic
openstack-ironic-common-13.0.3-0.20200305145746.73ec9b8.el7.noarch
openstack-ironic-python-agent-builder-1.1.1-0.20200302130802.b32f4ea.el7.noarch
python2-ironic-inspector-client-3.7.0-1.el7.noarch
python2-ironic-lib-2.21.0-1.el7.noarch
openstack-ironic-api-13.0.3-0.20200305145746.73ec9b8.el7.noarch
python2-ironicclient-3.1.1-1.el7.noarch
openstack-ironic-conductor-13.0.3-0.20200305145746.73ec9b8.el7.noarch
```

### Ironic.conf
```
[DEFAULT]
auth_strategy = noauth
enabled_hardware_types = ipmi
enabled_bios_interfaces = no-bios,redfish,ilo
enabled_boot_interfaces = ipxe,pxe
enabled_management_interfaces = ipmitool
enabled_network_interfaces=noop
enabled_power_interfaces = ipmitool
enabled_vendor_interfaces = ipmitool,no-vendor
rpc_transport = json-rpc
debug = true

[agent]
deploy_logs_collect = always
image_download_source = http

[api]
api_workers = 5

[conductor]
automated_clean = false

[database]
connection = mysql+pymysql://ironic:IRONIC_DBPASSWORD@127.0.0.1/ironic?charset=utf8

[deploy]
http_url = http://10.1.1.25:8080
http_root = /tftpboot
[disk_utils]
dd_block_size = 4M
[dhcp]
dhcp_provider = none

[pxe]
pxe_append_params = systemd.journald.forward_to_console=yes ipa-debug=1 nofb nomodeset vga=normal
tftp_server = 10.1.1.25
tftp_root = /tftpboot
tftp_master_path = /tftpboot/master_images
pxe_bootfile_name = undionly.kpxe
pxe_config_template = $pybasedir/drivers/modules/ipxe_config.template
pxe = true
instance_master_path = /tftpboot/master_images
images_path = /tftpboot/cache
uefi_pxe_bootfile_name=ipxe.efi
uefi_pxe_config_template = $pybasedir/drivers/modules/ipxe_config.template
[service_catalog]
endpoint_override = http://10.1.1.25:6385/
```
### Подготавлаиваем загрузку по iPXE
Создаем все нужные директории и сокпируем фалы согластно иструкции
Все действия с директорией /httpboot меняем на /tftpboot
Шаги с изменинями конфигов не нужны (уже настроили выше)

https://docs.openstack.org/ironic/latest/install/configure-pxe.html#ipxe-setup

#### Скачиваем IPA vmzlinuz и initram
На момент тестов работаь с много томными имаджами может только ipa-centos8
**IPA:** https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/

#### Файл boot.ipxe
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

```
ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
systemctl enable openstack-ironic-api openstack-ironic-conductor
systemctl restart openstack-ironic-api openstack-ironic-conductor
export OS_AUTH_TYPE=none
export OS_ENDPOINT=http://localhost:6385/

```

#### Примерное содержимое директории /tftpboot/
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
## Установка пакетов дополнительных пакетов
```
yum install -y squashfs-tools gdisk policycoreutils-python
```

## Установка узлов в режиме Legacy Boot
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
### Скрипт созадния образа
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
    mount:
      mount_point: /
      fstab:
        options: "defaults"
- mkfs:
    name: fs_var_log
    base: var_log
    type: ext4
    label: "var_log"
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
      --name test-mnode \
      --driver-info ipmi_address=10.1.2.67 \
      --driver-info ipmi_username=admin \
      --driver-info ipmi_password=admin \
      --deploy-interface direct \
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
openstack baremetal node manage $UUID
openstack baremetal node provide $UUID

openstack baremetal node set --instance-info image_source=http://10.1.1.25:8080/ubuntu.qcow2 \
    --boot-interface ipxe \
    --instance-info image_checksum=$MD5SUM \
    --instance-info capabilities='{"boot_option": "local"}' \

```
### Деплоим узел
 ```
 openstack baremetal node deploy $UUID --config-drive '{"meta_data": {"hostname": "server1.cluster"}}'
 ```

## Установка узлов в режиме UEFI
### Создаине образа
#### Добавлем катстомные хуки
```
├── elements
    └── ubuntu-custom
        └── finalise.d
            └── 60-grub
```
**!!ВАЖНО!!**
На текущий момент Скритп заточен под ubuntu. При создании обзара не заполнятеся /boot/efi, по этому заполняем его этим скриптом
```shell
cat elements/ubuntu-custom/finalise.d/60-grub

#!/bin/bash

if [ ${DIB_DEBUG_TRACE:-1} -gt 0 ]; then
    set -x
fi
set -eu
set -o pipefail

#centos
#yum reinstall -y grub2-efi shim 
#grub2-mkconfig --output=/boot/efi/EFI/centos/grub.cfg

#UBUNTU
apt-get install --reinstall grub-efi-amd64

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
grub-install --efi-directory=/boot/efi
```
### Скрипт для создиня образа диска
**!!ВАЖНО!!**

Обрати внимаение что есть единицы измерения GB и GiB (маркетологи пишут GB на дисках)

Типы разделов обязатяельны при UEFI+GPT.
```
export DIB_BLOCK_DEVICE_CONFIG='''
- local_loop:
    name: image0
    size: 238GB
- partitioning:
    base: image0
    label: gpt
    partitions:
      - name: ESP
        type: 'EF00'
        size: 512MiB
        mkfs:
          type: vfat
          mount:
            mount_point: /boot/efi
            fstab:
              options: "defaults"
              fsck-passno: 1
      - name: boot
        size: 512MiB
        type: 'EF02'
        mkfs:
          type: ext4
          mount:
            mount_point: /boot
            fstab:
              options: "defaults"
              fsck-passno: 1
      - name: lvm_data
        type: '8e00'
        size: 100%
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
    mount:
      mount_point: /
      fstab:
        options: "defaults"
- mkfs:
    name: fs_var_log
    base: var_log
    type: ext4
    label: "var_log"
    mount:
      mount_point: /var/log
      fstab:
        options: "defaults"
'''
```

```
cat test-efi.sh
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
disk-image-create ubuntu devuser baremetal cloud-init-nocloud grub2 block-device-efi ubuntu-custom -o ubuntu.qcow2 -x --logfile dib_`date "+%Y_%m_%d-%H:%M:%S"`.log
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
    --name test-mnode \
    --driver-info ipmi_address=10.1.2.67 \
    --driver-info ipmi_username=admin \
    --driver-info ipmi_password=admin \
    --deploy-interface direct \
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
openstack baremetal node manage $UUID
openstack baremetal node provide $UUID

openstack baremetal node set --instance-info image_source=http://10.1.1.25:8080/ubuntu.qcow2 \
    --boot-interface ipxe \
    --instance-info image_checksum=$MD5SUM \
    --instance-info disk_format=qcow2 \
    --deploy-interface direct \
    --driver-info deploy_kernel=http://10.1.1.25:8080/ipa-centos8-master.kernel \
    --driver-info deploy_ramdisk=http://10.1.1.25:8080/ipa-centos8-master.initramfs \
    --instance-info capabilities='{"boot_option": "local", "boot_mode":"uefi", "secure_boot": "true"}' \
    --property capabilities='boot_mode:uefi,disk_label:gpt' \
    $UUID
```
### Деплоим узел
```
openstack baremetal node deploy $UUID --config-drive '{"meta_data": {"hostname": "server1.cluster"}}'
```
