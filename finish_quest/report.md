# День 7

## Разворот полного стека LEMP

### 0.1 Подготовка

#### Создаем отказоустойчивое хранилище

###### Добавляем HDD в ВМ

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

    ![lsblk_RAID](/finish_quest/img/0.1.2.png)

- Получаем **UUID** томов для **fstab**.

    `sudo blkid /dev/web_data/logs`

    `sudo blkid /dev/web_data/site_files`

    `sudo blkid /dev/web_data/db_data`

    ![UUID](/finish_quest/img/0.1.3.png)

- Редактируем **fstab**.

    `sudo nano /etc/fstab`

    > Дописываем **UUID** в конец файла.

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

        ![Grants_for_wp_app](/finish_quest/img/1.1.png)

        `SHOW GRANTS FOR 'backup_user'@'localhost';`

        ![Grants_for_backup_user](/finish_quest/img/1.2.png)

    `EXIT;`
    
    > В работе с Mariadb возникли трудности в плане синтаксиса. я не сразу заметил ";" в конце строки, а потом при вводе пароля забыл "`" добавить в конце.. из за чего потерял не мало времени...

### 2. Установка WordPress и phpMyAdmin.

#### 2.1 WordPress (Сайт).

- Идем в корень веб-сервера

`cd /var/www/html` 

> Тут возникла странность, по какой то причине **/html** у меня не окозалось... 
>
> Cоздал дирректорию.
>
> `sudo mkdir html`
>
> Соответсветнно, **index.nginx-debian.html ** не окозалось, пропустил шаг.
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

![Word_Press](/finish_quest/img/2.1.1.png)

#### 2.2 **phpMyAdmin**

> Внимание: Мы ставим его вручную, чтобы не тянуть зависимости **Apache**.

- Скачиваем.

`sudo wget https://files.phpmyadmin.net/phpMyAdmin/5.2.3/phpMyAdmin-5.2.3-all-languages.tar.gz`

`sudo tar -xzvf phpMyAdmin-5.2.3-all-languages.tar.gz`

- Переименовываем для удобства в **pma**.

`sudo mv phpMyAdmin-5.2.3-all-languages pma`

![pma](/finish_quest/img/2.2.0.png)

- Удаляем gz-архив.

`sudo rm phpMyAdmin-5.2.3-all-languages.zip`

- Создаем конфиг из шаблона.

`cp pma/config.sample.inc.php pma/config.inc.php`

- Генерируем **blowfish_secret**.

`openssl rand -base64 32`

- Вставим полученный **blowfish_secret** в конфиг.

`sudo nano pma/config.inc.php`

![config.inc.php_blowfish_secret](/finish_quest/img/2.2.1.png)

