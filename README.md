# Installing Nix & Home Manager on Raspberry-Pi 5 Alpine OS
This document is meant to describe how I setup a Raspberry Pi 5 with Alpine that uses Nix+Home-Manager. Also sandboxing does not work on Alpine's `musl libc` by default, because sandboxing works on linux namespaces (user, mount, pid, etc.) to isolate builds. I'll find a work-around an publish it here at a later date.

## Why?
Because I wanted to my RPi to have a light-weight linux distro that had Nix, but RPi's don't work out-the-box with NixOS nor Arch. Alpine seemed like the next best thing and was very easy to install with the `rpi-imager`, but I soon found out that Nix doesn't work out-the-box with Alpine. Nevertheless, here's how to do it:

## 1. Installing Alpine OS on the Pi 5:

I used the rpi-imager from the raspberry pi website to install Alpine OS

```bash
rpi-imager
```

## 2. Setting up Alpine OS

Now that we are in the alpine tty we can begin configuring it and making it our own:
```ash
setup-alpine
```
Configure according to your keyboard, language, timezone, etc.

Now we need to install a couple of apps necessary for installing nix
```ash
apk add bash curl git xz build-base shadow sudo
```
*note: I recommend using the `bash` shell from this point foward, because `ash` is very minimal and lacks important features.

Let's also install a simple network manager, since many times Alpine won't come with one out-the-box (escpecially on rapberry pi's)
```bash
apk add dhcpcd
```
Now let's make the network manager start always on boot-up
```bash
rc-update add dhcpcd default
```
Start the service
```bash
rc-service dhcpcd start
```

You’ll also want to ensure /etc/nsswitch.conf exists (some Alpine minimal builds don’t include it):
```ash
echo 'hosts: files dns' > /etc/nsswitch.conf
```

## 3. Installing Nix

Let's this command from the Nix website to install the 'mutli-user' version of nix:
```
sh <(curl -L https://nixos.org/nix/install) --daemon
```
The command above will run successfully, but there is a big issue, it will install Nix, but won't be able to run the `nix-daemon` service. Alpine OS doesn't use `systemd`, it uses `openrc` as its service manager and Nix only starts the service if `systemd` or `systemctl` is being used. We can work around this by writing a openrc daemon for Nix.

Here's a command to create the file `nix-daemon` in the `/etc/init.d/` directory:
```bash
cat <<'EOF' > /etc/init.d/nix-daemon
#!/sbin/openrc-run
description="Nix Daemon"

command="/nix/var/nix/profiles/default/bin/nix-daemon"
command_user="root"
pidfile="/run/nix-daemon.pid"

depend() {
    use net
}
EOF
```
Now we need to set the permissions to execute the service:
```bash
chmod +x /etc/init.d/nix-daemon
```

Now we add the `nix-daemon` to our startup and initiate the service:
```bash
rc-update add nix-daemon default
```
```bash
rc-service nix-daemon start
```
Check if the service started:
```bash
ps aux | grep nix-daemon
```

Finally, ensure your user is in the nixbld group:
```bash
groups <username>
```

```bash
addgroup <username> nixbld
```
