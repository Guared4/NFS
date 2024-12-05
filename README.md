# NFS
 
Данный стенд мной брался из методички домашнего задания NFS, после чего был отредактирован, а именно
В конфигурации Vagrantfile модуль network, подсеть выбрал исходя из настроек DHCP сервера VirtualBox на хосте;
так же, чтобы избежать ошибки ""rsync" could not be found on your PATH. Make sure that rsync is properly installed on your system and available on the PATH.", следуя руководству по ссылке https://qna.habr.com/q/271364, необходимо установить плагин vagrant-vbguest и добавить запись в Vagrantfile: 
На хосте выполнил команду: vagrant plugin install vagrant-vbguest, 
и дописал в Vagrantfile: config.vm.synced_folder ".", "/vagrant", type: "virtualbox".

Цели домашнего задания
Научиться самостоятельно развёртывать сервис NFS и подключать к нему клиента(ов)

Описание домашнего задания
Основная часть:

vagrant up должен поднимать 2 настроенных виртуальных машины (сервер NFS и клиента) без дополнительных ручных действий;
на сервере NFS должна быть подготовлена и экспортирована директория; 
в экспортированной директории должна быть поддиректория с именем upload с правами на запись в неё; 
экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальной машины (systemd, autofs или fstab — любым способом);
монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3.
#Для самостоятельной реализации: 
#настроить аутентификацию через KERBEROS с использованием NFSv4.


1. Настраиваю сервер NFS 
Захожу на сервер:
vagrant ssh nfss

Дальнейшие действия выполняю от имени root. 

2. Проверил наличие слушающих портов 2049/udp, 2049/tcp, 111/udp, 111/tcp:

root@nfss:~# ss -tnplu | grep -E '2049|111'
udp   UNCONN 0      0               0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=431,fd=5),("systemd",pid=1,fd=37))
udp   UNCONN 0      0                  [::]:111           [::]:*    users:(("rpcbind",pid=431,fd=7),("systemd",pid=1,fd=39))
tcp   LISTEN 0      64              0.0.0.0:2049       0.0.0.0:*
tcp   LISTEN 0      4096            0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=431,fd=4),("systemd",pid=1,fd=36))
tcp   LISTEN 0      64                 [::]:2049          [::]:*
tcp   LISTEN 0      4096               [::]:111           [::]:*    users:(("rpcbind",pid=431,fd=6),("systemd",pid=1,fd=38))

3. Создал и настроил директорию, которая будет экспортирована в будущем:

root@nfss:~# mkdir -p /srv/share/upload
root@nfss:~# chown -R nobody:nogroup /srv/share
root@nfss:~# chmod 0777 /srv/share/upload

4. Cоздал в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию:

root@nfss:~# cat << EOF > /etc/exports
> /srv/share 192.168.56.11/32(rw,sync,root_squash)
> EOF

5. Экспортировал ранее созданную директорию:

root@nfss:~# exportfs -r
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.56.11/32:/srv/share".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x

6. Проверил экспортированную директорию следующей командой:

root@nfss:~# exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

7. Настроил клиент NFS

vagrant ssh nfsc

8. Дальнейшие действия выполняю от имени root. 
Установил пакет с NFS-клиентом:

root@nfsc:~# apt install nfs-common -y

9. Добавил в /etc/fstab строку:

root@nfsc:~# echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab

10. И выполнил команды:

root@nfsc:~# systemctl daemon-reload
root@nfsc:~# systemctl restart remote-fs.target

в данном случае происходит автоматическая генерация systemd units в каталоге /run/systemd/generator/, которые производят монтирование при первом обращении к каталогу /mnt/.

root@nfsc:~# cat /run/systemd/generator/mnt.mount
# Automatically generated by systemd-fstab-generator

[Unit]
Documentation=man:fstab(5) man:systemd-fstab-generator(8)
SourcePath=/etc/fstab

[Mount]
Where=/mnt
What=192.168.56.10:/srv/share/
Type=nfs
Options=vers=3,noauto,x-systemd.automount

12. Перешёл в директорию /mnt/ и проверил успешность монтирования:

root@nfsc:~# ls /mnt
upload

root@nfsc:~# mount | grep mnt
nsfs on /run/snapd/ns/lxd.mnt type nsfs (rw)
systemd-1 on /mnt type autofs (rw,relatime,fd=49,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=26696)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=54920,mountproto=udp,local_lock=none,addr=192.168.56.10)

Обратим внимание на vers=3, что соответствует NFSv3, как того требует задание.

13. Проверка работоспособности
захожу на сервер vagrant ssh nfss
захожу в каталог root@nfss:~# cd /srv/share/upload

создал тестовый файл root@nfss:/srv/share/upload# touch check_file
-----------------
захожу на клиент vagrant ssh nfsc
захожу в каталог root@nfsc:/mnt/upload#

проверяю наличие ранее созданного файла 
root@nfsc:/mnt/upload# ls
check_file

создал тестовый файл root@nfsc:/mnt/upload# touch client_file
проверил, что файл успешно создан и доступен на сервере
root@nfsc:/mnt/upload# ls
check_file  client_file

Если вышеуказанные проверки прошли успешно, это значит, что проблем с правами нет.

14. Предварительно проверяю клиент:

перезагружаю клиент 
захожу на клиент
захожу в каталог

vagrant ssh nfsc
root@nfsc:~# cd /mnt/upload
root@nfsc:/mnt/upload# ls
check_file  client_file

Проверяю сервер:

перезагружаю сервер 
захожу на сервер
захожу в каталог

vagrant ssh nfss
cd /srv/share/upload/
root@nfss:/srv/share/upload# ls
check_file  client_file

проверил экспорты exportfs -s
root@nfss:/srv/share/upload# exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

проверил работу RPC showmount -a 192.168.56.10

root@nfss:/srv/share/upload# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share

Проверяю клиент:

возвращаюсь на клиент;
перезагружаю клиент;
захожу на клиент;
проверяем работу RPC
root@nfsc:~# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share

захожу в каталог root@nfsc:/mnt/upload#
проверяю статус монтирования 
root@nfsc:/mnt/upload# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=60,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=15691)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=43807,mountproto=udp,local_lock=none,addr=192.168.56.10)
nsfs on /run/snapd/ns/lxd.mnt type nsfs (rw)

проверяю наличие ранее созданных файлов
root@nfsc:/mnt/upload# ls
check_file  client_file

создаю тестовый файл root@nfsc:/mnt/upload# touch final_check

проверяю, что файл успешно создан
root@nfsc:/mnt/upload# ls
check_file  client_file  final_check


Далее создал 2 bash-скрипта, nfss_script.sh - для конфигурирования сервера и nfsc_script.sh - для конфигурирования клиента, в которых описал bash-командами все выполненные шаги выше

----------------
end