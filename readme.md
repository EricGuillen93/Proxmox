<h1 align="center">🚀 Despliegue de Servicios en Máquina Cliente vía X11 y MobaXterm</h1>

### 💻 Escenario de Trabajo

En esta práctica operamos sobre una arquitectura de red segmentada en **Proxmox**. El flujo de trabajo se divide en tres capas:

1.  **PC Host (Windows):** Nuestra estación base con **MobaXterm**. Aquí es donde recibimos la señal gráfica y gestionamos el terminal.
2.  **Router (Ubuntu):** El encargado de gestionar el tráfico mediante IPTables para que la red externa llegue al nodo interno.
3.  **Cliente (Ubuntu):** Es el núcleo de la práctica (IP `10.10.10.2`). En este nodo es donde se orquestan los **servicios de mensajería** (Postfix y Dovecot) y donde corre la aplicación de usuario **Thunderbird**.

> **Objetivo:** Configurar la Máquina Cliente para que funcione como un nodo de mensajería completo, permitiendo el acceso gráfico remoto y la gestión de identidades corporativas bajo el dominio `cherjo.com`.

---

### 🛡️ Paso 1: Túneles y Redirección de Puertos (Infraestructura)

Lo primero que necesitamos es preparar la infraestructura para que nos deje entrar y, además, permita exportar la interfaz gráfica desde la red privada.

#### 1.1 Reglas de IPTables en el Router
Como la Máquina Cliente está "escondida" en una red privada, el Router debe mapear los puertos para que cuando toquemos su IP, la petición salte directamente al Cliente.

```bash
# Limpieza de reglas y mapeo de puertos (DNAT)
sudo iptables -t nat -F

# Redirigimos SSH (2222), SMTP (25) e IMAP (143) hacia la IP del Cliente 10.10.10.2
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 2222 -j DNAT --to-destination 10.10.10.2:22
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 25 -j DNAT --to-destination 10.10.10.2:25
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 143 -j DNAT --to-destination 10.10.10.2:143
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

#### 1.2 Configuración de la Máquina Cliente (SSH)
Con el puerto ya abierto en el Router, configuramos el servicio SSH en el Cliente para que sea capaz de enviar ventanas gráficas a través del túnel.

```bash
# Entramos a la configuración del servicio SSH
sudo nano /etc/ssh/sshd_config

# --- AJUSTES TÉCNICOS ---
# X11Forwarding yes -> Habilita el envío de paquetes gráficos a aplicaciones externas.
# AddressFamily inet -> Forzamos IPv4 para evitar conflictos de resolución de red.
# ------------------------

sudo systemctl restart ssh
```

#### 1.3 Acceso desde el PC Local (MobaXterm)
Una vez que el Router tiene las puertas abiertas y el Cliente permite el paso de gráficos, configuramos nuestro terminal en Windows. En **MobaXterm**, creamos una nueva sesión SSH con los siguientes parámetros críticos para que el "salto" sea exitoso:

* **Remote Host:** `192.168.109.53` (IP externa del Router).
* **Port:** `2222` (El puerto que configuramos en IPTables para que nos redirija internamente a la Máquina Cliente).
* **Username:** `cliente` (El usuario con el que operaremos en el nodo interno).
* **X11-Forwarding:** Esta casilla debe estar **marcada**. Es la que permite que, al ejecutar `thunderbird &`, la ventana "viaje" por el túnel SSH y aparezca en nuestro escritorio de Windows.

<div align="center">
  <table>
    <tr>
      <td align="center">
        <b>⚙️ Configuración de Sesión en MobaXterm</b><br>
        <img src="https://github.com/user-attachments/assets/5a386371-05eb-45f6-afbe-14e277e3f95d" width="550">
      </td>
    </tr>
  </table>
</div>

> **Nota técnica:** Al conectar por el puerto 2222, el Router recibe el paquete y, gracias a la regla de PREROUTING que definimos antes, lo reenvía de forma transparente al puerto 22 de la Máquina Cliente (`10.10.10.2`).

---

### 📧 Paso 2: Setup de los Servicios en la Máquina Cliente

Aquí es donde configuramos la lógica de mensajería. Necesitamos que los dos servicios principales (Postfix y Dovecot) se sincronicen perfectamente.

#### 2.1 Postfix (Gestión de envío SMTP)
Configuramos el MTA para que sepa que él es el responsable de los correos de `cherjo.com`.

```bash
sudo nano /etc/postfix/main.cf

