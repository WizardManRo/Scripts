...
For Ubuntu 22.04+ with GNOME:

**Enable Remote Desktop (RDP):**

```bash
gsettings set org.gnome.desktop.remote-desktop.rdp enable true
gsettings set org.gnome.desktop.remote-desktop.rdp view-only false
```

**Enable VNC:**

```bash
gsettings set org.gnome.desktop.remote-desktop.vnc enable true
gsettings set org.gnome.desktop.remote-desktop.vnc view-only false
```

**Set credentials:**

```bash
grdctl rdp set-credentials <username> <password>
grdctl vnc set-password <password>
```

**Enable the remote desktop service:**

```bash
systemctl --user enable --now gnome-remote-desktop.service
```

**Check status:**

```bash
systemctl --user status gnome-remote-desktop.service
```

**Note:** This requires an active GNOME session (user must be logged in). If you need headless access, you may still want a traditional VNC server like TigerVNC
...
