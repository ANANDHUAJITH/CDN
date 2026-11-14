Good — this project already ships a full testing guide (`docs/TESTING.md`), so here's the practical path, focused on what you'd actually want to check: the fixes I made, and the rolling-update/generations system ("Time Machine and Upgrades").

## 1. First boot — verify the fixes

After installing (do this in a VM first, see below):

```bash
# Wallpaper
plasma-apply-wallpaperimage --list    # or just look: should be the crimson-night scene

# Boot splash actually using plymouth+kms now
systemctl status plymouth-start.service
cat /etc/mkinitcpio.conf.d/ketchup.conf     # should show: HOOKS=(... kms plymouth ...)
cat /etc/kernel/cmdline; cat /etc/ketchup/boot-extra   # boot-extra should now include "splash"

# Boot speed
systemd-analyze
ketchup boot-audit        # flags slow units + exact fix for each
ketchup status            # "boot time" line warns if over ~30s
```

If `boot-extra` already existed from an older install (it only gets written once, on first install), it won't have `splash` in it — just append it: `echo "quiet splash loglevel=3 nowatchdog zswap.enabled=0 noirqdebug fbcon=nodefer" | sudo tee /etc/ketchup/boot-extra && sudo ketchup-limine update`.

## 2. Testing "rolling updates" — this is `ketchup upgrade` / channels

This is the atomic-generations system (like NixOS/Windows feature updates), separate from plain `pacman`. You don't need the real internet or a real release server — `docs/TESTING.md` has a fully offline, disposable test release:

```bash
# Make yourself a throwaway signing key + a fake release server
gpg --batch --passphrase '' --quick-generate-key "Test <t@t>" default default never
sudo mkdir -p /srv/kt
mkdir -p /tmp/files/etc/ketchup && echo "hello=world" > /tmp/files/etc/ketchup/demo.conf

tools/make-release.py --channel stable --version 2099.01 --archive 2026/11/01 \
  --defaults /tmp/files --notes "test release" --sign Test \
  --export-keyring /tmp/ring.gpg --out /srv/kt

sudo cp /tmp/ring.gpg /etc/ketchup/test-ring.gpg
printf 'base_url=file:///srv/kt\nkeyring=/etc/ketchup/test-ring.gpg\n' | sudo tee /etc/ketchup/release.conf
sudo ketchup channel stable

ketchup upgrade --check        # should list the release + the new config file as "add"
sudo ketchup upgrade           # builds generation g1 BESIDE your running system — nothing changes yet
ketchup generations            # shows g0 (running) and g1 (default on next boot)
sudo reboot                    # boots g1; g0 stays in the menu as "known good"
```

After reboot (wait ~90s, or run `sudo ketchup boot-ok` immediately): `ketchup generations` should show `g1` as **good**, `cat /etc/ketchup/version` → `2099.01`, and your own files/packages untouched. Repeat with `--version 2099.02`/`2099.03` to see it keep only the newest two generations and prune automatically.

To go back to rolling: `sudo ketchup channel rolling`.

## 3. The other two safety layers, quickly

```bash
sudo ketchup checkpoint create before-test   # Time Machine snapshot
sudo ketchup update --check                  # safe pacman update, dry-run
sudo ketchup update                          # checkpoint → download → install → verify → auto-rollback if broken
```

## 4. Don't do any of this on your real machine first

```bash
./buildiso.sh
qemu-img create -f qcow2 ketchup.qcow2 40G
qemu-system-x86_64 -enable-kvm -m 6G -smp 4 -cpu host -bios /usr/share/ovmf/OVMF.fd \
  -drive file=ketchup.qcow2,if=virtio -cdrom out/ketchup-*.iso -boot d
```

Install with Calamares, **pick Btrfs** (generations/rollback need it — there's an in-place fallback for other filesystems but you lose atomicity), then run everything above inside the VM.

There's also an automated unit-test suite (`python3 tools/test-ketchup.py`, 73 tests, runs against a real loop-mounted Btrfs) if you want to sanity-check the underlying logic before even booting a VM — good first step, takes about 2 minutes. Full details and failure-testing scenarios (simulated corruption, disk-full, crash-mid-rollback, etc.) are in `docs/TESTING.md`.
