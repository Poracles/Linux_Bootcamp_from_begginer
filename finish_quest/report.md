# Финальное задание.

- **!!! Некоторые комментарии направлены для получения *comeback* информации и адресованы куратору и участникам курса !!!**

## Разворот полного стека LEMP

## Оглавление
0. [Подготовка](#01-подготовка)

    0.1. [Создаем отказоустойчивое хранилище](#создаем-отказоустойчивое-хранилище)

1. [ Подготовка Базы Данных (MariaDB).](#1-подготовка-базы-данных-mariadb)

2. [ Установка WordPress и phpMyAdmin.](#2-установка-wordpress-и-phpmyadmin)

    2.1 [WordPress (Сайт).](#21-wordpress-сайт)

    2.2 [**phpMyAdmin**](#22-phpmyadmin)

    2.3 [Права доступа.](#23-права-доступа)

3. [Настройка Nginx.](#3-настройка-nginx)

    3.1 [Сертификаты.](#31-сертификаты)

    3.2 [Конфиг виртуалхоста.](#32-конфиг-виртуалхоста)

    3.3 [Активация.](#33-активация)

4. [Финальная настройка.](#4-финальная-настройка)

    4.1 [Подключение.](#41-подключение)

    4.2 [Установки в браузере.](#42-установки-в-браузере)

    4.3 [Проверка защиты. ](#43-проверка-защиты)
    
5. [Бэкап.](#5-бэкап)

### 0.1 Подготовка

#### Создаем отказоустойчивое хранилище

- Добавляем HDD в ВМ

- Список дисков.

![Список_дисков](/finish_quest/img/0.1.1.png)

- Создаем **RAID** массив.

    `sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde`

    - Сохраняем конфигурацию в **mdadm.conf**.

    `sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf`

    - Обновляем **initramfs**.

    `sudo update-initramfs -u`

> Я решил отступить от задания, и поднять class=10.

- Создаём **PV**

    `sudo pvcreate /dev/md0`

- Создаём **VG**.

    `sudo vgcreate web_data /dev/md0`

- Создаем **LV**.

    `sudo lvcreate -L 2G -n logs web_data`

    `sudo lvcreate -L 2G -n site_files web_data`

    `sudo lvcreate -L 2G -n db_date web_data`

    > Тут я ошибся в названии "datЕ" вместо "datа".

- Форматируем тома в ext4.

    `sudo mkfs.ext4 /dev/web_data/logs`

    `sudo mkfs.ext4 /dev/web_data/site_files`

    `sudo mkfs.ext4 /dev/web_data/db_data`

- Монтируем **LV** в файловую систему.

    `sudo mkdir -p /var/log`

    `sudo mkdir -p /var/www`

    `sudo mkdir -p /var/lib/mysql`

- Делаем миграцию **/var/log**.

    > Пишу данный отчет после проделанных действий, с вашего позволения пропущу описание данного шага, тем более, что он отлично описан в задании, добавить нечего.

- Проверяем нашу работу.

    `lsblk`

    - Получившееся дерево LVM.

    ![lsblk_RAID](/finish_quest/img/0.1.2.png)

- Получаем **UUID** томов для **fstab**.

    `sudo blkid /dev/web_data/logs`

    `sudo blkid /dev/web_data/site_files`

    `sudo blkid /dev/web_data/db_data`

    - Список всех UUID.

    ![UUID](/finish_quest/img/0.1.3.png)

- Редактируем **fstab**.

    `sudo nano /etc/fstab`

    > Дописываем **UUID** в конец файла.

    - Редактированный **fstab**

    ![fstab](/finish_quest/img/0.1.4.png)

### 1. Подготовка Базы Данных (MariaDB).

- Создадим базу данных и пользователей.

    `sudo mariadb `

    - Создаем базу данных.

    `CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`

    -  Пользователь для приложения.

    `CREATE USER 'wp_app'@'localhost' IDENTIFIED BY '-----!'`

    - Полные права на базу wordpress.

    `GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_app'@'localhost';` 

    - Пользователь для бэкапов.

    `CREATE USER 'backup_user'@'localhost' IDENTIFIED BY '-----!'; `

    - Права только на чтение и блокировку для дампа.

    `GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backup_user'@'localhost';`

    - Применяем права и выходим.

        `SHOW GRANTS FOR 'wp_app'@'localhost';`
        
        - Права для **wp_app**.

        ![Grants_for_wp_app](/finish_quest/img/1.1.png)

        `SHOW GRANTS FOR 'backup_user'@'localhost';`

        - Права для **backup_user**.

        ![Grants_for_backup_user](/finish_quest/img/1.2.png)

    `EXIT;`
    
    > В работе с Mariadb возникли трудности в плане синтаксиса. я не сразу заметил ";" в конце строки, а потом при вводе пароля забыл "`" добавить в конце.. из за чего потерял не мало времени...

### 2. Установка WordPress и phpMyAdmin.

#### 2.1 WordPress (Сайт).

- Идем в корень веб-сервера

`cd /var/www/html` 

> Тут возникла странность, по какой то причине **/html** у меня не оказалось... 
>
> Cоздал директорию.
>
> `sudo mkdir html`
>
> Соответсветнно, **index.nginx-debian.html ** не оказалось, пропустил шаг.
>
> Ох... чувствую не к добру это...

- Скачиваем и распаковываем **WordPress**.

    `sudo wget https://wordpress.org/latest.tar.gz`

    `sudo tar -xzvf latest.tar.gz`

- Переносим файлы из подпапки в корень.

`sudo mv wordpress/* .` 

- Удаляем ненужную (пустую) директорию.

`sudo rmdir wordpress`
    
- Удаляем скачанный и распакованный архив.

`sudo rm latest.tar.gz`

- Содержимое нашего сайта - wordpress.

![Word_Press](/finish_quest/img/2.1.1.png)

#### 2.2 **phpMyAdmin**

> Внимание: Мы ставим его вручную, чтобы не тянуть зависимости **Apache**.

- Скачиваем.

`sudo wget https://files.phpmyadmin.net/phpMyAdmin/5.2.3/phpMyAdmin-5.2.3-all-languages.tar.gz`

`sudo tar -xzvf phpMyAdmin-5.2.3-all-languages.tar.gz`

- Переименовываем для удобства в **pma**.

`sudo mv phpMyAdmin-5.2.3-all-languages pma`

- **phpMyAdmin** скачан и переименован.

![pma](/finish_quest/img/2.2.0.png)

- Удаляем gz-архив.

`sudo rm phpMyAdmin-5.2.3-all-languages.zip`

- Создаем конфиг из шаблона.

`cp pma/config.sample.inc.php pma/config.inc.php`

- Генерируем **blowfish_secret**.

`openssl rand -base64 32`

- Вставим полученный **blowfish_secret** в конфиг.

`sudo nano pma/config.inc.php`

- **config.inc.php** строка с **blowfish_secret** заполнена.

![config.inc.php_blowfish_secret](/finish_quest/img/2.2.1.png)

- Проверим **config.inc.php** на синтаксичечкие ошибки.

`php -l pma/config.inc.php`

![All_OK](/finish_quest/img/2.2.2.png)

#### 2.3. Права доступа.

- Проверим от какого имени работают слыжбы **php-fpm**

![php-fpm_users](/finish_quest/img/2.3.1.png)

- Поменяем владельца файлов и директорий и дадим права

![chown_chmod](/finish_quest/img/2.3.2.png)

> www-data - этот пользователь будет выполнять PHP-скрипты.
> Ему нужны права для загрузки картинок и обновлений, установки плагинов.

- Проверим нововведения, создадим файл от пользователя www-data

`sudo -u www-data touch test.txt`

![new_test](/finish_quest/img/2.3.3.png)

### 3. Настройка Nginx.

#### 3.1. Сертификаты.

-  Генерируем самоподписной сертификат.

    `sudo mkdir -p /etc/nginx/ssl`
    
    `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \`

    `-keyout /etc/nginx/ssl/nginx.key \` 

    `-out /etc/nginx/ssl/nginx.crt`

    - Ввод данных для сертификата.

    ![ssl](/finish_quest/img/3.1.1.png)

#### 3.2. Конфиг виртуалхоста.

- Создаем файл конфига и редактируем его.

`sudo nano /etc/nginx/sites-available/wordpress.`

![conf_wordpress.](/finish_quest/img/3.2.1.png)

> Я не смог настроить общий буфер обмена с виртуальной машиной, поэтому много времени потратил на переписывание всех конфигов. на момент написания данного коммита надеюсь, что правильно посчитал количество пробелов в табуляции и не совершил синтаксических ошибок.
>
> Еще желательно бы полностью разобрать структуру конфига, что зачем следует, и прочее... но это уже будет другая история...

#### 3.3 Активация.

- Выполняем команды:

`sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/`

`sudo rm /etc/nginx/sites-enabled/default`

`sudo nginx -t`

`sudo systemctl reload nginx`

### 4. Финальная настройка.

#### 4.1. Подключение.

- Нам нужно прописать **IP** нашего сервера в **/etc/hosts**.

- Добавляем **Адаптер 2** сетевой мост в ***Сеть*** в **VirtualBox**

![adpter_2](/finish_quest/img/4.1.1.png)

- Редактируем **netplan** конфиг.

`sudo nano /etc/netplan/50-cloud-init.yaml`

![netplan](/finish_quest/img/4.1.2.png)

> Я уже создавал до этого локальную сеть с другой ВМ для проверки предыдущих заданий, поэтому тут есть закоминченый постоянный IP.
>
> Кстати, был опыт работы с Ubuntu v.20.04 там не было подсветки синтаксиса конфига netplan, по мне, тут совершен огромный прорыв...

- Обновляем **netplan** перезапускаем **enp0s8**.

`sudo netplan apply`

`sudo ip link set enp0s8`

- смотрим результат:

`sudo a show enp0s8`

![a](/finish_quest/img/4.1.3.png)

> Служба **DHCP** выдала нам **IP** адрес, значит у нас есть локальный коннект с хостом.

- Записываем полученный IP в конфиг:

![hosts](/finish_quest/img/4.1.4.png)

- Открываем порт 443 в **iptables**.

`sudo ufw allow 443/tcp`

- Проверяем правила для фаервола.

`Sudo ufw status`

![ufw_status](/finish_quest/img/4.1.5.png)

#### 4.2. Установки в браузере.

- Открываем **https://mysite.local/** в браузере и выбираем язык.

![open_my-site](/finish_quest/img/4.2.6.png)

![select_language](/finish_quest/img/4.2.6.2.png)

- Заполняем данные для **инсталлятора wordpress**.

![Install_wordpress](/finish_quest/img/4.2.7.png)

- Подтверждение введенных данных.

![aproved](/finish_quest/img/4.2.8.png)

#### 4.3. Проверка защиты. 

- Заходим на **https://mysite.local/pma/**.

![pma_white_list](/finish_quest/img/4.2.9.png)

- Меняем **IP** доступа к сайту в конфиге.

![IP_change](/finish_quest/img/4.2.10.1.png)

- проверяем синтаксис (на всякий случай).

`sudo nginx -t`

- Перезагружаем сервер.

`sudo systemctl reload nginx`


![pma_not_aproved](/finish_quest/img/4.2.10.2.png)

> IP не принадлежит нашей локальной сети.

### 5. Бэкап.

- Добавляем права на запуск скрипта.

`sudo chmod +x /usr/local/bin/backup_full.sh`

![chmod_+x](/finish_quest/img/5.1.1.png)

- Редактируем скрипт для работы от backup_user.

![backup_script](/finish_quest/img/5.1.2.png)

- Запускаем, проверяем работу.

![script_run](/finish_quest/img/5.1.3.png)

- Добавляем в **cron**, бэкап 1 раз в сутки, в 3:00.

![auto_backup](/finish_quest/img/5.1.4.png)

- Для теста меняем время бэкапа на каждую минуту.

![1m_auto_backup](/finish_quest/img/5.1.4.2.png)

- Проверяем как работает авто-бэкап через **Cron**.

![cron_job](/finish_quest/img/5.1.5.png)


# Все задачи выполнены.