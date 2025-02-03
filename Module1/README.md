# Презентация: Настройка Linux и Администрирование

## Слайд 1: Введение
**Тема:** Базовая настройка и администрирование Linux
**Цель:** Ознакомление с основами установки, настройки сети, SSH и автоматизации
**Аудитория:** Начинающие системные администраторы и пользователи Linux

---

## Слайд 2: Установка Linux в виртуальную среду
- Используем **VirtualBox** или **VMware**
- Выбираем подходящий дистрибутив (например, Ubuntu, Debian, AlmaLinux)
- Настройка ресурсов (CPU, RAM, диск)
- Разделение диска (LVM или стандартные разделы)
- Локализация и часовой пояс

---

## Слайд 3: Настройка сетевых интерфейсов
### Одноразовая настройка
- `ifconfig` (устаревшее) → используем `ip addr`
- Настройка маршрутизации: `route` или `ip route`
- DNS: редактируем `/etc/resolv.conf`

### Статическая настройка через файлы
- `nmcli` (NetworkManager CLI)
- `/etc/network/interfaces` (Debian-based)
- **Netplan** (Ubuntu 18.04+)
```yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
  version: 2
```

---

## Слайд 4: Изменение имени хоста
- `hostnamectl set-hostname new-hostname`
- Временная смена: `hostname new-hostname`
- `/etc/hostname` – постоянное имя хоста
- `/etc/hosts` – локальное разрешение DNS
```bash
127.0.0.1 localhost
127.0.1.1 new-hostname
```

---

## Слайд 5: Установка и настройка OpenSSH
- Установка: `sudo apt-get install openssh-server`
- Проверка работы: `systemctl status ssh`
- Открытые порты: `ss -tlnp`
- Генерация ключей:
```bash
ssh-keygen -t rsa -b 4096
```
- Копирование ключа на сервер:
```bash
ssh-copy-id user@server
```

---

## Слайд 6: Конфигурация OpenSSH
- Файл `/etc/ssh/sshd_config`
- Отключение паролей:
```bash
PasswordAuthentication no
```
- Ограничение пользователей:
```bash
AllowUsers admin user1
```
- Перезапуск SSH: `sudo systemctl restart ssh`

---

## Слайд 7: Клонирование системы
- Полное клонирование диска:
```bash
sudo dd if=/dev/sda of=/dev/sdb bs=4M
```
- Копирование файлов:
```bash
rsync -av /home/ user@remote:/backup/
```
- Архивация:
```bash
tar czf backup.tar.gz /home/
```

---

## Слайд 8: Автоматизация обновлений и ротация логов
- Автоматическое обновление:
```bash
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```
- Ротация логов с `logrotate`
- Конфигурация `/etc/logrotate.d/custom-log`:
```bash
/var/log/custom.log {
  weekly
  rotate 4
  compress
  missingok
  notifempty
}
```

---

## Слайд 9: Заключение
- Настроили Linux-сервер
- Обеспечили доступ через SSH
- Автоматизировали обновления и управление логами
- Следующие шаги: изучение **firewall**, **мониторинга**, **Docker/Kubernetes**

**Спасибо за внимание!**


