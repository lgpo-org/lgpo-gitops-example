# lgpo GitOps example repo

## Policies: examples

The early MVP version of [lgpod](https://github.com/lgpo-org/lgpod) includes three kinds of policies:
- **ModprobePolicy** → kernel module allow/deny (e.g., block USB mass storage)
- **PolkitPolicy** → controls privileged actions (who can do what)  
- **DconfPolicy** → GNOME settings + locks (opinionated desktop security)  

### Block USB storage (ModprobePolicy for most Linux distributions)

```yaml
apiVersion: lgpo.io/v1
kind: ModprobePolicy
metadata: { name: block-removable-storage }
selector:
  tags:
    group: ["laptops", "kiosk"]
spec:
  blacklist: ["usb_storage", "uas", "firewire_ohci", "sbp2"]
  installFalse: true       # install <mod> /bin/false → hard-block
  updateInitramfs: true    # rebuild so block applies early
```

**Effect** → `/etc/modprobe.d/60-lgpo-block-removable-storage.conf` + `update-initramfs -u`.

---

### Snap admin only (PolkitPolicy for Ubuntu)

```yaml
apiVersion: lgpo.io/v1
kind: PolkitPolicy
metadata: { name: snap-admin-only }
selector:
  tags:
    group: ["laptops", "workstations"]
spec:
  rules:
    - name: snapd-admin
      matches:
        - action_id: io.snapcraft.snapd.manage
      subject: { group: sudo }          # only sudoers
      result: AUTH_ADMIN_KEEP           # auth once, keep session
      default_result: NO                # everyone else: deny
```

**Effect** → `/etc/polkit-1/rules.d/60-lgpo-snap-admin-only.rules`.

---

### Lockscreen baseline (DconfPolicy for GNOME)

```yaml
apiVersion: lgpo.io/v1
kind: DconfPolicy
metadata: { name: gnome-security-baseline }
selector:
  tags:
    group: ["laptops"]
spec:
  settings:
    org/gnome/desktop/session:
      idle-delay: "uint32 300"
    org/gnome/desktop/screensaver:
      lock-enabled: "true"
      lock-delay: "uint32 0"
    org/gnome/desktop/media-handling:
      automount: "false"
      automount-open: "false"
    org/gnome/desktop/privacy:
      remember-recent-files: "false"
  locks:
    - /org/gnome/desktop/session/idle-delay
    - /org/gnome/desktop/screensaver/lock-enabled
    - /org/gnome/desktop/screensaver/lock-delay
    - /org/gnome/desktop/media-handling/automount
    - /org/gnome/desktop/media-handling/automount-open
    - /org/gnome/desktop/privacy/remember-recent-files
```

**Effect** →  
`/etc/dconf/db/local.d/60-lgpo-gnome-security-baseline` and  
`/etc/dconf/db/local.d/locks/60-lgpo-gnome-security-baseline`, then `dconf update`.

---

## Facts & tags (targeting)

**facts** (auto-discovered):  
`hostname`, `os.id`, `os.version`, `has_gnome`, …

**tags** (you control):  
keys such as `group`, `ou`, `team` that contain values defined in your GitOps repo 

Use in a policy selector:

```yaml
selector:
  facts:
    has_gnome: "true"
  tags:
    group: ["developers", "devops"]
```

## Inventory: example

Your policies only will only be applied on devices with matchigs tags.

```yaml
apiVersion: lgpo.io/v1
kind: DeviceInventory
items:
  - device_pub_sha256: "7a93be12cd34ef56ab78cd90ef12ab34cd56ef78ab90cd12ef34ab56cd78ef90"
    identity: "alice@example.com"
    tags:
      group: "laptops"
      ou: "finance"
      site: "vienna"
```

Read more about device enrollement in the [lgpod repo](https://github.com/lgpo-org/lgpod). 
