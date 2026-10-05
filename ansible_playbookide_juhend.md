# Ansible Playbookide ja Play'de Konspekt

See juhend annab selge ja praktilise ülevaate **Ansible** playbookide ehitusest, põhilistest ülesannetest (*tasks*) ja kasulikest käskudest.

---

## 📋 Sisukord

1. [Struktuur](#1-struktuur)
2. [Põhikomponendid](#2-põhikomponendid)
3. [Sagedasemad Ülesanded (Tasks)](#3-sagedasemad-ülesanded-tasks)
4. [Tingimused ja Tsüklid](#4-tingimused-ja-tsüklid)
5. [Reageerijad (Handlers)](#5-reageerijad-handlers)
6. [Käivitamine ja Kontroll](#6-käivitamine-ja-kontroll)

---

## 1. Struktuur

Ansible'i **Playbook** koosneb ühest või mitmest **play**'st. Iga *play* seob konkreetse masinate grupi spetsiifiliste tegevustega.

```yaml
---
- name: Veebiserveri seadistamine
  hosts: webservers
  become: true
  vars:
    http_port: 80

  tasks:
    - name: Paigalda Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Veendu, et Nginx töötab
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

---

## 2. Põhikomponendid

| Komponent | Kirjeldus | Näide |
| :--- | :--- | :--- |
| **`name`** | Kirjeldab tegevust või *play*'d. | `name: Paigalda veebiserver` |
| **`hosts`** | Määrab sihtmasinad (`inventory` failist). | `hosts: webservers` või `hosts: all` |
| **`become`** | Käivitab ülesande administraatori õigustes (`sudo`). | `become: true` |
| **`vars`** | Määratleb lokaalsed muutujad. | `vars: app_port: 8080` |
| **`tasks`** | Nimekiri käivitatavatest moodulitest. | Järjestikused tegevused |
| **`handlers`** | Ülesanded, mida käivitatakse vaid muudatuse tekkimisel. | Teenuste taaskäivitamine |

---

## 3. Sagedasemad Ülesanded (Tasks)

### Tarkvarapakettide haldus (`package`)
```yaml
- name: Paigalda põhitööriistad
  ansible.builtin.package:
    name:
      - git
      - curl
      - htop
    state: present # present, absent, latest
```

### Failid ja Mallid (`copy` / `template`)
* **`copy`**: Kopeerib faili muutumatul kujul.
* **`template`**: Asendab Jinja2 (`.j2`) mallis muutujad enne kopeerimist.

```yaml
- name: Loo seadistusfail mallist
  ansible.builtin.template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: Taaskäivita Nginx
```

### Teenuste haldus (`service`)
```yaml
- name: Käivita ja automatiseeri teenus
  ansible.builtin.service:
    name: nginx
    state: started # started, stopped, restarted, reloaded
    enabled: true
```

### Käsureakäsud (`command` / `shell`)
```yaml
- name: Käivita skript käsurealt
  ansible.builtin.command: /usr/local/bin/setup.sh
  args:
    creates: /var/log/setup.log # Ei käivitu uuesti, kui fail on olemas
```

---

## 4. Tingimused ja Tsüklid

### Tingimuslaused (`when`)
```yaml
- name: Uuenda paketid ainult Debian-põhistes süsteemides
  ansible.builtin.apt:
    update_cache: true
  when: ansible_facts['os_family'] == "Debian"
```

### Tsüklid (`loop`)
```yaml
- name: Loo mitu kasutajat
  ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - peeter
    - mari
    - jaan
```

---

## 5. Reageerijad (Handlers)

*Handler* käivitub alles *play* lõpus ja **ainult juhul**, kui teda teavitanud ülesanne tegi süsteemis reaalne muudatuse (olek `changed`).

```yaml
tasks:
  - name: Muuda konfiguratsiooni
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Taaskäivita Nginx

handlers:
  - name: Taaskäivita Nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

---

## 6. Käivitamine ja Kontroll

* **Süntaksi kontroll:**
  ```bash
  ansible-playbook --syntax-check playbook.yml
  ```

* **Testkäivitus ehk simulatsioon (*Dry-run*):**
  ```bash
  ansible-playbook playbook.yml --check
  ```

* **Playbooki käivitamine:**
  ```bash
  ansible-playbook -i inventory.ini playbook.yml
  ```