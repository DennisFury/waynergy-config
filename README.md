# waynergy-config

See reddit post on why this is created: https://www.reddit.com/r/pop_os/comments/1rm9a77/cosmic_desktop_how_to_get_keyboard_mouse_share/

See config.ini file for keyboard mappings.

pop-os needs to be the client (no mouse / keyboard)
other machine needs to be the server (with mouse / keyboard)

POP-OS waynergy install (long version)

1) Install dependencies
```
sudo apt update
sudo apt install -y git meson ninja-build pkg-config libwayland-dev libxkbcommon-dev libtls-dev
```
3) Clone and build
```
git clone https://github.com/r-c-f/waynergy.git
cd waynergy
meson setup build --prefix=/usr/local
ninja -C build
sudo ninja -C build install
```
4) Create the uinput group and add your user
```
sudo groupadd -r uinput
sudo usermod -aG uinput "$USER"
```
#### Log out and back in after this, or reboot, so your new group membership applies.
#### Configure /dev/uinput access

5) Create a udev rule for /dev/uinput
```
echo 'KERNEL=="uinput", GROUP="uinput", MODE="0660"' | sudo tee /etc/udev/rules.d/99-uinput.rules
```
6) Reload udev rules
```
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo modprobe uinput
```
7) Make the Waynergy binary setuid root
This lets Waynergy start as root just long enough to open /dev/uinput, then drop privileges.
```
sudo chown root:uinput /usr/local/bin/waynergy
sudo chmod 4750 /usr/local/bin/waynergy
```
#### Verify the basic setup
8) Check permissions and rules
```
ls -l /dev/uinput
groups
grep -RIn --color=always "uinput" /etc/udev/rules.d /lib/udev/rules.d 2>/dev/null
```
Expected /dev/uinput ownership and mode:
```
crw-rw---- 1 root uinput ...
```
Your user should also appear in the uinput group when you run:
```
groups
```
Optional: Lock down uinput access further
#### Some systems also apply uaccess rules that grant the active desktop user access to /dev/uinput. If you want a stricter setup where only the uinput group can access it, do the following.
9) Add a local override to remove uaccess
```
sudo tee /etc/udev/rules.d/99-uinput-local.rules >/dev/null <<'EOF'
KERNEL=="uinput", SUBSYSTEM=="misc", TAG-="uaccess"
KERNEL=="uinput", GROUP="uinput", MODE="0660"
EOF
```
Reload rules:
```
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo modprobe -r uinput 2>/dev/null
sudo modprobe uinput
```
Check current ACLs:
```
getfacl /dev/uinput
```
A clean result looks like:
```
# owner: root
# group: uinput
user::rw-
group::rw-
other::---
```
If you still see an extra ACL such as:
```
user:yourusername:rw-
```
then continue with the optional sections below.
Optional: Disable Steam’s uaccess rule

#### If Steam adds uaccess to /dev/uinput on your system and you do not need Steam Input, you can mask its rule.
9) Mask Steam’s udev rule
```
sudo ln -s /dev/null /etc/udev/rules.d/60-steam-input.rules
```
Reload rules:
```
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo modprobe -r uinput 2>/dev/null
sudo modprobe uinput
```
Remove any current extra ACLs:
```
sudo setfacl -b /dev/uinput
```
Verify:
```
ls -l /dev/uinput
getfacl /dev/uinput
```
#### Optional: Disable early /dev/uinput uaccess
POP OS ships an early-creation rule for /dev/uinput that adds uaccess.
10) Mask the early-creation rule
```
sudo ln -s /dev/null /etc/udev/rules.d/71-uinput-dev-early-creation.rules
```
Reload rules again:
```
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo modprobe -r uinput 2>/dev/null
sudo modprobe uinput
sudo setfacl -b /dev/uinput
```
Verify again:
```
ls -l /dev/uinput
getfacl /dev/uinput
grep -RIn --color=always "uinput" /etc/udev/rules.d /lib/udev/rules.d 2>/dev/null
```
Final desired state:
```
crw-rw---- 1 root uinput ...
```
and:
```
# owner: root
# group: uinput
user::rw-
group::rw-
other::---
```

