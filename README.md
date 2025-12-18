# DZ_borg
# Устанавливаем на client и backup сервере borgbackup
apt install borgbackup

# На сервере backup
создание пользователя borg:
sudo adduser borg

создание директрории backup и сделать владельцем borg:
mkdir /var/backup
chown borg:borg /var/backup/

настройка аутитентификации по ключу:
su - borg
mkdir /home/borg/.ssh
touch /home/borg/.ssh/authorized_keys
chmod 700 /home/borg/.ssh
chmod 600 /home/borg/.ssh/authorized_keys

# На сервере client

генерация ключа:
sudo ssh-keygen

копирование лкюча на сервер backup:
borg init --encryption=repokey borg@192.168.56.10:/var/backup/

создание бэкапа /etc:
borg create --stats --list borg@192.168.56.10:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc

проверка результата:

borg list borg@192.168.56.10:/var/backup/
etc-2025-12-18_08:08:17              Thu, 2025-12-18 08:08:32 [d5fcef02325df578c843453e7183f203e3fe4ad4d6ca1ae219d04d2d739b8b32]

# Автоматизируем создание бэкапов с помощью systemd
nano /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

#Парольная фраза
Environment="BORG_PASSPHRASE=28978"
#Репозиторий
Environment=REPO=borg@192.168.56.10:/var/backup/
#Что бэкапим
Environment=BACKUP_TARGET=/etc

#Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

#Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

#Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}


#/etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target

#Включаем и запускаем службу таймера
systemctl enable borg-backup.timer 
systemctl start borg-backup.timer

# проверка создания таймера:

vagrant@client:~$ systemctl list-timers | grep borg*
Thu 2025-12-18 09:02:07 UTC 2s left       Thu 2025-12-18 08:57:07 UTC 4min 57s ago borg-backup.timer            borg-backup.service
