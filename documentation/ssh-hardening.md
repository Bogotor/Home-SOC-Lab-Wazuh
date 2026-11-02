# Hardening SSH — SOC-WAZUH-SERVER

## 1. Objetivo y contexto

Reducir la superficie de ataque del acceso remoto a la VM SOC-WAZUH-SERVER antes de instalar Wazuh: sustituir la autenticación por contraseña por autenticación mediante claves y bloquear por configuración el inicio de sesión SSH directo como `root`.

**Fase del proyecto:** Fase 2 — Preparación de infraestructura (en proceso).

Este documento cubre:

- Los conceptos necesarios para entender el cambio.
- La configuración del acceso por clave desde macOS.
- La configuración aplicada en el servidor y su validación.
- Las decisiones técnicas, las limitaciones y las evidencias pendientes.

La instalación de Wazuh **no forma parte de este hito**.

**Estado:** hardening aplicado. El acceso por clave funciona y un intento controlado con contraseña fue rechazado. `PermitRootLogin no` está confirmado en la configuración, pero **no se ha probado directamente un inicio de sesión SSH como `root`**. **Los componentes de Wazuh todavía no están instalados.**

**Convenciones de este documento:**

- **Resultado obtenido:** dato comprobado en este laboratorio.
- **No registrado:** paso realizado cuyo comando, método o salida exactos no quedaron anotados.
- **Comando de referencia:** forma habitual de realizar o repetir un paso. **No implica que se ejecutara exactamente así.**
- **Información técnica general:** comportamiento documentado del software, incluido para entender los resultados, pero **no verificado** en este laboratorio.

**Marcadores:**

- `<IP_SERVIDOR>`: IP de la VM. La asignación actual es temporal (DHCP), por lo que no se usa como valor fijo.
- `<CLAVE_LAB>`: nombre del archivo de la clave privada en macOS. No se publica.

> **Relación con otros documentos:** el acceso SSH por contraseña registrado en [`vm-01-setup.md`](vm-01-setup.md) (sección 3.7) corresponde al estado anterior a este hito.

## 2. Infraestructura y versiones

| Elemento | Valor |
|---|---|
| Servidor | VM SOC-WAZUH-SERVER |
| Sistema operativo | Ubuntu Server 24.04.5 LTS ARM64 |
| Usuario administrativo | `socadmin` |
| Cliente SSH | MacBook Pro, macOS |
| Tipo de clave | Ed25519, específica del laboratorio y protegida con passphrase |
| Snapshot previo | `baseline-before-ssh-hardening` |
| Archivo de configuración creado | `/etc/ssh/sshd_config.d/01-home-soc-hardening.conf` |
| Versión de OpenSSH | No registrada (ver sección 6.4) |
| Componentes de Wazuh | No instalados |

## 3. Conceptos fundamentales

Esta sección es **información técnica general**. Los párrafos «En este laboratorio» la relacionan con lo comprobado.

### 3.1 Autenticación

La autenticación es el proceso de demostrar una identidad. Responde a la pregunta «¿quién eres y cómo lo demuestras?».

SSH admite varios métodos, entre ellos `publickey` (clave), `password` (contraseña) y `keyboard-interactive` (diálogo de preguntas y respuestas que en Linux suele pasar por PAM y también puede solicitar una contraseña).

**En este laboratorio:** `socadmin` se autentica con una clave Ed25519.

### 3.2 Autorización

La autorización decide qué puede hacer una identidad. Responde a la pregunta «¿qué se te permite?». Tener una credencial válida no implica tener permiso: son dos controles distintos.

Las cuatro directivas aplicadas se clasifican así:

| Directiva | Qué controla | Categoría |
|---|---|---|
| `PubkeyAuthentication` | Si se acepta el método de clave pública | Autenticación |
| `PasswordAuthentication` | Si se acepta el método de contraseña | Autenticación |
| `KbdInteractiveAuthentication` | Si se acepta el método `keyboard-interactive` | Autenticación |
| `PermitRootLogin` | Si `root` puede abrir una sesión SSH, aunque presente una credencial válida | Autorización |

