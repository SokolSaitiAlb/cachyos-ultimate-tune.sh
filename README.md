# cachyos-ultimate-tune.sh
cachyos-ultimate-tune script

✓ Instructions

1. Save the script:

Bash

nano cachyos-ultimate-tune.sh


2. Paste the entire script :


Script

#!/bin/bash

# === CHECK ZENITY ===
if ! command -v zenity &> /dev/null; then
    echo "🧩 Installing Zenity GUI support..."
    pacman -S --noconfirm zenity
fi

# === ZENITY FEATURE SELECTION MENU ===
choices=$(zenity --list --checklist \
  --title="CachyOS Gaming Setup" \
  --width=500 --height=600 \
  --text="Select the features you want to enable:" \
  --column="Enable" --column="Feature" \
  TRUE "Install Steam, Wine, ProtonUp-Qt" \
  TRUE "Install OBS Studio + 720p Profile" \
  TRUE "Enable MangoHud + GameMode" \
  TRUE "Apply RX 6800 Fan Curve on Login" \
  FALSE "Install VKBasalt with CAS Shader" \
  FALSE "Install Lutris & Heroic Games Launcher" \
  FALSE "Enable KDE Gaming Layout (Latte Dock)" \
  FALSE "Add Backup Reminder Popup" \
  FALSE "Disable KDE File Indexer (Baloo)" \
  FALSE "Configure HugePages for Game Load Boost" \
  FALSE "Install CoreCtrl GPU OC Manager" \
  --separator="|")

# Convert string to flags
IFS="|" read -r -a features <<< "$choices"


# === CACHYOS ULTIMATE TUNING FOR RX 6800 + KDE GAMING ===
# By Alb Kestrel

echo "🔧 Starting Ultimate Tune for CachyOS (RX 6800 + KDE)..."

# --- REQUIRE ROOT ---
if [ $EUID -ne 0 ]; then
  echo "🚫 Please run as root"
  exit 1
fi

# === 0. REMOVE TIMESHiFT IF PRESENT ===
echo "🗑️ Removing Timeshift..."
pacman -Rns --noconfirm timeshift || echo "Timeshift not installed"

# === 1. GRUB TIMEOUT TO 2 SECONDS ===
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=2/' /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg

# === 2. CPU GOVERNOR TO PERFORMANCE ===
echo "performance" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat <<EOF > /etc/tmpfiles.d/cpu-governor.conf
w /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor - - - - performance
EOF

# === 3. SNAPSHOT SYSTEM (SNAPPER) ===
echo "📸 Setting up Snapper..."
pacman -Sy --noconfirm snapper

mkdir -p /etc/pacman.d/hooks
cat <<EOF > /etc/pacman.d/hooks/50-snapper-pre.hook
[Trigger]
Operation = Upgrade
Type = Package
Target = *
[Action]
Description = Creating pre-upgrade snapshot...
When = PreTransaction
Exec = /usr/bin/snapper create --type pre --description "pacman upgrade"
EOF

cat <<EOF > /etc/pacman.d/hooks/50-snapper-post.hook
[Trigger]
Operation = Upgrade
Type = Package
Target = *
[Action]
Description = Creating post-upgrade snapshot...
When = PostTransaction
Exec = /usr/bin/snapper create --type post --description "pacman upgrade"
EOF

# === 4. AMDGPU OVERCLOCK SERVICE FOR RX 6800 ===
mkdir -p /etc/systemd/system/amdgpu-oc.service.d
cat <<EOF > /usr/local/bin/amdgpu-oc-rx6800.sh
#!/bin/bash
echo "Applying RX 6800 OC settings..."
echo "manual" > /sys/class/drm/card0/device/power_dpm_force_performance_level
# echo "2000 900" > /sys/class/drm/card0/device/pp_od_clk_voltage  # Example
EOF

chmod +x /usr/local/bin/amdgpu-oc-rx6800.sh

cat <<EOF > /etc/systemd/system/amdgpu-oc.service
[Unit]
Description=RX 6800 Overclock Service
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/amdgpu-oc-rx6800.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
EOF

systemctl enable amdgpu-oc.service

# === 5. CORECTRL PROFILE SETUP FOR OC ===
pacman -S --noconfirm corectrl
mkdir -p /etc/polkit-1/rules.d
cat <<EOF > /etc/polkit-1/rules.d/90-corectrl.rules
polkit.addRule(function(action, subject) {
    if ((action.id == "org.corectrl.helper.init" ||
         action.id == "org.corectrl.helperkiller.run") &&
         subject.isInGroup("wheel")) {
        return polkit.Result.YES;
    }
});
EOF

# === 6. INSTALL GAMEMODE + MANGOHUD ===
pacman -S --noconfirm gamemode mangohud lib32-gamemode lib32-mangohud

cat <<EOF > /etc/environment
MANGOHUD=1
GAMEMODERUNEXEC=1
EOF

# === 7. KDE GAMING LAYOUT + THEME ===
echo "🎮 Installing KDE gaming themes & layout..."
plasma-apply-lookandfeel org.kde.breezedark.desktop

