# Nginx Gotchas

## [Official pitfalls and common mistakes](https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/)

## `alias` doesn't use MIME type detection
- Need an explicit `default_type`:
```nginx
location /cv {
    default_type text/html;
    alias /etc/nginx/cv.html;
}
```

## Directive matching order
### `server_name`
- > ... If none of the above steps are able to satisfy the request, then the request will be passed to the `default_server` for the matching IP address and port.
- If no `default_server` is specified, the **first** server block will be chosen
### `location`
- `=`, `~`, `~*`, `^~`, and then prefixes
### [DigitalOcean article](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)

## Jumping location blocks
```nginx
location /example1 {
    ...
    try_files DUMMY @login;
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

## `proxy_pass`
```nginx
# don't pass request headers
# e.g. If-Modified will result in 412
proxy_pass_request_headers off;
# only pass the required
proxy_set_header Authorization $http_Authorization;
proxy_set_header Cookie $http_cookie;

# pass proper hostname
proxy_set_header Host $http_host;
# pass proper client IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
# pass proper protocol
proxy_set_header X-Forwarded-Proto $scheme;

proxy_pass ...; 
```