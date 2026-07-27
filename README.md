# Dual boot Linux Mint 22 + Windows 11 en ASUS VivoBook TP470EZ (soluciones a posibles problemas críticos)


Buenas, soy Juan. Argentino de 23 años viviendo en Capital e introduciéndome en este espacio el cuál me sirvió para diversas cosas, este vez publicando más que especteando.

Paso a dejar una breve documentación técnica del proceso completo de instalación de Linux Mint 22 en paralelo con Windows 11 para usuarios de Vivabook de la marca ASUS, incluyendo el diagnóstico y resolución de tres bloqueos críticos: cifrado BitLocker activo, controlador Intel VMD habilitado, y fallo en la instalación del gestor de arranque GRUB. Quizás a alguien que posee alguna notebook de esta marca pueda presentar algunos de los problemas que les voy a mostrar acá así que les sirve, genial.

Especifiaciones de la notebook

| Componente | Especificación |
|---|---|
| Equipo | ASUS VivoBook Flip TP470EZ |
| CPU | Intel Core i7-1165G7 (11th Gen) |
| RAM | 16 GB |
| Almacenamiento | Intel SSDPEKNW512G8 — NVMe 512 GB |
| BIOS | American Megatrends, versión 311 |
| SO preinstalado | Windows 11 (build 26200) |
| SO a instalar | Linux Mint 22 "Wilma" Cinnamon |

---

Resumen 

El objetivo era instalar Linux Mint de forma nativa conservando Windows 11 intacto. El proceso presentó tres bloqueos que no aparecen en la mayoría de las guías de instalación estándar y que, de haberse ignorado, habrían resultado en pérdida de acceso a Windows o en un sistema Linux no booteable.

Los tres problemas fueron identificados mediante verificación previa antes de modificar la tabla de particiones, siguiendo el principio de **verificar antes de cortar**.

---

Problema 1 — BitLocker activo sin indicación visual clara

Síntoma

El ícono del disco C: en el Explorador de Windows mostraba un candado **abierto**, lo que sugiere que el volumen no está cifrado. Esta interpretación es incorrecta.

Diagnóstico

Verificación por línea de comandos con privilegios elevados:

```cmd
manage-bde -status C:
```

Salida relevante:

```
Volumen C: [OS]
[Volumen del sistema operativo]

    Tamaño:                    475,31 GB
    Versión de BitLocker:      2.0
    Estado de conversión:      Cifrado solo de espacio usado
    Porcentaje cifrado:        100,0%
    Método de cifrado:         XTS-AES 128
    Estado de protección:      Protección activada
    Estado de bloqueo:         Desbloqueado
    Protectores de clave:
        Contraseña numérica
        TPM
```

Interpretación propia

El candado abierto indica únicamente que el volumen está **desbloqueado en la sesión actual** (el TPM liberó la clave al arrancar Windows). No indica ausencia de cifrado. El campo determinante es `Estado de protección: Protección activada` junto con `Porcentaje cifrado: 100,0%`.

**Por qué importa:** redimensionar o modificar la tabla de particiones de un volumen con BitLocker activo invalida las mediciones PCR del TPM. En el siguiente arranque, Windows solicita la clave de recuperación de 48 dígitos. Si esa clave no está respaldada, el acceso al sistema se pierde de forma definitiva.

Resolución ¿cómo fue el proceso?

1. **Respaldo previo de la clave de recuperación** antes de cualquier modificación:

```cmd
manage-bde -protectors -get C:
```

Se verificó adicionalmente que la clave estuviera respaldada en la cuenta Microsoft asociada (`account.microsoft.com/devices/recoverykey`).

2. **Descifrado del volumen:**

```cmd
manage-bde -off C:
```

3. **Monitoreo del progreso** — el proceso corre en segundo plano y admite el uso normal del equipo:

```cmd
manage-bde -status C:
```

El campo `Porcentaje cifrado` desciende progresivamente de 100% a 0%.

4. **Verificación de estado final** antes de continuar:

