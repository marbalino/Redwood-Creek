# Deployment Guide — Redwood Creek Map (Linux + nginx)

This document is written for a Claude Code agent deploying this static map to a Linux VM. Follow each section in order. All commands assume a Debian/Ubuntu-based distro running as a user with `sudo` access.

---

## 1. Prerequisites

Install nginx and git if not already present:

```bash
sudo apt update && sudo apt install -y nginx git
```

Verify:

```bash
nginx -v
git --version
```

---

## 2. Clone the Repository

Pick a web root. `/var/www/redwood-creek` is recommended:

```bash
sudo mkdir -p /var/www/redwood-creek
sudo chown $USER:$USER /var/www/redwood-creek
git clone https://github.com/marbalino/Redwood-Creek.git /var/www/redwood-creek
```

Confirm the key files are present:

```bash
ls /var/www/redwood-creek
# Expected: index.html  data/  tiles/  README.md  DEPLOY.md
```

---

## 3. Configure nginx

Create a new site config:

```bash
sudo nano /etc/nginx/sites-available/redwood-creek
```

Paste the following — replace `YOUR_DOMAIN_OR_IP` with the server's public IP or domain name:

```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    root /var/www/redwood-creek;
    index index.html;

    # Serve GeoJSON with correct MIME type
    types {
        application/geo+json geojson;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    # Cache tile PNGs aggressively — they never change
    location /tiles/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Cache GeoJSON for a short time
    location /data/ {
        expires 1h;
        add_header Cache-Control "public";
    }

    # Gzip text assets
    gzip on;
    gzip_types text/html application/javascript application/json application/geo+json;
}
```

Enable the site and disable the default:

```bash
sudo ln -s /etc/nginx/sites-available/redwood-creek /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t        # must print "syntax is ok" and "test is successful"
sudo systemctl reload nginx
```

The map should now be accessible at `http://YOUR_DOMAIN_OR_IP`.

---

## 4. HTTPS with Let's Encrypt (Required for Geolocation)

The live GPS location feature requires HTTPS — browsers block the Geolocation API on plain HTTP. Skip this section only if the VM has no public domain name.

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Obtain and install a certificate (replace `yourdomain.com`):

```bash
sudo certbot --nginx -d yourdomain.com
```

Certbot will automatically update the nginx config to redirect HTTP → HTTPS and add the SSL block. Verify auto-renewal works:

```bash
sudo certbot renew --dry-run
```

---

## 5. File Permissions

Ensure nginx can read all files:

```bash
sudo chown -R www-data:www-data /var/www/redwood-creek
sudo chmod -R 755 /var/www/redwood-creek
```

---

## 6. Updating the Map

When new commits are pushed to GitHub, pull them on the server:

```bash
cd /var/www/redwood-creek
git pull origin master
```

No build step is required — the site is pure static files.

To automate updates, add a cron job:

```bash
crontab -e
# Add this line to pull every 5 minutes:
*/5 * * * * cd /var/www/redwood-creek && git pull origin master --quiet
```

---

## 7. Verify the Deployment

Run through this checklist after deploying:

- [ ] `http://YOUR_DOMAIN_OR_IP` loads the map
- [ ] Basemap tiles (OpenStreetMap, ESRI Satellite) render correctly
- [ ] Elevation hillshade layer loads (tiles served from `/tiles/redwood_rd_surface/`)
- [ ] All GeoJSON layers load (creek bank, drains, inlets, etc.)
- [ ] V1/V2 switcher toggles the creek bank/centerline geometry
- [ ] Right-click on the map shows coordinates
- [ ] Compass rose drag rotates the map
- [ ] HTTPS is active (padlock in browser) — required for geolocation button to work

---

## 8. Troubleshooting

**Blank map / tiles not loading**
- Check nginx error log: `sudo tail -f /var/log/nginx/error.log`
- Confirm file ownership: `ls -la /var/www/redwood-creek/tiles/`

**GeoJSON layers not appearing**
- Open browser devtools → Network tab — look for failed `fetch()` calls to `/data/*.geojson`
- Confirm the `data/` directory is present and readable

**Geolocation button does nothing**
- The browser silently blocks geolocation on HTTP. HTTPS is required — complete Step 4.

**nginx config test fails**
- Run `sudo nginx -t` and read the error — usually a missing semicolon or wrong file path
- Check that the symlink in `sites-enabled` points to the correct file: `ls -la /etc/nginx/sites-enabled/`

**Port 80 or 443 already in use**
- Check what's running: `sudo ss -tlnp | grep ':80'`
- Stop any conflicting service before starting nginx

---

## 9. Project-Specific Notes for the Agent

- **No build step.** This is a single `index.html` file with CDN dependencies. Do not run npm, webpack, or any build tool.
- **Tile format.** The elevation tiles in `tiles/redwood_rd_surface/` are in **TMS format** (Y-axis flipped). The Leaflet config already handles this with `tms: true` — do not reprocess the tiles.
- **GeoJSON coordinate system.** All `data/*.geojson` files are in EPSG:4326 (WGS84). Do not reproject them.
- **CDN dependencies.** The map loads Leaflet and leaflet-rotate from unpkg.com. The VM needs outbound internet access, or those scripts must be downloaded and served locally.
- **Large files.** The repo contains PNG tile images under `tiles/`. A fresh `git clone` pulls ~1 MB — this is expected and normal.
- **Source QGIS project.** The original `.qgz` project files live separately at `C:\Users\andre\OneDrive\Desktop\Redwood Ave\Project\` on the owner's Windows machine and are not part of this repo.
