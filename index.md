---
layout: default
title: "Jetson Orin Nano Headless GUI + VNC Setup"
---

# Instructions to setup a Virtual Network Computing (VNC) system for Jetson Orin Nano  

These instructions were tested in **Jetson Orin Nano Developer Kit with JetPack 6.1** to set up a VNC system, allowing a laptop to access a **GUI on a headless Jetson**.  
The objective is to enable a **GUI on a Jetson Orin Nano running headless**, meaning it operates without a **directly attached display, keyboard, or mouse**.

These steps use **x11vnc** to create a server that connects to the existing X server (active graphical session). **x11vnc mirrors the screen** and forwards it over **VNC**. The client used is **TigerVNC**.

---

## 1️⃣ Access the Jetson Orin Nano via SSH
```sh
ssh username-remote-machine@ip-address-or-name-jetson
```
The IP of the Jetson in the network with a connected display can be obtained using:
```sh
hostname -I
```
For a more detailed output:
```sh
ip a
```
If the Jetson is connected to the network and you want to check the IP from a different machine:
```sh
arp -a
```

---

## 2️⃣ Install x11vnc and Set a New VNC Password
```sh
sudo apt-get update
sudo apt-get install x11vnc
x11vnc -storepasswd
```

---

## 3️⃣ Confirm that Jetson Orin Nano is Running a Graphical Session
```sh
systemctl get-default
```
If the output is `"graphical.target"`, continue. Otherwise, set it manually:
```sh
sudo systemctl set-default graphical.target
sudo reboot
```
After reboot, the **display manager should be active**, giving a desktop session on `:0`, even **without a monitor**.

---

## 4️⃣ Enable a Fake Display (EDID)
On a headless Jetson Orin Nano, there's no real monitor connected. But some programs (like VNC, remote desktops, and some GUI apps) expect an Xorg display with a proper resolution. Xorg (X.Org Server) is the main implementation of the X11 display server for Linux and UNIX-based systems. It manages graphical output and user input by allowing applications (X clients) to draw windows, handle mouse/keyboard input, and interact with the display hardware. Xorg is the software that makes the GUI work on Linux. Without it, only a command-line interface would be available, unless using Wayland. If **Xorg doesn't detect a monitor**, it might **default to low resolution** (e.g., `640×480`) or **fail to start**.

A solution is to use a dummy video driver. However, a purely software‐emulated video device does not use hardware acceleration. This causes poor performance or missing hardware-accelerated features, and can conflict with the NVIDIA driver on Jetsons.

A better solution involves the use of a EDID (Extended Display Identification Data) from a display, which is a small binary data structure that a display (monitor, TV, laptop screen, etc.) sends to the GPU to describe its capabilities. This allows the system to automatically configure the correct resolution, refresh rate, and other display settings. Forcing an EDID on the NVIDIA driver tells the real NVIDIA driver to load a “fake monitor” descriptor (EDID file), so the hardware believes an HDMI/DP display is connected. This option is best because it retains GPU acceleration and you can pick any resolution and refresh rate that the driver supports (e.g. 1920×1080@60 Hz). 

