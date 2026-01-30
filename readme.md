# 🛡️ Paso 1: Infraestructura de Red y Acceso Remoto

El primer objetivo fue establecer una vía de comunicación segura entre el host externo (PC Windows) y el servidor interno privado (`10.10.10.2`) a través del nodo Router.

---

## 1️⃣ Habilitación de X11 Forwarding (MÁQUINA: CLIENTE)

Para poder visualizar la interfaz gráfica de Thunderbird en el PC local, modificamos la configuración del servicio SSH en el servidor de correo.

```bash
# Comando para editar la configuración de SSH:
sudo nano /etc/ssh/sshd_config

# --- CAMBIOS A REALIZAR DENTRO DEL ARCHIVO ---
# Asegúrate de que estas líneas no tengan un '#' al principio:
X11Forwarding yes
AddressFamily inet
# ---------------------------------------------

# Reiniciamos el servicio para aplicar los cambios:
sudo systemctl restart ssh
```
2️⃣ Configuración de Redirecciones DNAT (MÁQUINA: ROUTER)
Puesto que el Servidor de Correo está en una red privada detrás del Router, ejecutamos reglas de iptables para mapear los puertos necesarios hacia la IP interna 10.10.10.2.

```Bash
# 1. Redirección para el túnel SSH y gestión remota (Puerto 2222 -> 22)
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 2222 -j DNAT --to-destination 10.10.10.2:22

# 2. Redirección para el protocolo SMTP (Envío de correo - Puerto 25)
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 25 -j DNAT --to-destination 10.10.10.2:25

# 3. Redirección para el protocolo IMAP (Recepción de correo - Puerto 143)
sudo iptables -t nat -A PREROUTING -i ens18 -p tcp --dport 143 -j DNAT --to-destination 10.10.10.2:143

# 4. Enmascaramiento para permitir la salida a Internet del servidor interno
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

# 🛠️ Paso 2: Instalación y Configuración de Servicios de Correo

Una vez preparada la red, procedemos a instalar el cliente gráfico y configurar los servidores de envío (Postfix) y recepción (Dovecot) en la máquina **CLIENTE**.

> **⚠️ NOTA IMPORTANTE:** En la siguiente configuración verás el dominio `cherjo.com`. Este es el nombre que hemos asignado a nuestro proyecto. **Debes sustituir `cherjo.com` por el nombre de dominio que tú hayas elegido** (ej. `miempresa.com`, `proyecto.local`, etc.).

---

## 1️⃣ Instalación de Software
Actualizamos los repositorios e instalamos **Thunderbird** (Cliente de correo) y **xauth** (necesario para que la interfaz gráfica se autorice a través de SSH).

```bash
sudo apt update
sudo apt install thunderbird xauth -y
```
2️⃣ Configuración de Postfix (SMTP)
Configuramos el agente de transporte de correo para que acepte correos dirigidos a nuestro dominio y los guarde en formato de carpeta (Maildir/) en lugar de un archivo único.

```Bash
# Editamos la configuración principal de Postfix:
sudo nano /etc/postfix/main.cf

# --- CAMBIOS A REALIZAR DENTRO DEL ARCHIVO ---
# 1. Busca 'mydestination'. Aquí definimos qué correos se queda el servidor.
# SUSTITUYE 'cherjo.com' por TU DOMINIO:
mydestination = $myhostname, cherjo.com, localhost.cherjo.com, localhost

# 2. Añade (o modifica) esta línea al final para usar formato carpeta:
home_mailbox = Maildir/
# ---------------------------------------------

# Reiniciamos el servicio para aplicar cambios:
sudo systemctl restart postfix
```
3️⃣ Configuración de Dovecot (Autenticación)
Modificamos Dovecot para permitir el inicio de sesión sin cifrado SSL (entorno de laboratorio) y para que gestione correctamente los nombres de usuario con formato de correo completo.

```Bash
# Editamos el archivo de autenticación:
sudo nano /etc/dovecot/conf.d/10-auth.conf

# --- CAMBIOS A REALIZAR DENTRO DEL ARCHIVO ---
# Descomenta y modifica estas líneas:

# %n es CRÍTICO: le dice al sistema que ignore la parte del dominio.
# Si te logueas como 'usuario@cherjo.com', el sistema solo leerá 'usuario'.
auth_username_format = %n

# Permite contraseñas en texto plano (necesario porque no configuramos SSL)
disable_plaintext_auth = no
# ---------------------------------------------
```
4️⃣ Configuración de Dovecot (Buzón)
Es crítico sincronizar Dovecot para que busque los correos en la misma carpeta donde Postfix los está guardando (~/Maildir).

```Bash
# Editamos el archivo de ubicación del correo:
sudo nano /etc/dovecot/conf.d/10-mail.conf

# --- CAMBIOS A REALIZAR DENTRO DEL ARCHIVO ---
# Busca la línea mail_location y déjala así (asegúrate de comentar con # la que pone mbox):
mail_location = maildir:~/Maildir
# ---------------------------------------------

# Reiniciamos Dovecot para aplicar todos los cambios de configuración:
sudo systemctl restart dovecot
```

# ✉️ Paso 3: Lanzamiento y Configuración de Thunderbird

Finalmente, ejecutamos el cliente de correo y configuramos la cuenta manualmente para que conecte con nuestros servicios internos (Postfix/Dovecot).

> **⚠️ NOTA IMPORTANTE:** En el ejemplo utilizamos el usuario `cherjo` y el dominio `cherjo.com`. **Sustituye estos valores por tu propio usuario y dominio.**

---

## 1️⃣ Ejecución del Cliente
Lanzamos Thunderbird desde la terminal. Gracias al X11 Forwarding configurado en el Paso 1, la ventana aparecerá en nuestro escritorio.

```bash
# El símbolo '&' permite que la terminal siga operativa
thunderbird &
2️⃣ Configuración de Cuenta (GUI)Al abrirse el asistente de configuración, selecciona "Configuración Manual" e introduce los siguientes datos. Es crítico usar la IP interna y el nombre de usuario completo.AjusteValor (Ejemplo)ConfiguraciónNombreCherjo TeamTu nombre o el del equipoCorreocherjo@cherjo.comusuario@tudominio.comContraseña[La del sistema]La contraseña del usuario LinuxProtocolo EntranteIMAPPuerto 143Servidor Entrante10.10.10.2SSL: NoneProtocolo SalienteSMTPPuerto 25Servidor Saliente10.10.10.2SSL: NoneUsuario Entrantecherjo@cherjo.comImportante: Formato completoUsuario Salientecherjo@cherjo.comImportante: Formato completo3️⃣ Ajuste Final de PermisosUna vez configurada la cuenta, ejecutamos estos comandos para asegurar que el usuario tenga propiedad total sobre su buzón y evitar errores de lectura (bandeja vacía).Bash# Aseguramos que el usuario sea dueño de su carpeta de correo
# (Cambia 'cherjo' por tu usuario)
sudo chown -R cherjo:cherjo /home/cherjo/Maildir

# Restringimos permisos (Solo el dueño puede leer/escribir)
sudo chmod -R 700 /home/cherjo/Maildir
