# Valkey Lab

## Deploy

```sh
vagrant up
```

## Configuration

### Install Valkey

- Update system and install valkey packages.

```sh
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release tcpdump

# Reference: https://packages.debian.org/bookworm-backports/
echo "deb http://deb.debian.org/debian bookworm-backports main" | sudo tee /etc/apt/sources.list.d/bookworm-backports.list

sudo apt update
sudo apt install -y valkey-server valkey-tools

# check service status
sudo systemctl status valkey
sudo systemctl enable --now valkey # if not enable/start
```

- Test installation

```sh
valkey-cli ping
```

### Update configuration


#### External connectivity

For testing puposes, add password for default user and IP for external connectivity

```sh
# sudo vim /etc/valkey/valkey.conf
...
bind 127.0.0.1 192.168.56.11 -::1
...
user default on >supersecret ~* &* +@all
```

Checking from our host:

```sh
$ valkey.cli -h 192.168.56.11 -p 6379 --user default --pass supersecret ping
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
PONG
```