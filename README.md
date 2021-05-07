# wsl-vpnkit

Uses [VPNKit](https://github.com/moby/vpnkit) and [npiperelay](https://github.com/jstarks/npiperelay) to provide network connectivity to the WSL 2 VM. This requires no settings changes or admin privileges on the Windows host.

## Setup

The following steps will use WSL to setup `wsl-vpnkit`. If you do not have connectivity in WSL 2, you can [switch your WSL version](https://docs.microsoft.com/en-us/windows/wsl/install-win10#set-your-distribution-version-to-wsl-1-or-wsl-2) to WSL 1 for setup and back to WSL 2 once done. Alternatively, you can refer to [this post to setup `wsl-vpnkit` from the Windows side](https://github.com/sakai135/wsl-vpnkit/issues/11#issuecomment-777806102).

### Install `vpnkit.exe` and `vpnkit-tap-vsockd`

This will download and extract `vpnkit.exe` and `vpnkit-tap-vsockd` from the [Docker Desktop for Windows installer](https://docs.docker.com/docker-for-windows/install/). Alternatively, build `vpnkit.exe` and `vpnkit-tap-vsockd` from [VPNKit](https://github.com/moby/vpnkit).

```sh
sudo apt install p7zip genisoimage
```

```sh
wget https://desktop.docker.com/win/stable/amd64/Docker%20Desktop%20Installer.exe
7zr e Docker\ Desktop\ Installer.exe resources/vpnkit.exe resources/wsl/docker-for-wsl.iso
mkdir -p /mnt/c/bin
mv vpnkit.exe /mnt/c/bin/wsl-vpnkit.exe
isoinfo -i docker-for-wsl.iso -R -x /containers/services/vpnkit-tap-vsockd/lower/sbin/vpnkit-tap-vsockd > ./vpnkit-tap-vsockd
chmod +x vpnkit-tap-vsockd
sudo mv vpnkit-tap-vsockd /sbin/vpnkit-tap-vsockd
sudo chown root:root /sbin/vpnkit-tap-vsockd
rm Docker\ Desktop\ Installer.exe docker-for-wsl.iso
```

### Install `npiperelay.exe`

Download from [npiperelay](https://github.com/jstarks/npiperelay).

```sh
sudo apt install unzip
```

```sh
wget https://github.com/jstarks/npiperelay/releases/download/v0.1.0/npiperelay_windows_amd64.zip
unzip npiperelay_windows_amd64.zip npiperelay.exe
rm npiperelay_windows_amd64.zip
mkdir -p /mnt/c/bin
mv npiperelay.exe /mnt/c/bin/
```

### Install `socat`

```sh
sudo apt install socat
```

### Configure DNS for WSL

Disable WSL from generating and overwriting `/etc/resolv.conf`.

```sh
sudo tee /etc/wsl.conf <<EOL
[network]
generateResolvConf = false
EOL
```

Manually set DNS servers to use when not running this script. `1.1.1.1` is provided as an example.

```sh
sudo tee /etc/resolv.conf <<EOL
nameserver 1.1.1.1
EOL
```

## Run

```sh
sudo ./wsl-vpnkit
```

Keep this terminal open.

In some environments, explicitly pass the environment variable `WSL_INTEROP` to `sudo`.

```sh
sudo --preserve-env=WSL_INTEROP ./wsl-vpnkit
```

Services on the WSL2 VM should be accessible from the Windows host using `localhost` through [the WSL networking integrations](https://devblogs.microsoft.com/commandline/whats-new-for-wsl-in-insiders-preview-build-18945/#use-localhost-to-connect-to-your-linux-applications-from-windows). Services on the Windows host should be accessible using the IP from `VPNKIT_HOST_IP` (`192.168.67.2`).

## Run in the Background

This is an example setup to run `wsl-vpnkit` in the background.

### Use `start-stop-daemon`

This will use `start-stop-daemon` to create a daemon from the `wsl-vpnkit` script.

```sh
sudo /sbin/start-stop-daemon --startas /path/to/wsl-vpnkit --make-pidfile --remove-pidfile --pidfile /var/run/wsl-vpnkit.pid --background --start
```

```sh
sudo /sbin/start-stop-daemon --startas /path/to/wsl-vpnkit --make-pidfile --remove-pidfile --pidfile /var/run/wsl-vpnkit.pid --stop
```

### Setup Sudoers

This allows running the `wsl-vpnkit` daemon without entering a password every time.

This step can be dangerous. Read [Sudoers](https://help.ubuntu.com/community/Sudoers) before doing this step.

```sh
sudo visudo -f /etc/sudoers.d/wsl-vpnkit
```

```
yourusername ALL=(ALL) NOPASSWD: /sbin/start-stop-daemon --startas /path/to/wsl-vpnkit --make-pidfile --remove-pidfile --pidfile /var/run/wsl-vpnkit.pid *
```

### Run in the Background

Starting the `wsl-vpnkit` daemon from Windows using `wsl.exe` allows the daemon to keep running in the background. Add the following to your `.profile` or `.bashrc` to start `wsl-vpnkit` when you open your WSL terminal.

```sh
wsl.exe -- sudo /sbin/start-stop-daemon --startas /path/to/wsl-vpnkit --make-pidfile --remove-pidfile --pidfile /var/run/wsl-vpnkit.pid --quiet --oknodo --background --start
```

## Troubleshooting

### Configure VS Code Remote WSL Extension

If VS Code takes a long time to open your folder in WSL, [enable the setting "Connect Through Localhost"](https://github.com/microsoft/vscode-docs/blob/main/remote-release-notes/v1_54.md#fix-for-wsl-2-connection-issues-when-behind-a-proxy).

### Configure `http_proxy.json` and `gateway_forwards.json`

This step is only necessary for using a HTTP proxy or exposing some service from the Windows host to the WSL 2 VM through VPNKit.

Set the variables `VPNKIT_HTTP_CONFIG` and/or `VPNKIT_GATEWAY_FORWARD_CONFIG` to the Windows host path to these configuration files. Example values are provided in `./wsl-vpnkit` for using the configuration generated by Docker Desktop. If using Docker Desktop's `http_proxy.json` and `gateway_forwards.json`, ensure that Docker Desktop is setup for WSL 2 integration and is running Docker.

`http_proxy.json` points to any HTTP proxies that might be configured on the Windows host. See an [example `http_proxy.json` from VPNKit](https://github.com/moby/vpnkit/blob/v0.5.0/src/bin/main.ml#L714-L721).

`gateway_forwards.json` points to any services to forward to the WSL 2 VM. See an [example `gateway_forwards.json` from VPNKit](https://github.com/moby/vpnkit/blob/bfd0458bb811027cb9bd45f9ed8d63984b5d4a33/go/pkg/vpnkit/config_test.go#L28).

### Try shutting down WSL VM to reset

```sh
wsl.exe --shutdown
```

Check for a `vpnkit.exe` process listening on `\\.\pipe\wsl-vpnkit` and kill that too.

### Check for the required processes

```sh
ps aux | grep wsl-vpnkit
```

* `socat ... npiperelay.exe`
* `wsl-vpnkit.exe`
* `vpnkit-tap-vsockd`