> **Cuidado con el nombre:** `authorized_keys` lista las claves públicas con las que se puede autenticar una cuenta. Pese a su nombre, no determina qué podrá hacer esa cuenta después de entrar; eso depende, por ejemplo, de los permisos de archivos y de `sudo`.

### 3.3 Hardening

El hardening (bastionado) consiste en configurar un sistema para reducir su superficie de ataque: desactivar lo que no se necesita y aplicar opciones más restrictivas. No hace invulnerable al sistema, y cada medida tiene un coste operativo.

| Con estas medidas se mitiga | No se mitiga |
|---|---|
| Fuerza bruta y *password spraying* contra SSH | Robo de la clave privada junto con su passphrase |
| Reutilización de contraseñas filtradas para entrar por SSH | Compromiso del equipo cliente |
| Acceso SSH directo a la cuenta `root` | Vulnerabilidades del propio servidor OpenSSH |

### 3.4 Claves públicas y privadas

La autenticación por clave usa criptografía asimétrica: un par de claves relacionadas matemáticamente, donde lo que firma una solo puede verificarse con la otra.

| Elemento | Dónde reside | ¿Es secreto? | Función |
|---|---|---|---|
| Clave privada | Cliente (macOS) | Sí | Firma el desafío de autenticación |
| Clave pública | `~/.ssh/authorized_keys` en el servidor | No | Verifica la firma |
| Passphrase | Solo la conoce el usuario | Sí | Cifra la clave privada en disco |

Flujo simplificado:

1. El cliente indica qué clave pública quiere usar.
2. El servidor comprueba si esa clave está en `authorized_keys` de la cuenta.
3. El cliente firma con la clave privada unos datos que incluyen un identificador único de la sesión.
4. El servidor verifica la firma con la clave pública.

La clave privada **nunca se envía al servidor**. Por eso, a diferencia de una contraseña, conocer la clave pública no permite iniciar sesión.

La passphrase se comprueba localmente en el cliente. Protege el archivo si alguien lo copia, pero el servidor no la ve ni puede exigirla: no es un segundo factor controlado por el servidor.

> SSH también usa **claves de host**, que autentican al servidor ante el cliente (`~/.ssh/known_hosts`). Son independientes de la clave de usuario que documenta este hito.

### 3.5 Prueba positiva y prueba negativa

- **Prueba positiva:** comprueba que lo que debe funcionar funciona. Aquí, que el administrador conserva el acceso.
- **Prueba negativa:** comprueba que lo que debe bloquearse se bloquea. Aquí, que no se acepta una contraseña.

Se necesitan ambas, porque cada una cubre lo que la otra no ve. Una prueba negativa, además, solo es concluyente si el rechazo se debe al control evaluado y no a otra causa, como una contraseña mal escrita o un fallo de red.

**En este laboratorio:** la conexión por clave ya funcionaba antes del cambio, por lo que por sí sola no demuestra que las contraseñas estén desactivadas. A la inversa, un rechazo de contraseñas sin una prueba de clave no garantizaría que el administrador siguiera pudiendo entrar.

## 4. Procedimiento ejecutado

### 4.1 Punto de restauración

**Equipo:** MacBook Pro (VMware Fusion).

Se creó el snapshot `baseline-before-ssh-hardening` antes de aplicar el hardening.

**Resultado obtenido:** snapshot creado.

**No registrado:** el estado de la VM al crearlo y su orden exacto respecto a la creación de la clave. **La restauración no se ha probado.**

### 4.2 Creación de la clave

**Equipo:** MacBook Pro (macOS).

Se generó un par de claves Ed25519 específico del laboratorio, protegido con passphrase.

