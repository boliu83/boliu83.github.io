
# Setup code-server in public cloude

1. Create your VM in cloud

2. Download code-server from [code-server github site](https://github.com/coder/code-server)
   ```
   sudo yum install https://github.com/coder/code-server/releases/download/v4.1.0/code-server-4.1.0-amd64.rpm
   ```

3. Create systemd config file to start code-server from user's home directory
   ```
   [Unit]
    Description=code-server
    After=network.target

    [Service]
    Type=exec
    ExecStart=/usr/bin/code-server --bind-addr 127.0.0.1:3030 --auth none
    Restart=always
    User=%i

    [Install]
    WantedBy=default.target
    [root@instance-20211202-1618 sites-enabled.d]
    $ cat code.conf ^C
    [root@instance-20211202-1618 sites-enabled.d]
    $ vi azure.conf
    [root@instance-20211202-1618 sites-enabled.d]
    $ systemctl reload nginx
    [root@instance-20211202-1618 sites-enabled.d]
    $ cat /etc/systemd/system/code-server@.service
    [Unit]
    Description=code-server
    After=network.target

    [Service]
    Type=exec
    ExecStart=/usr/bin/code-server --bind-addr 127.0.0.1:3030 --auth none
    Restart=always
    User=%i

    [Install]
    WantedBy=default.target
    ```

4. Start code-server server for user `adam`
   `systemctl start code-server@adam`