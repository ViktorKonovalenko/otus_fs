<h1>РЕШЕНИЕ</h1>
Запускаем Vagrant файл, в котором автоматически создается RAID и 5 партиций

<h1>Путь к решению ДЗ</h1>
Добавляем в Vagrantfile два новых диска
<pre>
  MACHINES = {
    :otuslinux => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.101',
        :disks => {
            .
            .
            .
            .
            :sata5 => {
                :dfile => './sata5.vdi',
                :size => 250,
                :port => 5
            },
            :sata6 => {
                :dfile => './sata6.vdi',
                :size => 250,
                :port => 6
            }
            
</pre>

Добавляем в Vagrantfile SHELL команды на создание RAID10 и автоматическую конфигурацию при загрузке системы
<pre>
mdadm --create --verbose /dev/md0 -l 10 -n 6 /dev/sd{b,c,d,e,f,g}
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> \ /etc/mdadm/mdadm.conf
</pre>

Добавляем в Vagrantfile SHELL команды на создание 5 партиций GPT и автоматическое их монтирование при загрузке системы в определенные каталоги 
<pre>
parted -s /dev/md0 mklabel gpt
for i in 0 20 40 60 80;do parted /dev/md0 mkpart primary $i% $(( $i+20 ))% -s;done
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mkdir -p /raid/part{1,2,3,4,5}
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
echo -e "$(for i in {1..5};do echo "$(blkid|grep md0p$i |awk '{print $2 }') /raid/part_$i  ext4 defaults 0 0" ;done)\n" >> /etc/fstab
</pre>

Запускаем vagrant
<pre>
  vagrant up
</pre>
ВМ создалась с RAID10 и смонтированными каталогами
<pre>
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdg[5] sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 512K chunks 2 near-copies [6/6] [UUUUUU]
      
unused devices: <none>
</pre>
<pre>
[root@otuslinux vagrant]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.8M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
/dev/sda1        40G  5.1G   35G  13% /
/dev/md0p1      139M  1.6M  127M   2% /raid/part1
/dev/md0p2      140M  1.6M  128M   2% /raid/part2
/dev/md0p3      142M  1.6M  130M   2% /raid/part3
/dev/md0p4      140M  1.6M  128M   2% /raid/part4
/dev/md0p5      139M  1.6M  127M   2% /raid/part5
tmpfs           100M     0  100M   0% /run/user/1000
</pre>

<h2>Пробуем сломать RAID</h2>
Производим имитацию сломанного диска 
<pre>
[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sdf
mdadm: set /dev/sdf faulty in /dev/md0
</pre>
Прверяем RAID 
<pre>
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdg[5] sdf[4](F) sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 512K chunks 2 near-copies [6/5] [UUUU_U]
      
unused devices: <none>
</pre>
Видим что один диск неактивен<br>
Удаляем сломанный диск из массива 
<pre>
[root@otuslinux vagrant]# mdadm /dev/md0 --remove /dev/sdf
mdadm: hot removed /dev/sdf from /dev/md0
</pre>
Добавляфем новый диск на место старого 
<pre>
[root@otuslinux vagrant]# mdadm /dev/md0 --add /dev/sdf
mdadm: added /dev/sdf
</pre>
Проверяем наш массив 
<pre>
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdf[6] sdg[5] sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 512K chunks 2 near-copies [6/6] [UUUUUU]
      
unused devices: <none>  
</pre>
Видим что массив восстановелн и все диске активны

