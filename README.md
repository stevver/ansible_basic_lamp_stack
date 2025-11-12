# LAMP Stack Ansible

## Autor
Stever Heinsaar, IT-23

## Kirjeldus
See projekt paigaldab ja seadistab **LAMP stack’i (Linux, Apache, MySQL, PHP)** ühele või mitmele **Ubuntu 24.04** serverile **Ansible** abil.  
Struktuur järgib paremaid tavasid: muutujad `group_vars/` all, teenused eraldi playbook’ides, Apache vhost ja PHP testileht on Jinja2 template’id, ning katus-play `site.yml` impordib kõik alammängud.  
Lahendus on **konfigureeritav**: õpetaja muudab ainult **`inventory.ini`** ja **`group_vars/all/main.yml`**, seejärel käivitab **ühe käsu**.

---

## Eeldused
- Juhtmasinas Ansible (testitud Ubuntu 24.04 peal)
- Sihtmasinad: **Ubuntu Server 24.04**
- SSH ligipääs juhtmasinast sihtmasinasse
- Sudo-õigused sihtmasinates (Ansible küsib sudo parooli automaatselt — vt `ansible.cfg`)

---

## Failide struktuur
```
ansible_lamp/
├── ansible.cfg
├── inventory.ini
├── playbooks/
│   ├── site.yml          # master-play (impordib kõik ülejäänud)
│   ├── system.yml        # süsteemi ettevalmistus (uuendused, tööriistad, UFW, ajavöönd)
│   ├── apache.yml        # Apache paigaldus + vhost
│   ├── mysql.yml         # MySQL paigaldus + DB + kasutaja (TCP üle 127.0.0.1:3306)
│   └── php.yml           # PHP + laiendused + testileht
├── group_vars/
│   ├── all/
│   │   └── main.yml      # globaalsed muutujad (muudab õpetaja)
│   └── lamp_servers/
│       └── main.yml      # grupipõhised muutujad (vajadusel)
├── templates/
│   ├── vhost.conf.j2     # Apache virtual host template
│   └── index.php.j2      # PHP testilehe template
└── docs/
    └── screenshot.png    # ekraanipilt töötavast lehetest
```

---

## Seadistamine

### 1) Inventory (`inventory.ini`)
Määra sihtserver(id) ja SSH kasutaja:
```ini
[lamp_servers]
server1 ansible_host=192.168.56.102 ansible_user=kasutaja
```

### 2) Globaalsed muutujad (`group_vars/all/main.yml`)
Muuda vastavalt oma keskkonnale:
```yaml
# Failide omanik veebijuurikas
student_username: "kasutaja"

# Veeb / vhost seaded
domain_name: "lamp.local"
server_admin: "webmaster@lamp.local"
document_root: "/var/www/lamp"

# Ansible privileegi tõstmine
ansible_become: true
# Sudo parooli küsitakse automaatselt (vt ansible.cfg -> become_ask_pass=True)
```

---

## Käivitamine

Repo juurkataloogist:
```bash
ansible-playbook playbooks/site.yml
```

- Kuvatakse **sudo parooli** küsimine (sest `become_ask_pass=True`).
- **MySQL** ja **PHP** playbook’id küsivad:
  - **MySQL root parool**
  - **Rakenduse andmebaasi kasutaja parool**  
  (Paroole ei hoita koodis.)

---

## Mida playbookid teevad

- **System**
  - Turvalised uuendused, baas-tööriistad, ajavöönd
  - **UFW**: lubatud pordid **22, 80, 443**
- **Apache**
  - `apache2` + moodulid `rewrite`, `ssl`, `headers`
  - Vhost Jinja2 templatega (`ServerName`, `DocumentRoot`, jm)
- **MySQL**
  - `mysql-server`, `python3-pymysql`
  - Andmebaas ja kasutaja luuakse **TCP** kaudu (127.0.0.1:3306), vältimaks soketiga seotud tõrkeid
- **PHP**
  - PHP 8.x + laiendused (mysql, curl, gd, mbstring, xml, zip)
  - Testileht `index.php` serveri info ja **MySQL ühenduse testiga**

---

## Testimine

### 1) Leht on kättesaadav
Ava brauseris:
- `http://<sihtserveri-ip>/` või `http://<sihtserveri-ip>/index.php`

### 2) Oodatav tulemus
Lehel kuvatakse **“LAMP Stack Test”** ja **“MySQL connection: OK”**.

### 3) Ekraanipilt
Tõendus asub eraldi failina juur kaustas `Screenshot 2025-11-12 161734.png`

---

## Idempotentsus

Käivita uuesti kontrollrežiimis:
```bash
ansible-playbook playbooks/site.yml
```
Teisel käivitamisel peaks enamik samme olema **ok** (mitte **changed**).

---

## Tõrkeotsing

- **“Missing sudo password”**  
  Veendu, et jooksutad repo juurkataloogist ja `ansible.cfg` loetakse sisse.  
  Kontrolli:
  ```bash
  ansible --version
  ```
  Otsi rida:
  ```
  config file = /home/kasutaja/ansible_lamp/ansible.cfg
  ```

- **Apache annab 403** vahetult pärast vhost’i lubamist  
  See on normaalne enne, kui `index.php` on paigas. Apache testi task lubab **200 või 403**, kuni PHP on laetud.

- **MySQL soketi/taaskäivituse veidrused**  
  Projekt kasutab meelega **TCP** ligipääsu (127.0.0.1:3306), et vältida soketi võistlust.  
  Kui oled käsitsi proovinud ja teenus ei käivitu, sihtmasinas:
  ```bash
  sudo pkill -f mysqld_safe || true
  sudo pkill -f mysqld || true
  sudo rm -f /run/mysqld/mysqld.sock /run/mysqld/mysqld.pid
  sudo install -o mysql -g mysql -m 755 -d /run/mysqld
  sudo chown -R mysql:mysql /var/lib/mysql
  sudo systemctl start mysql
  ```


## Refleksioon

**1. Mis oli kõige raskem ja kuidas lahendasid?**  
Kõige keerulisem oli MySQL algseadistus ja soketiga seotud “race” olukorrad. Lahendasin selle, vahetades tegevused **TCP** ühenduse peale (127.0.0.1:3306) ning lisades selged oote- ja kontrollisammud.

**2. Suurim “ahaa!” Ansible’is ja miks?**  
**Idempotentsus** — sama playbook’i võib turvaliselt korduvalt käivitada ning tulemus on alati sama. See teeb automatiseerimise usaldusväärseks.

**3. Kus kasutaksid Ansible’it edaspidi?**  
Kodulabori serverite kiirseadistamiseks ja uute VM-ide ülesehitamiseks minutitega. Uuendused ja kordustööd muutuvad deklaratiivseks ja kiireks.

**4. Kuidas seletaksid sõbrale, mis on Ansible?**  
See on nagu **kaugjuhtimispult serveritele** — kirjeldad soovitud seisu ning Ansible viib üks või sada serverit sellele vastavusse.

**5. Mis oli kõige huvitavam?**  
Template’ide kasutamine. Ühest mallist saab luua erinevatele keskkondadele õiged konfiguratsioonifailid — justkui “kood konfi jaoks”.

---

Ansible küsib sudo parooli automaatselt. MySQL ja PHP playbook’id küsivad vajadusel andmebaasi paroole.
