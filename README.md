#  PAM
### Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

#### Запускаем вертуальную машину используя [Vagrant-file](https://github.com/SalnikovAnton/pam/blob/main/Vagrantfile "Vagrant-file").  
После поднятия машины создаём пользователя otusadm и otus, и назначаем пользователям пароли:  
```
[root@pam ~]# sudo useradd otusadm && sudo useradd otus
[root@pam ~]# echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.
```
Создаём группу admin и добавляем в нее пользователей vagrant,root и otusadm. После проверяем что все прошло успешно:
```
[root@pam ~]# sudo groupadd -f admin
[root@pam ~]# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
[root@pam ~]# cat /etc/group | grep admin
printadmin:x:994:
admin:x:1003:otusadm,root,vagrant
```
После создания пользователей, нужно проверить, что они могут подключаться по SSH к нашей ВМ. Для этого пытаемся подключиться с хостовой машины:
```
 anton@ml110  ~/pam  ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ED25519 key fingerprint is SHA256:JJUYdk3mvAeEIBtWTlwJwn+uQ6+9HS9XJIpqGwhvKoc.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
Warning: Permanently added '192.168.57.10' (ED25519) to the list of known hosts.
otus@192.168.57.10's password: 
[otus@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.
```
Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни. Выберем метод PAM-аутентификации, так как у нас используется только ограничение по времени, то было бы логично использовать метод pam_time, однако, данный метод не работает с локальными группами пользователей, и, получается, что использование данного метода добавит нам большое количество однообразных строк с разными пользователями. В текущей ситуации лучше написать небольшой скрипт контроля и использовать модуль pam_exec.  
Создадим файл-скрипт /usr/local/bin/login.sh
```
[root@pam ~]# vi /usr/local/bin/login.sh

#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```
Добавим права на исполнение файла:
```
[root@pam ~]# chmod +x /usr/local/bin/login.sh
```
Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:
```
[root@pam ~]# vi /etc/pam.d/sshd 

#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
account    required     dad
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```
На этом настройка завершена.   








