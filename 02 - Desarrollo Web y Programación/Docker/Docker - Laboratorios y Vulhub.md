---
title: "Docker - Laboratorios y Vulhub"
tags:
  - lang/docker
  - concept/vulnerable-environments
  - concept/ctf
  - concept/security-labs
  - level/intermedio
  - type/reference
  - area/security
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Docker - Laboratorios y Vulhub

> **Objetivo**: Desplegar entornos vulnerables controlados para práctica de hacking ético, CTFs, y testing de seguridad.

---

## 1. Vulhub - Repositorio Principal

**Vulhub** (https://github.com/vulhub/vulhub) es la colección más completa de entornos vulnerables dockerizados.

### 1.1 Estructura del Repositorio

```
vulhub/
├── apache/
│   ├── CVE-2021-41773/          # Path traversal
│   └── CVE-2021-42013/
├── docker/
│   ├── CVE-2019-5736/           # runc escape
│   └── unauthorized-access/     # Docker daemon sin auth
├── elasticsearch/
│   └── CVE-2014-3120/           # RCE via script
├── jenkins/
│   └── CVE-2018-1999002/        # RCE
├── kafka/
│   └── CVE-2018-1266/
├── kubernetes/
│   └── CVE-2018-1002105/
├── mysql/
│   └── CVE-2016-6662/
├── nginx/
│   ├── CVE-2013-2028/
│   └── CVE-2017-7529/
├── redis/
│   ├── CVE-2022-0543/           # Lua sandbox escape
│   └── unauthorized-access/
├── spring/
│   ├── CVE-2018-1273/
│   └── CVE-2022-22965/          # Spring4Shell
├── struts2/
│   ├── CVE-2017-5638/           # Jakarta Multipart RCE
│   └── CVE-2017-9805/
├── tomcat/
│   ├── CVE-2017-12615/          # PUT RCE
│   └── CVE-2020-1938/           # Ghostcat
└── wordpress/
    ├── CVE-2019-6977/
    └── CVE-2021-29447/
```

### 1.2 Uso Básico

```bash
# Clonar solo lo necesario (recomendado)
# Opción A: downgit (descarga subcarpeta)
pip install downgit
downgit https://github.com/vulhub/vulhub/tree/master/apache/CVE-2021-41773

# Opción B: svn export (sin git history)
svn export https://github.com/vulhub/vulhub/trunk/apache/CVE-2021-41773

# Opción C: Clonar completo (pesado ~2GB)
git clone https://github.com/vulhub/vulhub.git
```

### 1.3 Levantar Entorno

```bash
cd apache/CVE-2021-41773

# Ver docker-compose.yml primero
cat docker-compose.yml

# Levantar en background
docker compose up -d

# Ver logs
docker compose logs -f

# Verificar funcionamiento
curl http://localhost:8080

# Parar y limpiar
docker compose down
docker compose down -v  # Con volúmenes
```

---

## 2. Entornos Vulnerables Populares

### 2.1 Web Applications

| Entorno | Puerto | Vulnerabilidad | Dificultad |
|---------|--------|----------------|------------|
| `dvwa` | 80 | XSS, SQLi, CSRF, File Upload | Principiante |
| `bWAPP` | 80 | 100+ vulns web | Principiante |
| `webgoat` | 8080 | OWASP Top 10 interactivo | Intermedio |
| `mutillidae` | 80 | OWASP Top 10 | Principiante |
| `juice-shop` | 3000 | OWASP Top 10, moderna | Intermedio |

```bash
# DVWA (Damn Vulnerable Web App)
docker run -d --name dvwa -p 80:80 vulnerables/web-dvwa

# OWASP Juice Shop
docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop

# bWAPP
docker run -d --name bwapp -p 80:80 raesene/bwapp

# WebGoat
docker run -d --name webgoat -p 8080:8080 -p 9090:9090 webgoat/webgoat-8.0
```

### 2.2 CMS Vulnerables

```bash
# WordPress vulnerable
docker run -d --name wp -p 80:80 wordpress:4.9.8

# Joomla vulnerable
docker run -d --name joomla -p 80:80 joomla:3.8.13

# Drupal vulnerable (Drupalgeddon2)
docker run -d --name drupal -p 80:80 drupal:7.58
```

### 2.3 Servicios y Protocolos

| Entorno | Puerto | Vulnerabilidad |
|---------|--------|----------------|
| `msfconsole` | - | Metasploit framework |
| `metasploitable2` | Múltiples | Linux vulnerable completo |
| `metasploitable3` | Múltiples | Windows/Linux vulnerable |
| `samba` | 445 | CVE-2017-7494 (SambaCry) |
| `ftp` | 21 | vsftpd 2.3.4 backdoor |
| `ssh` | 22 | Credenciales débiles |
| `ldap` | 389 | Anonymous bind |

```bash
# Metasploitable2
docker run -d --name msf2 -p 21:21 -p 22:22 -p 80:80 -p 445:445 -p 3306:3306 tleemcjr/metasploitable2

# Metasploitable3 (Windows)
docker run -d --name msf3 -p 80:80 -p 445:445 -p 3389:3389 rapid7/metasploitable3-win2k8

# Samba vulnerable
docker run -d --name samba -p 445:445 vulhub/samba:4.5.9
```

---

## 3. Descarga Selectiva con `downgit` / `svn`

### 3.1 Script Automatizado

```bash
#!/bin/bash
# download_lab.sh - Descargar lab específico de Vulhub
# Uso: ./download_lab.sh "apache/CVE-2021-41773"

LAB_PATH="$1"
BASE_URL="https://github.com/vulhub/vulhub/trunk"
DEST_DIR="labs/$(basename "$LAB_PATH")"

if [[ -z "$LAB_PATH" ]]; then
    echo "Uso: $0 <ruta/lab>"
    echo "Ejemplo: $0 apache/CVE-2021-41773"
    exit 1
fi

mkdir -p "$(dirname "$DEST_DIR")"
svn export "$BASE_URL/$LAB_PATH" "$DEST_DIR"
echo "Lab descargado en: $DEST_DIR"
```

### 3.2 Listar Labs Disponibles

```bash
# Ver estructura sin clonar
svn list https://github.com/vulhub/vulhub/trunk

# Buscar por tecnología
svn list https://github.com/vulhub/vulhub/trunk | grep -E "(apache|nginx|tomcat|spring)"
```

---

## 4. Otras Fuentes de Labs

### 3.1 VulnHub (VMs Completas)

```bash
# Descargar OVA/VMware y convertir a Docker (avanzado)
# O usar imágenes preconvertidas:
docker pull vulnub/basic-pentesting
docker pull vulnub/kioptrix
```

### 3.2 HackTheBox / TryHackMe

```bash
# Algunas máquinas liberadas como Docker
docker pull hackthebox/academy
docker pull tryhackme/room-name
```

### 3.3 CTFd (Plataforma CTF)

```bash
# Plataforma completa para CTFs
docker run -d --name ctf -p 8000:8000 ctfd/ctfd
```

---

## 4. Crear tu Propio Lab Vulnerable

### 4.1 Estructura Mínima

```
mi-lab/
├── docker-compose.yml
├── Dockerfile
├── vulnerable-app/
│   └── app.py
└── README.md
```

### 4.2 Dockerfile Vulnerable (Ejemplo: RCE)

```dockerfile
# Dockerfile - App Flask con RCE intencional
FROM python:3.11-slim

WORKDIR /app
COPY vulnerable-app/ .

RUN pip install flask

# Usuario no-root (buena práctica incluso en labs)
RUN useradd -r -s /bin/bash labuser
USER labuser

EXPOSE 5000
CMD ["python", "app.py"]
```

### 4.3 App Vulnerable (Flask RCE)

```python
# vulnerable-app/app.py
from flask import Flask, request
import subprocess

app = Flask(__name__)

@app.route("/ping")
def ping():
    # VULNERABLE: Inyección de comandos
    host = request.args.get("host", "127.0.0.1")
    # Sin validación ni sanitización
    result = subprocess.check_output(f"ping -c 3 {host}", shell=True, text=True)
    return f"<pre>{result}</pre>"

@app.route("/")
def index():
    return '''
    <h1>Lab RCE - Ping</h1>
    <form action="/ping">
        Host: <input name="host" value="127.0.0.1">
        <input type="submit" value="Ping">
    </form>
    <p>Prueba: <code>127.0.0.1; id</code></p>
    '''

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 4.4 docker-compose.yml

```yaml
version: "3.8"
services:
  rce-lab:
    build: .
    ports:
      - "5000:5000"
    restart: unless-stopped
    # Sin red externa para aislamiento
    networks: []
```

---

## 5. Aislamiento y Seguridad en Labs

### 5.1 Red Aislada (Sin Internet)

```yaml
version: "3.8"
services:
  vulnerable:
    build: .
    networks:
      - lab-net
    # Sin ports publicados al host
    # Acceso solo via docker exec o contenedor atacante

  attacker:
    build: ./attacker  # Kali, Parrot, o custom
    networks:
      - lab-net
    cap_add:
      - NET_ADMIN
      - NET_RAW

networks:
  lab-net:
    driver: bridge
    internal: true  # ¡Clave! Sin salida a internet
```

### 5.2 Contenedor Atacante (Kali/Parrot)

```dockerfile
# attacker/Dockerfile
FROM kalilinux/kali-rolling

RUN apt-get update && apt-get install -y \
    nmap \
    curl \
    netcat \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /root
CMD ["bash"]
```

### 5.3 Uso del Lab Aislado

```bash
# Levantar lab
docker compose up -d

# Atacar desde contenedor atacante
docker compose exec attacker bash
# Dentro:
nmap -sV vulnerable
curl http://vulnerable:5000/ping?host=127.0.0.1\;id

# O desde host via docker exec
docker compose exec vulnerable bash
```

---

## 6. Herramientas de Escaneo en Docker

```bash
# Trivy (vulnerabilidades en imágenes)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image miapp:latest

# Grype (Syft + Grype)
docker run --rm anchore/grype miapp:latest

# Hadolint (Dockerfile linting)
docker run --rm -i hadolint/hadolint < Dockerfile

# Dockle (security audit)
docker run --rm goodwithtech/dockle miapp:latest

# Checkov (IaC scanning)
docker run --rm -v $(pwd):/tf bridgecrew/checkov -d /tf
```

---

## 7. Checklist de Lab Seguro

```
[ ] Red interna (internal: true) sin salida a internet
[ ] Contenedor vulnerable sin privilegios innecesarios
[ ] Contenedor atacante con tools necesarias
[ ] Volúmenes para persistencia de evidencias
[ ] Logs centralizados
[ ] Documentación del exploit/POC
[ ] Limpieza automática (docker compose down -v)
[ ] Versión fija de imágenes (tags, no latest)
```

---

## 8. Referencias Cruzadas

- [Fundamentos y Conceptos](./Docker%20-%20Fundamentos%20y%20Conceptos.md)
- [Docker Compose](./Docker%20-%20Docker%20Compose.md)
- [Redes y Volúmenes](./Docker%20-%20Redes%20y%20Vol%C3%BAmenes.md)
- [Buenas Prácticas y Seguridad](./Docker%20-%20Buenas%20Pr%C3%A1cticas%20y%20Seguridad.md)

---

*Etiquetas: #docker #vulhub #labs #vulnerable-environments #ctf #dvwa #juice-shop #metasploitable #security-training #aislamiento*