```
Estado de conversión:      Totalmente descifrado
Porcentaje cifrado:        0,0%
Estado de protección:      Protección desactivada
```

Nota operativa

El proceso de descifrado fue interrumpido una vez por apagado del equipo al 44,6%. BitLocker reanuda automáticamente al siguiente inicio de sesión, pero **las operaciones de cifrado/descifrado no deben interrumpirse deliberadamente**. En otros sistemas de cifrado (LUKS, cifrado empresarial) una interrupción mal manejada puede dejar el volumen irrecuperable.

---
Problema 2 — Intel VMD impide la detección del disco

### Síntoma

Riesgo anticipado: en equipos con CPU Intel de 11ª generación en adelante, el controlador **Intel Volume Management Device (VMD)** viene habilitado de fábrica. Linux no incluye soporte nativo para VMD en la mayoría de las distribuciones, por lo que el instalador no detecta el SSD NVMe y reporta que no hay discos disponibles.

### Diagnóstico

Verificación en BIOS:

```
BIOS → F7 (Advanced Mode) → pestaña Advanced → VMD setup menu
    Enable VMD controller: [Enabled]
```

### El conflicto

Deshabilitar VMD directamente resuelve el problema para Linux, pero **rompe el arranque de Windows**: el sistema quedó instalado con el driver de almacenamiento VMD/RST cargado, y al cambiar el modo del controlador arranca con `INACCESSIBLE_BOOT_DEVICE` (pantalla azul).

### Resolución — método safeboot

El procedimiento fuerza a Windows a arrancar en Modo Seguro, donde carga drivers de almacenamiento genéricos (AHCI estándar) y reconfigura su stack de drivers automáticamente.

**Paso 1** — En Windows, CMD con privilegios elevados:

```cmd
bcdedit /set {current} safeboot minimal
```

**Paso 2** — Reiniciar y entrar a BIOS (F2). Deshabilitar VMD:

```
Advanced → VMD setup menu → Enable VMD controller: [Disabled]
F10 → Save Changes and Exit
```

**Paso 3** — Windows arranca en Modo Seguro y reconfigura los drivers de almacenamiento.

**Paso 4** — Revertir el flag de arranque seguro:

```cmd
bcdedit /deletevalue {current} safeboot
```

**Paso 5** — Reiniciar. Windows arranca normalmente con el controlador en modo AHCI, y Linux ahora detecta el disco.

### Configuración adicional de BIOS

| Opción | Valor | Motivo |
|---|---|---|
| Secure Boot Control | Disabled | Evita conflictos con módulos de kernel de terceros |
| Fast Boot | Disabled | Permite acceder al menú de arranque |
| VMD Controller | Disabled | Habilita la detección del NVMe por parte de Linux |

---

## Problema 3 — Espacio no reducible en el volumen NTFS

### Síntoma

Al intentar reducir el volumen C: desde Administración de discos, Windows ofrecía únicamente **24 GB** de reducción, pese a existir más de 140 GB de espacio libre real en el sistema de archivos.

### Diagnóstico

El espacio libre en NTFS no equivale a espacio reducible. Windows solo puede liberar bloques **contiguos al final del volumen**. Determinados archivos del sistema se marcan como inmovibles y quedan anclados en esa zona, actuando como barrera. Antes de identificar la causa real, intenté la solución obvia: liberar espacio. Borré cerca de 89 GB de archivos y volví a intentar la reducción. Windows seguía ofreciéndome los mismos 24 GB. Ese fue el momento en que entendí que el problema no era de cantidad de espacio libre, sino de dónde estaba ubicado ese espacio dentro del volumen.

Verificación de los tres candidatos habituales:

```cmd
wmic pagefile list /format:list      :: archivo de paginación
powercfg /a                          :: estado de hibernación
vssadmin list shadows                :: puntos de restauración
```

Resultado: hibernación ya deshabilitada, sin instantáneas de volumen, pero **el archivo de paginación seguía activo** (`AllocatedBaseSize: 2944`).

