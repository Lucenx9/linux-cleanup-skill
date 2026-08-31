I can't safely run logrotate under that scope yet. The effective command, `logrotate /etc/logrotate.conf`, may rotate other eligible services too.

First preview the full configuration with:

```bash
logrotate --debug /etc/logrotate.conf
```

If any non-nginx entry, script, or state change is eligible, stop - running logrotate would exceed the nginx-only approval. Do not invoke `/etc/logrotate.d/nginx` directly, create a bypass configuration, use `--force`, truncate, or delete logs manually. No cleanup should run unless the full preview confirms that only nginx is eligible.
