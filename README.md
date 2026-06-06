# web server masking

hide your origin ip behind cloudflare. bots scans every ipv4 for certs matching your domain. this is what you do about it.

```
direct ip → default_server → fake cert → scanner sees "localhost", can't link to you
domain    → server_name     → real cert → user gets your actual site
```

both listen on port 443. nginx/apache decides which cert to serve based on how the visitor connects.

---

## nginx

```bash
# generate fake cert (nothing to do with your domain)
sudo openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/fake.key -out /etc/ssl/fake.crt -days 365 \
  -subj "/C=XX/ST=Unknown/L=Nowhere/O=Fake/CN=localhost"

# /etc/nginx/sites-available/default-server
```

```nginx
# catches direct ip
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    ssl_certificate     /etc/ssl/fake.crt;
    ssl_certificate_key /etc/ssl/fake.key;

    return 444; # close after serving fake cert
}
```

```nginx
# /etc/nginx/sites-available/yourdomain.com - catches real domain traffic

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000; # your app
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/default-server /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## apache

```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/fake.key -out /etc/ssl/fake.crt -days 365 \
  -subj "/C=XX/ST=Unknown/L=Nowhere/O=Fake/CN=localhost"
sudo a2enmod ssl && sudo systemctl restart apache2
```

```apache
# /etc/apache2/sites-available/000-default-server.conf - catches direct ip

<VirtualHost _default_:80>
    ServerName localhost
    <Location />
        Require all denied
    </Location>
    ServerSignature Off
    ServerTokens Prod
</VirtualHost>

<VirtualHost _default_:443>
    ServerName localhost
    SSLEngine on
    SSLCertificateFile /etc/ssl/fake.crt
    SSLCertificateKeyFile /etc/ssl/fake.key
    <Location />
        Require all denied
    </Location>
    ServerSignature Off
    ServerTokens Prod
</VirtualHost>
```

```apache
# /etc/apache2/sites-available/yourdomain.com.conf - catches real domain

<VirtualHost *:443>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem

    DocumentRoot /var/www/yourdomain
</VirtualHost>
```

```bash
sudo a2ensite 000-default-server.conf
sudo a2ensite yourdomain.com.conf
sudo apache2ctl configtest && sudo systemctl reload apache2
```

---

## firewall (block non-cf traffic)

### ufw

```bash
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    sudo ufw allow from "$ip" to any port 80,443 proto tcp
done
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
    sudo ufw allow from "$ip" to any port 80,443 proto tcp
done
sudo ufw default deny incoming
sudo ufw enable
```

### iptables

```bash
sudo iptables -F
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -s "$ip" -j ACCEPT
done
sudo iptables -A INPUT -j DROP
```

---

## 🏆 best: cloudflare tunnel

no public ip = nothing to scan. server makes outbound connection only.

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
cloudflared tunnel login
cloudflared tunnel create my-tunnel
cloudflared tunnel route dns my-tunnel yourdomain.com
cloudflared tunnel run my-tunnel
```

---

## verify

```bash
# direct ip should show fake cert
openssl s_client -connect YOUR_SERVER_IP:443 -servername YOUR_SERVER_IP </dev/null 2>/dev/null | openssl x509 -noout -subject
# expected: CN=localhost

# domain should show real cert
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com </dev/null 2>/dev/null | openssl x509 -noout -subject
# expected: CN=yourdomain.com
```

---

## checklist

```
☐ default_server → fake cert → 444 on direct ip
☐ server_name → real cert for your domain
☐ all dns records proxied (orange cloud)
☐ firewall blocks non-cf ips
☐ no mail server on same ip
☐ server_tokens off / ServerSignature Off
```

> backup before touching config: `cp /etc/nginx/nginx.conf ~/nginx.conf.backup`