**No registrado:** el comando exacto. Comando de referencia:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/<CLAVE_LAB> -C "home-soc-lab"
```

- `-t ed25519`: tipo de clave.
- `-f`: archivo de la clave privada. La pública se guarda con el mismo nombre y la extensión `.pub`. Un nombre propio evita sobrescribir otra clave.
- `-C`: comentario que se añade a la clave pública. Si se omite, suele incluir el usuario y el equipo locales.
- La passphrase se solicita de forma interactiva, por lo que no queda en el historial de la shell.

### 4.3 Instalación de la clave pública

**Equipos:** MacBook Pro y VM SOC-WAZUH-SERVER.

**Resultado obtenido:** la clave pública se agregó a `~/.ssh/authorized_keys` en el servidor.

**No registrado:** el método utilizado. Comando de referencia (desde el MacBook Pro):

```bash
ssh-copy-id -i ~/.ssh/<CLAVE_LAB>.pub socadmin@<IP_SERVIDOR>
```

- `-i`: clave pública que se instalará. Solo se copia la parte pública.
- En ese momento la conexión todavía se autentica con el método disponible, la contraseña.

> **Información técnica general (no verificada en este laboratorio):** con la opción `StrictModes`, activa por defecto, `sshd` ignora `authorized_keys` si los permisos del archivo o de sus directorios son demasiado abiertos. Los valores habituales son `700` para `~/.ssh` y `600` para `authorized_keys`.

### 4.4 Prueba de la clave antes de desactivar contraseñas

**Equipo:** MacBook Pro (macOS).

**Resultado obtenido:** autenticación mediante claves correcta.

**No registrado:** el comando exacto. Comando de referencia:

```bash
ssh -i ~/.ssh/<CLAVE_LAB> -o IdentitiesOnly=yes socadmin@<IP_SERVIDOR>
```

- `-i`: clave privada que se usará.
- `-o IdentitiesOnly=yes`: limita el intento a la clave indicada, para que el resultado no se deba a otra clave disponible en el agente SSH.

Este paso va antes de tocar la configuración del servidor: desactivar las contraseñas sin una clave probada puede dejar al administrador sin acceso remoto.

### 4.5 Hallazgo: `50-cloud-init.conf` habilitaba contraseñas

**Equipo:** VM SOC-WAZUH-SERVER.

**Resultado obtenido:** `/etc/ssh/sshd_config.d/50-cloud-init.conf` contenía `PasswordAuthentication yes`.

**No registrado:** el comando con el que se localizó. Comando de referencia:

```bash
sudo grep -rn 'PasswordAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

- `-r`: busca de forma recursiva.
- `-n`: muestra el número de línea.

> **Información técnica general (no verificada en este laboratorio):** en Ubuntu 24.04, `/etc/ssh/sshd_config` incluye `/etc/ssh/sshd_config.d/*.conf` al principio. Los archivos se procesan en orden lexicográfico y **`sshd` conserva el primer valor que obtiene para cada directiva**. Por eso, añadir `PasswordAuthentication no` al final de `sshd_config`, o en un archivo ordenado después de `50-cloud-init.conf`, no tendría efecto. cloud-init puede generar este archivo a partir de su opción `ssh_pwauth`.

### 4.6 Creación del archivo de hardening

**Equipo:** VM SOC-WAZUH-SERVER.

**Resultado obtenido:** se creó `/etc/ssh/sshd_config.d/01-home-soc-hardening.conf` con este contenido:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

**No registrado:** el editor utilizado, y el propietario y los permisos del archivo. No consta ninguna modificación de `50-cloud-init.conf`.

**Información técnica general** sobre cada directiva:

| Directiva | Efecto |
|---|---|
| `PubkeyAuthentication yes` | Permite la autenticación por clave. Es el valor por defecto de OpenSSH; declararlo deja explícita la intención. |
| `PasswordAuthentication no` | Desactiva el método `password`. |
| `KbdInteractiveAuthentication no` | Desactiva `keyboard-interactive`. Sin esto, PAM podría seguir pidiendo la contraseña por esta vía aunque `PasswordAuthentication` esté en `no`. |
| `PermitRootLogin no` | Impide que `root` inicie sesión por SSH con cualquier método, incluidas las claves. El valor por defecto de OpenSSH, `prohibit-password`, sí permite las claves. |

