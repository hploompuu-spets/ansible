# K1 Praktikum: Esimene playbook — Konspekt ja Juhend

## 1. Eesmärgid ja Õpiväljundid

### Peamine eesmärk

Selle praktikumi lõpuks viiakse kolm serverit (`vm1`, `vm2`, `vm3`) täpselt samasse soovitud olekusse:

1. Süsteemikasutaja `saidi` on loodud.

2. Veebiserver `nginx` on paigaldatud, käivitatud ning seadistatud automaatselt taaskäivituma buutimisel.

3. Tulemüüris (`firewalld`) on lubatud HTTP liiklus.

4. Nginxi avaleht kuvab serveri inventari nime (`inventory_hostname`).

5. Korduv playbooki käivitamine on täielikult **idempotentne** (`changed=0`).

### Õpiväljundid

Praktikumi läbimisel oskad:

* Tõestada Bashi skripti puudusi ja mitte-idempotentsust ning asendada selle declarative Ansible playbookiga.

* Seadistada Ansible'i keskkonda (`ansible.cfg`, `inventory.ini`), kasutada ad-hoc käske ning koguda süsteemi fakte (`setup` moodul).

* Kirjutada playbooke sisseehitatud idempootsete moodulitega (`user`, `package`, `copy`, `service`, `firewalld`).

* Ennetada ja eelvaadata muudatusi käsuga `ansible-playbook --check --diff` ning tuvastada/parandada süsteemi "triivi" (*drift*).

* Konfigureerida võtmepõhist SSH-ühendust lühinimedega (`~/.ssh/config`) ja rakendada sama playbooki mitmele sihtmasinale korraga.

* Dokumenteerida töö tulemusi, hallata logifaile ning läbida automaatkontrollid (Autograde K1–K5).

## 2. Keskkonna Arhitektuur ja Struktuur

### Control Node ja Sihtmasinad

* **Control Node (Juhtmasin):** Masin, kus Ansible jookseb. Selles praktikumis on juhtmasinaks **`vm1`**.

* Target Nodes **(Sihtmasinad):** `vm1`, `vm2`, `vm3`. Märkus: `vm1` on ühtaegu nii juhtmasin kui ka üks sihtmasinatest.

* **Märkus opsüsteemide kohta:** Windows ei sobi Ansible control node'iks (Ansible ei toeta Windowsi juhtmasinana), seetõttu tehakse töö Linux keskkonnas (`vm1` või WSL).

### Projekti Failipuu Juurkaustas

```
/
├── ansible.cfg          # Ansible'i konfiguratsioon
├── inventory.ini        # Masinate ja gruppide inventar
├── bootstrap.yml        # Peamine paigaldus- ja seadistusplaybook
├── halb.sh              # Näidisskript mitte-idempotentsuse demonstreerimiseks
├── README.md            # Projekti dokumentatsioon
└── logid/
    ├── teine_jooks.txt  # Localhosti idempotentsuse tõend (changed=0)
    └── kolm_masinat.txt # Kolme masina idempotentsuse tõend (changed=0)

```

## 3. A-Osa: Käsitsi Seadistus ja Kontrolltabel

Enne automatiseerimist tuleb selgeks teha käsitsisammud ja nende kontrollimehhanismid.

### A1. Seadistuse Kontrolltabel

| Samm | Käsitsikäsk | Oodatav tulemus / Objekt | Kontrollkäsk | 
| ----- | ----- | ----- | ----- | 
| **1. Kasutaja** | `sudo useradd -m saidi` | Kasutaja `saidi` ja kodukaust `/home/saidi` tekitatud | `id saidi` | 

`ls -d /home/saidi` | | **2. Pakett** | `sudo dnf install -y nginx` | Pakett `nginx` paigaldatud süsteemi | `rpm -q nginx` | | **3. Avaleht** | \`echo "

# Tere käsitsi

" | sudo tee /usr/share/nginx/html/index.html`| Avalehe fail sisuga olemas |`cat /usr/share/nginx/html/index.html`| | **4. Teenus** |`sudo systemctl enable --now nginx`| Teenus töötab ja on automaatselt käivituv |`systemctl is-active nginx\`

`systemctl is-enabled nginx` |

### Veaotsing käsitsi seadistusel:

Kui veebiserver ei vasta, kontrolli logisid ja olekut:

```
sudo systemctl status nginx
sudo journalctl -u nginx -n 20
sudo tail -n 20 /var/log/nginx/error.log

```

## 4. Miks Shell Skript Pole Idempotentne? (A2)

**Idempotentsus:** Omadus, kus tegevuse mitmekordne rakendamine annab täpselt sama tulemuse kui ühekordne rakendamine, ilma kõrvalmõjudeta.

### Näidisfail `halb.sh`:

```
#!/usr/bin/env bash
useradd -m raporteerija
mkdir /srv/raport
echo "seade=1" >> /srv/raport/conf

```

### Probleemid topeltkäivitamisel:

1. `useradd` annab vea: `user 'raporteerija' already exists`.

2. `mkdir` annab vea: `cannot create directory '/srv/raport': File exists`.

3. `echo ... >>` lisab **vaikselt iga kord uue rea** faili `/srv/raport/conf`. Kuu aja pärast igapäevaselt cronist käies oleks seal 30 korduvat rida `seade=1`.

> **Põhimõte:** Bashi skriptis peaks iga rea ette kirjutama manuaalsed kontrollid (`id ... ||`, `mkdir -p`, `grep -qx ... ||`). Ansible moodulid teevad need kontrollid automaatselt süsteemi olekut kontrollides (*declarative approach*).

## 5. Konfiguratsioon ja Ad-hoc Käsud (A3)

### `inventory.ini`

```
[kohalik]
localhost ansible_connection=local

```

### `ansible.cfg`

```
[defaults]
inventory = inventory.ini 
callback_result_format = yaml 

[privilege_escalation]
become_ask_pass = True 

[ssh_connection]
pipelining = True 

```

### Konfiguratsiooni kontroll ja ad-hoc käsud

```
# Kontrolli, et kasutatakse õiget cfg-faili
ansible --version | grep "config file"
ansible-inventory --graph

# Pingi testimine (kasutab pythonit sihtmasinas)
ansible kohalik -m ping

# Süsteemi faktide pärimine
ansible kohalik -m setup -a "filter=ansible_distribution*"

# Süsteemi käsu käivitamine (Command on ALATI CHANGED status-ega!)
ansible kohalik -m command -a "uptime"

# Paketi paigaldus ad-hoc käskkirjaga
ansible kohalik -b -m package -a "name=tree state=present"

```

## 6. Täielik Playbook: `bootstrap.yml` (A4–A8 & B4–B5)

See playbook hõlmab muutujate kasutamist, kasutaja haldust, pakettide paigaldust,