### Download an EDID file  
EDID files can be obtained from the web. You can obtain EDID files from:
🔗 [Linux EDID Repository](https://github.com/linuxhw/EDID?tab=readme-ov-file#how-to-install-edid-file)

### Apply the EDID file:
Once you decided which EDID .bin file to use, move it to the Jetson Nano storage as follows:
```sh
scp EDID.bin username-remote-machine@ip-address-or-name-jetson:/tmp
```
In the Jetson, move the file from /tmp to a edid folder:
```sh
sudo mkdir -p /lib/firmware/edid
sudo cp /tmp/EDID.bin /lib/firmware/edid/EDID.bin
```

---

## 5️⃣ Modify Xorg Configuration
Modify xorg.conf to use the “fake monitor” descriptor (EDID file). A **1080p** resolution will be used as an example.  
```sh
sudo nano /etc/X11/xorg.conf
```
Add the following in the "Device" section (**remember to change the path in the CustomEDID option to the correct one**):
```
Section "Device"
    Identifier  "Tegra0"
    Driver      "nvidia"
    Option      "PrimaryGPU" "Yes"
    Option      "AllowEmptyInitialConfiguration" "true"
    Option      "ConnectedMonitor" "DFP-0"
    Option      "CustomEDID" "DFP-0:/lib/firmware/edid/EDID.bin"
    Option      "UseDisplayDevice" "DFP-0"
    Option      "IgnoreEDIDChecksum" "DFP-0"
    Option      "AllowNonEdidModes" "true"
EndSection
```
Next, concatenate the following at the end of the file:
```
Section "Monitor"
    Identifier "Monitor0"
    VendorName "FakeVendor"
    ModelName  "Fake1920x1080"
EndSection

Section "Screen"
    Identifier "Screen0"
    Device     "Tegra0"
    Monitor    "Monitor0"
    DefaultDepth 24
    SubSection "Display"
        Depth 24
        Modes "1920x1080"
    EndSubSection
EndSection
```
Save the file and close it.

---

## 6️⃣ Enable Auto-Login, Reboot and Verify
Edit GDM's custom config (as root):
```sh
sudo nano /etc/gdm3/custom.conf
```
Look for the daemon section and add or uncomment lines like:
```
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=replace_with_your_username
```
Save, exit and reboot.
```sh
sudo reboot
```
**Important Note**: having auto-login enabled ensures that a full desktop session is started on display :0. VNC mirrors an active X session, and without auto-login, the system only shows a login screen (the greeter) that doesn't have the complete user session and X credentials needed for VNC to display the desktop properly. However, auto-login bypasses the login screen, so if someone gains physical access to the device, they can immediately access the desktop without needing to enter any credentials. This security risk is why **auto-login is recommended only for development purposes and should be disabled in production environments**. 
After you reboot, Xorg should load this configuration. Even without a real monitor, it will create a 1920×1080 desktop on :0. That can be verified with the following command:
```sh
DISPLAY=:0 xrandr
```

---

## 7️⃣ Start x11vnc 
Test the setup manually by initiating x11vnc server. 
```sh
x11vnc -usepw -forever -display :0
```
By default, x11vnc starts on port **5900**.

---

## 8️⃣ Connect Using TigerVNC
To connect to a Jetson (or any remote machine) running a VNC server, you need to install a VNC client (viewer). TigerVNC is a fast and reliable option.
On your **client machine**, install TigerVNC by running:
```sh
sudo apt update
sudo apt install tigervnc-viewer -y
```
Then, try to establish a connection and launch the GUI:
```sh
vncviewer ip-address-or-name-jetson:5900
```
It is possible to use the IP of the Jetson in the network or the name that you configured to be visualized in the network. 5900 is the port number.
Once you confirmed manually that the VNC works and that you have access to the Jetson’s GUI, it is possible to create a systemd service so that the VNC server always runs at boot.

---

## 9️⃣ Enable x11vnc as a Systemd Service  
```sh
sudo nano /etc/systemd/system/x11vnc.service
```
Paste this inside and update the username field:
```ini
[Unit]
Description=Start x11vnc at startup
Requires=display-manager.service
After=display-manager.service

[Service]
Type=simple
User=replace_with_your_username
ExecStart=/usr/bin/x11vnc -auth guess -display WAIT:0 -loop -usepw -forever -rfbport 5900
Restart=on-failure

[Install]
WantedBy=graphical.target
```
The -display flag is present in ExecStart because of a possible race issue. x11vnc might try to attach :0 before the GUI session is ready. The WAIT:0 option tells x11vnc to wait until :0 actually appears (rather than failing immediately if the X server isn’t fully up).
Next, enable the service so it launches on startup:
```sh
sudo systemctl enable x11vnc.service
```

---

## 🔄 Restart the Service  
```sh
sudo systemctl daemon-reload
sudo systemctl restart x11vnc.service
sudo reboot
```
Subsequently, check if the service is running:
```sh
systemctl status x11vnc.service
```
If it says active (running), then your VNC client should be able to connect immediately. Therefore, connect using:
```sh
vncviewer ip-address-or-name-jetson:5900
```

---

## ✅ Done!
Now, you should be able to access your **Jetson Orin Nano’s GUI headless over VNC!** 🎉

---
**Author**: Mauro Francisco Arcidiacono
