# Nextcloud AIO — Troubleshooting

> A running log of issues encountered and how they were diagnosed/resolved. Will keep growing as new issues come up.

---

## Table of Contents
- [Grant-Access Loop with the Desktop App](#grant-access-loop-with-the-desktop-app)
- [Mobile Download Stopping at ~1.2GB](#mobile-download-stopping-at-12gb)

---

## Grant-Access Loop with the Desktop App

**Context:** Uploading via the web browser was slow, so the desktop client was installed for faster syncing.

**Symptom:** Login → redirected to browser → click "Grant Access" → loops back without completing.

### Attempt 1 — Check Apache/PHP config

Reference: [community thread on the grant-access loop](https://help.nextcloud.com/t/macos-and-ios-clients-stuck-in-grant-access-loop/52279/3)

Suspected misconfigured values in `config.php`: `overwrite.cli.url`, `overwriteprotocol`, `trusted_proxies`.

```bash
docker exec -it nextcloud-aio-nextcloud bash
cat -n /var/www/html/config/config.php | grep "overwrite.cli.url"
```

Checked the values — all were already correct:

```php
'overwrite.cli.url' => 'https://cloud.example.com/',
'overwriteprotocol' => 'https',
'trusted_proxies' =>
array (
  0 => '127.0.0.1',
  1 => '::1',
  10 => '172.20.0.0/16',
),
```

Ruled out — not the cause.

### Attempt 2 — Check the Nextcloud logs

```bash
docker exec -it nextcloud-aio-nextcloud bash
tail -f /var/www/html/data/nextcloud.log
```

Log output showed:

```
"message":"Login failed: admin (Remote IP: 172.71.214.221)"
```

### Root Cause

The password was simply wrong — the browser's autofill was silently submitting an incorrect saved password. Not a config issue at all.

**Resolution:** Manually re-enter the password instead of relying on browser autofill.

---

## Mobile Download Stopping at ~1.2GB

**Context:** Downloading a 2.4GB video file to a phone via the Nextcloud mobile app. Download consistently stalled at ~53%.

### Things tried

| Attempt | Reasoning | Result |
| :--- | :--- | :--- |
| Checked AIO/Nextcloud logs | Look for explicit errors | Nothing unusual found |
| Disabled Cloudflare proxy mode | Free plan caps requests at 100MB | Didn't fix it |
| Reviewed PHP config | Suspected `upload_max_filesize`, `post_max_size`, `memory_limit` | Already set high enough — not the cause |
| Added NPM timeout/buffering config | Suspected NPM pausing the connection mid-transfer | Didn't fix it |
| Tested download via browser (desktop) | Rule out a server-side issue | Worked fine — pointed at the mobile app specifically |
| Tested via a different app (WebDAV) | Isolate whether it's the official app | **Worked** — confirms the issue is with the mobile app |

NPM configuration applied during testing (kept for reference, did not resolve the issue but is still reasonable to keep for large file transfers):

```nginx
client_max_body_size 0;
proxy_request_buffering off;
proxy_buffering off;
proxy_connect_timeout 3600;
proxy_send_timeout 3600;
proxy_read_timeout 3600;
send_timeout 3600;
proxy_max_temp_file_size 0;
keepalive_timeout 3600;
keepalive_requests 100;
proxy_set_header Connection "";
```

**Workaround found:** Used a third-party file manager (EX File Explorer) configured with the Nextcloud WebDAV link (**Files → Settings → WebDAV** in the Nextcloud app to get the link) instead of downloading through the official mobile app. Download completed successfully.

**Status:** Workaround confirmed, root cause still unknown — likely an issue specific to the official mobile app's download handling on large files. Not yet reproduced or reported upstream.

---

**Back to:** [Nextcloud AIO Setup](./Nextcloud-AIO-Setup.md) · [Domain Setup](./Nextcloud-Domain-Setup.md)
