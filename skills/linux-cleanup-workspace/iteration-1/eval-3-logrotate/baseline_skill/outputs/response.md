I'll isolate the operation to nginx and will not run `/etc/logrotate.conf`, since that could rotate every included service.

First, preview only the nginx stanza:

```bash
logrotate --debug /etc/logrotate.d/nginx
```

After verifying it contains only nginx log paths and reviewing its pre/post-rotate scripts, run:

```bash
sudo logrotate /etc/logrotate.d/nginx
```

I would not add `--force`, delete logs directly, or truncate active logs. This follows nginx's configured schedule and retention, updates logrotate state, and may run its configured reload script; it may reclaim nothing if no nginx log is currently eligible.