> **Información técnica general (no verificada en este laboratorio):** `PasswordAuthentication no` solo afecta a SSH. No elimina la contraseña de `socadmin`, que sigue usándose, por ejemplo, en la consola local y con `sudo`.

### 4.7 Validación de la sintaxis

**Equipo:** VM SOC-WAZUH-SERVER.

```bash
sudo sshd -t
```

- `-t`: modo de prueba. Comprueba la validez de la configuración y de las claves de host sin aplicar cambios.

**Resultado obtenido:** finalizó sin errores.

### 4.8 Comprobación de la configuración efectiva

**Equipo:** VM SOC-WAZUH-SERVER.

```bash
sudo sshd -T
```

- `-T`: modo de prueba extendido. Procesa la configuración, incluidos los archivos incluidos y la precedencia, y muestra los valores resultantes.

**Resultado obtenido:** confirmó los cuatro valores esperados.

| Directiva | Valor confirmado |
|---|---|
| `PubkeyAuthentication` | `yes` |
| `PasswordAuthentication` | `no` |
| `KbdInteractiveAuthentication` | `no` |
| `PermitRootLogin` | `no` |

**No registrado:** la salida literal y si se filtró. Comando de referencia para mostrar solo estas directivas (`sshd -T` las escribe en minúsculas):

```bash
sudo sshd -T | grep -Ei '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

> **Qué demuestra y qué no.** `sshd -T` lee la configuración del disco: muestra lo que aplicaría un proceso nuevo, no lo que está haciendo el servicio en ejecución. Por eso hacen falta el reinicio y las pruebas de comportamiento.

### 4.9 Aplicación de la configuración

**Equipo:** VM SOC-WAZUH-SERVER.

**Resultado obtenido:** se reinició `ssh.service` para aplicar la configuración.

**No registrado:** el comando exacto. Comando de referencia:

```bash
sudo systemctl restart ssh.service
```

> **Información técnica general (no verificada en este laboratorio):** el reinicio no suele cerrar las sesiones SSH ya abiertas. Mantener una abierta sirve de red de seguridad, pero no valida la configuración nueva: para eso hace falta una conexión nueva. No consta si se mantuvo una sesión abierta durante este paso.

### 4.10 Prueba positiva tras el reinicio

**Equipo:** MacBook Pro (macOS).

**Resultado obtenido:** una nueva conexión SSH mediante claves fue exitosa.

**No registrado:** el comando exacto. El comando de referencia es el de la sección 4.4.

### 4.11 Prueba negativa con contraseña

**Equipo:** cliente SSH (equipo de origen no registrado).

**Resultado obtenido:** una prueba controlada con autenticación por contraseña fue rechazada.

**Referencia comparativa:** antes del hardening, `socadmin` sí podía entrar con contraseña ([`vm-01-setup.md`](vm-01-setup.md), sección 3.7). Ese cambio de comportamiento es lo que relaciona el rechazo con la configuración aplicada.

**No registrado:** el comando exacto, el mensaje mostrado y si el cliente llegó a solicitar la contraseña. Comando de referencia:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password socadmin@<IP_SERVIDOR>
```

- `-o PubkeyAuthentication=no`: el cliente no intenta usar claves.
- `-o PreferredAuthentications=password`: el cliente solo intenta el método de contraseña.

> **Información técnica general (no verificada en este laboratorio):** el mensaje habitual en este escenario es `Permission denied (publickey).`, sin llegar a pedir la contraseña. Con `ssh -v`, la línea `Authentications that can continue:` muestra los métodos que ofrece el servidor. Si solo aparece `publickey`, el rechazo se debe a la política del servidor y no a una contraseña incorrecta. **Este mensaje es una referencia, no la salida registrada en este laboratorio.**

