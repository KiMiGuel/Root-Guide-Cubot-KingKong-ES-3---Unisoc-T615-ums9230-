**For English, click [here](./README.md).**

# 🔓 Cubot KingKong ES 3 — Guía de Desbloqueo y Root

**Unisoc T615 (UMS9230_6h10) · UFS · slots A/B**

Son dos partes: **desbloquear el bootloader** (Pasos 1–5, la secuencia clásica BROM/FDL2) y **rootear con Magisk** (Paso 6), con una preparación de herramientas antes (Paso 0).

| 📱 Build | ✅ Probado |
|---|---|
| `CUBOT_KINGKONG_ES_3_F071_V16_20260309` | Pasos 1–5 (desbloqueo) y Paso 6 (root) |
| `CUBOT_KINGKONG_ES_3_F071_V17_20260924` | Paso 6 (root) |

¿Ya actualizaste a un build más nuevo o vas a hacerlo? Ve directo a [Después de una actualización OTA](#-después-de-una-actualización-ota-v16--v17-y-posteriores).

> [!WARNING]
> **Lee esto antes de empezar**
> - 🟡 Dificultad: media/avanzada. Si no sabes lo que estás haciendo, detente aquí.
> - 💥 **Esto borra todo el dispositivo** — respalda tus datos primero.
> - 🔌 **Solo puerto y cable USB 2.0** — el USB 3.0 causa fallos de conexión en este dispositivo.
> - 🐧 Usa **Linux** (Kali). Ni macOS, ni Windows.
> - 📁 **Ejecuta cada comando desde dentro de la carpeta del paquete** — `spd_dump` toma los archivos desde el directorio actual; trabajar desde otro lugar falla en silencio.
> - 🎯 **Solo probado en los builds de arriba.** Para saber cuál tienes: **Ajustes → Acerca del teléfono → Número de compilación**. En cualquier otro, compruébalo antes.
> - 🧯 ¿Se brickeó a la mitad? El modo BROM siempre responde — siempre puedes recuperarlo.
> - ☢️ `prodnv` / `nvitem` / `l_fixnv` contienen datos de calibración de fábrica — nunca los leas, escribas ni borres, en ningún paso.

---

## 🗺️ Los personajes — lee esto una vez y cada comando de abajo tendrá sentido

Cada comando de esta guía hace una de dos cosas: hablar con **BROM**, o (una vez cargados los ayudantes) hablar con **FDL2**. Aquí está quién es quién:

- **BROM (Boot ROM)** — código grabado en el chip Unisoc de fábrica. Nunca se puede borrar ni brickear, y por eso es tu botón de pánico. Con el teléfono apagado, mantener presionada una tecla de volumen mientras conectas el USB lo despierta; el teléfono aparece en `lsusb` como `1782:4d00`. **BROM no hace nada por sí solo** — solo espera a que una PC le envíe código.
- **El exploit (`exec_addr 0x65015f08`)** — normalmente BROM se niega a ejecutar cualquier cosa que no venga firmada de fábrica. CVE-2022-38694 es un bug en el BROM de este chip que nos permite colar nuestro propio código saltándonos esa verificación. Verás `exec_addr 0x65015f08` al inicio de casi todos los comandos — esa es la palanca que abre BROM. Funciona cargando `custom_exec_no_verify_65015f08.bin`, que `spd_dump` toma automáticamente de la carpeta actual (una razón más para trabajar siempre dentro de la carpeta del paquete).
- **FDL1 / FDL2** — los dos programas ayudantes que enviamos a través del BROM ya abierto. **FDL1** despierta la memoria RAM del teléfono; **FDL2** es el verdadero trabajador: puede leer, escribir y borrar cualquier partición de la memoria flash. El patrón que se repite `fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec` simplemente significa: *"carga FDL1, carga FDL2, arráncalos."*
- **Verbos de spd_dump** — una vez que FDL2 está corriendo: `r <partición>` = copia una partición del teléfono a un archivo · `w <partición> <archivo>` = escribe un archivo en una partición · `e <partición>` = borra · `read_part` = lee bytes crudos a un archivo · `reset` = reinicia el teléfono.
- **splloader** — la primera etapa de la cadena de arranque *normal* del teléfono, guardada en su propia partición. **Contiene la verificación del bloqueo del bootloader** — por eso es el objetivo. Nunca "extraes" el splloader de ningún archivo: en el Paso 1 lo copias desde el teléfono (`r splloader`) y parcheas esa copia.
- **uboot** — el bootloader normal (lo que te da fastboot). En el Paso 2 se reemplaza temporalmente con una copia cooperativa (`fdl2-cboot.bin`) para que el desbloqueo pueda correr; el Paso 5 restaura tu original.
- **miscdata** — una partición pequeña y oculta que guarda la bandera de desbloqueo. "Desbloquear" = voltear esa bandera. El Paso 4 la lee de vuelta como prueba.
- **La cadena de arranque, en orden:** `BROM → splloader → uboot → Android`. El desbloqueo (Pasos 1–5) neutraliza la verificación de bloqueo en **splloader**. El root (Paso 6) es un truco aparte que parchea **`init_boot`**, una de las imágenes de arranque propias de Android.

---

## 🧰 Paso 0 — Prepara las herramientas

Descarga el paquete de desbloqueo desde [Releases](https://github.com/KiMiGuel/Cubot-KingKong-ES3-Unisoc-T615-Root/releases) — `cubot_es3_unlock_package_amd64.zip` para una PC normal (x86-64), `cubot_es3_unlock_package_arm64.zip` para máquinas ARM64 — descomprímelo y compila:

```
cd cubot_es3_unlock_package
make
gcc gen_spl-unlock.c -o gen_spl-unlock
gcc chsize.c -o chsize
```

Verifica que compilaste lo correcto — ambas comprobaciones deben pasar:

```
./spd_dump --help          # NO debe listar un comando "baudrate"
grep reopen_port common.c  # debe encontrar el fix de reconexión
```

> [!TIP]
> **Cómo escribir los comandos:** la guía pone `spd_dump …`, `gen_spl-unlock …` y `chsize …` solo para que las líneas queden cortas. Al escribirlos tú, añade `./` al principio (`./spd_dump`, `./gen_spl-unlock`, `./chsize`) y ejecuta `spd_dump` con `sudo` (`sudo ./spd_dump`), porque necesita permisos de root para acceder al USB.

También necesitarás `adb` y `fastboot` de Android para el Paso 6 — instálalos ahora para no quedarte atascado a mitad de la guía:

```
sudo apt install android-tools-adb android-tools-fastboot
adb --version && fastboot --version
```

### 📦 De dónde viene cada archivo

| Archivo | De dónde viene |
|---|---|
| `spd_dump`, `gen_spl-unlock`, `chsize` | Los acabas de compilar desde el código fuente del paquete (comandos de arriba) |
| `fdl1-dl.bin`, `fdl2-dl.bin`, `fdl2-cboot.bin`, `misc-wipe.bin`, `custom_exec_no_verify_65015f08.bin` | Vienen **listos dentro del paquete** — no hay nada que compilar, no los renombres ni los muevas |
| `splloader.bin`, `uboot.bin` | Los extraes de tu propio teléfono en el Paso 1 |
| `spl-unlock.bin` | `gen_spl-unlock` lo crea a partir de tu `splloader.bin` extraído (Paso 1) — es tu splloader con la verificación de bloqueo neutralizada |
| `u-boot-spl-16k-sign.bin`, `uboot_bak.bin` | Tus respaldos renombrados del Paso 1 — se usan para restaurar el teléfono en el Paso 5 |
| `init_boot.img` | **NO viene en el paquete** — lo extraes de tu propio PAC stock (abajo) |
| `magisk_patched-*.img` | La app de Magisk lo crea en el teléfono (Paso 6) |

### 📥 Cómo sacar `init_boot.img` del PAC stock

Un archivo `.pac` es el contenedor de firmware todo-en-uno de Unisoc — un solo archivo grande que contiene todas las imágenes de partición. Tú solo necesitas una imagen de ahí.

1. Descarga el firmware oficial **de la versión que tienes instalada en el teléfono** (**Ajustes → Acerca del teléfono → Número de compilación**; por ejemplo `CUBOT_KINGKONG_ES_3_F071_V16_20260309` o `..._V17_20260924`) desde la página de soporte de Cubot: [cubot.net/soporte](https://cubot.net/Support/id/135/cid/27.html) → KingKong ES 3. El zip pesa unos 3 GB; al descomprimirlo sale el archivo `.pac` (unos 4 GB). ⚠️ **Importante:** si usas un `init_boot.img` de una versión distinta a la del teléfono, se queda en bucle de arranque.
2. Extrae el PAC en Linux:

```
git clone https://github.com/bismoy-bot/PAC-Extractor
cd PAC-Extractor
python3 extractor.py /ruta/a/tu/firmware.pac extracted
```

3. Dentro de `extracted/`, busca `init_boot.img` (si se llama `init_boot_a.img`, renómbralo) y cópialo a la carpeta del paquete.

Trabaja siempre desde dentro de la carpeta del paquete — los respaldos se guardan en el directorio actual.

---

## 💾 Paso 1 — Respalda y parchea tus loaders

Entra a **BROM**: apaga el teléfono → mantén presionado **Volumen Abajo** → conecta el USB (o presiona Encender) → sigue presionando. Si no aparece `1782:4d00`, reintenta con **Volumen Arriba** — la tecla que funciona puede cambiar según el estado del dispositivo. Confirma:

```
lsusb | grep -i '1782:4d00'
```

Extrae `splloader` + `uboot` (cópialos del teléfono a archivos en la carpeta actual):

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset
```

> [!NOTE]
> **Qué hace:** abre BROM con el exploit → carga FDL1+FDL2 → lee `splloader` y `uboot` a `splloader.bin` / `uboot.bin` → borra las ranuras del splloader (se reescriben en el Paso 5) → reinicia.


Espera el reinicio. Si ves `find port failed` → cierra y vuelve a ejecutar — **no** continúes después de ese error.

Genera el payload de desbloqueo, ajusta el tamaño y guarda los respaldos:

```
gen_spl-unlock splloader.bin
chsize uboot.bin
mv splloader.bin u-boot-spl-16k-sign.bin
mv uboot.bin uboot_bak.bin
```

---

## ⚡ Paso 2 — Escribe el FDL2 de trabajo

> [!NOTE]
> **Qué hace:** reemplaza temporalmente la partición `uboot` con `fdl2-cboot.bin`, una copia cooperativa de FDL2 — el uboot stock se negaría a participar en lo que sigue. Tu uboot original está respaldado a salvo y regresa en el Paso 5.


Vuelve a entrar a BROM (Paso 1) y luego:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
```

Espera ~10 segundos después del reinicio.

---

## 🔓 Paso 3 — Ejecuta el desbloqueo

> [!NOTE]
> **Qué hace:** envía el `spl-unlock.bin` neutralizado a través de BROM y lo ejecuta. Es el código del splloader **con la verificación de bloqueo eliminada** — y el simple hecho de que corra voltea la bandera de desbloqueo en `miscdata`. Ese volteo de bandera ES el desbloqueo; no hay ninguna danza de "restauración" después.


Vuelve a entrar a BROM y luego:

```
spd_dump exec_addr 0x65015f08 fdl spl-unlock.bin 0x65000800
```

**Lo que vas a ver, y por qué todo va bien:**

| | Verás | Significa |
|---|---|---|
| 🔄 | `CHECK_BAUD FAIL` | Es normal. El build parcheado se reconecta solo. |
| ✅ | `CHECK_BAUD FDL1` → `BSL_REP_VER: "SPRD3"` → `CMD_CONNECT FDL1` | **Esta ES la señal de éxito.** El desbloqueo ya se aplicó. |
| ⏱️ | `timeout reached` después de eso | **Es exactamente lo que quieres ver.** No es un error. **No** lo vuelvas a ejecutar; pasa al Paso 4. (Solo si nunca ves `CMD_CONNECT FDL1`, ejecútalo una vez más.) |

---

## ✅ Paso 4 — Verifica el desbloqueo

No confías en lo que la herramienta imprimió — lees la bandera directamente desde `miscdata`:

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec verbose 2 read_part miscdata 8192 64 m.bin reset
```

Revisa `m.bin` (`xxd m.bin`): 64 bytes en cero = sigue bloqueado (repite el Paso 3) · una cadena de 32 bytes + dos hashes de 16 bytes = **desbloqueado** ✅. Un timeout al final de este paso es normal.

---

## 🧹 Paso 5 — Restaura y borra

> [!NOTE]
> **Qué hace:** vuelve a armar el teléfono — reescribe tus respaldos originales de `splloader` y `uboot` (el splloader neutralizado y `fdl2-cboot` eran solo invitados temporales), y escribe `misc-wipe.bin` para forzar el restablecimiento de fábrica que un desbloqueo requiere. El `r boot` del comando también guarda una copia de seguridad de tu partición `boot` como `boot.bin`.


```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

Deja que el teléfono haga el restablecimiento de fábrica y arranque. 🟠 **La pantalla naranja de advertencia (unlocked) = éxito.**

---

## 🪄 Paso 6 — Root con Magisk

El bootloader ya está desbloqueado — ahora el root es simplemente que Magisk parchee `init_boot.img` (por eso lo extrajiste en el Paso 0) y flashear la copia parcheada. Se hace en **fastbootd** (fastboot en espacio de usuario), no en el bootloader normal.

Si todavía no lo tienes, instala la app de Magisk en el teléfono: descarga el APK más reciente desde [los releases de Magisk en GitHub](https://github.com/topjohnwu/Magisk/releases) e instálalo (permite "instalar de orígenes desconocidos" si te lo pide).

Arranca Android → Ajustes → Acerca del teléfono → toca **Número de compilación** ×7 → Opciones de desarrollador → **Depuración USB**. Al conectar el teléfono a la PC te preguntará **¿Permitir depuración USB?** — toca **Permitir** (y marca *Permitir siempre*). Después:

```
adb push init_boot.img /sdcard/Download/
```

En la app de Magisk: **Install → Select and Patch a File → `init_boot.img`**, trae de vuelta la imagen parcheada y flasheala — en **fastbootd**, no en el bootloader normal:

```
adb pull /sdcard/Download/magisk_patched-*.img .
mv magisk_patched-*.img init_boot_a.img
adb reboot fastboot                 # va DIRECTO a fastbootd — salta el bootloader por completo.
                                    # Confirmación en pantalla: se ve como Android Recovery —
                                    # "FASTBOOTD" en amarillo arriba, letras rojas debajo.
fastboot devices                    # debe mostrar "<serie> fastbootd" — confirma antes de flashear
fastboot getvar current-slot        # muestra "current-slot: a" o "b" — el root tiene que ir en el slot con el que arrancas
fastboot flash init_boot_a init_boot_a.img    # si dijo b: flashea init_boot_b y renombra el archivo a init_boot_b.img
fastboot reboot
```

Abre Magisk → root activo. 🎉 **Listo — desbloqueado y rooteado.**

---

## 🔄 Después de una actualización OTA (V16 → V17 y posteriores)

Una OTA reemplaza `init_boot`, así que **pierdes el root, pero el bootloader sigue desbloqueado (OEM unlock)**. Normalmente solo hay que repetir el Paso 6 — **salvo que reflashees el firmware completo desde el PAC**, porque eso puede borrar la bandera de desbloqueo (vive en `miscdata`) y tendrías que repetir también los Pasos 1–5.

**Para recuperar el root:**

1. Consigue el PAC de la versión **nueva** y saca de ahí su `init_boot.img` original (Paso 0 → *Cómo sacar `init_boot.img` del PAC stock*).
2. Repite el Paso 6 con el `init_boot.img` nuevo (comprueba antes el `current-slot`).

> [!CAUTION]
> **Lo que no debes hacer**
> - 🚫 **Reutilizar una imagen parcheada de una versión anterior** — deja el teléfono en bootloop.
> - 🚫 **Flashear el `super.img` de un PAC.** Solo trae las particiones de sistema del slot A; las del slot B vienen vacías, así que el slot B se quedaría sin sistema.
> - 🚫 **Desinstalar `com.android.devicelockcontroller`** (aparece en muchas listas de debloat). En V17, sin esa app el sistema se cae una y otra vez y el teléfono se queda para siempre en la animación de arranque.

🩺 **¿Ya lo desinstalaste?** Ejecuta esto desde una PC con adb autorizado y repítelo hasta que responda `installed for user: 0` (falla mientras el sistema sigue reiniciándose):

```
adb shell cmd package install-existing --user 0 com.android.devicelockcontroller
```

---

## 🩹 Solución de problemas

| Síntoma | Solución |
|---|---|
| `LIBUSB_ERROR_BUSY` | `sudo systemctl stop ModemManager`, reconecta |
| `LIBUSB_ERROR_NO_DEVICE` | Inicia `spd_dump` primero, **después** conecta el teléfono |
| `find port failed` | Cierra y vuelve a ejecutar el mismo comando |
| Se desconecta / no detecta el teléfono | Asegúrate de estar en USB 2.0 |
| Dos `CHECK_BAUD FAIL` → `device removed, exiting...` | A tu build de `spd_dump` le falta el fix de reconexión — recompila desde el código fuente del paquete (`common.c` debe contener `reopen_port`) |
| `adb devices` muestra `unauthorized` | El teléfono no reconoce la llave de esta PC. Toca **Permitir** en el teléfono, o copia `adbkey` y `adbkey.pub` (carpeta `~/.android/`) desde una PC que ya habías autorizado, y ejecuta `adb kill-server` |
| Atascado en la animación de arranque después de quitar apps | Mira [Después de una actualización OTA](#-después-de-una-actualización-ota-v16--v17-y-posteriores) — vuelve a instalar `com.android.devicelockcontroller` |
| Boot loop / soft brick en cualquier punto | BROM (Paso 1) siempre responde — reflashea el firmware stock y repite la guía |
| ☠️ En `spd_dump` | **Nunca** ejecutes `repartition`, `erase_all` ni `ufs repart/rawerase/setattr` — truncan la GPT |

---

## 📚 ¿Quieres saber el porqué? La investigación completa está aquí

- [`ROOTING_GUIDE.md`](ROOTING_GUIDE.md) — guía extendida con la evidencia detrás de cada paso (en inglés)
- [`INVESTIGACION_COMPLETA_ES.md`](INVESTIGACION_COMPLETA_ES.md) — investigación completa en español

Este repositorio guarda **solo documentos** — nada de imágenes de firmware, PACs, volcados de particiones, manifiestos de hashes ni archivos `.ko`. Documenta el estado de un dispositivo/build específico; no lo tomes como una receta universal de rooteo.

🙏 Construido sobre el [toolkit de CVE-2022-38694 de TomKing062](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader) (`spd_dump`/`spreadtrum_flash`, `gen_spl-unlock`, `chsize`), [PAC-Extractor](https://github.com/bismoy-bot/PAC-Extractor) de Bismoy Ghosh, y [Magisk](https://github.com/topjohnwu/Magisk) de topjohnwu.
