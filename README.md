# 🌐 WebDAV Server on Termux (HTTPS + Multi-User + Auto-Restart)

Host a full **WebDAV server directly on Android (Termux)** using **Nginx 1.29.2** with WebDAV + DAV-Ext modules.  
Supports:
- 🔒 HTTPS (self-signed certificate)
- 👥 Multi-user authentication
- ♻️ Auto-restart via `termux-job-scheduler`
- 💾 Storage-root `/sdcard/WebDAV`

---

## ⚙️ Requirements

| Item | Notes |
|------|-------|
| **Termux** | Latest version (F-Droid recommended) |
| **Storage access** | Run `termux-setup-storage` |
| **OpenSSL** | Used to generate TLS certificates |
| **Custom Nginx build** | Must include `ngx_http_dav_ext_module` |

---

## 🧩 Directory Layout (Ubuntu-Style)
```bash
PREFIX=/data/data/com.termux/files/usr
ETC=$PREFIX/etc/nginx
VAR=$PREFIX/var
RUN=$VAR/run
LOG=$VAR/log/nginx
LIB=$VAR/lib/nginx

mkdir -p $ETC/{sites-available,sites-enabled,conf.d,snippets,modules-available,modules-enabled,auth,ssl}
mkdir -p $LOG $RUN $LIB/{client_body_temp,proxy_temp,fastcgi_temp,uwsgi_temp,scgi_temp}
```

---

## 📦 Install Binary
```bash
cp ~/nginx_full_web_dav_termux_arm64/bin/nginx $PREFIX/bin/
chmod +x $PREFIX/bin/nginx
nginx -V
```

---

## 📁 Enable Storage
```bash
termux-setup-storage
mkdir -p ~/storage/shared/WebDAV
```

---

## 🔐 Create Auth User
```bash
read -p "Username: " USER && read -s -p "Password: " PW && echo && HASH=$(openssl passwd -apr1 "$PW") && echo "$USER:$HASH" > $ETC/auth/dav.htpasswd
```

---

## 🧱 Load DAV-EXT Module
```bash
cat > $ETC/modules-available/dav_ext.conf <<EOF
load_module $HOME/nginx_full_web_dav_termux_arm64/modules/ngx_http_dav_ext_module.so;
EOF
ln -sf $ETC/modules-available/dav_ext.conf $ETC/modules-enabled/dav_ext.conf
```

---

## ⚙️ Main Config (`nginx.conf`)
```nginx
include /data/data/com.termux/files/usr/etc/nginx/modules-enabled/*.conf;

worker_processes  1;
error_log  /data/data/com.termux/files/usr/var/log/nginx/error.log info;
pid        /data/data/com.termux/files/usr/var/run/nginx.pid;

events { worker_connections 1024; }

http {
    include       /data/data/com.termux/files/usr/etc/nginx/mime.types;
    default_type  application/octet-stream;
    charset       utf-8;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" "$http_user_agent"';
    access_log  /data/data/com.termux/files/usr/var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 20g;
    client_body_temp_path /data/data/com.termux/files/usr/var/lib/nginx/client_body_temp 1 2;

    include /data/data/com.termux/files/usr/etc/nginx/conf.d/*.conf;
    include /data/data/com.termux/files/usr/etc/nginx/sites-enabled/*.conf;
}
```

---

## 📜 MIME Types
```nginx
types {
  text/html html htm shtml;
  text/css  css;
  image/png png;
  image/jpeg jpg jpeg;
  video/mp4 mp4;
  audio/mpeg mp3;
  application/javascript js;
  application/json json;
  application/pdf pdf;
  application/zip zip;
}
```

---

## 🔒 Self-Signed TLS Certificate
```bash
SSL=$ETC/ssl
openssl req -x509 -nodes -newkey rsa:2048   -keyout $SSL/webdav.key -out $SSL/webdav.crt   -days 3650 -subj "/CN=localhost"
```

---

