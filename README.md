# systemd-slack-notifications

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Linux](https://img.shields.io/badge/platform-linux-blue)
![Slack](https://img.shields.io/badge/slack-notifications-4A154B?logo=slack&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Send Slack notifications via Webhooks whenever your Linux server OR MySQL service stops, starts, or crashes unexpectedly — using `systemd` hooks.

---

## 🚀 Features

- Slack notifications for:
  - Server shutdown
  - Server startup
  - MySQL service stop
  - MySQL service start
  - MySQL crash / unexpected failure
- Easy to install on any Ubuntu / Debian / Linux server
- No external application required — uses systemd + curl
- Lightweight & production safe
- Color-coded Slack messages with icons and timestamps

---

## 📦 Requirements

- Linux system with systemd  
- Slack Incoming Webhook URL  
- curl installed

---

## 🔧 Installation

### 1. Save Your Slack Webhook URL

Create a secure file to store your webhook:

```bash
echo "https://hooks.slack.com/services/XXXX/YYYY/ZZZZ" | sudo tee /etc/slack_webhook_url
sudo chmod 600 /etc/slack_webhook_url
```

---

### 2. Universal Slack Notification Script

All hooks will call this script.

Create:

```bash
sudo nano /usr/local/bin/slack-notify.sh
```

Paste:

```bash
#!/bin/bash

WEBHOOK_URL=$(cat /etc/slack_webhook_url)

TITLE="$1"
MESSAGE="$2"
EVENT_TYPE="$3"  # start, stop, crash, server_start, server_shutdown

case "$EVENT_TYPE" in
  mysql_start)
    ICON=":large_green_circle:"
    COLOR="#36a64f"
    ;;
  mysql_stop)
    ICON=":red_circle:"
    COLOR="#dc3545"
    ;;
  mysql_crash)
    ICON=":fire:"
    COLOR="#ff0000"
    ;;
  server_start)
    ICON=":rocket:"
    COLOR="#4a90e2"
    ;;
  server_shutdown)
    ICON=":warning:"
    COLOR="#ffae42"
    ;;
  *)
    ICON=":information_source:"
    COLOR="#dddddd"
    ;;
esac

HOSTNAME=$(hostname)
TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")

payload=$(cat <<EOF
{
  "attachments": [
    {
      "color": "$COLOR",
      "blocks": [
        {
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": "$ICON *$TITLE*\n$MESSAGE"
          }
        },
        {
          "type": "context",
          "elements": [
            {
              "type": "mrkdwn",
              "text": "*Server:* $HOSTNAME"
            },
            {
              "type": "mrkdwn",
              "text": "*Event:* $EVENT_TYPE"
            },
            {
              "type": "mrkdwn",
              "text": "*Time:* $TIMESTAMP"
            }
          ]
        }
      ]
    }
  ]
}
EOF
)

curl -X POST -H 'Content-type: application/json' --data "$payload" "$WEBHOOK_URL" >/dev/null 2>&1
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/slack-notify.sh
```

---

## 🛠 MySQL Notifications

### 3. MySQL Start Hook

```bash
sudo nano /usr/local/bin/mysql-start-hook.sh
```

```bash
#!/bin/bash
/usr/local/bin/slack-notify.sh \
  "MySQL Started" \
  "The MySQL service has successfully started." \
  "mysql_start"
```

```bash
sudo chmod +x /usr/local/bin/mysql-start-hook.sh
```

---

### 4. MySQL Stop Hook

```bash
sudo nano /usr/local/bin/mysql-stop-hook.sh
```

```bash
#!/bin/bash
/usr/local/bin/slack-notify.sh \
  "MySQL Stopped" \
  "The MySQL service has been stopped." \
  "mysql_stop"
```

```bash
sudo chmod +x /usr/local/bin/mysql-stop-hook.sh
```

---

### 5. MySQL Crash Hook

```bash
sudo nano /usr/local/bin/mysql-crash-detect.sh
```

```bash
#!/bin/bash
/usr/local/bin/slack-notify.sh \
  "MySQL Crash Detected" \
  "MySQL crashed and systemd is attempting to restart it." \
  "mysql_crash"
```

```bash
sudo chmod +x /usr/local/bin/mysql-crash-detect.sh
```

---

### Edit MySQL service to add hooks

Create override:

```bash
sudo mkdir -p /etc/systemd/system/mariadb.service.d
sudo nano /etc/systemd/system/mariadb.service.d/override.conf
```

Add:

```ini
[Service]
ExecStartPost=/usr/local/bin/mysql-start-hook.sh
ExecStopPost=/usr/local/bin/mysql-stop-hook.sh
ExecStartPre=/usr/local/bin/mysql-crash-detect.sh
```

To verify override was saved:

```bash
sudo systemctl cat mariadb
```

You should see this at the bottom:

```ini
# /etc/systemd/system/mariadb.service.d/override.conf
[Service]
ExecStartPost=/usr/local/bin/mysql-start-hook.sh
ExecStopPost=/usr/local/bin/mysql-stop-hook.sh
ExecStartPre=/usr/local/bin/mysql-crash-detect.sh
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

---

## 🖥 Server Notifications

### 6. Server Shutdown Notification

```bash
sudo nano /lib/systemd/system-shutdown/slack-shutdown.sh
```

```bash
#!/bin/bash
/usr/local/bin/slack-notify.sh \
  "Server Shutdown" \
  "The server is shutting down. Reason: $1" \
  "server_shutdown"
```

Make executable:

```bash
sudo chmod +x /lib/systemd/system-shutdown/slack-shutdown.sh
```

---

### 7. Server Boot Notification

Create systemd service:

```bash
sudo nano /etc/systemd/system/server-start-hook.service
```

```ini
[Unit]
Description=Slack notification when server boots
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/server-start-hook.sh

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl enable server-start-hook.service
```

---

### Boot script

```bash
sudo nano /usr/local/bin/server-start-hook.sh
```

```bash
#!/bin/bash
/usr/local/bin/slack-notify.sh \
  "Server Booted" \
  "The server has started successfully." \
  "server_start"
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/server-start-hook.sh
```

---

## 🧪 Test Notifications

```bash
/usr/local/bin/slack-notify.sh "Test Message" "This is a test." "test_event"
sudo systemctl restart mysql
sudo reboot
```

---

## 🖼 Example Slack Notifications

### MySQL Crash
```
🔥 MySQL Crash Detected
MySQL crashed and systemd is attempting to restart it.

Server: prod-db-01
Event: mysql_crash
Time: 2025-02-19 14:12:05
```

### Server Boot
```
🚀 Server Booted
The server has started successfully.

Server: app-server-01
Event: server_start
Time: 2025-02-19 14:12:05
```

### MySQL Stop
```
🔴 MySQL Stopped
The MySQL service has been stopped.

Server: prod-db-01
Event: mysql_stop
Time: 2025-02-19 14:12:05
```

---

## 📄 License

MIT

---

## 👨‍💻 Author

Fabmedia — DevOps & Laravel Engineering Team
