# Инструкция по установке и настройке почтового сервера с помощью iRedMail на Debian 12 с MySQL

## Введение

В этом руководстве описаны шаги по развертыванию собственного почтового сервера на **Debian** 12 с использованием **iRedMail** и базы данных **MySQL**. Все действия выполняются с правами root или через sudo.

- **Debian** — популярное семейство дистрибутивов Linux.
- **iRedMail** — комплексное ПО для развёртывания почтового сервера с веб-интерфейсом, антиспамом, антивирусом и поддержкой управления пользователями.
- **MySQL** — одна из самых распространённых систем управления базами данных (СУБД).

---

## 1. Подготовка сервера

### 1.1. Обновление системы

```sh
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 1.2. Настройка hostname и /etc/hosts

Допустим, ваш домен: mail.example.com.

```sh
sudo hostnamectl set-hostname mail.example.com
sudo nano /etc/hosts
```

Добавьте строку, если её нет:

```
127.0.0.1   mail.example.com mail localhost
```

### 1.3. Отключение **exim4**

- **exim4** — это стандартный почтовый сервер (**MTA**, Mail Transfer Agent) в Debian, который по умолчанию обрабатывает почту на сервере. Его нужно отключить, чтобы не было конфликта с новым почтовым сервером.

```sh
sudo systemctl stop exim4
sudo systemctl disable exim4
sudo apt remove --purge exim4 exim4-base exim4-config exim4-daemon-light -y
```

---

## 2. Открытие необходимых портов

Если используется **UFW** (Uncomplicated Firewall):

- **UFW** — это удобный инструмент для управления встроенным в Linux файрволом (iptables).

```sh
sudo ufw allow 22,25,80,443,587,110,995,143,993,465/tcp
sudo ufw reload
sudo ufw status
```

---

## 3. Установка необходимых зависимостей

```sh
sudo apt install curl wget nano lsb-release ca-certificates -y
```
- **curl**/**wget** — утилиты для загрузки файлов по сети.
- **nano** — текстовый редактор в терминале.
- **lsb-release** — утилита для определения версии ОС.
- **ca-certificates** — набор корневых сертификатов для работы с HTTPS.

---

## 4. Скачивание и подготовка **iRedMail**

```sh
cd /opt
wget https://github.com/iredmail/iRedMail/archive/refs/tags/1.7.0.tar.gz
tar xzf 1.7.0.tar.gz
cd iRedMail-1.7.0
```
*(Проверьте актуальную версию на https://github.com/iredmail/iRedMail/releases)*

---

## 5. Запуск установки **iRedMail**

```sh
sudo bash iRedMail.sh
```

### 5.1. Ответы на вопросы установщика

- Directory for mail storage: обычно по умолчанию /var/vmail (можете изменить).
- Веб-сервер: **Nginx** или **Apache** (выберите по желанию).
  - **Nginx**/**Apache** — популярные веб-сервера для обслуживания веб-интерфейса.
- Mail database backend: выбираем **MySQL**.
- **MySQL root password**: введите пароль root MySQL или создайте новый.
- Mail domain name: укажите ваш домен, например example.com.
- Mail admin password: придумайте сложный пароль.
- Дополнительные опции: выберите нужные (антиспам, антивирус, веб-интерфейс и т.д.).
  - Антиспам и антивирус реализованы с помощью **Amavisd**, **ClamAV**, **SpamAssassin**:
    - **Amavisd** — фильтрация и обработка почты.
    - **ClamAV** — антивирус.
    - **SpamAssassin** — фильтрация спама.

---

## 6. Завершение установки

- После установки откроется окно с итоговой информацией: пароли, локации конфигов и т.д.
- Скопируйте эти данные в безопасное место.
- Разрешите перезагрузку системы или перезапустите службы по требованию установщика.

---

## 7. **DNS**-записи для домена

- **DNS** (Domain Name System) — система доменных имён, которая переводит доменные имена в IP-адреса.

### Обязательные записи:

- **MX** (Mail Exchange): mail.example.com с приоритетом 10
  - Указывает, какой сервер получает почту для домена.
- **A** (Address): mail.example.com → IP вашего сервера
  - Связывает имя с IP-адресом.
- **SPF** (Sender Policy Framework, тип TXT):
  ```txt
  v=spf1 mx a ip4:ВАШ_IP -all
  ```
  - Технология для борьбы с подделкой отправителя.
- **DKIM** (DomainKeys Identified Mail):
  - Ключ будет сгенерирован после установки, инструкцию найдете в /var/log/iredmail/install.log или через **iRedAdmin**.
  - Технология электронной подписи исходящих писем, защищает от подделки и изменений.
- **DMARC** (Domain-based Message Authentication, Reporting & Conformance, тип TXT):
  ```txt
  _dmarc.example.com  IN  TXT  "v=DMARC1; p=none; rua=mailto:postmaster@example.com"
  ```
  - Политика для обработки писем, основанная на результатах SPF и DKIM, и отчёты о доставке.

---

## 8. Проверка и запуск сервисов

```sh
sudo systemctl status postfix dovecot mariadb nginx
```
# или
```sh
sudo systemctl status postfix dovecot mysql apache2
```

- **systemctl** — инструмент управления сервисами (службами) в Linux.
- **Postfix** — почтовый сервер (MTA, Mail Transfer Agent), обрабатывающий отправку и приём писем.
- **Dovecot** — сервер почтовых протоколов **IMAP**/**POP3**:
  - **IMAP** (Internet Message Access Protocol) — доступ к почте на сервере.
  - **POP3** (Post Office Protocol 3) — загрузка почты с сервера.
- **MariaDB**/**MySQL** — сервер СУБД.
- **Nginx**/**Apache** — веб-серверы.

---

## 9. Доступ к веб-интерфейсу

- **iRedAdmin**:  
  https://mail.example.com/iredadmin  
  (логин: postmaster@example.com, пароль вы выбрали при установке)
  - **iRedAdmin** — административный веб-интерфейс для управления почтовым сервером.
- **Roundcube** (веб-почта):  
  https://mail.example.com/mail
  - **Roundcube** — современный веб-клиент для работы с электронной почтой через браузер.

---

## 10. Управление пользователями

Добавлять почтовых пользователей и домены можно через **iRedAdmin**.

---

## 11. Дополнительные рекомендации

- Включите двуфакторную аутентификацию (**2FA**, Two-Factor Authentication) для админских аккаунтов.
- Настройте резервное копирование **MySQL** и почтовых данных.
- Обновляйте систему и **iRedMail** своевременно.

---

## 12. Полезные ссылки

- Официальная документация iRedMail: https://docs.iredmail.org/
- Часто задаваемые вопросы: https://forum.iredmail.org/forum6.html

---

## Готово!

Ваш почтовый сервер работает. Проверьте доставку и отправку писем с помощью внешнего почтового ящика.

---

**Не забудьте:**  
- Внести сервер в белые списки крупных почтовиков (Google, Yandex, Mail.ru), если потребуется.  
- Следите за логами: /var/log/maillog, /var/log/mail.err, /var/log/iredapd/iredapd.log.
  - **iredapd** — служба политики почты (policy daemon), используемая в iRedMail.

---

## Дополнение: интеграция с уже существующей БД **Roundcube**

- **БД** — база данных.
- **Roundcube** — веб-клиент для почты, использующий отдельную базу для хранения пользовательских настроек, папок, адресной книги.

### 1. Подготовка к установке

#### 1.1. Бэкап текущей БД

Перед началом установки обязательно сделайте дамп вашей базы данных Roundcube:

```sh
mysqldump -u roundcube_user -p roundcube_db > ~/roundcube_backup.sql
```
- **mysqldump** — стандартная утилита для резервного копирования MySQL.

---

### 2. Установка **iRedMail**

#### 2.1. На этапе выбора **MySQL**

- Укажите данные для подключения к вашему существующему серверу MySQL (host, user, пароль root).

#### 2.2. На этапе выбора компонентов

- НЕ выбирайте установку **Roundcube**, если установщик предлагает такой пункт (обычно можно снять галочку).
- Если такой опции нет, после установки выполните шаги по удалению лишних новых таблиц (см. ниже).

---

### 3. После установки iRedMail

#### 3.1. Проверка БД

Ваша существующая база данных **Roundcube** должна остаться нетронутой. Однако, если iRedMail создал новую базу **roundcubemail** или изменил существующую, сравните дамп с новым состоянием:

```sh
mysqldump -u root -p roundcubemail > ~/iredmail_roundcube_postinstall.sql
diff ~/roundcube_backup.sql ~/iredmail_roundcube_postinstall.sql
```

Если структура или данные изменились, восстановите из бэкапа:

```sh
mysql -u root -p roundcubemail < ~/roundcube_backup.sql
```

---

#### 3.2. Настройка подключения iRedMail к вашей БД **Roundcube**

- Откройте файл конфигурации Roundcube (например, /opt/www/roundcubemail/config/config.inc.php или /var/www/roundcube/config/config.inc.php):

```sh
nano /var/www/roundcube/config/config.inc.php
```

- Убедитесь, что параметры подключения к MySQL соответствуют вашей существующей базе и пользователю:

```php
$config['db_dsnw'] = 'mysql://roundcube_user:roundcube_password@localhost/roundcube_db';
```

---

#### 3.3. Проверьте права пользователя БД

Пользователь MySQL, используемый для Roundcube, должен иметь права на SELECT/INSERT/UPDATE/DELETE в вашей базе данных.

```sql
GRANT ALL PRIVILEGES ON roundcube_db.* TO 'roundcube_user'@'localhost' IDENTIFIED BY 'PASSWORD';
FLUSH PRIVILEGES;
```

---

#### 3.4. (Если нужно) Обновление схемы **Roundcube**

Если версии Roundcube различаются (ваша старая и установленная iRedMail), рекомендуется выполнить обновление схемы:

```sh
cd /var/www/roundcube
php bin/updatedb.sh
```

---

#### 3.5. Интеграция пользователей

**Roundcube** — это только веб-клиент. Основные почтовые ящики и домены будут храниться в базе iRedMail (например, в таблицах vmail).

Пользователи Roundcube и почтовые ящики в iRedMail — это разные сущности.

Почтовые ящики должны совпадать с теми, что были ранее, чтобы пользователи могли войти в Roundcube с теми же логинами и паролями.

---

### 4. Итоговые проверки

- Попробуйте войти в Roundcube под существующим пользователем.
- Проверьте отправку и получение писем.
- Проверьте, что письма, папки и настройки пользователей сохранены.

---

### 5. Особенности

- iRedMail не перезаписывает существующую базу Roundcube, если вы укажете другую БД для установки или снимете галочку с установки Roundcube.
- При обновлениях iRedMail следите за совместимостью версий Roundcube.
- Если что-то пошло не так — восстановите базу из бэкапа.

---

## Пример пути конфигурации после установки iRedMail:

- **Roundcube**: /var/www/roundcube/
- Конфигурация: /var/www/roundcube/config/config.inc.php
- База: та, что была у вас (roundcube_db)

---

## Важно

- Не используйте одну и ту же БД для двух разных установок Roundcube (старой и новой).
- Не переименовывайте таблицы и не смешивайте схему iRedMail и старую схему Roundcube вручную.

---

### Если вы используете **Docker** или отдельные контейнеры, путь к файлам и доступ к БД могут отличаться.

- **Docker** — популярная технология контейнеризации приложений.

---

## Готово!

Теперь ваш сервер работает с **iRedMail** и использует уже существующую базу данных **Roundcube** — без потери данных и настроек пользователей.

---
