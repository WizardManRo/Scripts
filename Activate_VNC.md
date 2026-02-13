# x11vnc VNC Server Setup — Ubuntu Unity with GDM3

A step-by-step guide to set up x11vnc as a persistent systemd service, allowing VNC access at any time without intervention.

---

## Step 1: Install x11vnc

```bash
sudo apt update
sudo apt install x11vnc -y
```

## Step 2: Set the VNC password

```bash
sudo mkdir -p /root/.vnc
sudo x11vnc -storepasswd /root/.vnc/passwd
```

Enter and verify your desired VNC password when prompted.

## Step 3: Create the startup wrapper script

This script dynamically finds the correct GDM Xauthority file, so it survives reboots and updates.

```bash
sudo nano /usr/local/bin/x11vnc-start.sh
```

Paste the following:

```bash
#!/bin/bash
AUTH=$(find /run/user -name "Xauthority" -path "*/gdm/*" 2>/dev/null | head -1)
exec /usr/bin/x11vnc -usepw -display :0 -forever -noxdamage -auth "$AUTH" -rfbport 5900
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/x11vnc-start.sh
```

## Step 4: Create the systemd service

```bash
sudo nano /etc/systemd/system/x11vnc.service
```

Paste the following:

```ini
[Unit]
Description=x11vnc VNC Server
After=display-manager.service
Requires=display-manager.service

[Service]
Type=simple
ExecStartPre=/bin/sleep 5
ExecStart=/usr/local/bin/x11vnc-start.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## Step 5: Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable x11vnc.service
sudo systemctl start x11vnc.service
```

## Step 6: Verify

```bash
sudo systemctl status x11vnc.service
```

You should see `active (running)` and the line `PORT=5900` in the logs.

## Connecting

From any VNC client (RealVNC Viewer, TigerVNC, etc.), connect to:

```
<your-server-ip>:5900
```

## Troubleshooting

| Issue | Fix |
|---|---|
| Blank/frozen screen over VNC | Already handled by `-noxdamage` flag |
| Service fails to start | Check logs: `sudo journalctl -u x11vnc.service -e` |
| Password issues | Re-run: `sudo x11vnc -storepasswd /root/.vnc/passwd` |
| Auth file not found | Verify GDM is running: `systemctl status gdm3` |
| Change VNC port | Edit `-rfbport 5900` in the wrapper script |