## 5. Decisiones técnicas

Las decisiones son hechos del laboratorio. Las alternativas y justificaciones son razonamiento técnico, no resultados medidos.

| Decisión | Alternativas | Justificación |
|---|---|---|
| Clave Ed25519 | RSA, ECDSA | Claves cortas, firma rápida y amplio soporte en OpenSSH. |
| Clave específica del laboratorio | Reutilizar una clave personal | Aísla el laboratorio: se puede revocar sin afectar a otros sistemas, y un compromiso externo no alcanza a este servidor a través de esa clave. |
| Clave protegida con passphrase | Clave sin passphrase | Protege la clave si se copia el archivo. Coste: hay que introducirla o usar un agente SSH. |
| Snapshot antes del cambio | Ningún punto de restauración | Ofrece una vía de vuelta si la configuración deja la VM inaccesible. |
| Probar la clave antes de desactivar contraseñas | Aplicar todo de una vez | Evita el bloqueo del acceso remoto. |
| Archivo propio `01-home-soc-hardening.conf` | Editar `sshd_config` o `50-cloud-init.conf` | El prefijo `01-` se procesa antes que `50-`, por lo que sus valores prevalecen. Agrupa los cambios en un archivo identificable y fácil de revertir, y no depende de un archivo asociado a cloud-init. |
| Declarar `KbdInteractiveAuthentication no` | Confiar en el valor existente | Cierra la otra vía por la que podría solicitarse una contraseña y deja la intención explícita. |
| `PermitRootLogin no` | `prohibit-password` | La administración se hace con `socadmin` y no se necesita acceso SSH directo como `root`. También bloquea el acceso con claves. |
| Secuencia `sshd -t`, `sshd -T`, reinicio y pruebas | Reiniciar directamente | Detecta errores antes de aplicar el cambio y valida primero la configuración y después el comportamiento. |

## 6. Verificaciones y resultados

### 6.1 Verificaciones realizadas

| Comprobación | Tipo | Equipo | Comando o método | Resultado obtenido |
|---|---|---|---|---|
| Clave antes del cambio | Positiva | MacBook Pro | No registrado | Autenticación correcta |
| Sintaxis de la configuración | Configuración | VM | `sudo sshd -t` | Sin errores |
| Valores efectivos | Configuración | VM | `sudo sshd -T` | Los cuatro valores esperados |
| Reinicio del servicio | Aplicación | VM | Reinicio de `ssh.service` (comando no registrado) | Realizado |
| Clave después del reinicio | Positiva | MacBook Pro | Nueva conexión SSH (comando no registrado) | Conexión exitosa |
| Contraseña después del reinicio | Negativa | No registrado | Prueba controlada (comando no registrado) | Rechazada |

### 6.2 Qué demuestra cada comprobación

| Comprobación | Demuestra | No demuestra |
|---|---|---|
| `sshd -t` | Que la configuración es válida | Que se esté aplicando o que tenga el efecto deseado |
| `sshd -T` | Los valores que resultan de la configuración en disco | Que el servicio en ejecución los esté usando |
| Conexión con clave | Que se conserva el acceso administrativo | Que las contraseñas estén desactivadas |
| Contraseña rechazada | Que el servicio rechazó ese intento | El comportamiento con `root`. La causa exacta tampoco puede documentarse sin el mensaje del cliente o los registros del servidor |

### 6.3 Sin comprobar

- **Inicio de sesión SSH directo como `root`.** `PermitRootLogin no` solo está confirmado por `sshd -T`.
- **Persistencia tras reiniciar la VM.** Solo consta el reinicio de `ssh.service`.
- **Restauración del snapshot.**
- **Acceso de emergencia por la consola de VMware Fusion** después del cambio.
- **Registros del servidor** correspondientes a las pruebas.