# === 8. ZRAM + SYSCTL TWEAKS ===
pacman -S --noconfirm systemd-zram-generator
cat <<EOF > /etc/systemd/zram-generator.conf
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
EOF

cat <<EOF >> /etc/sysctl.d/99-performance-tweaks.conf
vm.swappiness=10
vm.vfs_cache_pressure=50
EOF

# === 9. DAVINCI RESOLVE INSTALL (RX 6800 PATCHED) ===
echo "🎥 Installing DaVinci Resolve with RX 6800 patching..."

pacman -S --noconfirm davinci-resolve lib32-opencl-mesa ocl-icd opencl-amd

mkdir -p /etc/OpenCL/vendors
echo "/opt/amdgpu-pro/lib64/libamdocl64.so" > /etc/OpenCL/vendors/amdocl64.icd

echo "export AMDGPU_PRO=1" >> /etc/environment
echo "export LD_LIBRARY_PATH=/opt/amdgpu-pro/lib64:\$LD_LIBRARY_PATH" >> /etc/environment

pacman -S --noconfirm vulkan-radeon opencl-mesa lib32-vulkan-radeon lib32-opencl-mesa

# === 10. EXTRA APPS: STEAM, WINE, OBS, VLC, KRITA ===
echo "🎮 Installing extra tools: Steam, Wine, OBS, VLC, Krita..."
pacman -S --noconfirm steam wine-staging vlc obs-studio krita

# === 11. PROTONUP-QT APPIMAGE INSTALL ===
echo "⬇️ Installing ProtonUp-Qt..."
mkdir -p /opt/protonup-qt
curl -Lo /opt/protonup-qt/ProtonUp-Qt.AppImage https://github.com/DavidoTek/ProtonUp-Qt/releases/download/v2.12.0/ProtonUp-Qt-2.12.0-x86_64.AppImage
chmod +x /opt/protonup-qt/ProtonUp-Qt-2.12.0-x86_64.AppImage
ln -sf /opt/protonup-qt/ProtonUp-Qt-2.12.0-x86_64.AppImage /usr/local/bin/protonup-qt

echo "🔧 Optimizing for 1366x768 screen resolution..."

# OBS: lower resolution + bitrate preset
mkdir -p ~/.config/obs-studio/basic/profiles/YouTube720p
cat <<EOF > ~/.config/obs-studio/basic/profiles/YouTube720p/basic.ini
[General]
Name=YouTube720p
[Video]
BaseCX=1280
BaseCY=720
OutputCX=1280
OutputCY=720
[Output]
Mode=Advanced
Bitrate=4500
Encoder=x264
EOF

# Steam: small mode & scale fix
mkdir -p ~/.steam/steam
echo "STEAM_FORCE_DESKTOPUI_SCALE=0.75" >> ~/.steam/steam/env

# KDE: adjust scale factor
kwriteconfig5 --file kdeglobals --group KScreen --key ScaleFactor "0.9"

# === 12. AUTO-APPLY CORECTRL FAN CURVE WITH NOTIFICATION ===
echo "🌀 Setting up auto fan curve for RX 6800 with notify-send popup..."

# Ensure notify-send works
pacman -S --noconfirm libnotify

# Create directory for CoreCtrl fan curve profile
mkdir -p ~/.config/corectrl/profiles

# Copy the fan curve profile
cat <<EOF > ~/.config/corectrl/profiles/rx6800_oc_fancurve.json
{
  "version": 1,
  "profiles": [
    {
      "name": "RX 6800 OC + Fan Curve",
      "device": "AMD Radeon RX 6800",
      "power_dpm_force_performance_level": "manual",
      "fan_control": true,
      "fan_curve": [
        {"temp": 30, "speed": 20},
        {"temp": 50, "speed": 40},
        {"temp": 65, "speed": 60},
        {"temp": 75, "speed": 80},
        {"temp": 85, "speed": 100}
      ]
    }
  ]
}
EOF

# Create the apply script
mkdir -p ~/.local/bin

cat <<'EOF' > ~/.local/bin/apply-rx6800-fancurve.sh
#!/bin/bash
corectrl-cli --import ~/.config/corectrl/profiles/rx6800_oc_fancurve.json
notify-send -i graphics-card "CoreCtrl" "✅ RX 6800 fan curve applied successfully"
EOF

chmod +x ~/.local/bin/apply-rx6800-fancurve.sh

# Autostart on KDE login
mkdir -p ~/.config/autostart

cat <<EOF > ~/.config/autostart/rx6800-fancurve.desktop
[Desktop Entry]
Name=RX 6800 Fan Curve
Exec=/home/$USER/.local/bin/apply-rx6800-fancurve.sh
Type=Application
X-GNOME-Autostart-enabled=true
EOF

zenity --info --title="CachyOS Gaming Tuner" \
  --text="✅ Your selected gaming features have been installed.\nPlease reboot to apply all system-level tweaks."


# === DONE ===
echo "✅ Ultimate tuning complete. Please reboot to apply all changes."


3. Make it executable:

Bash

chmod +x cachyos-ultimate-tune.sh



4. Run the script as root:

Bash

sudo ./cachyos-ultimate-tune.sh
