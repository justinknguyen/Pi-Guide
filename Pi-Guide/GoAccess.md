# GoAccess

View NGINX access log in a UI/dashboard

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

[NGINX](/Pi-Guide/NGINX.md)

## Installation

1. Install GoAccess:
   ```
   sudo apt-get install goaccess
   ```

## Configuration

1. Open the config file for GoAccess:
   ```
   sudo nano /etc/goaccess/goaccess.conf
   ```
2. Uncomment the following lines:
   ```
   time-format %H:%M:%S
   date-format %d/%b/%Y
   ```
3. Under the `Log Format Options` section, enter the following line:
   ```
   log-format %h %^[%d:%t %^] "%m %U" %s %b "%R" "%u"
   ```

## Testing

To output to a terminal and generate an interactive report:

```
goaccess /var/log/nginx/access.log
```

## Sources

- https://hub.docker.com/r/allinurl/goaccess
- https://github.com/allinurl/goaccess/issues/1635