# --- AJUSTES REALIZADOS ---
# mydestination -> Añadimos el dominio para que el servidor acepte los correos como locales.
# home_mailbox = Maildir/ -> Cambiamos a formato Maildir para que cada correo sea un archivo.
# --------------------------

sudo systemctl restart postfix
```

<div align="center">
  <table>
    <tr>
      <td align="center">
        <b>⚙️ Configuración Postfix</b><br>
        <img src="https://github.com/user-attachments/assets/11aebd5a-5491-47ac-ae92-ab1bccc2a851" width="550">
      </td>
    </tr>
  </table>
</div>

#### 2.2 Dovecot (Gestión de recepción IMAP)
Aquí ajustamos la forma en la que el usuario se identifica.

```bash
# Ajustamos el parseo del nombre de usuario
sudo nano /etc/dovecot/conf.d/10-auth.conf
# auth_username_format = %n -> Esto descarta el @dominio y se queda solo con el 'user' local.

# Ajustamos la ruta de lectura
sudo nano /etc/dovecot/conf.d/10-mail.conf
# mail_location = maildir:~/Maildir -> Sincronizamos con la carpeta que usa Postfix.

sudo systemctl restart dovecot
```

<div align="center">
  <table>
    <tr>
      <td align="center">
        <b>⚙️ Configuración Dovecot</b><br>
        <img src="https://github.com/user-attachments/assets/1df72ec8-9135-4cdc-a33e-fd52123d2b16" width="550">
      </td>
    </tr>
  </table>
</div>

---

### ✉️ Paso 3: Thunderbird y Despliegue Gráfico

Llegó el momento de ver los resultados. Lanzamos Thunderbird desde la Máquina Cliente, pero lo controlamos desde nuestro Windows.

#### 3.1 Lanzamiento X11

```bash
# Lanzamos el proceso en background para no bloquear la consola
thunderbird &
```

<div align="center">
  <table>
    <tr>
      <td align="center">
        <b>🖥️ Visualización Remota</b><br>
        <img src="https://github.com/user-attachments/assets/a658ba18-a3d5-4139-a8d3-f6c94cd67f9b" width="550">
      </td>
    </tr>
  </table>
</div>

#### 3.2 Configuración de la Cuenta
Introducimos los parámetros de red. Es vital hacerlo manual para apuntar directamente a las IPs del entorno de laboratorio.

* **Identidad:** `cherjo@cherjo.com`
* **Incoming (IMAP):** IP `10.10.10.2` | Puerto `143` | SSL `None`
* **Outgoing (SMTP):** IP `10.10.10.2` | Puerto `25` | SSL `None`

<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://github.com/user-attachments/assets/524d9074-ca57-468a-baf7-b21169e06f72" width="400">
      </td>
      <td align="center">
        <img src="https://github.com/user-attachments/assets/2f12eb9a-c953-4d15-9ae6-553d1cc5227a" width="400">
      </td>
    </tr>
  </table>
</div>

#### 3.3 Fix de Permisos y Sincronización
Si la bandeja de entrada no carga, es un problema de permisos en el sistema de archivos o de caché del cliente.

```bash
# Aseguramos que el usuario es el propietario legal del buzón
sudo chown -R cherjo:cherjo /home/cherjo/Maildir
sudo chmod -R 700 /home/cherjo/Maildir
```

---

### ✅ Paso 4: Verificación y Prueba de Envío

Para comprobar que el flujo de datos es correcto, realizamos una prueba de envío y recepción. El proceso técnico es el siguiente:

1.  **Envío (SMTP):** El correo sale por el puerto **25** hacia Postfix.
2.  **Almacenamiento:** Postfix lo deposita en la carpeta `Maildir`.
3.  **Recepción (IMAP):** Dovecot detecta el nuevo archivo y lo sirve por el puerto **143** a la interfaz de Thunderbird.

<div align="center">
  <img src="https://github.com/user-attachments/assets/bd4a7533-333e-482d-984e-091da72ca734" width="600">
  <br><i>Éxito: Recepción del "Correo de Prueba" en la Inbox de cherjo@cherjo.com.</i>
</div>

**Resultado:** El sistema es 100% operativo. Hemos logrado que un mensaje viaje a través de toda nuestra arquitectura de red y sea visualizado correctamente en el cliente gráfico.
