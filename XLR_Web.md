# XLR.cz — Projektová dokumentace

## Aktuální stav

Web `xlr.cz` běží jako černá stránka s Matomo trackingem.
GitHub repozitář: https://github.com/xray3131/www

---

## Infrastruktura

### Síť
```
Návštěvník
    ↓
Cloudflare (proxy, HTTPS, CDN)
    ↓
MikroTik router — veřejná IP: 188.75.181.136
    ↓
Ubuntu server — lokální IP: 10.0.0.24
    ├── Caddy (webový server, port 80)
    │     └── /opt/matomo  → Matomo (PHP + MariaDB)
    └── cloudflared (Cloudflare Tunnel, systemd service)
```

### DNS záznamy (Cloudflare, zóna xlr.cz)
| Typ | Název | Cíl | Proxy |
|-----|-------|-----|-------|
| A | xlr.cz | 185.199.108–111.153 | ✅ | 
| CNAME | www.xlr.cz | xray3131.github.io | ✅ |
| CNAME | matomo.xlr.cz | 2457ab17-....cfargotunnel.com | ✅ |
| MX | xlr.cz | wes1-mx1/2.wedos.net | ❌ |
| CNAME | ftp/imap/pop3/smtp/webmail | WEDOS | ✅ |

---

## Komponenty

### 1. Web (GitHub Pages)
- Repozitář: `xray3131/www` (veřejný)
- Soubory: `index.html`, `CNAME`
- Hostování: GitHub Pages → větev `main`
- Custom doména: `xlr.cz` (soubor CNAME obsahuje `xlr.cz`)
- Obsah: černá stránka + Matomo tracking

### 2. Matomo (analytika)
- Umístění: `10.0.0.24:/opt/matomo`
- Webový server: Caddy `:80`
- PHP: php8.3-fpm (unix socket)
- Databáze: MariaDB, databáze `matomo`
- Přístup: https://matomo.xlr.cz
- Site ID: `1`

**Konfigurace** (`/opt/matomo/config/config.ini.php`):
```ini
[General]
proxy_client_headers[] = "HTTP_CF_CONNECTING_IP"
proxy_client_headers[] = "HTTP_X_FORWARDED_FOR"
proxy_host_headers[] = "HTTP_X_FORWARDED_HOST"
proxy_scheme_headers[] = "HTTP_X_FORWARDED_PROTO"
trusted_hosts[] = "10.0.0.24"
trusted_hosts[] = "matomo.xlr.cz"
trusted_hosts[] = "xlr.cz"
enable_hostname_resolver = 1
```

**Zapnuto/vypnuto:**
- IP anonymizace: ❌ vypnuto
- Hostname resolver: ✅ zapnuto

### 3. Cloudflare Tunnel
- Název: `matomo`
- Tunnel ID: `2457ab17-be79-40c0-9ad5-3f9dd71ed3cd`
- Service: `systemd` (`cloudflared.service`)
- Config: `/etc/cloudflared/config.yml`

```yaml
# /etc/cloudflared/config.yml
ingress:
  - hostname: matomo.xlr.cz
    service: http://localhost:80
  - service: http_status:404
```

---

## Tracking kód (v index.html)
```html
<!-- Matomo -->
<script>
  var _paq = window._paq = window._paq || [];
  _paq.push(['trackPageView']);
  _paq.push(['enableLinkTracking']);
  (function() {
    var u="https://matomo.xlr.cz/";
    _paq.push(['setTrackerUrl', u+'matomo.php']);
    _paq.push(['setSiteId', '1']);
    var d=document, g=d.createElement('script'), s=d.getElementsByTagName('script')[0];
    g.async=true; g.src=u+'matomo.js'; s.parentNode.insertBefore(g,s);
  })();
</script>
<!-- End Matomo Code -->
```

---

## Plán — Přechod na lokální git + Caddy

### Cíl
Zbavit se závislosti na GitHub Pages. Nasazení bude:
```
git push local → post-receive hook → /var/www/xlr.cz → Caddy → Cloudflare Tunnel → xlr.cz
```

### Kroky
1. **Bare git repozitář** na `10.0.0.24` (nebo LXC na Proxmoxu)
   ```bash
   git init --bare /opt/git/www.git
   ```

2. **Post-receive hook** — automatické nasazení po `git push`
   ```bash
   # /opt/git/www.git/hooks/post-receive
   GIT_WORK_TREE=/var/www/xlr.cz git checkout -f main
   ```

3. **Caddy** — přidat nový virtualhost pro `xlr.cz`
   ```
   xlr.cz {
       root * /var/www/xlr.cz
       file_server
       encode gzip
   }
   ```

4. **Cloudflare Tunnel** — přidat ingress pravidlo pro `xlr.cz`
   ```yaml
   ingress:
     - hostname: xlr.cz
       service: http://localhost:80
     - hostname: www.xlr.cz
       service: http://localhost:80
     - hostname: matomo.xlr.cz
       service: http://localhost:80
     - service: http_status:404
   ```

5. **DNS** — změnit A záznamy `xlr.cz` z GitHub Pages IP na tunel
   - A záznamy pro xlr.cz smazat
   - CNAME www.xlr.cz změnit na `2457ab17-....cfargotunnel.com`

6. **Git remote** — přidat lokální remote na vývojovém počítači
   ```bash
   git remote add local x2@10.0.0.24:/opt/git/www.git
   git push local main
   ```

### Výhody oproti GitHub Pages
- ✅ Privátní repozitář (není veřejný na GitHubu)
- ✅ Okamžité nasazení (žádné čekání na GitHub Actions)
- ✅ Plná kontrola nad serverem
- ✅ Možnost PHP/serverové logiky v budoucnu
- ✅ Vše běží na vlastní infrastruktuře

---

## Přístupy & účty
- **GitHub**: xray3131
- **Cloudflare**: zóna xlr.cz, zone ID `f565499d1e08a2f059370c9f564fe198`
- **Server SSH**: `x2@10.0.0.24`
- **Proxmox**: lokální síť