### Resolución

**Deshabilitar el archivo de paginación:**

```cmd
wmic computersystem set AutomaticManagedPagefile=False
wmic pagefileset delete
```

**Deshabilitar hibernación:**

```cmd
powercfg /h off
```

**Eliminar instantáneas de volumen:**

```cmd
vssadmin delete shadows /all /quiet
```

**Deshabilitar Windows Search** (su índice también se ancla al final del volumen):

```
services.msc → Windows Search → Detener → Tipo de inicio: Deshabilitado
```

**Deshabilitar compresión de memoria** (PowerShell elevado):

```powershell
Disable-MMAgent -mc
```

**Reorganizar el volumen:**

```cmd
defrag C: /L
```

**Reiniciar** — sin reinicio los archivos permanecen bloqueados en memoria y los cambios no se materializan en disco.

### Resultado

La capacidad de reducción pasó de **24 GB → 52 GB → 94 GB** de forma progresiva a medida que se liberaban los archivos anclados. Se reservaron finalmente ~94 GB para la instalación de Linux. La reducción no se destrabó de golpe. Fue por etapas: al desactivar el archivo de paginación pasó de 24 a 52 GB, y recién después de deshabilitar Windows Search y correr el defrag llegó a 94 GB. Cada archivo anclado que se libera corre la barrera un poco más atrás.

---

## Esquema de particionado

Se optó por particionado manual (`Más opciones` en el instalador) en lugar del automático, para mantener control explícito sobre qué particiones se modifican.

### Particiones preexistentes — no modificadas

| Dispositivo | Tipo | Tamaño | Función |
|---|---|---|---|
| `/dev/nvme0n1p1` | FAT32 | 272 MB | Partición EFI (compartida) |
| `/dev/nvme0n1p2` | — | 16 MB | Microsoft Reserved |
| `/dev/nvme0n1p3` | NTFS | 415 GB | Windows 11 |
| `/dev/nvme0n1p4` | NTFS | 1,2 GB | Windows Recovery |
| `/dev/nvme0n1p5` | FAT32 | 209 MB | Recovery secundaria |

### Particiones creadas

| Dispositivo | Tipo | Tamaño | Punto de montaje |
|---|---|---|---|
| `/dev/nvme0n1p6` | ext4 | 40 GB | `/` |
| `/dev/nvme0n1p7` | ext4 | 46 GB | `/home` |
| `/dev/nvme0n1p8` | swap | 8,9 GB | — |

**Criterio de separación `/` y `/home`:** permite reinstalar el sistema operativo o migrar a otra distribución formateando únicamente la raíz, conservando archivos personales y configuraciones de usuario intactos.

**Dispositivo de instalación del gestor de arranque:** `/dev/nvme0n1` (disco completo, no una partición individual). Seleccionar una partición específica en este campo es una causa frecuente de sistemas no booteables.

---

## Problema 4 — Fallo en la instalación de GRUB. (Este fue el punto donde estuve más cerca de reinstalar todo desde cero. El instalador usa la palabra "fatal", que da a entender que el proceso completo se perdió. Lo primero que hice fue frenar y verificar qué se había escrito realmente al disco antes de tocar nada.)

### Síntoma

Con todos los archivos del sistema ya copiados al disco, el instalador falló en el paso final:

```
No se pudo instalar GRUB en /dev/nvme0n1
La ejecución de «grub-install /dev/nvme0n1» falló.
Esto es un error fatal.
```

### Diagnóstico

Es importante distinguir el alcance real del fallo: **el sistema de archivos se copió correctamente**. Lo que falló fue exclusivamente la escritura del gestor de arranque en la partición EFI. El sistema está completo pero no tiene forma de arrancar.

Verificación del estado de los montajes desde la sesión Live:

```bash
lsblk -f
```

Confirmó que la partición EFI estaba correctamente montada en `/target/boot/efi`, descartando un problema de montaje y apuntando a un rechazo del firmware al escribir la entrada de arranque.

