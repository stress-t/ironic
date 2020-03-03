**Tags:** ironic standelon baremetal
1. Скачиваем IPA vmzlinuz и initram c https://tarballs.opendev.org/openstack/ironic-python-agent/dib/files/
1. Добавляем узел
  ```shell
  openstack baremetal node create --driver ipmi \
      --driver-info ipmi_address=10.220.4.67 \
      --driver-info ipmi_username=Administrator \
      --driver-info ipmi_password=Admin@9000 \
      --driver-info deploy_kernel=http://10.220.36.25:8080/ipa-centos7-stable-train.kernel \
      --driver-info deploy_ramdisk=http://10.220.36.25:8080/ipa-centos7-stable-train.initramfs
  ```
1. Добавляем порт(ы) ноды. С этих маков будет разрешена загрузка по iPXE
```shell
openstack baremetal port create XX:XX:XX:XX:AE:6E --node $UUID
openstack baremetal port create XX:XX:XX:XX:AE:6F --node $UUID
```
1. далее