> **Cómo diseñar la prueba de `root`.** Un `ssh root@<IP_SERVIDOR>` rechazado no sería concluyente por sí solo: sin contraseñas y sin una clave autorizada para `root`, el acceso se rechazaría también con otro valor de `PermitRootLogin`. Para atribuir el rechazo a esta directiva, `root` tendría que presentar una credencial válida, lo que implica un cambio temporal en la VM que debe autorizarse y revertirse. **Esta prueba no se ha realizado y queda pendiente de decisión.**

### 6.4 Comprobaciones complementarias recomendadas

Estos comandos **no se han ejecutado** en este hito. Se proponen para completar la validación y generar evidencias.

| Comprobación | Equipo | Comando recomendado |
|---|---|---|
| Versión de OpenSSH | VM | `dpkg-query -W openssh-server` |
| Directiva `Include` | VM | `grep -n '^Include' /etc/ssh/sshd_config` |
| Propietario y permisos de la configuración | VM | `ls -l /etc/ssh/sshd_config.d/` |
| Permisos de `~/.ssh` y `authorized_keys` | VM | `ls -ld ~/.ssh ~/.ssh/authorized_keys` |
| Estado del servicio | VM | `systemctl status ssh.service` |
| Métodos ofrecidos por el servidor | Cliente SSH | `ssh -v -o PubkeyAuthentication=no -o PreferredAuthentications=password socadmin@<IP_SERVIDOR>` |
| Registros de las pruebas | VM | `sudo journalctl -u ssh.service --since "<FECHA>"` |

Explicación de argumentos:

- `dpkg-query -W`: muestra la versión instalada del paquete.
- `ls -ld`: muestra los permisos de un directorio en lugar de su contenido. Estos comandos no muestran las claves.
- `ssh -v`: muestra el detalle de la negociación. Incluye rutas locales, que deben revisarse antes de publicar.
- `journalctl -u ssh.service --since "<FECHA>"`: muestra los registros del servicio desde una fecha. Es posible que los registros de las pruebas originales ya no se conserven.

## 7. Incidencias y resolución

No constan incidencias en los datos registrados de este hito. El contenido de `50-cloud-init.conf` se trata como un hallazgo de configuración (sección 4.5).

## 8. Evidencias pendientes

Las evidencias se guardarán en `evidence/screenshots/ssh-hardening/` (en la raíz del repositorio) una vez revisadas y anonimizadas. Ese directorio todavía no existe.

**Estado: pendientes.** Ninguna evidencia está incorporada al repositorio todavía. Los resultados de este documento proceden de las comprobaciones registradas por el estudiante.

| # | Evidencia necesaria | Qué demuestra | Revisión antes de publicar | Estado |
|---|---|---|---|---|
| 1 | Lista de snapshots de VMware Fusion | Existencia de `baseline-before-ssh-hardening` | — | Pendiente |
| 2 | Línea `PasswordAuthentication yes` de `50-cloud-init.conf` | Hallazgo de la sección 4.5 | — | Pendiente |
| 3 | Contenido de `01-home-soc-hardening.conf` | Configuración aplicada | — | Pendiente |
| 4 | Salida de `sudo sshd -t` | Sintaxis válida | — | Pendiente |
| 5 | Salida filtrada de `sudo sshd -T` | Los cuatro valores efectivos | Publicar solo las cuatro líneas | Pendiente |
| 6 | Salida de `systemctl status ssh.service` | Servicio activo tras el reinicio | Revisar IP y puertos en las líneas de registro | Pendiente |
| 7 | Nueva conexión mediante clave | Prueba positiva | Ocultar el usuario de macOS, el nombre de la clave y su huella | Pendiente |
| 8 | Intento con contraseña rechazado, con el mensaje del cliente | Prueba negativa | Ocultar el usuario de macOS y las rutas locales | Pendiente |
| 9 | Registros del servidor de ambas pruebas, si se conservan | Las pruebas vistas desde el servidor | Ocultar huellas de clave y revisar IP | Pendiente |

