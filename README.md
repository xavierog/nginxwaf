nginxwaf is an attempt to implement a basic, positive security WAF within nginx, without loading additional modules.
Technically, it turns a YAML description of known endpoints into nginx directives (mostly locations and rewrite).

Usage:

```sh
./nginxwaf < myapp-waf.yaml > myapp-waf.nginx.conf
```

Format: see [FORMAT.md](FORMAT.md)
