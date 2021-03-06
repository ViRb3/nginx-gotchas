# Nginx Gotchas

## [Official pitfalls and common mistakes](https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/)

## SSL best practices
- ### [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- ### [Bettercrypto](https://bettercrypto.org/#_nginx)
- ### [Cipherli.st](https://cipherli.st/)

## Directive matching order
- ### [DigitalOcean article](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)
  - #### `server_name`
    - > ... If none of the above steps are able to satisfy the request, then the request will be passed to the `default_server` for the matching IP address and port.
    - If no `default_server` is specified, the **first** server block will be chosen

## MIME type detection
- ### `alias`, `proxy_pass` and jumps won't recognize the destination MIME type. You need an explicit `default_type`:
    ```nginx
    location /cv {
        default_type text/html;
        alias /etc/nginx/cv.html;
    }
    ```

## Snippets
### Jump location block
> The order of execution for each approach is different, test which works for your use case.
```nginx
location /example1 {
    ...
    try_files /dev/null @login;
}

location /example2 {
    ...
    error_page 404 = @login;
    return 404;
}

location @login {
    internal;
    ...
}
```

### Access `http` block from `.config` file
```nginx
# will go in parent (http) block
limit_req_zone $binary_remote_addr zone=userlimit:10m rate=1r/s;

server {
    ...
}
```

### Reverse proxy
```nginx
# pass proper hostname
proxy_set_header Host $http_host;
proxy_set_header X-Forwarded-Host $http_host;
# pass proper client IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Real-IP $remote_addr;
# pass proper protocol
proxy_set_header X-Forwarded-Proto $scheme;

# don't automatically fix "location" and "redirect" headers
proxy_redirect off;
proxy_buffering off;

proxy_pass ...; 
```

### Rate limit
```nginx
limit_req_status 403;
limit_req zone=serverlimit burst=10 nodelay;
limit_req zone=userlimit burst=5 nodelay;
```

### Disable search engine crawling
```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /\n";
}
```

### Cookie-based auth proxy
```nginx
auth_request /auth;

# pass auth cookie to client
auth_request_set $saved_set_cookie $upstream_http_set_cookie;
add_header Set-Cookie $saved_set_cookie;

# use = to take precedence over other ~ locations
location = /auth {
    internal;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
    # the "reverse proxy" section discussed before
    include reverse-proxy.conf;

    # don't pass request headers
    # e.g. If-Modified will result in 412
    proxy_pass_request_headers off;
    # only pass the required
    proxy_set_header Authorization $http_Authorization;
    proxy_set_header Cookie $http_cookie;

    proxy_pass https://auth.example.com; 
}
```

### Don't respond if invalid URL
```nginx
error_page 404 403 @putoff;

location @putoff {
    return 444;
}

location / {
    error_page 418 @putoff;
    return 418;
}
```
### `proxy_pass` trailing slash ([Source](https://stackoverflow.com/questions/22759345/nginx-trailing-slash-in-proxy-pass-url))
- No URI (i.e. `http://server:1234`) will forward the URI from the original request exactly as it was with all double slashes, `../` and so on
-  With URI (i.e. `http://server:1234/a/`) acts like the `alias` directive, meaning nginx will replace the part that matches the location prefix with the URI in the `proxy_pass` directive. For example:
    ```nginx
    location /one/ {
        proxy_pass http://127.0.0.1:8080/two;
    }
    ```
    Accessing `http://yourserver.com/one/path/here?param=1` will become `http://127.0.0.1/twopath/here?param=1`