> **Capturas originales y repeticiones.** Si una salida se obtiene ahora, debe indicarse la fecha y que se trata de una repetición. Demuestra el estado en ese momento, no sustituye a la ejecución original.

Recomendaciones:

- Nombres de archivo descriptivos, por ejemplo `ssh-05-sshd-t-filtrado.png`.
- Eliminar los metadatos de las imágenes antes de publicarlas.
- No publicar nunca la clave privada, la passphrase, contraseñas ni el contenido de `~/.ssh/`.

## 9. Seguridad

- **Datos no documentados:** este repositorio no contiene la clave privada, la clave pública, su huella, el nombre del archivo de la clave, la passphrase ni contraseñas.
- **Riesgo operativo:** perder la clave privada u olvidar la passphrase impide el acceso SSH. Las vías de recuperación previstas son la consola de VMware Fusion y el snapshot; **ninguna se ha comprobado en este hito**. Restaurar el snapshot revertiría también cualquier cambio posterior a su creación.
- **Medidas no aplicadas:** restricción por usuarios (`AllowUsers`), límite de intentos (`MaxAuthTries`), cambio de puerto, cortafuegos y Fail2Ban no forman parte de los cambios registrados y no deben darse por aplicadas.
- **Implicación para el LAB 01:** con las contraseñas desactivadas, no se espera que este servidor genere eventos `Failed password` por intentos SSH (información técnica general, no verificada en este laboratorio). La arquitectura prevé las prácticas de autenticación en SOC-LINUX-ENDPOINT, cuyo servidor SSH sigue sin comprobarse. Conviene tenerlo en cuenta al diseñar ese laboratorio.

## 10. Estado del hito

### Completado y verificado por el estudiante

- [x] Snapshot `baseline-before-ssh-hardening` creado antes del hardening.
- [x] Par de claves Ed25519 específico del laboratorio, protegido con passphrase.
- [x] Clave pública agregada a `~/.ssh/authorized_keys` en el servidor.
- [x] Autenticación mediante claves probada.
- [x] `PasswordAuthentication yes` identificado en `50-cloud-init.conf`.
- [x] `01-home-soc-hardening.conf` creado con las cuatro directivas.
- [x] `sudo sshd -t` finalizado sin errores.
- [x] `sudo sshd -T` confirma los cuatro valores esperados.
- [x] `ssh.service` reiniciado para aplicar la configuración.
- [x] Prueba positiva: nueva conexión mediante claves exitosa.
- [x] Prueba negativa: autenticación por contraseña rechazada.

### Pendiente

- [ ] Decidir cómo comprobar el comportamiento de `PermitRootLogin no` (sección 6.3) y, si procede, ejecutar la prueba.
- [ ] Comprobar que la configuración persiste tras reiniciar la VM.
- [ ] Comprobar el acceso de emergencia por la consola de VMware Fusion.
- [ ] Ejecutar las comprobaciones complementarias de la sección 6.4.
- [ ] Incorporar las evidencias de la sección 8 en `evidence/screenshots/ssh-hardening/`.

## 11. Conclusiones y limitaciones

SOC-WAZUH-SERVER ha pasado del acceso SSH por contraseña al acceso mediante una clave Ed25519 específica del laboratorio. La configuración se validó antes de aplicarla, y su efecto se comprobó con una prueba positiva y una negativa tras reiniciar el servicio.

Limitaciones actuales:

- `PermitRootLogin no` está confirmado en la configuración, pero no mediante una prueba de inicio de sesión.
- Varios pasos se realizaron sin registrar el comando o la salida exactos.
- El hardening se limita a cuatro directivas de autenticación y acceso; no es un bastionado completo del servidor.
- Las evidencias todavía no están incorporadas al repositorio.
- La red NAT y la IP por DHCP siguen siendo provisionales.
- **Los componentes de Wazuh todavía no están instalados.**
