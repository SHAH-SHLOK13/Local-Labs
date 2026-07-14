# Installing, Configuring & Managing Nginx

## What is Nginx?

Nginx is a powerful, high-performance web server that helps websites handle a large number of visitors efficiently. By default, it listens on port 80. Beyond serving web content, it's commonly used as a reverse proxy, a load balancer, a mail proxy, and an HTTP cache, which is why it shows up everywhere from small personal projects to large-scale production infrastructure.

**Reverse proxy:** think of it as a middleman sitting between users and a web server. Instead of users talking directly to the backend server, they talk to the reverse proxy, which forwards the request on their behalf.

---

## Part 1: Installing and Configuring Nginx

### 1. Check for existing installation, then install

```bash
rpm -qa | grep nginx
dnf install nginx -y
```

### 2. Start, enable, and check status

```bash
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

### 3. Open port 80

```bash
systemctl stop firewalld
systemctl disable firewalld
```

> This is only acceptable in a lab environment. In a corporate environment, you'd instead grant the specific service permission through the firewall rather than disabling it outright.

### 4. Create the server config

```bash
vi /etc/nginx/nginx.conf
vi /etc/nginx/conf.d/shloklinux.conf   # create this file
```

```nginx
server {
    listen 80;
    server_name 10.194.213.166;   # your machine's IP

    root /var/www/shloklinux/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

> **Note:** Nginx may fail to start if port 80 is already in use, for example by Apache. If that happens, stop the conflicting service first, then start Nginx.
>
> ```bash
> lsof -i :80              # check which service is using port 80
> systemctl stop <service>  # stop whatever service is running on it
> ```

### 5. Create the web root and a test page

```bash
mkdir -p /var/www/shloklinux/html
cd /var/www/shloklinux/html
vi index.html
```

```html
<h1>Hello from ShlokLinux</h1>
```

### 6. Verify and restart

```bash
nginx -t                  # check syntax, must return "ok"
systemctl restart nginx
```

### 7. Test it

Open in a browser:

```
http://10.194.213.166
```

---

## Part 2: Setting Up Nginx as a Reverse Proxy

**Setup:**

- `ShlokLinux` — `10.194.213.166` (the reverse proxy / middleman)
- `ShlokProxy-serv` — `10.194.213.90` (the backend content server)
- The user connects to `ShlokLinux` (.161), which forwards the request to `ShlokProxy-serv` (.162) and returns the response.

### Step 1: Repeat the base install on the new machine

Set up the backend machine (`10.194.213.90`) using steps 1 through 6 from Part 1: install Nginx, create the web root, and add its own `index.html`.

**If you hit a "403 Forbidden" error**, it's usually SELinux blocking access to the content directory:

```bash
sestatus   # check SELinux status, shows details
```

If it's in `enforcing` mode, restore the correct SELinux context on the content directory:

```bash
chcon -R -t httpd_sys_content_t /var/www/shloklinux/html
```

Set the test page's message to something distinguishable, for example "Hello from shlok-proxy-server", so you can visually confirm later which server is responding.

### Step 2: Configure the reverse proxy on the first machine (.161)

```nginx
server {
    listen 80;
    server_name 10.194.213.166;   # this machine's own IP

    location / {
        proxy_pass http://10.194.213.90;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Repeat the check-and-restart steps (`nginx -t` and `systemctl restart nginx`) as before.

### Testing and troubleshooting

Visit `http://10.194.213.166` in the browser.

**If you get a "502 Bad Gateway" error:**

```bash
tail -f /var/log/nginx/error.log
```

Check the last line of the log. If it shows "permission denied," SELinux is blocking Nginx from making outbound network connections, which it needs to do to reach the backend server. Fix it with:

```bash
setsebool -P httpd_can_network_connect 1
systemctl restart nginx
```

Once that's applied, reloading `http://10.194.213.166` should return the message from `10.194.213.90` (the backend) instead of the local page on `.166`, confirming the reverse proxy is working correctly.
