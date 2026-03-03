# Ansible Szerver Management

Ansible alapú szerver management projekt, amely Docker konténereken demonstrálja a felhasználókezelést, csomagelemzést és biztonsági auditot.

---

## Előfeltételek

```bash
# Docker
sudo apt update && sudo apt install docker.io -y
sudo systemctl start docker && sudo systemctl enable docker

# Ansible
sudo apt install ansible -y
```

---

## Első indítás

### 1. Docker konténerek felépítése

```bash
docker build -t ansible-node .
docker run -d --name node1 -p 2221:22 ansible-node
docker run -d --name node2 -p 2222:22 ansible-node
```

### 2. Ansible Vault beállítása

A vault titkosítja az érzékeny adatokat (pl. jelszavak). Hozd létre a vault jelszófájlt:

```bash
echo "vault_jelszavad" > .vault_pass
chmod 600 .vault_pass
```

> `.vault_pass` szerepel a `.gitignore`-ban — soha ne commitold be.

### 3. SSH host key-ek begyűjtése

```bash
ssh-keyscan -H -p 2221 localhost >> ~/.ssh/known_hosts
ssh-keyscan -H -p 2222 localhost >> ~/.ssh/known_hosts
```

### 4. Ansible service user létrehozása

Ez a lépés:
- Létrehozza az `ansible` usert a node-okon SSH kulcsos hozzáféréssel
- Beállítja a jelszó nélküli sudo jogot
- Letiltja a root SSH logint

```bash
# SSH kulcspár generálása (ha még nincs)
ssh-keygen -t ed25519 -f ~/.ssh/ansible_id -N "" -C "ansible automation key"

# Setup playbook futtatása (egyszer, root userrel)
ansible-playbook -i hosts.yml setup_ansible_user.yml \
  -e ansible_user=root \
  -e ansible_password=root \
  -e "ansible_ssh_private_key_file="

# Kapcsolat tesztelése
ansible all -m ping -i hosts.yml
```

### 5. Csomag baseline létrehozása

```bash
ansible-playbook -i hosts.yml get_packets.yml
```

---

## Playbook referencia

### `create_user.yml` – Felhasználó létrehozása

Létrehoz egy felhasználót a szervereken jelszóval és opcionális csoporttagsággal. Az `admins` csoportba kerülő felhasználók sudo jogot kapnak.

**Jelszó követelmények:** minimum 12 karakter, kis- és nagybetű, szám.

```bash
# Egyszerű felhasználó
ansible-playbook -i hosts.yml create_user.yml \
  -e "user=janos" \
  -e "password=TitkosJelszo123"

# Admin felhasználó sudo joggal
ansible-playbook -i hosts.yml create_user.yml \
  -e "user=adminuser" \
  -e "password=AdminPass456" \
  -e "user_groups=['admins']"
```

---

### `remove_user.yml` – Felhasználó eltávolítása

Biztonságosan eltávolít egy felhasználót: leállítja a folyamatait (SIGTERM, majd SIGKILL), törli a home könyvtárát és a sudoers bejegyzését.

> A `root` felhasználó törlése blokkolva van.

```bash
ansible-playbook -i hosts.yml remove_user.yml -e "user=janos"

# Dry-run (csak megmutatja, mit tenne)
ansible-playbook -i hosts.yml remove_user.yml -e "user=janos" --check
```

---

### `get_users.yml` – Felhasználói audit riport

Részletes riportot készít minden szerver felhasználóiról:
- Felhasználók adatai, csoporttagságok
- Jelszó állapot és lejárati adatok
- Bejelentkezési előzmények (`last`, `lastlog`, `lastb`)
- Auth log és sudo napló

A riport titkosítva (`ansible-vault`) és `chmod 600` védelemmel kerül mentésre.

```bash
ansible-playbook -i hosts.yml get_users.yml

# Bash history begyűjtéssel (alapból ki van kapcsolva)
ansible-playbook -i hosts.yml get_users.yml -e "collect_history=true"

# Riport megtekintése
ansible-vault view user_report_node1.txt
```

---

### `get_packets.yml` – Csomag baseline mentése

Elmenti az összes telepített csomag nevét és verzióját baseline fájlba (`./baselines/`). Ezt kell először futtatni, mielőtt a `check_packets.yml`-t használnád.

```bash
# Baseline létrehozása (csak ha még nem létezik)
ansible-playbook -i hosts.yml get_packets.yml

# Baseline frissítése – régi diff fájlok automatikusan törlődnek
ansible-playbook -i hosts.yml get_packets.yml -e "force_update=true"
```

---

### `check_packets.yml` – Csomag változáskeresés

Összehasonlítja az aktuális csomaglistát a baseline-nal. Ha van eltérés, titkosított diff fájlt ment (`./diffs/`). A 30 napnál régebbi diff fájlok automatikusan törlődnek.

```bash
# Változások ellenőrzése
ansible-playbook -i hosts.yml check_packets.yml

# Egyéni retention időszak
ansible-playbook -i hosts.yml check_packets.yml -e "diff_retention_days=90"

# Diff megtekintése
ansible-vault view diffs/diff_node1_2026-03-03.txt
```

---

## Automatizálás (cron)

```bash
# Naponta futtatva: változásfigyelés és régi diffek törlése
0 6 * * * cd /home/ansible && ansible-playbook -i hosts.yml check_packets.yml

# Baseline frissítése tervezett karbantartás után
ansible-playbook -i hosts.yml get_packets.yml -e "force_update=true"
```

---

## Konténerek újraindítása

```bash
docker stop node1 node2 && docker rm node1 node2
ssh-keygen -f "$HOME/.ssh/known_hosts" -R "[localhost]:2221"
ssh-keygen -f "$HOME/.ssh/known_hosts" -R "[localhost]:2222"
docker run -d --name node1 -p 2221:22 ansible-node
docker run -d --name node2 -p 2222:22 ansible-node

# SSH key-ek újbóli begyűjtése
ssh-keyscan -H -p 2221 localhost >> ~/.ssh/known_hosts
ssh-keyscan -H -p 2222 localhost >> ~/.ssh/known_hosts

# setup_ansible_user.yml újrafuttatása (root userrel)
ansible-playbook -i hosts.yml setup_ansible_user.yml \
  -e ansible_user=root \
  -e ansible_password=root \
  -e "ansible_ssh_private_key_file="
```

---

## Biztonsági megjegyzések

| Terület | Megvalósítás |
|---|---|
| SSH autentikáció | Kulcsalapú (`~/.ssh/ansible_id`) |
| Root SSH login | Tiltva (`PermitRootLogin no`) |
| Host key ellenőrzés | Bekapcsolva (`StrictHostKeyChecking=yes`) |
| Érzékeny adatok | Ansible Vault AES256 titkosítással |
| Riportfájlok | Titkosítva + `chmod 600` |
| Ansible service user | `NOPASSWD: ALL` sudo (kulcs védi) |

> `~/.ssh/ansible_id` (privát kulcs) és `.vault_pass` biztonságos tárolása elengedhetetlen.
