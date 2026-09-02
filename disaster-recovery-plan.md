Отказ MediaWiki main-ноды.
При отказе MediaWiki‑ноды сервис остаётся доступен за счёт балансировщика, перенаправляющего трафик на рабочую ноду. Восстановление — через пересоздание ВМ и применение конфигурации без повторной инициализации.
Основные шаги и команды:
    1. Пересоздание ноды в Vagrant:
vagrant destroy mediawiki-1 -f
vagrant up mediawiki-1
    2. Применение Ansible‑конфигурации (без запуска install.php):
ansible-playbook playbook.yml --tags=mediawiki_1 -e mediawiki_run_installer=false
    3. Восстановление LocalSettings.php с рабочей ноды:
    • Копирование файла на временную локацию и смена владельца:
vagrant ssh mediawiki-2 -c "sudo cp /var/www/mediawiki/LocalSettings.php /tmp/LocalSettings.php && sudo chown vagrant:vagrant /tmp/LocalSettings.php"
    • Скачивание файла на хост:
vagrant ssh mediawiki-2 -c "cat /tmp/LocalSettings.php" > /tmp/LocalSettings.php
    • Передача файла на восстанавливаемую ноду:
vagrant ssh mediawiki-1 -c "cat > /tmp/LocalSettings.php" < /tmp/LocalSettings.php
    • Размещение файла и настройка прав:
vagrant ssh mediawiki-1 -c "sudo cp /tmp/LocalSettings.php /var/www/mediawiki/LocalSettings.php && sudo chown www-data:www-data /var/www/mediawiki/LocalSettings.php && sudo chmod 600 /var/www/mediawiki/LocalSettings.php"

   4. Восстановление из бэкапа (альтернативный вариант):
    • Проверка содержимого архива:
vagrant ssh backup-1 -c "tar -tzf /srv/backups/mediawiki/latest.tar.gz | head"
    • Передача архива через base64:
vagrant ssh backup-1 -c "sudo base64 -w0 /srv/backups/mediawiki/latest.tar.gz" | \
vagrant ssh mediawiki-1 -c "base64 -d > /tmp/latest.tar.gz"
    • Проверка архива на целевой ноде:
vagrant ssh mediawiki-1 -c "tar -tzf /tmp/latest.tar.gz | head"
    • Распаковка, настройка ссылок, прав и перезапуск сервисов:
vagrant ssh mediawiki-1 -c "
  sudo rm -rf /var/www/mediawiki /var/www/mediawiki-1.42.1 &&
  sudo tar -xzf /tmp/latest.tar.gz -C /var/www/ &&
  sudo ln -sfn /var/www/mediawiki-1.42.1 /var/www/mediawiki &&
  sudo chown -R www-data:www-data /var/www/mediawiki-1.42.1 &&
  sudo chmod 600 /var/www/mediawiki/LocalSettings.php &&
  sudo systemctl restart nginx php8.1-fpm"


Отказ MediaWiki ноды.
Если одна из нод MediaWiki выходит из строя, сервис не прерывает работу: балансировщик автоматически перенаправляет запросы на исправную ноду.
Способ восстановления зависит от причины сбоя. Если ошибку не удаётся устранить, ноду пересоздают и настраивают заново — для этого применяют ограниченный набор команд:
vagrant destroy mediawiki-2 -f
vagrant up mediawiki-2
ansible-playbook playbook.yml --tags=mediawiki_2

Отказ Nginx балансировщика.
Если балансировщик на базе Nginx выходит из строя, сервис становится недоступен — трафик больше не распределяется между нодами.
Для восстановления работоспособности достаточно пересоздать виртуальную машину балансировщика и применить к ней нужную конфигурацию:
vagrant destroy nginx-lb -f
vagrant up nginx-lb
ansible-playbook playbook.yml --tags=nginx
*Сначала удаляется старая ВМ (флаг -f отключает подтверждение), затем создаётся и запускается новая, после чего Ansible настраивает на ней Nginx согласно роли с тегом nginx.
Отказ Backup/Zabbix сервера
Сервера не влияют на работоспособность сервиса
Для восстановления можно пересоздать ВМ и запустить плейбук
vagrant destroy backup-1 -f
vagrant up backup-1
ansible-playbook playbook.yml --tags=backup
vagrant destroy zabbix-1 -f
vagrant up zabbix-1
ansible-playbook playbook.yml --tags=zabbix
Отказ DB Replica
При отказе реплики сервис будет продолжать работать
Для восстановления в зависимости от ошибки можно пересоздать ВМ и запустить плейбук, запустить плейбук с pg_basebackup с DB Primary или просто перезапустить плейбук
vagrant destroy db-replica -f
vagrant up db-replica
ansible-playbook playbook.yml --tags=db_replica
ansible-playbook playbook.yml --tags=db_replica -e force_reinit_replica=true
vagrant ssh db-replica -c "sudo -u postgres psql -tAc 'SELECT pg_is_in_recovery();'"
vagrant ssh db-primary -c "sudo -u postgres psql -c 'SELECT client_addr, state FROM pg_stat_replication;'"
Отказ DB Primary
При отказе праймари сервис не работает
Для восстановления можно переключить реплику в праймари
vagrant halt db-primary
vagrant ssh db-replica -c "sudo -u postgres pg_ctlcluster 14 main promote"
vagrant ssh db-replica -c "sudo -u postgres psql -tAc 'SELECT pg_is_in_recovery();'"
И поменять DB host в LocalSettings.php
vagrant ssh mediawiki-1 -c "sudo sed -i 's/192.168.56.10/192.168.56.11/' /var/www/mediawiki/LocalSettings.php"
vagrant ssh mediawiki-2 -c "sudo sed -i 's/192.168.56.10/192.168.56.11/' /var/www/mediawiki/LocalSettings.php"
vagrant ssh mediawiki-1 -c "sudo systemctl restart php8.1-fpm nginx"
vagrant ssh mediawiki-2 -c "sudo systemctl restart php8.1-fpm nginx"
vagrant ssh db-replica -c "echo 'host    my_wiki    wikiuser    192.168.56.12/32    md5' | sudo tee -a /etc/postgresql/14/main/pg_hba.conf"
vagrant ssh db-replica -c "echo 'host    my_wiki    wikiuser    192.168.56.13/32    md5' | sudo tee -a /etc/postgresql/14/main/pg_hba.conf"
vagrant ssh db-replica -c "sudo systemctl reload postgresql"
Для восстановления также можно удалить DB Primary и DB replica, создать их снова и запустить плейбук. БД взять из бэкапа
Проверка работоспособности бэкапа:
Добавим тестовых страниц в MediaWiki
vagrant ssh db-primary -c "sudo -u postgres psql -d my_wiki -c 'SELECT COUNT(*) FROM mediawiki.page;'"
vagrant ssh backup-1 -c "sudo /opt/backup/backup_db.sh"
vagrant ssh backup-1 -c "sudo base64 -w0 /srv/backups/postgresql/latest.dump" | \
vagrant ssh db-primary -c "base64 -d > /tmp/latest_from_backup.dump"
vagrant ssh db-primary -c "sudo -u postgres dropdb my_wiki"
vagrant ssh db-primary -c "
sudo -u postgres createdb -O wikiuser my_wiki &&
sudo -u postgres pg_restore -d my_wiki /tmp/latest_from_backup.dump
