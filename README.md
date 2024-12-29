# Настройка Xray-core с VLESS и Reality на AlmaLinux

Руководство по установке и настройке Xray-core с использованием протоколов VLESS и Reality на серверах AlmaLinux или RHEL подобных.

---

## 1. **Обновите сервер**
Убедитесь, что все пакеты обновлены:
```bash
sudo dnf update -y
```

---

## 2. **Установите необходимые зависимости**
Установите базовые утилиты:
```bash
sudo dnf install -y curl wget vim unzip openssl
```

---

## 3. **Скачайте и установите Xray-core**
1. Скачайте последнюю версию Xray-core с официального репозитория:
   ```bash
   curl -L https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip -o Xray.zip
   ```
2. Распакуйте архив:
   ```bash
   unzip Xray.zip -d /usr/local/bin/
   ```
3. Сделайте бинарный файл исполняемым:
   ```bash
   chmod +x /usr/local/bin/xray
   ```

---

## 4. **Генерация UUID, ShortID и приватного ключа**
1. Сгенерируйте UUID:
   ```bash
   cat /proc/sys/kernel/random/uuid
   ```
2. Сгенерируйте ключи для Reality:
   ```bash
   xray x25519
   ```
**Пример вывода:**
   ```
   Private key: sK9VV_qkrI5Aex7YCsFt9lgddBVK9DIUyzN9R9PYIF9
   Public key: C2jbG73fEkDX190hplzPkDmUZ0vJYowxzZoLu-ylgD9
   ```
3. Создайте ShortID:
ShortID — это произвольная строка, которая используется для идентификации клиента. Вы можете создать ShortID, используя случайные символы. Например, вы можете использовать команду `openssl` для генерации случайной строки:

    ```bash
    openssl rand -base64 8 | tr -d '=' | tr -d '/' | tr -d '+' | cut -c1-14
    ```

    Эта команда создаст случайную строку длиной 14 символов, которая может быть использована в качестве ShortID. Вы также можете использовать любой другой метод генерации случайных строк, который вам удобен.

С помощью командной строки:
```bash
cat /dev/urandom | tr -dc 'a-f0-9' | head -c 14
```

Вы можете указать несколько Short ID, если хотите поддерживать разные идентификаторы для разных клиентов:

{
  "shortIds": ["1a2b3c4d", "e4f2a1b3"]
}

   **Используйте сгенерированные ключи в конфигурации.**
---

### 5. **Создайте конфигурацию для VLESS и Reality**
1. Создайте директорию для конфигурации:
   ```bash
   mkdir -p /usr/local/etc/xray
   ```
2. Создайте файл конфигурации:
   ```bash
   vim /usr/local/etc/xray/config.json
   ```

Пример конфигурации для VLESS + Reality:
```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "vless-reality",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "ВАШ-UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none",
        "fallbacks": [
          {
            "dest": 80
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.google.com:443",
          "serverNames": ["www.google.com"],
          "privateKey": "ВАШ-ПРИВАТНЫЙ-КЛЮЧ",
          "shortIds": ["ВАШ-SHORTID"],
          "spiderX": "/"
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

**Объяснение ключевых параметров:**
- `id`: Уникальный UUID для клиента.
- `privateKey`: Ваш приватный ключ, сгенерированный ранее.
- `shortIds`: Список Short ID для идентификации клиентов.
- `fallbacks`: Перенаправление трафика, если запрос не относится к VLESS.

---

### 6. **Настройте автозапуск Xray**
1. Создайте systemd-сервис:
   ```bash
   vim /etc/systemd/system/xray.service
   ```
2. Вставьте следующий текст:
   ```ini
   [Unit]
   Description=Xray Service
   After=network.target

   [Service]
   User=root
   ExecStart=/usr/local/bin/xray -config /usr/local/etc/xray/config.json
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```
3. Активируйте и запустите сервис:
   ```bash
   sudo systemctl enable xray
   sudo systemctl start xray
   ```

4. Проверьте статус сервиса:
   ```bash
   sudo systemctl status xray
   ```

---

### 7. **Откройте порт на сервере**
Откройте порт 443 в вашем файерволе:
```bash
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

---

### 8. **Тестируйте подключение**
- Проверка логов:
  ```bash
  journalctl -u xray -f
  ```

Для подключения используйте ваш клиент VLESS, указав `UUID`, серверный адрес и порт 443.

---
Поздравляю! Вы успешно настроили Xray-core с VLESS и Reality. Теперь ваш сервер готов к безопасному соединению. 🎉