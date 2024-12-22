# Настройка Xray-core с VLESS и Reality на AlmaLinux

В этом руководстве мы рассмотрим, как установить и настроить Xray-core с VLESS и Reality на серверах AlmaLinux и RHEL.

---

### 1. **Обновите сервер**
Убедитесь, что все пакеты обновлены:
```bash
sudo dnf update -y
```

---

### 2. **Установите необходимые зависимости**
Установите базовые утилиты и curl:
```bash
sudo dnf install -y curl wget vim unzip
```

---

### 3. **Скачайте и установите Xray-core**
1. Скачайте последнюю версию Xray-core:
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

### 4. **Создайте UUID, ShortID и приватный ключ**
1. Сгенерируйте UUID:
   ```bash
   cat /proc/sys/kernel/random/uuid
   ```
2. Сгенерируйте ключи для Reality:
   ```bash
   xray x25519
   ```
   Пример вывода:
   ```
   Private key: sK9VV_qkrI5Aex7YCsFt9lgddBVK9DIUyzN9R9PYIF9
   Public key: C2jbG73fEkDX190hplzPkDmUZ0vJYowxzZoLu-ylgD9
   ```
3. ShortID — это произвольная строка, которая используется для идентификации клиента. Вы можете создать ShortID, используя случайные символы. Например, вы можете использовать команду `openssl` для генерации случайной строки:

    ```bash
    openssl rand -base64 8 | tr -d '=' | tr -d '/' | tr -d '+' | cut -c1-8
    ```

    Эта команда создаст случайную строку длиной 8 символов, которая может быть использована в качестве ShortID. Вы также можете использовать любой другой метод генерации случайных строк, который вам удобен.

   Используйте сгенерированные ключи в конфигурации.

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
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "ВАШ-UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.google.com:443",
          "serverNames": ["www.google.com"],
          "privateKey": "ВАШ-ПРИВАТНЫЙ-КЛЮЧ",
          "shortIds": ["ВАШ-SHORTID"]
        }
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

---

### 6. **Настройте автозапуск Xray**
1. Создайте systemd-сервис:
   ```bash
   vim /etc/systemd/system/xray.service
   ```
   Вставьте следующий текст:
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
2. Активируйте и запустите сервис:
   ```bash
   sudo systemctl enable xray
   sudo systemctl start xray
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
Проверьте статус сервиса:
```bash
sudo systemctl status xray
```

Для подключения используйте ваш клиент VLESS, указав `UUID`, серверный адрес и порт 443.

---
