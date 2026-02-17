# Firewall de Red (Host) para Extagram

Esta configuración establece un perímetro de seguridad a nivel de **Sistema Operativo (Host)**. A diferencia de un WAF (que analiza el contenido HTTP), este firewall actúa como una barrera física que decide qué conexiones de red se aceptan o rechazan antes de que lleguen siquiera a los contenedores Docker (S1).

**Objetivo:** Proteger el Gateway S1 asegurando que solo el tráfico web legítimo (y la administración SSH) pueda tocar el servidor.

---

### 1. Política de Seguridad "Zero Trust" (Por Defecto)

Lo primero es establecer una política de "Lista Blanca": se prohíbe todo por defecto y solo se permite lo estrictamente necesario.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

```

* **`deny incoming`**: Cierra **todas** las puertas del servidor. Si alguien intenta conectar a cualquier puerto, el servidor ignorará la petición silenciosamente (DROP). Esto oculta servicios que podrían estar corriendo accidentalmente.
* **`allow outgoing`**: Permite que tu servidor inicie conexiones hacia afuera. Esto es vital para que el sistema pueda descargar actualizaciones de seguridad, o para que los contenedores puedan enviar correos o conectarse a APIs externas si fuera necesario.

---

### 2. Asegurar el Acceso Administrativo (Anti-Lockout)

⚠️ **CRÍTICO:** Antes de activar el firewall, debes garantizar tu propia entrada.

```bash
sudo ufw allow 22/tcp

```

* **Explicación:** Abre el puerto estándar de **SSH**. Sin esta regla, al activar el firewall, tu propia sesión de terminal se cortaría y perderías el control remoto del servidor para siempre.

---

### 3. Exposición Controlada de Servicios (S1 Gateway)

El servicio **S1 (Proxy)** es el único punto de entrada autorizado para la aplicación Extagram. Debemos abrir los canales para que los usuarios accedan a la web.

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

```

* **`80/tcp` (HTTP)**: Necesario para recibir la primera petición del usuario, que luego S1 redirigirá automáticamente a HTTPS (Redirección 301).
* **`443/tcp` (HTTPS)**: El canal principal encriptado donde S1 realiza la terminación SSL.

> **Nota técnica:** Al abrir estos puertos en el Host, permitimos que el tráfico fluya hacia el mapeo de puertos de Docker (`-p 443:443`) definido en S1.

---

### 4. Mitigación de Ataques (Rate Limiting)

Una capa extra de protección contra ataques de fuerza bruta o denegación de servicio simple.

```bash
sudo ufw limit 443/tcp

```

* **Explicación:** En lugar de un simple "allow", el comando `limit` vigila la frecuencia de conexión. Si una misma dirección IP intenta iniciar 6 o más conexiones en un intervalo de 30 segundos, UFW bloqueará esa IP temporalmente. Es muy efectivo contra bots agresivos.

---

### 5. Activación y Verificación

Una vez definidas las reglas, encendemos el sistema.

```bash
sudo ufw enable

```

Para confirmar que todo está correcto, usa:

```bash
sudo ufw status verbose

```

**Resultado Final:**
El Host ahora actúa como un escudo. Cualquier escaneo de puertos o intento de conexión a servicios internos (como la base de datos S7 en el puerto 3306) será rechazado inmediatamente por el Host, protegiendo la infraestructura Docker subyacente.