### Resolución — reinstalación de GRUB vía chroot

La técnica consiste en montar el sistema instalado desde la sesión Live y ejecutar los comandos como si se estuviera dentro de él.

```bash
# Montar la raíz del sistema instalado
sudo mount /dev/nvme0n1p6 /mnt

# Montar la partición EFI en su ubicación relativa
sudo mount /dev/nvme0n1p1 /mnt/boot/efi

# Enlazar los sistemas de archivos virtuales del kernel
sudo mount --bind /dev  /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys  /mnt/sys
sudo mount --bind /run  /mnt/run

# Cambiar la raíz al sistema instalado
sudo chroot /mnt

# Instalar el gestor de arranque
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --recheck

# Regenerar la configuración detectando todos los SO presentes
update-grub

# Salir y reiniciar
exit
sudo reboot
```

### Resultado

GRUB detectó ambos sistemas operativos. El menú de arranque presenta correctamente Linux Mint y Windows Boot Manager, permitiendo alternar entre ambos.

---

## Verificación post-instalación

```bash
juan@juan-VivoBook-ASUSLaptop-TP470EZ:~$ fastfetch
```

```
OS: Linux Mint 22 x86_64
Host: VivoBook_ASUSLaptop TP470EZ
Kernel: 6.8.0-38-generic
Shell: bash 5.2.21
DE: Cinnamon 6.2.9
CPU: 11th Gen Intel i7-1165G7 (8)
GPU: Intel TigerLake-LP GT2 [Iris Xe]
Memory: 3447MiB / 15685MiB
```

Verificaciones funcionales completadas: conectividad Wi-Fi, audio, touchpad, distribución de teclado latinoamericano, control de brillo, y arranque dual operativo en ambos sentidos.

---

## Conclusiones

**Verificar antes de modificar.** El único motivo por el que Windows sobrevivió a este proceso fue haber ejecutado `manage-bde -status` antes de tocar la tabla de particiones. El indicador visual del sistema operativo (el candado abierto) inducía a error; la verificación por línea de comandos dio el estado real.

**El espacio libre no es espacio disponible.** En NTFS, la fragmentación y los archivos inmovibles determinan cuánto se puede reducir realmente un volumen, independientemente de cuánto espacio libre reporte el sistema.

**Delimitar el alcance de un fallo antes de reaccionar.** El error de GRUB se presenta como "error fatal", lo que sugiere reinstalar desde cero. En realidad afectaba solo al último paso de un proceso que ya había completado el 95% del trabajo. Identificar qué falló exactamente evitó repetir toda la instalación.

**Los procesos de cifrado no se interrumpen.** BitLocker tolera interrupciones, pero es una excepción y no debe asumirse como norma en otros sistemas de cifrado de disco.

Concluyo acá mi primera documentación sobre la instalación nativa de Linux en mi Vivabook. Lo escribí porque cuando busqué estos errores encontré hilos de foro sin respuesta, y me pareció que valía la pena dejar el proceso completo en un solo lugar.

---

## Referencias de comandos

| Comando | Función |
|---|---|
| `manage-bde -status C:` | Estado de cifrado del volumen |
| `manage-bde -protectors -get C:` | Obtener protectores y clave de recuperación |
| `manage-bde -off C:` | Iniciar descifrado del volumen |
| `bcdedit /set {current} safeboot minimal` | Forzar arranque en Modo Seguro |
| `bcdedit /deletevalue {current} safeboot` | Revertir arranque en Modo Seguro |
| `powercfg /h off` | Deshabilitar hibernación |
| `vssadmin delete shadows /all` | Eliminar instantáneas de volumen |
| `defrag C: /L` | Optimización TRIM/retrim en SSD |
| `lsblk -f` | Listar dispositivos de bloque y puntos de montaje |
| `mount --bind` | Enlazar sistemas de archivos virtuales para chroot |
| `grub-install` | Instalar el gestor de arranque |
| `update-grub` | Regenerar configuración detectando SO instalados |
