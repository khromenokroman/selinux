# SELinux
DZ otus SELinux

Для проверки дз необходимо запустит Vagrantfile, который    
развернет 2 виртуалки(ns01 и client) и подцепит файлы ns01.sh и client.sh, которые    
доставит нам необходимые пакеты и настройки. Файл ns01.sh и client.sh должны    
находится в одной директории с Vagrantfile. Далее необходимо    
проделать шаги описанные ниже.    

1 Часть запустить nginx на нестандартном порту 3-мя разными способами.    
Заходим на сервер ns01    
```vagrant ssh ns01```    
И там выполяем:    

a) Переключатель setsebool    
Для этого поменяем порт на нестандартный и запуcтим и nginx    
```sed -i 's|listen       80 default_server;|listen       10000 default_server;|' /etc/nginx/nginx.conf>```
systemctl start nginx     
Что выдаст нам ошибку:     
```Oct 16 14:32:01 selinux nginx[6883]: nginx: [emerg] bind() to 0.0.0.0:10000 failed (13: Permission denied)```
Далее проанализируем лог:    
```audit2why < /var/log/audit/audit.log```     
Где нам и будет предложено решение:         
```setsebool nis_enabled 1```     
После этого делаем рестарт сервера nginx    
```systemctl restart nginx```    
И проверяем его работоспособность    
```curl localhost:10000```    

б) добавление порта в тип    
Прверяем наш порт:     
```semanage port -l | grep http```    
Ответ такой:     
```http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```    

Его нет, необходо добавить:     
```semanage port -a -t http_port_t -p tcp 10000```     

Проверяем:     
```semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      10000, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```    
Видим порт доступен. И снова проверяем       
```curl localhost:10000```    

Удаляем порт      
```semanage port -d -t http_port_t -p tcp 10000```     

в) Модуль SELinux     
Снова анализируем логи и компилируем модуль на основе файла аудита     
```audit2allow -M httpd_add --debug < /var/log/audit/audit.log```     
Получили модуль, теперь его необходимо загрузить     
```semodule -i httpd_add.pp```     
Прверяем встал ли наш модуль      
```semodule -l | grep http```     
Ответ:      
```httpd_add       1.0```    
Запускаем наш nginx:      
```systemctl start nginx```    
И проверяем:     
```curl localhost:10000```    
Чтобы потом удалить модуль     
```semodule -r httpd_add```    
На этом первая часть дз закончена.     

2 Часть. Для проверки второй необходимо зайти на ns01      
```vagrant ssh ns01```     
Там выполняем     
```ansible-playbook /vagrant/provisioning/playbook.yml```     
Что установит нам необходимые пакеты и настройки для выполнения дз      
После в новом терминале заходим на сервер client     
```vagrant ssh client```     
И там выполяем:     
```nsupdate -k /etc/named.zonetransfer.key
server 192.168.50.10
zone ddns.lab
update add www.ddns.lab. 60 A 192.168.50.15
send
```

Получим ошибку:     
```update failed: SERVFAIL```     

Далее идем на сервер ns01 и там выполняем:     
```audit2allow -M named-selinux --debug  < /var/log/audit/audit.log
semodule -i named-selinux.pp
```     
Пробуем снова с сервера client:    
```nsupdate -k /etc/named.zonetransfer.key
server 192.168.50.10
zone ddns.lab
update add www.ddns.lab. 60 A 192.168.50.15
send
```      
Снова ошибка переходим на сервак ns01 и там выполяем     
```cat /var/log/messages```     
Там находим подсказку что нужно сделать и поочередно выполняем      
```ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000 | semodule -i my-iscworker0000.pp
ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0001 | semodule -i my-iscworker0001.pp
ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0002 | semodule -i my-iscworker0002.pp
ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0003 | semodule -i my-iscworker0003.pp
ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0004 | semodule -i my-iscworker0004.pp
```      
После чего ошибка уйдет, но будет ошибка в named сервисе:      
```ns01 named[9299]: /etc/named/dynamic/named.ddns.lab.view1.jnl: open: permission denied```     

Для ее решания удаляем     
```rm /etc/named/dynamic/named.ddns.lab.view1.jnl```     

И делаем рестарт службы:     
```systemctl restart named```      

После всех манипуляций ошибка уходит     

Вывод:     
Selinux блокирует доступ к обновлению файлов для DNS сервера, а так же к файлам к которым обращается DNS-    
сервер во время своей работы.      
Для решения проблемы можно использовать скомпилировнанные модули SELinux или изменять контекст безопасности     
для файлов SELinux к которым обращается BIND.     