## 🌐 HTTPS WebDAV Site
**`/etc/nginx/sites-available/webdav.conf`**
```nginx
server {
    listen 1617 ssl;
    server_name _;

    ssl_certificate     /data/data/com.termux/files/usr/etc/nginx/ssl/webdav.crt;
    ssl_certificate_key /data/data/com.termux/files/usr/etc/nginx/ssl/webdav.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers   HIGH:!aNULL:!MD5;

    auth_basic "Restricted WebDAV";
    auth_basic_user_file /data/data/com.termux/files/usr/etc/nginx/auth/dav.htpasswd;

    add_header Access-Control-Allow-Origin * always;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS, PROPFIND, MKCOL, COPY, MOVE, DELETE, PUT" always;
    add_header Access-Control-Allow-Headers "Authorization, Depth, Content-Type, Destination, Overwrite" always;
    if ($request_method = OPTIONS) { return 204; }

    root /data/data/com.termux/files/home/storage/shared/WebDAV;

    dav_methods PUT DELETE MKCOL COPY MOVE;
    create_full_put_path on;
    dav_access user:rw group:rw all:rw;
    dav_ext_methods PROPFIND OPTIONS;

    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;

    location / {
        types { }
        default_type application/octet-stream;
    }
}
```

Enable it:
```bash
ln -s $ETC/sites-available/webdav.conf $ETC/sites-enabled/webdav.conf
```

---

## 🔁 Redirect HTTP → HTTPS (Optional)
```bash
cat > $ETC/sites-available/webdav-redirect.conf <<'EOF'
server {
    listen 1616;
    return 301 https://$host:1617$request_uri;
}
EOF
ln -s $ETC/sites-available/webdav-redirect.conf $ETC/sites-enabled/webdav-redirect.conf
```

---

## ▶️ Run / Reload / Stop
```bash
nginx -c /data/data/com.termux/files/usr/etc/nginx/nginx.conf -t
nginx -c /data/data/com.termux/files/usr/etc/nginx/nginx.conf
nginx -c /data/data/com.termux/files/usr/etc/nginx/nginx.conf -s reload
nginx -c /data/data/com.termux/files/usr/etc/nginx/nginx.conf -s stop
```

---

## 🧩 Add New User (Shared Access)
To add a new login that shares the same WebDAV root:
```bash
ETC=/data/data/com.termux/files/usr/etc/nginx
read -p "New username: " USER && read -s -p "Password: " PW && echo && HASH=$(openssl passwd -apr1 "$PW") && echo "$USER:$HASH" >> $ETC/auth/dav.htpasswd
```
✅ Adds directly to `dav.htpasswd`  
✅ No config edit or reload needed  
✅ Immediate access to `/sdcard/WebDAV`

---

## ⚡ Keep Alive & Auto-Start
```bash
termux-wake-lock

cat > /data/data/com.termux/files/usr/bin/start-nginx.sh <<'EOF'
#!/data/data/com.termux/files/usr/bin/bash
nginx -c /data/data/com.termux/files/usr/etc/nginx/nginx.conf
EOF
chmod +x /data/data/com.termux/files/usr/bin/start-nginx.sh

termux-job-scheduler --job-id 1 --period-ms 600000   --script /data/data/com.termux/files/usr/bin/start-nginx.sh
```

---

## 🧾 Logs & Troubleshooting
| Path | Purpose |
|------|----------|
| `/var/log/nginx/error.log` | Error logs |
| `/var/log/nginx/access.log` | Request logs |
| `/etc/nginx/auth/*.htpasswd` | User credentials |
| `/etc/nginx/ssl/` | TLS certs |

```bash
tail -f /data/data/com.termux/files/usr/var/log/nginx/error.log
```

---

## ✅ Quick Status
| Check | Command | Expected |
|--------|----------|-----------|
| Config | `nginx -t` | OK |
| Port | `nc -vz 127.0.0.1 1617` | open |
| Local | `curl -k https://127.0.0.1:1617/` | XML |
| Remote | `curl -k https://<IP>:1617/` | XML |

---

## 🎉 Done
Your Android phone now runs a **secure, multi-user WebDAV server**:
- 🔐 HTTPS secured  
- 👥 Multi-user ready  
- ♻️ Auto-restarting  
- 💾 Rooted on `/sdcard/WebDAV`

---
