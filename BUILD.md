# Opposing Force X — Build 32-bit (PC / Linux-WSL)

Compilación de **Xash3D FWGS** + **hlsdk-portable (rama `opfor`)** en **32-bit (i386)**
para jugar *Half-Life: Opposing Force* en PC.

Este es el **paso 1** del port a Xbox original. El motivo de forzar i386 es doble:
1. Los gamedlls originales de GoldSrc son x86 y la ABI del engine asume punteros de 32 bits.
2. El Xbox original es x86 (Pentium III 733 MHz), así que un build i386 limpio valida
   el toolchain y descarta asunciones de 64 bits en el código antes de tocar el target real.

---

## 0. Entorno

| Elemento | Valor |
|---|---|
| Host | Windows 11 Pro 10.0.26200 |
| Subsistema | WSL2 — distro `Ubuntu` (22.04.3 LTS, jammy) |
| Arch host | `amd64` + multiarch `i386` habilitado |
| CPU / RAM | 12 hilos disponibles |
| GCC / G++ | 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04.3) |
| glibc | 2.35 |
| Python | 3.10.12 (para waf) |

### Raíz de trabajo

```
~/opposing-force-x/
```

Desde Windows, esa misma carpeta es accesible en el Explorador como:

```
\\wsl.localhost\Ubuntu\home\<usuario>\opposing-force-x\
```

> **Por qué no se compiló dentro de `C:\...\Opposing Force X`:**
> el path contiene espacios (waf y varios scripts de FWGS lo toleran mal) y `/mnt/c`
> tiene un coste de I/O muy alto en WSL2 (el build tarda varias veces más).
> El repositorio de código y este `BUILD.md` viven en el proyecto de Windows;
> el árbol compilado vive en el filesystem nativo de WSL.

---

## 1. Dependencias instaladas

`sudo` en esta WSL pide contraseña, así que todos los comandos de sistema se
ejecutaron entrando directamente como root (WSL lo permite sin password):

```powershell
wsl -d Ubuntu -u root -- bash
```

### 1.1 Habilitar multiarch i386

```bash
dpkg --add-architecture i386
apt-get update
```

Verificación:

```bash
dpkg --print-foreign-architectures   # -> i386
```

### 1.2 Paquetes

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y --no-install-recommends \
  build-essential \
  gcc-multilib \
  g++-multilib \
  libsdl2-dev:i386 \
  libfontconfig-dev:i386 \
  libfreetype6-dev:i386 \
  zlib1g-dev:i386 \
  libbz2-dev:i386 \
  libopus-dev:i386 \
  libopusfile-dev:i386 \
  libogg-dev:i386 \
  libvorbis-dev:i386 \
  python3 \
  git \
  file \
  pkg-config \
  ca-certificates
```

### 1.3 Versiones exactas instaladas

| Paquete | Versión |
|---|---|
| build-essential | 12.9ubuntu3 |
| gcc-multilib | 4:11.2.0-1ubuntu1 |
| g++-multilib | 4:11.2.0-1ubuntu1 |
| libsdl2-dev:i386 | 2.0.20+dfsg-2ubuntu1.22.04.1 |
| libfontconfig-dev:i386 | 2.13.1-4.2ubuntu5 |
| libfreetype6-dev:i386 | 2.11.1+dfsg-1ubuntu0.3 |
| zlib1g-dev:i386 | 1:1.2.11.dfsg-2ubuntu9.2 |
| libbz2-dev:i386 | 1.0.8-5build1 |
| libopus-dev:i386 | 1.3.1-0.1build2 |
| libopusfile-dev:i386 | 0.9+20170913-1.1build1 |
| libogg-dev:i386 | 1.3.5-0ubuntu3 |
| libvorbis-dev:i386 | 1.3.7-1build2 |
| python3 | 3.10.6-1~22.04.1 |
| git | 1:2.34.1-1ubuntu1.17 |
| file | 1:5.41-3ubuntu0.1 |
| pkg-config | 0.29.2-1ubuntu3 |

### 1.4 Comprobación del toolchain 32-bit

```bash
printf 'int main(void){return 0;}\n' > /tmp/t32.c
gcc -m32 /tmp/t32.c -o /tmp/t32 && file /tmp/t32
```

Salida obtenida:

```
/tmp/t32: ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically linked,
          interpreter /lib/ld-linux.so.2, for GNU/Linux 3.2.0, not stripped
```

### 1.5 `PKG_CONFIG_PATH` (imprescindible)

Sin esto, `pkg-config` devuelve los `.pc` de **amd64** y el configure del motor
o bien falla o bien enlaza librerías de 64 bits. Hay que exportarlo **antes**
de cada `waf configure`:

```bash
export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
```

Comprobación:

```bash
for m in sdl2 opus opusfile ogg vorbis freetype2 fontconfig; do
  printf '%-12s ' "$m"; pkg-config --modversion "$m"; done
```

```
sdl2         2.0.20
opus         1.3.1
opusfile     0.9+20170913
ogg          1.3.5
vorbis       1.3.7
freetype2    24.1.18
fontconfig   2.13.1
```

---

## 2. Motor: Xash3D FWGS

**Commit compilado:** `46add4784b1a31ec46057d23d241ea47c813dd8a` (2026-08-05, tag `continuous`)

```bash
mkdir -p ~/opposing-force-x && cd ~/opposing-force-x
git clone --recursive https://github.com/FWGS/xash3d-fwgs
```

> `--recursive` es **obligatorio**. El README avisa explícitamente:
> *"NEVER USE GitHub's ZIP ARCHIVES. GitHub doesn't include external dependencies we're using!"*
> Se descargan ~15 submódulos (mainui, vgui_support, opus, vorbis, mbedtls, libbacktrace, gl4es…).

```bash
cd ~/opposing-force-x/xash3d-fwgs
export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig

./waf configure -T release
./waf build -j$(nproc)
./waf install --destdir=$HOME/opposing-force-x/game
```

### Nota sobre 32 vs 64 bits

**No hay que pasar ningún flag para 32-bit: es el comportamiento por defecto.**
Del README de xash3d-fwgs:

> *"If your CPU is x86 compatible and you're on Windows or Linux, we are building 32-bit code by default.
> This was done to maintain compatibility with Steam releases of Half-Life."*

El flag `-8` / `--64bits` es el que **opta por 64 bits** — justamente lo que NO queremos.

Líneas relevantes del configure que confirman el target:

```
Checking if 'gcc' generates 32-bit code                 : no
Configuring 'gcc' to generate 32-bit code               : yes
New target CPU                                          : x86
```

### Salida de la instalación

```
+ install .../game/filesystem_stdio.so
+ install .../game/valve/extras.pk3
+ install .../game/libref_gl.so
+ install .../game/libref_soft.so
+ install .../game/libmenu.so
+ install .../game/libvgui_support.so
+ install .../game/xash3d
+ install .../game/libxash.so
```

Tiempo de build: ~21 s con `-j12`.

---

## 3. Gamedlls: Opposing Force (hlsdk-portable)

### Dónde vive el soporte de Op4

El soporte de las expansiones **no está en `master` ni en carpetas aparte**: cada mod
es una **rama** derivada de `master`. Para Opposing Force hay dos:

| Rama | Qué es |
|---|---|
| `opfor` | Implementación de Opposing Force. **La usada aquí.** |
| `opforfixed` | `opfor` + los macros de bugfix de FWGS activados (merge de `opfor`) |

Se eligió `opfor` por ser el comportamiento fiel al original — mejor punto de partida
para un port, donde interesa aislar los cambios propios de los de terceros.
(Otras ramas del repo: `bshift`, `asheep`, `poke646`, `theyhunger`, `decay-pc`, etc.)

**Commit compilado:** `613eb55d5bcd257219c881297d1d43c1da4a7445` (rama `opfor`, 2026-06-14)

### Comandos

```bash
cd ~/opposing-force-x
git clone https://github.com/FWGS/hlsdk-portable
cd hlsdk-portable

git checkout opfor
git submodule update --init --recursive      # <-- imprescindible (vgui_support)

export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
./waf configure -T release --prefix=/
./waf build -j$(nproc)
./waf install --destdir=$HOME/opposing-force-x/game
```

### Cómo sabe waf dónde instalar

La rama `opfor` trae un `mod_options.txt` que el `wscript` parsea en el configure:

```
GAMEDIR=gearbox                # Gamedir path
SERVER_INSTALL_DIR=dlls        # Where to put server dll
CLIENT_INSTALL_DIR=cl_dlls     # Where to put client or menu dll
SERVER_LIBRARY_NAME=opfor      # Default server dll name
```

De ahí sale `install_path = gearbox/dlls` y `gearbox/cl_dlls`, y el nombre `opfor.so`
(el `wscript` además quita el prefijo `lib` del patrón de shared library, por eso es
`opfor.so` / `client.so` y no `libopfor.so`). Los nombres coinciden con los que
espera el `liblist.gam` de la copia de Steam.

Confirmación en el configure:

```
-> processing mod options                                    : ...
* Gamedir path                                               : gearbox
```

Y en el build se ven las fuentes específicas de Op4, prueba de que la rama es la correcta:

```
Compiling dlls/gearbox/displacer.cpp
Compiling dlls/gearbox/sporelauncher.cpp
Compiling dlls/gearbox/penguin.cpp
Compiling dlls/gearbox/eagle.cpp
Compiling dlls/gearbox/m249.cpp
```

### Resultado

```
+ install .../game/gearbox/dlls/opfor.so
+ install .../game/gearbox/cl_dlls/client.so
```

---

## 4. Verificación

### 4.1 Todos los binarios son ELF 32-bit i386

```bash
cd ~/opposing-force-x/game
file xash3d *.so gearbox/dlls/*.so gearbox/cl_dlls/*.so
```

```
xash3d:                    ELF 32-bit LSB pie executable, Intel 80386, ...
filesystem_stdio.so:       ELF 32-bit LSB shared object, Intel 80386, ...
libmenu.so:                ELF 32-bit LSB shared object, Intel 80386, ...
libref_gl.so:              ELF 32-bit LSB shared object, Intel 80386, ...
libref_soft.so:            ELF 32-bit LSB shared object, Intel 80386, ...
libvgui_support.so:        ELF 32-bit LSB shared object, Intel 80386, ...
libxash.so:                ELF 32-bit LSB shared object, Intel 80386, ...
gearbox/dlls/opfor.so:     ELF 32-bit LSB shared object, Intel 80386, ...
gearbox/cl_dlls/client.so: ELF 32-bit LSB shared object, Intel 80386, ...
```

**Recuento: 9 objetos ELF 32-bit / 0 objetos ELF 64-bit.** ✅

Chequeo automatizable (debe imprimir `0`):

```bash
find . -type f \( -name '*.so' -o -name xash3d \) -exec file {} + | grep -c 'ELF 64-bit'
```

### 4.2 El motor arranca y sólo se queja de assets

Sin assets, con `-game gearbox`:

```
[14:58:57] Console initialized.
[14:58:57] ~/opposing-force-x/game is working directory now
[14:58:57] Adding directory: ./
[14:58:57] execing gearbox/vfs.cfg
[14:58:57] Couldn't find game directory 'gearbox'
[14:58:57] Note: Issuing host shutdown due to reason "caught an error"
```

Sin `-game`, el fallo es el equivalente sobre `valve`:

```
[14:59:33] Couldn't find game directory 'valve'
```

Es decir: el ejecutable carga, inicializa la consola, monta el VFS y **muere
exactamente por falta de datos del juego**, no por linkado, arquitectura ni SDL.

### 4.3 Prueba extra: el motor i386 monta la jerarquía de Op4

Para descartar que el error anterior tapara algún problema más profundo, se creó
un `liblist.gam` **mínimo y temporal** en `valve/` y `gearbox/` (ya borrados) y se
relanzó el motor:

```
[14:59:33] FS_AddGameHierarchy( valve )
[14:59:33] Adding directory: valve/
[14:59:33] FS_AddGameHierarchy( gearbox )
[14:59:33] Adding directory: gearbox_downloads/
[14:59:33] Adding directory: gearbox/
[14:59:33] Adding directory: gearbox/custom/
[14:59:33] Host_InitCommon: couldn't load gfx.wad
```

El motor **reconoce `gearbox` como mod válido y monta la cadena `valve` → `gearbox`
correctamente**. El único fallo restante es `gfx.wad`, que es un asset de la copia
de Steam. Confirma que lo único que falta son los datos del juego.

---

## 5. Estructura final de directorios

```
~/opposing-force-x/
├── xash3d-fwgs/            (392 MB) fuentes del motor + submódulos
├── hlsdk-portable/         ( 85 MB) fuentes del SDK, en rama `opfor`
└── game/                   ( 40 MB) <-- ÁRBOL DE JUEGO, aquí se ejecuta todo
    ├── xash3d                      ejecutable (launcher i386)
    ├── libxash.so                  motor
    ├── libref_gl.so                renderer OpenGL
    ├── libref_soft.so              renderer software
    ├── libmenu.so                  menú principal (mainui)
    ├── libvgui_support.so          soporte VGUI1
    ├── vgui.so                     lib VGUI1 (copiada a mano, ver §6.4)
    ├── filesystem_stdio.so         módulo de filesystem
    ├── valve/                      <-- COPIAR AQUÍ los assets de Half-Life
    │   └── extras.pk3              (lo instala el propio motor)
    └── gearbox/                    <-- COPIAR AQUÍ los assets de Opposing Force
        ├── dlls/
        │   └── opfor.so            gamedll de servidor (Op4)
        └── cl_dlls/
            └── client.so           gamedll de cliente (Op4)
```

---

## 6. Dónde copiar tus carpetas `valve/` y `gearbox/`

### 6.1 Origen (tu copia de Steam)

Opposing Force se instala **dentro** de la carpeta de Half-Life:

```
<Steam>\steamapps\common\Half-Life\valve      <-- assets de Half-Life
<Steam>\steamapps\common\Half-Life\gearbox    <-- assets de Opposing Force
```

(Normalmente `<Steam>` = `C:\Program Files (x86)\Steam`.)
Ambas hacen falta: `gearbox` **depende** de `valve` (lo verás en el log
`FS_AddGameHierarchy( valve )` antes de `gearbox`).

### 6.2 Destino

Desde el Explorador de Windows, pega en:

```
\\wsl.localhost\Ubuntu\home\<usuario>\opposing-force-x\game\
```

de forma que quede:

```
...\game\valve\      (fusionado con el valve\ que ya existe)
...\game\gearbox\    (fusionado con el gearbox\ que ya existe)
```

Elige **"Combinar / Merge"**, no "Reemplazar carpeta".

### 6.3 Importante: no te cargues los `.so`

`game\gearbox\` **ya contiene** `dlls\opfor.so` y `cl_dlls\client.so`. La copia de
Steam para Windows trae `dlls\opfor.dll` y `cl_dlls\client.dll`, que tienen nombres
distintos, así que en principio no hay colisión.

Aun así, la forma **a prueba de balas** es copiar primero los assets y después
reinstalar los gamedlls encima:

```bash
cd ~/opposing-force-x/hlsdk-portable
./waf install --destdir=$HOME/opposing-force-x/game
```

Alternativa por línea de comandos (si prefieres copiar desde WSL en vez del Explorador):

```bash
STEAM="/mnt/c/Program Files (x86)/Steam/steamapps/common/Half-Life"
cp -rn "$STEAM/valve/."   ~/opposing-force-x/game/valve/
cp -rn "$STEAM/gearbox/." ~/opposing-force-x/game/gearbox/
```

(`-n` = no sobrescribir lo existente, así los `.so` quedan intactos.)

### 6.4 Comprobación previa al lanzamiento

```bash
cd ~/opposing-force-x/game
ls valve/liblist.gam gearbox/liblist.gam valve/halflife.wad gearbox/gfx.wad
```

Si `gearbox/liblist.gam` no aparece, el motor volverá a decir
`Couldn't find game directory 'gearbox'`.

---

## 7. Comando final para lanzar Opposing Force

Desde una terminal WSL (WSLg proporciona el display; aquí `DISPLAY=:0` y
`WAYLAND_DISPLAY=wayland-0` ya estaban disponibles):

```bash
cd ~/opposing-force-x/game
./xash3d -game gearbox
```

Desde PowerShell en Windows, en una sola línea:

```powershell
wsl -d Ubuntu -- bash -lc "cd ~/opposing-force-x/game && ./xash3d -game gearbox"
```

Flags útiles:

| Flag | Para qué |
|---|---|
| `-game gearbox` | selecciona el mod Opposing Force (**obligatorio**) |
| `-console` | abre la consola del motor al arrancar |
| `-dev 2` | nivel de developer/verbosidad (útil para depurar el port) |
| `-ref soft` | fuerza el renderer software (referencia interesante para Xbox) |
| `-ref gl` | fuerza el renderer OpenGL |
| `-width 640 -height 480` | resolución (640×480 ≈ el target de Xbox) |
| `-windowed` | modo ventana |

Comando recomendado para depurar durante el port:

```bash
./xash3d -game gearbox -console -dev 2 -windowed -width 640 -height 480
```

---

## 8. Errores encontrados y cómo se resolvieron

### 8.1 `sudo: a password is required` en WSL

**Síntoma:** `sudo -n true` falla; la WSL no tiene sudo sin contraseña.
**Solución:** ejecutar los comandos de sistema entrando como root directamente,
que WSL permite sin password:

```powershell
wsl -d Ubuntu -u root -- bash -c "<comando>"
```

### 8.2 Falsos "paquete no disponible" al sondear apt desde PowerShell

**Síntoma:** un bucle de comprobación de paquetes reportó *todos* los paquetes
como `MISSING`, incluidos los que sí existen.
**Causa:** no era un problema de apt sino de **escapado de comillas**: PowerShell
se comía los `$` de las variables de shell al pasar el comando a `wsl -- bash -lc "..."`.
**Solución:** escribir los comandos a un `.sh` y ejecutarlos como fichero. Como los
ficheros se crean en Windows, hay que quitar los CRLF antes:

```powershell
wsl -d Ubuntu -- bash -c "tr -d '\r' < /mnt/c/ruta/script.sh > /tmp/script.sh && bash /tmp/script.sh"
```

Merece la pena recordarlo: **cualquier fallo raro al pilotar WSL desde PowerShell,
sospecha primero del quoting/CRLF y no del sistema.**

### 8.3 `libsdl2-dev:i386` no instalable sin habilitar multiarch

**Síntoma:** los paquetes `:i386` no existen para apt.
**Solución:** `dpkg --add-architecture i386 && apt-get update` **antes** de instalar.

### 8.4 hlsdk-portable: submódulo `vgui_support` sin inicializar

**Síntoma:** `git submodule status` devuelve `-991085982209... vgui_support`
(el guion inicial = no inicializado). El `git clone` se hizo sin `--recursive`.
**Solución:**

```bash
git submodule update --init --recursive
```

Esto trae además el submódulo anidado `vgui_support/vgui-dev`.
**Lección:** clonar hlsdk-portable también con `--recursive`, igual que el motor.

### 8.5 hlsdk: `Checking if 'gcc' can target 32-bit : no` (falsa alarma)

**Síntoma:** el configure imprime:

```
WARNING: will build game for 32-bit target
Checking if 'gcc' can target 32-bit : no
```

y da la impresión de que el build va a salir en 64 bits.

**Explicación:** es el comportamiento normal de `force_32bit.py`. El primer test
comprueba si gcc genera 32 bits **sin flags** (no, porque el gcc por defecto es
amd64); al fallar, y como `BIT32_MANDATORY` está activo, reintenta con `-m32` y
ese sí pasa. Si ambos fallaran, el configure abortaría con
`Compiler can't create 32-bit code!` — no habría "finished successfully".

**Verificación de que efectivamente es 32-bit:**

```bash
grep -oE "DEST_SIZEOF_VOID_P = [0-9]+" build/c4che/_cache.py   # -> 4
grep -oE "'-m32'" build/c4che/_cache.py                        # -> aparece en CFLAGS/CXXFLAGS/LINKFLAGS
```

Y en última instancia, el `file` de §4.1.

### 8.6 xash3d-fwgs: `Checking for 'opus' : not found` (no bloqueante)

**Síntoma:** el configure no encuentra `opus` por pkg-config pese a tener
`libopus-dev:i386` instalado y `PKG_CONFIG_PATH` bien puesto (`opusfile`, `ogg` y
`vorbis` sí los encuentra).
**Impacto:** ninguno. El motor cae automáticamente al opus **incluido como submódulo**
en `3rdparty/opus`, que se compila junto al resto:

```
--> 3rdparty/opus : in progress
Checking for C99 lrint : yes
...
<-- 3rdparty/opus : done
```

**Solución aplicada:** ninguna, se deja el opus bundled. Para el port a Xbox esto
incluso es preferible: menos dependencias del sistema que portar.

### 8.7 `vgui.so => not found` en `libvgui_support.so`

**Síntoma:**

```bash
ldd libvgui_support.so | grep 'not found'
#   vgui.so => not found
```

`waf install` del motor **no copia** `vgui.so` al directorio de juego.
**Impacto real:** ninguno para Op4 — se comprobó que `client.so` de `opfor`
**no enlaza** VGUI:

```bash
ldd gearbox/cl_dlls/client.so | grep -i vgui   # (sin resultados)
```

**Solución aplicada (preventiva):** copiar el `vgui.so` i386 que viene en el
submódulo vgui-dev al directorio de juego:

```bash
cp ~/opposing-force-x/xash3d-fwgs/3rdparty/vgui_support/vgui-dev/lib/vgui.so \
   ~/opposing-force-x/game/
```

Verificado como `ELF 32-bit LSB shared object, Intel 80386`.

### 8.8 El README del motor recomienda `aptitude` en vez de `apt`

El README de xash3d-fwgs sugiere instalar con
`aptitude --without-recommends` por [este problema](https://github.com/FWGS/xash3d-fwgs/issues/1828).
**En este entorno no hizo falta:** `apt-get install --no-install-recommends`
resolvió todas las dependencias i386 sin conflictos. Se deja anotado por si en
una reinstalación apt intenta desinstalar medio sistema para satisfacer i386;
en ese caso, usar aptitude.

### 8.9 Error esperado (no es un bug): `Couldn't find game directory 'gearbox'`

No basta con que exista la carpeta `gearbox/`: el motor necesita el manifiesto
`liblist.gam` (o `gameinfo.txt`) dentro. Viene con la copia de Steam.
Ver §6.

---

## 9. Notas para el port a Xbox original

Observaciones de este build que afectan al paso 2:

- **La ABI de 32 bits está validada de punta a punta**: launcher, motor, renderers
  y ambos gamedlls son i386 puros. No ha hecho falta parchear nada del código para
  ello, lo que indica que el árbol de FWGS no arrastra asunciones de 64 bits.
- **Ambos proyectos compilan 32-bit por defecto** en x86; el flag `-8` es el que
  opta por 64 bits. Para el target Xbox conviene mantener el default.
- **`libref_soft.so` existe y se compila**: hay un renderer por software además
  del de OpenGL. Dado que la NV2A del Xbox necesitará un backend propio, el
  renderer software es un buen punto de partida para arrancar sin GPU y aislar
  problemas (`-ref soft`).
- **Dependencias del sistema a sustituir/portar:** SDL2 (vídeo/input/audio),
  freetype2 + fontconfig (fuentes TTF del menú), bz2 y zlib. Opus, Ogg, Vorbis,
  mbedtls y libbacktrace ya vienen como submódulos bundled, lo que reduce el
  trabajo de portado.
- **VGUI1 es prescindible para Op4** (§8.7): `client.so` no lo enlaza. Un candidato
  claro a recortar en el build de Xbox.
- **`opfor.so` se carga dinámicamente** vía `liblist.gam`. En Xbox, sin `dlopen`,
  habrá que enlazar el gamedll estáticamente contra el motor; el nombre y el punto
  de entrada están fijados en `mod_options.txt` (`SERVER_LIBRARY_NAME=opfor`).

---

## 10. Reconstruir desde cero (resumen ejecutable)

```bash
# --- root: dependencias ---
dpkg --add-architecture i386 && apt-get update
apt-get install -y --no-install-recommends build-essential gcc-multilib g++-multilib \
  libsdl2-dev:i386 libfontconfig-dev:i386 libfreetype6-dev:i386 zlib1g-dev:i386 \
  libbz2-dev:i386 libopus-dev:i386 libopusfile-dev:i386 libogg-dev:i386 \
  libvorbis-dev:i386 python3 git file pkg-config ca-certificates

# --- usuario: build ---
export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
mkdir -p ~/opposing-force-x && cd ~/opposing-force-x
GAME=$HOME/opposing-force-x/game

git clone --recursive https://github.com/FWGS/xash3d-fwgs
cd xash3d-fwgs
./waf configure -T release && ./waf build -j$(nproc) && ./waf install --destdir=$GAME
cp 3rdparty/vgui_support/vgui-dev/lib/vgui.so $GAME/
cd ..

git clone --recursive https://github.com/FWGS/hlsdk-portable
cd hlsdk-portable
git checkout opfor
git submodule update --init --recursive
./waf configure -T release --prefix=/ && ./waf build -j$(nproc) && ./waf install --destdir=$GAME
cd ..

# --- verificar ---
file $GAME/xash3d $GAME/*.so $GAME/gearbox/dlls/*.so $GAME/gearbox/cl_dlls/*.so

# --- copiar assets de Steam y jugar ---
STEAM="/mnt/c/Program Files (x86)/Steam/steamapps/common/Half-Life"
cp -rn "$STEAM/valve/."   $GAME/valve/
cp -rn "$STEAM/gearbox/." $GAME/gearbox/
cd $GAME && ./xash3d -game gearbox
```

---

## 11. Parches locales permanentes

**Decisión tomada: estos parches NO se upstreamean.** Se mantienen como parches
locales sobre los clones de trabajo. Esta sección es la referencia para reaplicarlos.

Viven como ficheros `.patch` en la raíz del proyecto:

```
~/opposing-force-x/
├── mingw-server-snprintf.patch              -> hlsdk-portable   (§11.1)
├── xash-static-linking-no-default-pie.patch -> xash3d-fwgs      (§11.2)
├── xash-xshlib-static-linking-fixes.patch   -> xash3d-fwgs      (§11.3)
├── hlsdk-vcs-info-inline.patch              -> hlsdk-portable   (§11.4)
└── xash-saverestore-offsets-priority.patch  -> xash3d-fwgs      (§11.5)
```

Los dos primeros son necesarios para los builds que ya funcionan (win32 y
`--static-linking` respectivamente). Los dos últimos solo hacen falta para el
experimento de §14 y **no afectan** a los builds de `game/` ni `game-win32/`.

### 11.1 hlsdk-portable — `mingw-server-snprintf.patch`

| | |
|---|---|
| Fichero | `dlls/CMakeLists.txt` |
| Qué arregla | El cross-compile mingw del gamedll de servidor no compila (§12.6.1) |
| Por qué | `-D_snprintf=snprintf` choca con la `stdio.h` de MinGW. `cl_dll/CMakeLists.txt` ya lo protege con `if(NOT MINGW)`; `dlls/` no |
| Obligatorio | **Sí** para el build win32. Irrelevante para el build Linux (waf no usa CMake) |
| Upstream | No enviado. Ver §12.6.5: upstream retiró el soporte de mingw en 2022 ([PR #283](https://github.com/FWGS/hlsdk-portable/pull/283)) |

### 11.2 xash3d-fwgs — `xash-static-linking-no-default-pie.patch`

| | |
|---|---|
| Fichero | `wscript` |
| Qué arregla | `--static-linking` no enlaza en toolchains con GCC `--enable-default-pie` (§13.5.1) |
| Por qué | `check_pic(False)` añade `-fno-PIC`, que no desactiva PIE; siguen emitiéndose los PC-thunks i386 en grupos COMDAT que `objcopy -G` rompe |
| Obligatorio | Solo si usas `--static-linking`. Inocuo en el resto de builds (el check ni se ejecuta) |
| Upstream | No enviado, por decisión propia. El bug es real y sigue en master |

### 11.3 xash3d-fwgs — `xash-xshlib-static-linking-fixes.patch`

| | |
|---|---|
| Fichero | `scripts/waifulib/xshlib.py` |
| Qué arregla | Dos bugs del mecanismo `--static-linking` (§14.3 y §14.4a) |
| Obligatorio | Solo para el experimento de §14 |

Contiene dos cambios independientes:

1. **`NAME` de objcopy.** `xshlib` generaba la tabla de exports con el nombre del
   *taskgen* (`lib_server_exports`) pero ejecutaba `objcopy -G` con el nombre del
   *target* (`lib_opfor_exports`). Cuando difieren, objcopy no encuentra el símbolo y
   **localiza absolutamente todo** (6683 globales → 0). Corregido a
   `env['NAME'] = self.generator.name`, que es de donde salen los nombres en
   `write_export_list` y `write_libraries_list`. Para los módulos del motor
   `name == target`, así que su comportamiento no cambia.
2. **`tg.post()` en `add_deps`**, porque el binario principal puede postearse antes
   que los módulos que va a absorber.

### 11.4 hlsdk-portable — `hlsdk-vcs-info-inline.patch`

| | |
|---|---|
| Ficheros | `dlls/wscript`, `cl_dll/wscript` |
| Qué arregla | Hace los gamedlls autocontenidos: elimina la dependencia `use='vcs_info'` |
| Por qué | `ld -r` no consume librerías, así que la estática `libvcs_info.a` nunca entraba y quedaban `undefined reference to g_VCSInfo_Commit` (§14.4b) |
| Cómo | Compila `game_shared/vcs_info.c` directamente en las fuentes de ambos targets |
| Obligatorio | Solo para el experimento de §14. Inocuo en los builds normales |

### 11.5 xash3d-fwgs — `xash-saverestore-offsets-priority.patch`

| | |
|---|---|
| Fichero | `engine/platform/posix/lib_posix.c` |
| Qué hace | Hace que `XASH_ALLOW_SAVERESTORE_OFFSETS` **realmente active** el guardado por offsets |
| Obligatorio | Solo para el build de validación de §15 (`game-offsets/`) |

**No es un arreglo de bug: es un cambio de comportamiento deliberado.** Upstream solo
llega al camino de offsets como *fallback* de que `dladdr()` falle, cosa que no ocurre
nunca con un gamedll dinámico. El parche invierte la prioridad cuando la macro está
definida. Ver §15.1.

Se usa junto con la macro en la línea de comandos:

```bash
CFLAGS="-DXASH_ALLOW_SAVERESTORE_OFFSETS" CXXFLAGS="-DXASH_ALLOW_SAVERESTORE_OFFSETS" \
  ./waf configure -T release -o build-offsets
```

### 11.6 Reaplicar tras un `git pull`

```bash
cd ~/opposing-force-x/hlsdk-portable
git pull
git apply ~/opposing-force-x/mingw-server-snprintf.patch
git apply ~/opposing-force-x/hlsdk-vcs-info-inline.patch      # solo para §14

cd ~/opposing-force-x/xash3d-fwgs
git pull
git apply ~/opposing-force-x/xash-static-linking-no-default-pie.patch
git apply ~/opposing-force-x/xash-xshlib-static-linking-fixes.patch      # solo para §14
git apply ~/opposing-force-x/xash-saverestore-offsets-priority.patch     # solo para §15
```

**Comprobación rápida de que están aplicados:**

```bash
grep -q 'NOT MINGW'            ~/opposing-force-x/hlsdk-portable/dlls/CMakeLists.txt   && echo 'hlsdk snprintf      : OK' || echo 'hlsdk snprintf      : FALTA'
grep -q 'fno-pie'              ~/opposing-force-x/xash3d-fwgs/wscript                  && echo 'xash no-default-pie : OK' || echo 'xash no-default-pie : FALTA'
grep -q 'self.generator.name'  ~/opposing-force-x/xash3d-fwgs/scripts/waifulib/xshlib.py && echo 'xash xshlib NAME    : OK' || echo 'xash xshlib NAME    : FALTA'
grep -q 'game_shared/vcs_info.c' ~/opposing-force-x/hlsdk-portable/dlls/wscript        && echo 'hlsdk vcs_info      : OK' || echo 'hlsdk vcs_info      : FALTA'
grep -q 'offsets take priority'  ~/opposing-force-x/xash3d-fwgs/engine/platform/posix/lib_posix.c && echo 'xash ofs priority   : OK' || echo 'xash ofs priority   : FALTA'
```

**Si `git apply` falla** porque upstream tocó las mismas líneas, aplica a mano
guiándote de §12.6.1 y §13.5.1 (los dos son de pocas líneas) y regenera el `.patch`:

```bash
cd <repo> && git diff > ~/opposing-force-x/<nombre>.patch
```

> **Ojo:** `git pull` en `hlsdk-portable` te devuelve a la rama en la que estés.
> El build de Op4 usa la rama **`opfor`**, no `master` (§3).

---

## 12. Build win32 (Windows nativo)

**Motivación:** el input de mando no atraviesa WSL, así que para jugar y probar de
verdad hace falta un build que corra en Windows nativo. Además, win32 nos acerca al
objetivo final: las APIs del Xbox original derivan de Win32.

**Estrategia (asimétrica a propósito):**

| Componente | Cómo se obtiene | Por qué |
|---|---|---|
| Motor (`xash3d.exe`, `xash.dll`, renderers…) | **Binario oficial descargado** de la release `continuous` | No aporta nada cross-compilarlo; el oficial es el de referencia y trae SDL2/ffmpeg ya resueltos |
| Gamedlls (`opfor.dll`, `client.dll`) | **Cross-compilados** con mingw-w64 i686 | Es *nuestro* código, el que acabará en el Xbox. Tiene que salir de nuestro toolchain |

### 12.1 Dependencias añadidas

```bash
# como root
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  binutils-mingw-w64-i686 \
  gcc-mingw-w64-i686 \
  g++-mingw-w64-i686 \
  mingw-w64-i686-dev \
  mingw-w64-common \
  cmake \
  ninja-build \
  p7zip-full
```

| Paquete | Versión |
|---|---|
| binutils-mingw-w64-i686 | 2.38-3ubuntu1+9build1 |
| gcc-mingw-w64-i686 | 10.3.0-14ubuntu1+24.3 |
| g++-mingw-w64-i686 | 10.3.0-14ubuntu1+24.3 |
| mingw-w64-i686-dev | 8.0.0-1 |
| mingw-w64-common | 8.0.0-1 |
| cmake | 3.22.1-1ubuntu1.22.04.2 |
| ninja-build | 1.10.1-1 |
| p7zip-full | 16.02+dfsg-8 (para descomprimir el `.7z` del motor) |

Toolchain resultante: `i686-w64-mingw32-gcc/g++ (GCC) 10-win32 20220113`
(variante de hilos **win32**, que es la que selecciona `update-alternatives` por defecto).

Comprobación:

```bash
printf 'int __stdcall DllMain(void*a,unsigned long b,void*c){return 1;}\n' > /tmp/d32.c
i686-w64-mingw32-gcc -shared /tmp/d32.c -o /tmp/d32.dll && file /tmp/d32.dll
# -> PE32 executable (DLL) (console) Intel 80386, for MS Windows
```

### 12.2 Cómo cross-compila FWGS para Windows: waf **no**, CMake **sí**

Esto hubo que investigarlo. Resumen de lo encontrado en el repo:

- `.github/workflows/build.yml` compila Windows con **MSVC (`cl`) + CMake** sobre un
  runner `windows-latest`. No hay job de cross-compile con mingw.
- `.travis.yml` (CI histórico) **sí** documenta el cross-compile con mingw, y lo hace
  con **CMake**:
  ```
  cmake ../ -DCMAKE_SYSTEM_NAME=Windows \
            -DCMAKE_C_COMPILER=i686-w64-mingw32-gcc \
            -DCMAKE_CXX_COMPILER=i686-w64-mingw32-g++ && make -j3
  ```
  y en el paso de instalación de dependencias instala exactamente
  `mingw-w64-i686-dev binutils-mingw-w64-i686 gcc-mingw-w64-i686 g++-mingw-w64-i686`.
- El `CMakeLists.txt` raíz tiene un bloque **`if (MINGW)`** específico:
  ```cmake
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++ -static-libgcc")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--add-stdcall-alias")
  ```
  Ambos flags son **imprescindibles**: el primero evita depender de
  `libstdc++-6.dll`/`libgcc_s_dw2-1.dll` en el sistema del usuario, y el segundo
  genera el alias `GiveFnptrsToDll@8` que el motor busca por nombre decorado stdcall.
- El `scripts/waifulib/xcompile.py` de waf **no** tiene ruta para mingw (solo Android
  y msvc-wine), y no aplicaría ninguno de esos dos flags.

**Conclusión: para win32 se usa CMake, no waf.** Es la única asimetría respecto al
build Linux de §3.

> Nota adicional: `vgui_support/wscript` y `scripts/waifulib/vgui.py` avisan de que
> *"vgui can't be linked with MinGW"* (incompatibilidad de ABI). No es problema aquí:
> `USE_VGUI` está **OFF por defecto** y ya se comprobó en §8.7 que el cliente de Op4
> no enlaza VGUI.

### 12.3 Comandos

Mismo repo, misma rama, **mismo commit `613eb55`** que el build Linux — solo cambia
el directorio de build (`build-win32/`), así que ambos builds conviven sin pisarse.

```bash
cd ~/opposing-force-x/hlsdk-portable
git branch --show-current     # opfor
git log -1 --format=%h        # 613eb55

cmake -B build-win32 -S . \
  -DCMAKE_SYSTEM_NAME=Windows \
  -DCMAKE_C_COMPILER=i686-w64-mingw32-gcc \
  -DCMAKE_CXX_COMPILER=i686-w64-mingw32-g++ \
  -DCMAKE_RC_COMPILER=i686-w64-mingw32-windres \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$HOME/opposing-force-x/game-win32

cmake --build build-win32 -j$(nproc)
cmake --install build-win32
```

Líneas del configure que confirman el target:

```
-- Gamedir path set to gearbox
-- Default server dll name set to opfor
-- Looking for XASH_64BIT - not found
-- Looking for XASH_WIN32 - found
-- Looking for XASH_X86 - found
-- Library postfix:
-- Building for 32 Bit
```

Resultado:

```
-- Installing: .../game-win32/gearbox/cl_dlls/client.dll
-- Installing: .../game-win32/gearbox/dlls/opfor.dll
```

> ⚠️ **Requiere un parche de una línea.** Ver §12.6.1. Sin él, el servidor no compila.

### 12.4 Motor oficial win32

Release: `continuous`, publicada `2026-08-05T23:10:23Z`.
Asset: **`xash3d-fwgs-win32-i386.7z`** (20.7 MB) — *ojo, `-i386`, no `-amd64`*.

```bash
mkdir -p ~/opposing-force-x/downloads && cd ~/opposing-force-x/downloads
curl -sSL -O https://github.com/FWGS/xash3d-fwgs/releases/download/continuous/xash3d-fwgs-win32-i386.7z
sha256sum xash3d-fwgs-win32-i386.7z
# 4a933508b10c5900e0af4881b96bcb56b5c4edb7fed6e89a43fcd2e9902cea8b

mkdir -p extract && 7z x -oextract xash3d-fwgs-win32-i386.7z
cp -r extract/. ~/opposing-force-x/game-win32/
```

El `.7z` no tiene carpeta raíz: descomprime los ficheros directamente. Se copia
*sobre* `game-win32/`, que ya contiene `gearbox/` con nuestros DLLs — no hay colisión
porque el release no trae `gearbox/`.

Versión del motor obtenida (leída de `engine.log`):

```
Xash3D FWGS 0.21 (4145, 15276ef, master, win32-i386)
```

### 12.5 Verificación

**Todos los binarios son PE32 i386:**

```bash
find ~/opposing-force-x/game-win32 -type f \( -name '*.dll' -o -name '*.exe' \) -exec file {} +
```

```
gearbox/dlls/opfor.dll:     PE32 executable (DLL) (console) Intel 80386, for MS Windows
gearbox/cl_dlls/client.dll: PE32 executable (DLL) (console) Intel 80386, for MS Windows
xash3d.exe:                 PE32 executable (GUI) Intel 80386, for MS Windows
xash.dll:                   PE32 executable (DLL) (GUI) Intel 80386, for MS Windows
... (18 binarios en total)
```

**Recuento: 18 binarios PE32 i386 / 0 PE32+ (64-bit).** ✅

Chequeo automatizable (debe imprimir `0`):

```bash
find ~/opposing-force-x/game-win32 -type f \( -name '*.dll' -o -name '*.exe' \) \
  -exec file {} + | grep -c 'PE32+'
```

**Los DLLs exportan la interfaz que GoldSrc espera:**

```bash
i686-w64-mingw32-objdump -p game-win32/gearbox/dlls/opfor.dll \
  | sed -n '/\[Ordinal\/Name Pointer\] Table/,/^$/p'
```

| Símbolo | opfor.dll |
|---|---|
| `GiveFnptrsToDll` | ✅ presente **y** con alias stdcall `GiveFnptrsToDll@8` |
| `GetEntityAPI` | ✅ |
| `GetEntityAPI2` | ✅ |
| `Server_GetPhysicsInterface` | ✅ |
| `GetNewDLLFunctions` | ausente — **correcto**, no existe en el código de esta rama; el `opfor.so` de Linux ya validado tampoco lo exporta |

`client.dll` exporta `Initialize`, `HUD_Init`, `HUD_VidInit`, `HUD_Redraw`,
`HUD_GetHullBounds`, `HUD_PlayerMove`… (71 exports).

> Recuento de exports: **`opfor.dll` 661**, **`client.dll` 71**.

**Dependencias limpias** — solo DLLs del sistema, ninguna de mingw:

```
opfor.dll  -> KERNEL32.dll, msvcrt.dll
client.dll -> KERNEL32.dll, msvcrt.dll, USER32.dll
```

Esto confirma que `-static-libstdc++ -static-libgcc` hizo su trabajo: los DLLs son
autocontenidos y no hace falta distribuir runtime de mingw.

**El motor arranca en Windows nativo** (probado copiando el árbol a un directorio
local de Windows y lanzando `xash3d.exe -game gearbox -log`):

```
Xash3D FWGS 0.21 (4145, 15276ef, master, win32-i386)
Couldn't find game directory 'gearbox'
```

Y con `liblist.gam` mínimos temporales, igual que en §4.3:

```
Host_InitCommon: couldn't load gfx.wad
```

**Paridad exacta con el build Linux**: muere en el mismo punto y solo por falta de
assets.

### 12.6 Errores encontrados

#### 12.6.1 El servidor no compila con MinGW: `conflicting declaration ... with 'C' linkage`

**Síntoma:** `client.dll` enlaza bien, pero el target `server` revienta en el primer
`.cpp`:

```
[ 34%] Building CXX object dlls/CMakeFiles/server.dir/gearbox/blkop_apache.cpp.obj
<command-line>: error: conflicting declaration of 'int snprintf(char*, size_t, const char*, ...)' with 'C' linkage
/usr/share/mingw-w64/include/stdio.h:437:5: note: previous declaration with 'C++' linkage
<command-line>: error: conflicting declaration of 'int vsnprintf(char*, size_t, const char*, va_list)' with 'C' linkage
/usr/share/mingw-w64/include/stdio.h:450:5: note: previous declaration with 'C++' linkage
gmake[2]: *** [dlls/CMakeFiles/server.dir/build.make:77: .../blkop_apache.cpp.obj] Error 1
```

**Causa — inconsistencia del propio repo.** Los defines
`-D_snprintf=snprintf -D_vsnprintf=vsnprintf` chocan con la `stdio.h` de mingw, que
declara `snprintf`/`vsnprintf` con linkage C++. El repo **ya sabe esto** y lo protege
en el cliente, pero no en el servidor:

| Fichero | Línea | Guarda `if(NOT MINGW)` |
|---|---|---|
| `cl_dll/CMakeLists.txt` | 36-37 | ✅ sí |
| `dlls/CMakeLists.txt` | 34 | ❌ **no** |

Por eso `client.dll` compilaba y `opfor.dll` no.

**Solución:** aplicar en `dlls/CMakeLists.txt` exactamente el mismo patrón que ya usa
`cl_dll/CMakeLists.txt`:

```diff
-	add_definitions(-Dstricmp=strcasecmp -Dstrnicmp=strncasecmp -D_snprintf=snprintf -D_vsnprintf=vsnprintf )
+	add_definitions(-Dstricmp=strcasecmp -Dstrnicmp=strncasecmp)
+	if(NOT MINGW)
+		add_definitions(-D_snprintf=snprintf -D_vsnprintf=vsnprintf)
+	endif()
```

El parche está guardado en **`~/opposing-force-x/mingw-server-snprintf.patch`** y se
reaplica con:

```bash
cd ~/opposing-force-x/hlsdk-portable && git apply ~/opposing-force-x/mingw-server-snprintf.patch
```

**Estado del árbol:** `hlsdk-portable` tiene esta modificación local sin commitear
(`git status` muestra `M dlls/CMakeLists.txt`). Es **el único** parche activo. **No
afecta al build Linux de §3**, porque waf no usa los `CMakeLists.txt`. Es candidato a
PR upstream.

#### 12.6.2 `waf` no sirve para cross-compilar a win32

Intento descartado antes de gastar tiempo: `scripts/waifulib/xcompile.py` no
contempla mingw y no aplicaría `-static-libstdc++` ni `--add-stdcall-alias`. Ver
§12.2. **No es un bug, es que esa ruta no existe en el proyecto.**

#### 12.6.3 Los exports parecían ausentes (falsa alarma de diagnóstico)

Al comprobar los símbolos con `objdump -p`, un primer intento reportó todos los
exports como ausentes pese a que el DLL tenía 662. El fallo era del comando de
diagnóstico, no del DLL: los nombres están en la tabla
`[Ordinal/Name Pointer] Table` (no en `Export Address Table`) y el formato es
`[   0] Nombre`, con espacios **dentro** de los corchetes. Regex correcta:

```bash
sed -E 's/^[[:space:]]*\[[[:space:]]*[0-9]+\][[:space:]]*//'
```

#### 12.6.4 Comillas y CRLF al pilotar WSL desde PowerShell

Vuelve a aplicar lo de §8.2. Añadido: PowerShell también rompe comandos con comillas
dobles anidadas (`grep -oE '"(name|url)"'`) y no reconoce `wc`, `head` etc. si el
comando se parte mal. **Todo comando no trivial va a un `.sh` y se ejecuta como
fichero.**

#### 12.6.5 Falsa alarma: `Can't find proc: DoorTouch@CBaseDoor`

**Resuelto — no era un bug del build.** Se deja documentado porque la investigación
dejó información útil, y porque la vía muerta merece estar escrita.

**Síntoma:** al cargar partida, el motor escupía `Can't find proc: DoorTouch@CBaseDoor`
(y decenas más) y las entidades quedaban sin lógica.

**Causa real:** contenido corrupto en las carpetas `valve/` y `gearbox/` copiadas.
Regenerándolas, el save/restore funciona correctamente con el `opfor.dll` de MinGW.
**Las puertas funcionan.**

**Intento descartado — `-Wl,--export-all-symbols`:** la hipótesis fue que el gamedll
solo exportaba los entry points, porque un `__declspec(dllexport)` explícito desactiva
la heurística de auto-export de `ld`. Se aplicó el flag y **se revirtió después**.
Datos medidos, por si alguna vez hace falta:

| | sin el flag | con el flag |
|---|---|---|
| exports de `opfor.dll` | **661** ← estado actual | 6035 |
| tamaño | **2.870.688 B** ← estado actual | 3.084.704 B |

El flag hace lo que promete, pero era innecesario: `DoorTouch` **ya estaba exportado**
sin él, como `_ZN9CBaseDoor9DoorTouchEP11CBaseEntity`. El problema nunca fue la
cantidad de símbolos.

**Lo que sí conviene saber del mecanismo** (esto es correcto y sigue siendo relevante):

- El save/restore de GoldSrc serializa los punteros a función de las entidades
  (`SetTouch`, `SetThink`, `SetUse`…) **por nombre**, vía `pfnFunctionFromName` /
  `pfnNameForFunction` (`dlls/enginecallback.h:116-117`).
- En win32 el motor guarda los nombres del export table **tal cual**: `lib_win.c:265`
  los pasa por `COM_GetMSVCName()`, que solo transforma nombres que empiezan por `?`
  (mangling MSVC) y **deja intacto todo lo demás** — incluidos los nombres Itanium de
  GCC/MinGW.
- Como `COM_NameForFunction` y `COM_FunctionFromName` (`lib_win.c:558` y `:542`) usan
  **esa misma lista**, un save escrito por nuestro build guarda
  `_ZN9CBaseDoor9DoorTouchEP11CBaseEntity` y al restaurar busca esa misma cadena.
  **Es autoconsistente y por eso funciona.**
- La forma neutra `DoorTouch@CBaseDoor` es la que produce **otro** build: en Linux,
  `lib_posix.c:154` devuelve `COM_GetPlatformNeutralName( info.dli_sname )`.

> ⚠️ **Regla práctica que sale de aquí:** los saves **no son intercambiables entre el
> árbol `game/` (Linux) y `game-win32/`**, ni con los de la copia retail de Steam. Cada
> build escribe los nombres en su propio formato. No compartas la carpeta `SAVE/` entre
> ellos: es precisamente el tipo de mezcla que produce este error.

#### 12.6.6 `SV_LoadProgs: couldn't get physics API` — benigno y por diseño

**No desaparece por más símbolos que exporte el gamedll, y no puede desaparecer.** Es
independiente de lo anterior.

Lado del gamedll (`dlls/cbase.cpp:130`):

```cpp
int Server_GetPhysicsInterface( int version, server_physics_api_t *api, physics_interface_t *interface )
{
	g_fIsXash3D = true;
	return FALSE; // do not tell engine to init physics interface, as we're not using it
}
```

Devuelve `FALSE` **siempre, a propósito**. Su única función real es poner
`g_fIsXash3D = true`: sirve para que el gamedll detecte que corre sobre Xash3D y no
sobre GoldSrc.

Lado del motor (`engine/server/sv_phys.c:2101`), y aquí está el detalle no obvio:

```c
pPhysIface = COM_GetProcAddress( svgame.hInstance, "Server_GetPhysicsInterface" );
if( pPhysIface ) {
    if( pPhysIface( ... )) { ...; return true; }     // presente y acepta -> OK
    ...
    return false;                                     // presente y rechaza -> WARNING
}
return true;                                          // ausente -> sin warning
```

Es decir: **el warning aparece precisamente porque el símbolo SÍ está exportado** y
declina la interfaz. Si no estuviera exportado, el motor devolvería `true` y no diría
nada. Exportar *más* símbolos no puede quitarlo — la única forma sería dejar de
exportar `Server_GetPhysicsInterface`, lo que rompería la detección de Xash3D.

**Es informativo (`S_WARN`, no `S_ERROR`) y se puede ignorar.** La API de físicas
extendida es una extensión opcional de Xash3D que hlsdk-portable decide no implementar.

### 12.7 Estructura final

```
~/opposing-force-x/
├── game/                       build Linux i386 (§1-§7), ya con tus assets
├── game-win32/                 <-- ÁRBOL WINDOWS
│   ├── xash3d.exe              motor oficial (PE32 i386)
│   ├── xash.dll
│   ├── ref_gl.dll / ref_soft.dll
│   ├── menu.dll / menu_tui.dll
│   ├── filesystem_stdio.dll
│   ├── vgui.dll / vgui_support.dll
│   ├── SDL2.dll
│   ├── avcodec-62.dll / avformat-62.dll / avutil-60.dll
│   ├── swresample-6.dll / swscale-9.dll
│   ├── mdldec.exe              (utilidad de decompilación de modelos)
│   ├── *.pdb                   símbolos de depuración del motor
│   ├── valve/                  <-- COPIAR AQUÍ assets de Half-Life
│   │   └── extras.pk3
│   └── gearbox/                <-- COPIAR AQUÍ assets de Opposing Force
│       ├── dlls/opfor.dll      NUESTRO build (cross-compilado)
│       └── cl_dlls/client.dll  NUESTRO build (cross-compilado)
├── downloads/                  .7z descargado + extracción
├── mingw-server-snprintf.patch parche de §12.6.1
└── hlsdk-portable/build-win32/ build tree de CMake (no tocar)
```

### 12.8 Qué tienes que copiar tú

Igual que en §6: las carpetas **`valve/`** y **`gearbox/`** de tu copia de Steam.

```
<Steam>\steamapps\common\Half-Life\valve
<Steam>\steamapps\common\Half-Life\gearbox
```

→ a `\\wsl.localhost\Ubuntu\home\<usuario>\opposing-force-x\game-win32\`

#### ⚠️ AVISO IMPORTANTE — colisión de DLLs que en Linux no existía

Tu copia de Steam **contiene los gamedlls originales de Valve**:

```
gearbox/dlls/opfor.dll         1.519.685 bytes   18-mar-2002
gearbox/cl_dlls/client.dll       614.472 bytes   14-dic-2001
```

En el build Linux no había problema: los nuestros eran `.so` y los de Valve `.dll`,
convivían. **En win32 los nombres son idénticos y se pisan.** Si copias los assets
encima sin cuidado, acabarás ejecutando el gamedll retail de Valve de 2002 en vez del
tuyo — el juego funcionaría, pero **no estarías probando tu código**, que es justo lo
que necesitamos validar para el port.

El `liblist.gam` de la copia retail declara la gamedll de servidor con ese mismo
nombre de fichero y activa la dll de cliente, así que no hay forma de renombrar para
evitar la colisión. (No se reproduce aquí el contenido del fichero: viene con tu copia
del juego.)

**Forma segura (recomendada), desde WSL:**

```bash
STEAM="/mnt/c/Program Files (x86)/Steam/steamapps/common/Half-Life"
G=~/opposing-force-x/game-win32

# guardar los DLLs retail de Valve por si quieres comparar A/B
mkdir -p ~/opposing-force-x/valve-retail-dlls
cp "$STEAM/gearbox/dlls/opfor.dll"     ~/opposing-force-x/valve-retail-dlls/
cp "$STEAM/gearbox/cl_dlls/client.dll" ~/opposing-force-x/valve-retail-dlls/

# -n = no sobrescribir: nuestros DLLs sobreviven a la copia
cp -rn "$STEAM/valve/."   $G/valve/
cp -rn "$STEAM/gearbox/." $G/gearbox/

# y por si acaso, reinstalar los nuestros encima
cd ~/opposing-force-x/hlsdk-portable && cmake --install build-win32
```

**Si copias desde el Explorador de Windows:** elige *Combinar*, y cuando pregunte por
`opfor.dll` y `client.dll` responde **"Omitir estos archivos"** (conservar los del
destino). Luego ejecuta el `cmake --install` de arriba para asegurarte.

**Cómo verificar que estás usando el tuyo y no el de Valve:**

```bash
ls -l ~/opposing-force-x/game-win32/gearbox/dlls/opfor.dll
```

El discriminador fiable es **la fecha**:

| | `opfor.dll` | `client.dll` |
|---|---|---|
| **Nuestro build** | 2.870.688 B — fecha de hoy | 1.081.596 B — fecha de hoy |
| **Retail de Valve** | 1.519.685 B — 18-mar-2002 | 614.472 B — 14-dic-2001 |

(Los nuestros son mayores porque `CMAKE_BUILD_TYPE=Release` solo aplica `strip` en
plataformas no-Windows, así que conservan los símbolos de depuración. Para el port
eso es una ventaja.)

### 12.9 Cómo lanzarlo desde Windows

**Recomendado: copia el árbol a disco local de Windows.** Ejecutar desde
`\\wsl.localhost\...` funciona, pero todo el I/O pasa por el share 9P de WSL y va
notablemente más lento (y el motor escribe configs y saves constantemente).

```powershell
# desde PowerShell
Copy-Item -Recurse "\\wsl.localhost\Ubuntu\home\<usuario>\opposing-force-x\game-win32" "C:\Games\OpposingForceX"
cd C:\Games\OpposingForceX
.\xash3d.exe -game gearbox
```

Flags equivalentes a los de §7 (mismos nombres):

```powershell
.\xash3d.exe -game gearbox -console -dev 2 -windowed -width 640 -height 480
```

| Flag | Para qué |
|---|---|
| `-game gearbox` | selecciona Opposing Force (**obligatorio**) |
| `-log` | escribe `engine.log` junto al `.exe` — **muy útil**, en Windows los errores salen en un MessageBox modal |
| `-console` | consola del motor |
| `-dev 2` | verbosidad de developer |
| `-ref soft` / `-ref gl` | fuerza renderer software / OpenGL |
| `-windowed`, `-width`, `-height` | ventana y resolución |

> **Truco de depuración:** en Windows un `Sys_Error` abre un cuadro de diálogo modal
> que bloquea el proceso. Lanza siempre con `-log` y lee `engine.log`; así tienes el
> error aunque mates el proceso.

**Mando:** una vez con assets, el gamepad lo gestiona SDL2. Si no responde, revisa
`joy_enable` / `joy_index` en la consola del motor (`-console`).

### 12.10 Notas para el port a Xbox

Lo que aporta este paso al objetivo final:

- **Nuestro código de juego ya compila y enlaza contra la ABI de Windows en x86.**
  Es el primer eslabón real hacia el Xbox: el gamedll cruza de SysV/ELF a
  Win32/PE sin tocar una sola línea de C++ (solo un flag de build).
- **La convención stdcall importa.** `GiveFnptrsToDll@8` es el punto de entrada que
  el motor resuelve por nombre decorado. En el Xbox, al enlazar estáticamente, ese
  detalle desaparece — pero conviene tenerlo fichado porque es donde fallará
  cualquier intento intermedio de carga dinámica.
- **`-static-libstdc++ -static-libgcc` ya elimina la dependencia de runtime externo**,
  que es exactamente el modelo que necesita el Xbox (binario autocontenido).
- **`msvcrt.dll` y `KERNEL32.dll` son las únicas dependencias del gamedll.** La
  superficie de CRT/Win32 que hay que proporcionar en el Xbox es pequeña: es la lista
  concreta a auditar en el siguiente paso.
- **MinGW GCC 10 traga el código sin cambios**, lo que sugiere que un toolchain GCC
  apuntando a Xbox (nxdk usa clang, pero la base es la misma) no debería encontrar
  incompatibilidades de lenguaje.
- **`ref_soft.dll` existe también en win32**: sigue siendo el mejor candidato para
  arrancar sin backend gráfico propio.

### 12.10.1 El save/restore es un problema de primer orden para el port

La investigación de §12.6.5 destapó algo que hay que resolver sí o sí en Xbox:
**GoldSrc serializa punteros a función por nombre**. En Xbox el gamedll irá enlazado
estáticamente, así que **no habrá tabla de exports en la que buscar**.

La buena noticia es que **Xash3D ya tiene ese caso contemplado**. En
`common/defaults.h:102-105`:

```c
#elif defined( XASH_STATIC_LIBS )
	#define XASH_LIB LIB_STATIC
	#define XASH_INTERNAL_GAMELIBS
	#define XASH_ALLOW_SAVERESTORE_OFFSETS
```

Con `XASH_STATIC_LIBS` se activa `XASH_ALLOW_SAVERESTORE_OFFSETS`, y entonces
(`engine/platform/misc/lib_static.c:64` + `engine/common/lib_common.c:41,78`) los
nombres se sustituyen por **desplazamientos relativos a `pfnGameInit`**:

```c
// al guardar
Q_snprintf( sname, MAX_STRING, "ofs:%zu", (size_t)((byte*)function - (byte*)svgame.dllFuncs.pfnGameInit ));
// al restaurar
if( !memcmp( pName, "ofs:", 4 ))
    return (byte*)svgame.dllFuncs.pfnGameInit + Q_atoi( pName + 4 );
```

Implicaciones para el port:

- **`XASH_STATIC_LIBS` es la configuración base del target Xbox**, no un detalle
  posterior: arrastra el gamedll interno *y* el save/restore por offsets.
- Los **saves dejan de ser portables** entre builds: cualquier recompilación mueve los
  offsets e invalida las partidas guardadas. Hay que asumirlo o versionar los saves.
- Con offsets **desaparece la dependencia del formato de los nombres** descrita en
  §12.6.5, y con ella la incompatibilidad de saves entre builds… a cambio de que los
  saves dejen de sobrevivir a una recompilación.
- Merece la pena mirar `lib_static.c` entero pronto: es, de facto, el esqueleto de
  la plataforma que hay que implementar.

---

## 13. Enlazado estático del gamedll (investigación)

**Objetivo:** determinar si FWGS permite meter el gamedll *dentro* del binario del
motor (un solo ejecutable, sin `dlopen`), como necesitará el target Xbox.

### 13.0 Veredicto

| Pregunta | Respuesta |
|---|---|
| ¿Existe el mecanismo? | **Sí.** `XASH_STATIC_LIBS` + `lib_static.c` + `scripts/waifulib/xshlib.py`, con la opción `./waf configure --static-linking=<lista>` |
| ¿Está documentado? | **No.** Cero menciones en `Documentation/` |
| ¿Lo usa algún target del repo? | **No.** El único consumidor es `scripts/waifulib/psp.py`, para el port de PSP, que vive en un fork externo y está "In progress" |
| ¿Funciona tal cual viene? | **Parcialmente.** Ver §13.5: funciona para un módulo autocontenido; falla para módulos con dependencias externas |
| ¿Se puede meter el gamedll (opfor/client)? | **No sin construir piezas que no existen.** Ver §13.6 |
| ¿Se creó `game-static/`? | **No.** No se alcanzó el criterio de éxito y no se improvisó un sistema propio |

**El árbol `build/` del motor y los directorios `game/` y `game-win32/` no se han
tocado.** Todas las pruebas se hicieron con `waf -o build-sN` en directorios aparte,
ya eliminados salvo uno (§13.5).

### 13.1 El mecanismo, en detalle

Tres piezas:

**a) `scripts/waifulib/xshlib.py`** — la maquinaria de build.

```python
MAIN_BINARY = 'xash'
opt.add_option('--static-linking', action='store', dest='STATIC_LINKING', default=None)
```

Recibe una lista de **nombres de target waf** separados por comas. Para cada uno:

1. Le quita las features `cshlib`/`cxxshlib` y le pone `xshlib`, que en vez de una
   librería compartida produce un **objeto relocalizable**:
   ```
   ${LD} -r -o <target>.unstripped.o ${LD_RELOCATABLE_FLAGS} <objetos>
   ```
   (con `-melf_i386` añadido automáticamente si hay `-m32`).
2. Lee un fichero **`exports.txt`** del directorio fuente del módulo y genera un
   `link_helper.c` que se inyecta como fuente adicional:
   ```c
   extern void GetFSAPI(void);
   struct {const char *name;void *func;} lib_filesystem_stdio_exports[] = {
   { "GetFSAPI", &GetFSAPI },
   {0,0}
   };
   ```
   *(esto es salida real de la prueba de §13.5, no un ejemplo inventado)*
3. Pasa el objeto por **`objcopy -G lib_<target>_exports`**, que deja global
   únicamente esa tabla y localiza todo lo demás. Es la medida anticolisión: permite
   que varios módulos con símbolos homónimos convivan en un binario.
4. Genera **`generated_library_tables.h`** con el índice de módulos:
   ```c
   extern table_t lib_filesystem_stdio_exports[];
   struct {const char *name;void *func;} libs[] = {
   { "filesystem_stdio", &lib_filesystem_stdio_exports },
   {0,0}
   };
   ```
5. Añade todos los `.o` resultantes a las fuentes del binario `xash`.

**b) `engine/platform/misc/lib_static.c`** — el cargador en tiempo de ejecución.
Sustituye a `dlopen`/`LoadLibrary` con búsqueda lineal en tablas:

```c
#include "generated_library_tables.h"

void *COM_LoadLibrary( const char *dllname, ... ) { return Lib_Find((table_t*)libs, dllname); }
void *COM_GetProcAddress( void *hInstance, const char *name ) { return Lib_Find( hInstance, name ); }
void *COM_FunctionFromName( void *hInstance, const char *pName ) { return Lib_Find( hInstance, pName ); }
void  COM_FreeLibrary( void *hInstance ) { /* impossible */ }
```

**c) `common/defaults.h:102-105`** — la conmutación de modo:

```c
#elif defined( XASH_STATIC_LIBS )
	#define XASH_LIB LIB_STATIC
	#define XASH_INTERNAL_GAMELIBS
	#define XASH_ALLOW_SAVERESTORE_OFFSETS
```

Activar `--static-linking` arrastra las tres cosas a la vez. Además
`wscript:443` pone `LAUNCHER = False` (no se construye `game_launch`, el ejecutable
pasa a ser `xash` directamente) y `wscript:296` desactiva `enforce_pic`.

### 13.2 Cómo resuelve el motor los nombres

Respuesta a una de las preguntas planteadas. Hay **tres** resoluciones distintas:

| Qué se resuelve | Cómo | Nombre usado |
|---|---|---|
| El módulo | `COM_LoadLibrary(name)` → búsqueda lineal en `libs[]` | Con `XASH_INTERNAL_GAMELIBS`, `COM_GenerateServerLibraryPath()` devuelve literalmente **`"server"`**, y el cliente y el menú usan **`"client"`** y **`"menu"`** (`lib_common.c:133,153`). Es decir: **el nombre del target waf debe ser exactamente `server` / `client` / `menu`** |
| Los entry points | `COM_GetProcAddress(hInstance, name)` → búsqueda lineal en la tabla `exports.txt` del módulo | Solo encuentra lo que esté listado en `exports.txt` |
| Los punteros a función del save/restore | **No** pasa por la tabla. `XASH_ALLOW_SAVERESTORE_OFFSETS` hace que `COM_NameForFunction` devuelva `"ofs:%zu"` (desplazamiento respecto a `pfnGameInit`) y que `COM_FunctionFromName_SR` lo intercepte antes de buscar | `ofs:12345` |

Ese tercer punto es la pieza elegante del diseño: **no hace falta listar los miles de
handlers de entidades en `exports.txt`**, porque el save/restore deja de usar nombres.
Es la contrapartida al problema de mangling de §12.6.5, y lo que hace viable el modo
estático. Coste: los saves dejan de sobrevivir a una recompilación.

Buena noticia para el gamedll: hlsdk-portable **ya** nombra sus targets
`name = 'server'` (`dlls/wscript:68`) y `name = 'client'` (`cl_dll/wscript:135`),
que es exactamente lo que el motor busca. La convención encaja.

### 13.3 Qué targets lo usan de verdad

Solo existen **tres** ficheros `exports.txt` en todo el repositorio, y los tres son de
módulos propios del motor:

| Fichero | Símbolos |
|---|---|
| `filesystem/exports.txt` | `GetFSAPI` |
| `ref/gl/exports.txt` | `GetRefAPI` |
| `3rdparty/mainui/exports.txt` | `GetMenuAPI`, `GetExtAPI` |

**No hay ningún `exports.txt` para un gamedll**, ni en el motor ni en hlsdk-portable.

`--static-linking` no aparece en ningún workflow de CI, script de plataforma ni
documento. La única referencia es `scripts/waifulib/psp.py:42`, del port de PSP
(estado "In progress", repositorio externo `Crow-bar/xash3d-fwgs`).

### 13.4 Lo que dice upstream

`Documentation/development/engine-porting-guide.md:17`, literal:

> *"The one of unsupported configurations at this time is when platform can't load
> dynamic libraries (`*.so` or DLLs). We can't help you as supporting full-static
> ports are violating the GPL license in various ways."*

Es decir: **no es solo una laguna técnica, es una postura declarada del proyecto**, con
argumento de licencia (enlazar lógica de juego propietaria dentro de un binario GPL).
Conviene tenerlo presente antes de plantear un PR upstream: el mecanismo existe pero
su uso para gamedlls no va a recibir apoyo.

### 13.5 Pruebas realizadas

Todas con Linux i386, `waf -o build-sN` (directorios separados, `build/` intacto).

| # | Configuración | Resultado |
|---|---|---|
| 1 | `--static-linking=filesystem_stdio,menu,ref_gl` | ❌ falla al enlazar `xash`: 148 errores `__x86.get_pc_thunk.*` + `undefined reference` (FreeType, `gEngfuncs`…) |
| 2 | `--static-linking=filesystem_stdio` (caso mínimo) | ❌ falla: 148 × `get_pc_thunk`, **0** referencias indefinidas |
| 3 | `--static-linking=filesystem_stdio` **con `-fno-pie`** | ✅ **compila y enlaza**. `xash`: ELF 32-bit i386, 13.536.372 B |
| 4 | `--static-linking=filesystem_stdio,menu,ref_gl` con `-fno-pie` | ❌ 529 referencias indefinidas (ya sin `get_pc_thunk`) |

Se conserva `xash3d-fwgs/build-s3/` como evidencia del caso que sí enlazó.

#### 13.5.1 Fallo 1 — PIE por defecto (resuelto, `-fno-pie`)

```
`__x86.get_pc_thunk.bx' referenced in section `.text' of filesystem/filesystem_stdio.o:
    defined in discarded section `.text.__x86.get_pc_thunk.bx[__x86.get_pc_thunk.bx]'
```

Los *thunks* PC-relativos del código PIC de 32 bits viven en grupos **COMDAT**.
`objcopy -G` localiza el símbolo que da nombre al grupo, el enlazador final descarta el
grupo, y las referencias quedan colgando.

El `wscript:296` ya intenta cubrirlo:

```python
enforce_pic = conf.env.DEST_OS != 'psvita' and not conf.env.STATIC_LINKING
```

pero eso solo evita **añadir** `-fPIC`; no anula el `--enable-default-pie` con el que
Ubuntu construye su GCC. Por eso los objetos siguen saliendo PIC.

**Arreglo verificado:**

```bash
CFLAGS="-fno-pie" CXXFLAGS="-fno-pie" LINKFLAGS="-no-pie" \
  ./waf configure -T release -o build-static --static-linking=filesystem_stdio
```

Es un bug real de upstream y el arreglo natural sería que el wscript añadiese
`-fno-pie`/`-no-pie` cuando `STATIC_LINKING` está activo. **Una línea.**

#### 13.5.2 Fallo 2 — las dependencias de enlace no se propagan (sin resolver)

Con `-fno-pie`, añadir `menu` y `ref_gl` produce 529 referencias indefinidas de tres
familias:

| Familia | Ejemplos | Origen |
|---|---|---|
| FreeType | `FT_Init_FreeType`, `FT_Load_Glyph`… | `menu` usa el uselib `FT2` |
| `ref/common` | `Matrix4x4_Concat`, `GL_ResampleTexture`, `CL_RunLightStyles` | `ref_gl` enlaza contra la estática `ref/common` |
| Propias | `GetRefAPI` | el `link_helper.c` de `ref_gl` la referencia |

**Causa:** `ld -r` produce un objeto relocalizable, **no consume librerías**. Todo lo
que el módulo obtenía de su `use = [...]` (uselibs y librerías estáticas) se queda sin
resolver, y el enlace final de `xash` no hereda esas dependencias porque pertenecen a
otro task generator.

Es una limitación estructural de `xshlib.py`: **el mecanismo solo funciona para
módulos autocontenidos.** Por eso `filesystem_stdio` (que no depende de nada externo)
sí enlaza y los otros dos no.

> Matiz relevante para el objetivo: los gamedlls de hlsdk **son razonablemente
> autocontenidos** (no usan uselibs externos; solo libm/libstdc++, que ya entran en
> `xash`). Es decir, este fallo concreto probablemente **no** afectaría a
> `server`/`client`. Pero no se ha podido comprobar por el bloqueo de §13.6.

### 13.6 Qué falta para meter el gamedll — estimación

El bloqueo no es de flags, es **estructural**: `xshlib.py` solo puede convertir
targets que sean *task generators de la misma ejecución de waf*. hlsdk-portable es un
proyecto waf **separado**, así que sus targets `server` y `client` no existen dentro
del build del motor.

Ahora bien, el motor tiene dos huecos de extensión preparados, en `wscript:107,133`:

```python
Subproject('stub/server'),
Subproject('stub/client', lambda x: x.env.CLIENT),
```

**Los directorios `stub/server/` y `stub/client/` no existen en el repositorio.** La
clase `Subproject.is_exists()` comprueba si hay un `<name>/wscript` y, si no, los salta
en silencio. Son ranuras vacías, casi con seguridad pensadas exactamente para esto.

Trabajo necesario, por orden de dificultad:

| # | Tarea | Tamaño | Notas |
|---|---|---|---|
| 1 | Añadir `-fno-pie`/`-no-pie` cuando `STATIC_LINKING` | **1 línea** | Ya verificado (§13.5.1) |
| 2 | `stub/server/exports.txt` | **6 líneas** | Símbolos exactos que pide el motor: `GiveFnptrsToDll`, `GetEntityAPI`, `GetEntityAPI2`, `GetNewDLLFunctions`, `Server_GetPhysicsInterface`, `SV_SaveGameComment` |
| 3 | `stub/client/exports.txt` | **~69 líneas** | El cliente de opfor **no** exporta `F` ni `GetClientAPI` (los entry points únicos que el motor prueba primero en `cl_game.c:4021,4028`), así que hay que listar los nombres individuales de `cdll_exports[]` + `cdll_new_exports[]`. Se pueden generar con `nm -D --defined-only client.so` (69 símbolos hoy) |
| 4 | `stub/server/wscript` y `stub/client/wscript` | **~60-100 líneas cada uno** | Lo caro. Hay que replicar dentro del árbol del motor lo que hacen `dlls/wscript` y `cl_dll/wscript` de hlsdk: globs de fuentes, includes (`gearbox`, `common`, `engine`, `pm_shared`, `game_shared`, `public`), y los ~30 defines que salen de `mod_options.txt`. Y mantenerlo sincronizado con hlsdk |
| 5 | Verificar el save/restore por offsets | **desconocido** | `XASH_ALLOW_SAVERESTORE_OFFSETS` se activa solo, pero nunca se ha ejercitado con un gamedll real |
| 6 | Posiblemente, propagación de uselibs en `xshlib.py` | **medio** | Solo si los gamedlls resultan no ser autocontenidos (§13.5.2) |

**Estimación global:** el 1-3 es trivial (data + un flag). El **4 es el trabajo real** y
es donde se decide si esto es razonable: no es "usar el mecanismo", es reimplementar el
sistema de build de hlsdk dentro del motor, con una duplicación que habría que mantener.

Por eso me he parado aquí en vez de improvisarlo, como acordamos.

**Alternativa que quizá evita el punto 4:** en lugar de reimplementar, hacer que
`stub/server/wscript` sea un envoltorio fino que reutilice los wscripts de hlsdk vía
`bld.add_subproject()` apuntando a un checkout de hlsdk-portable. No se ha probado y
tiene sus propios problemas (dos proyectos con `mod_options.txt`, `library_naming`,
opciones de configure y `public/build.h` propios). Es lo primero que exploraría si
decides seguir.

### 13.7 Limitaciones y respuestas concretas

**¿Pueden convivir menú + client + server en un mismo binario?**
Por diseño **sí**: `libs[]` es una lista y `objcopy -G` existe precisamente para evitar
colisiones de símbolos entre módulos. En la práctica **hoy no**, porque `menu` es uno
de los módulos que falla por el problema de propagación de dependencias (§13.5.2).
Lo único demostrado funcionando es **un** módulo autocontenido.

**¿Cómo resuelve el motor los nombres?** Ver §13.2: tres mecanismos distintos
(módulo por nombre literal, entry points por tabla `exports.txt`, punteros de
save/restore por offset).

**Otras limitaciones observadas:**

- `COM_FreeLibrary` es un no-op (`// impossible`): no hay descarga ni recarga de
  gamedll en caliente.
- Las búsquedas son **lineales** sobre las tablas. Irrelevante para 6 símbolos, a
  vigilar si alguna tabla creciera.
- `LAUNCHER = False`: desaparece el ejecutable `xash3d` y el binario pasa a ser `xash`.
- El modo estático **no** implica `-static` de libc: SDL2 y la libc siguen enlazándose
  dinámicamente. `STATIC_LINKING` y `STATIC` son variables distintas. Para Xbox habrá
  que resolver eso por separado.
- Los saves dejan de ser portables entre recompilaciones (offsets).

### 13.8 Comandos usados

```bash
cd ~/opposing-force-x/xash3d-fwgs
export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig

# el que funciona (un modulo autocontenido)
CFLAGS="-fno-pie" CXXFLAGS="-fno-pie" LINKFLAGS="-no-pie" \
  ./waf configure -T release -o build-static --static-linking=filesystem_stdio
./waf build -o build-static -j$(nproc)
file build-static/engine/xash
# -> ELF 32-bit LSB executable, Intel 80386

# inspeccionar lo que genera el mecanismo
cat build-static/engine/generated_library_tables.h
cat build-static/filesystem/link_helper.c
```

> **Nota:** usa siempre `-o <dir>` para no pisar el `build/` del árbol validado de §2.
> Tras estas pruebas, el fichero de lock de waf apunta al último `-o` usado; si
> necesitas reconstruir el build dinámico de §2, vuelve a lanzar su `configure`.

---

## 14. Build híbrido con gamedlls estáticos (experimento de §13.6)

**Objetivo:** motor con `server` y `client` de Op4 enlazados DENTRO del binario
`xash`, y el resto de módulos (menu, renderers, filesystem) como `.so` dinámicos —
para esquivar el bug 2 de §13, que solo afecta a módulos con uselibs externos.

### 14.0 Estado final: **BLOQUEADO**

| Fase | Estado |
|---|---|
| Envoltorio que delega en hlsdk sin duplicar su build | ✅ **funciona** |
| `configure` con `--static-linking=server,client` | ✅ **funciona** |
| Compilación de las ~230 fuentes de Op4 dentro del build del motor | ✅ **funciona** |
| Conversión a objetos relocalizables + tablas de exports | ✅ **funciona** |
| **Enlazado final del binario `xash`** | ❌ **falla** |
| `game-static/`, `map of0a0`, save/load | ⛔ **no alcanzado** |

**No se creó `game-static/`** y no se copió ningún asset: no hay binario que probar.
`game/` y `game-win32/` **no se han tocado** (verificado: fechas 5-ago 14:57 y
6-ago 03:04, sin cambios).

La premisa del experimento resultó ser **falsa**: los gamedlls de hlsdk **sí** tienen
dependencias `use=` (§14.4), así que el bug 2 de §13 les afecta igual. Además
aparecieron dos bugs nuevos de `xshlib` que no se conocían.

### 14.1 El envoltorio (esto sí funcionó)

Vive en **`xash3d-fwgs/stub/client/wscript`**. Delega en el wscript de hlsdk; no
reimplementa nada.

```python
def configure(conf):
	path = hlsdk_path(conf)
	for key, value in HLSDK_OPTION_DEFAULTS.items():
		if not hasattr(conf.options, key):
			setattr(conf.options, key, value)
	conf.msg('Delegating game libraries to', path)
	isolated_subproject(conf, path)

def build(bld):
	isolated_subproject(bld, hlsdk_path(bld))
```

Tres detalles no obvios que hubo que resolver:

**a) Una sola ranura, no dos.** El `configure()` de hlsdk termina con
`conf.add_subproject('game_shared dlls cl_dll')`, así que **un solo envoltorio trae
los dos targets** (`server` y `client`). Rellenar también `stub/server` configuraría
hlsdk dos veces.

**b) `stub/client`, no `stub/server`.** En `SUBDIRS` del motor las ranuras están en
posiciones distintas:

```
wscript:107   Subproject('stub/server')            <- ANTES de vgui_support
wscript:130   Subproject('3rdparty/vgui_support')
wscript:133   Subproject('stub/client')            <- DESPUES  ✔
```

Los dos proyectos traen módulos `waifulib` con el mismo nombre (`vgui.py`,
`xcompile.py`, `compiler_optimizations.py`). Esos módulos usan decoradores `@conf`,
que **parchean métodos en la clase de waf de forma global**; el último importado
gana. Con el envoltorio en `stub/server`, el `vgui.py` de hlsdk se cargaba antes y el
`vgui_support` del motor acababa buscando vgui-dev en la ruta de hlsdk:

```
Configuring VGUI by provided path : yes: [...'/xash3d-fwgs/vgui_support/vgui-dev/lib']
Checking for library VGUI sanity  : no
Can't compile simple program. Check your path to vgui-dev repository.
```

Poniéndolo en `stub/client` (después de que el motor haya hecho sus checks) el
problema desaparece. **No hizo falta tocar el `wscript` del motor.**

**c) Las opciones de hlsdk no se pueden registrar.** Su `options()` declara `-8`,
`-4`, `--disable-werror`, `--enable-msvcdeps` y `--enable-wafcache`, que el motor ya
declara; argparse rechaza duplicados. Como su `configure()` solo necesita los
*valores*, el envoltorio los inyecta:

```python
HLSDK_OPTION_DEFAULTS = {
	'USE_VGUI': False, 'USE_NOVGUI_MOTD': False, 'USE_NOVGUI_SCOREBOARD': False,
	'USE_VOICEMGR': False, 'GOLDSOURCE_SUPPORT': False, 'ANDROID_APK': False,
}
```

Son las 6 opciones que usa el configure de hlsdk y el motor no define. El resto
(`ALLOW64`, `FORCE32`, `DISABLE_WERROR`, `MSVC_WINE`, `VGUI_DEV`…) coinciden por
nombre de `dest` y se reutilizan tal cual.

**d) `PYTHONPATH` en el build.** waf reimporta al arrancar el build las herramientas
cargadas en configure. `library_naming` solo existe en hlsdk, así que:

```
ModuleNotFoundError: No module named 'library_naming'
```

Se resuelve invocando el build con:

```bash
PYTHONPATH=~/opposing-force-x/hlsdk-portable/scripts/waifulib ./waf build -o build-hybrid
```

### 14.2 Los `exports.txt`

Generados a partir de lo que el motor pide **intersecado** con lo que los `.so`
validados de §3 realmente exportan (así no se listan símbolos inexistentes, que
romperían el enlazado del `link_helper.c`).

`hlsdk-portable/dlls/exports.txt` — 4 símbolos:

```
GetEntityAPI
GetEntityAPI2
GiveFnptrsToDll
Server_GetPhysicsInterface
```

El motor pide dos más, `GetNewDLLFunctions` y `SV_SaveGameComment`, que **no existen
en la rama `opfor`** (el `opfor.so` validado tampoco los exporta) — son opcionales.

`hlsdk-portable/cl_dll/exports.txt` — 42 símbolos, la intersección de los 48 de
`cdll_exports[]` + `cdll_new_exports[]` con los 43 símbolos C de `client.so`. Los 6
que faltan (`HUD_ChatInputPosition`, `HUD_ClipMoveToEntity`, `HUD_GetRenderInterface`,
`HUD_GetSoundInterface`, `IN_ClientTouchEvent`, `Voice_StartChannel`) son exports
extendidos opcionales; el build dinámico validado funciona sin ellos.

Comando de regeneración:

```bash
nm -D --defined-only game/gearbox/cl_dlls/client.so | awk '$2=="T"{print $3}' | grep -vE '^_Z' | sort -u
```

### 14.3 Bug nuevo de xshlib nº1 — orden de post (parcheado)

```
AttributeError: 'task_gen' object has no attribute 'objcopy_task'
```

`add_deps` (que inyecta los objetos relocalizables en el binario principal) asume que
los taskgens de esos módulos ya se han posteado. Con hlsdk delegado aparecen dos
objetos taskgen para el mismo nombre y `get_tgen_by_name` acaba devolviendo uno sin
postear. Arreglo aplicado en `scripts/waifulib/xshlib.py` (local, no upstreameado):

```python
for t in reloc:
	tg = self.bld.get_tgen_by_name(t)
	tg.post()          # post() es idempotente
	self.source += [tg.objcopy_task.outputs[0]]
```

Con esto el build avanza hasta el enlazado final.

### 14.4 Los tres fallos del enlazado final

#### a) `objcopy` deja el objeto SIN ningún símbolo global — bug nuevo nº2

Este es el importante, y explica dos de los tres errores.

`xshlib` genera la tabla de exports con el **nombre del taskgen**:

```python
write_export_list(self.name, ...)   # -> lib_server_exports
```

pero ejecuta objcopy con el **nombre del target**:

```python
self.generator.objcopy_task.env['NAME'] = target   # -> 'opfor'
# => objcopy -G lib_opfor_exports
```

Para los módulos del motor `name == target` (`filesystem_stdio`, `menu`, `ref_gl`) y
el bug es invisible. Para hlsdk **`name='server'` pero `target='opfor'`**, así que
objcopy conserva un símbolo que no existe y localiza todo lo demás:

```
globales en opfor.unstripped.o : 6683
globales tras objcopy          :    0
```

De ahí salen los otros dos síntomas:

```
undefined reference to `lib_server_exports'
`_ZTV11CBasePlayer' referenced in section `.text.startup' of client.o:
    defined in discarded section `.rodata._ZTV11CBasePlayer[_ZTV11CBasePlayer]'
```

(las vtables y funciones inline de C++ viven en grupos COMDAT; al quedarse locales sus
símbolos de firma, el enlazador descarta los grupos).

**Arreglo evidente a probar:** usar `self.name` en vez de `target` para el `NAME` de
objcopy. Es una línea. **No se ha probado** — ver §14.5.

#### b) `use='vcs_info'` no se propaga — bug 2 de §13, que se creía esquivado

```
dlls/game.cpp:21: undefined reference to `g_VCSInfo_Commit'
cl_dll/cdll_int.cpp:175: undefined reference to `g_VCSInfo_Branch'
```

`hlsdk-portable/dlls/wscript:72` declara `use = 'vcs_info'` y
`cl_dll/wscript:139` declara `use = libs`. `ld -r` no consume librerías, así que la
estática `libvcs_info.a` no entra en ningún sitio.

**La premisa del experimento era que los gamedlls eran autocontenidos. No lo son.**
Es exactamente el bug 2 de §13.5.2, solo que con una estática propia de hlsdk en vez
de un uselib del sistema.

### 14.5 Por qué me paré aquí

El envoltorio —que era la parte que podía resultar imposible— funciona. Lo que bloquea
es el propio mecanismo `xshlib`, con dos bugs nuevos encima del ya conocido.

Arreglar (a) es plausiblemente una línea. Pero arreglar (b) significa **implementar la
propagación de dependencias de enlace en `xshlib`**, que es precisamente el bug 2 de
§13 y la pieza que el mecanismo nunca ha tenido. Eso ya no es *usar* el mecanismo: es
completarlo. Y con el precedente de §13.4 (upstream declara los ports full-static como
configuración no soportada), es trabajo que nadie más va a mantener.

Por eso lo dejo aquí y lo evaluamos, en vez de seguir tirando del hilo.

**Siguiente paso más barato si se retoma**, en este orden:

1. Cambiar `env['NAME'] = target` por el nombre del taskgen en
   `xshlib.py:objcopy_relocatable_lib` (1 línea). Debería eliminar los errores de
   `lib_server_exports` y de secciones COMDAT descartadas.
2. Ver qué queda de (b). Lo más barato sería hacer que hlsdk no use `vcs_info` como
   librería aparte (compilar `vcs_info.c` directamente en los targets), lo que evita
   tocar `xshlib`. Es un cambio en hlsdk, no en el motor.
3. Solo si eso funciona, retomar `game-static/`, `map of0a0` y el save/load por
   offsets, que siguen sin verificarse.

### 14.6 Estado del árbol

Modificaciones locales tras el experimento:

```
xash3d-fwgs/
  M wscript                      <- parche permanente de §11.2
  M scripts/waifulib/xshlib.py   <- arreglo de §14.3 (experimental)
  ? stub/client/wscript          <- el envoltorio
  ? build-hybrid/                <- build fallido, conservado como evidencia

hlsdk-portable/
  M dlls/CMakeLists.txt          <- parche permanente de §11.1
  ? dlls/exports.txt             <- §14.2
  ? cl_dll/exports.txt           <- §14.2
```

Para revertir solo lo experimental y quedarte con los parches permanentes:

```bash
cd ~/opposing-force-x/xash3d-fwgs
git checkout -- scripts/waifulib/xshlib.py
rm -rf stub build-hybrid
cd ~/opposing-force-x/hlsdk-portable
rm -f dlls/exports.txt cl_dll/exports.txt
```

---

### 14.7 Resultado del segundo intento (arreglos de §14.5)

**Estado: BLOQUEADO en `objcopy -G` vs. grupos COMDAT de C++.**

Se aplicaron los dos arreglos apuntados en §14.5. **Los dos funcionan.** El build
avanza más que antes, pero sigue sin enlazar por un tercer motivo, distinto de ambos.

### Arreglo 1 — `NAME` de objcopy ✅

`xshlib.py`: `env['NAME'] = self.generator.name` en vez de `target` (§11.3).

| | antes | después |
|---|---|---|
| globales en `opfor.o` tras objcopy | **0** | **1** |
| ¿es `lib_server_exports`? | no existía | ✅ sí |
| globales en `client.o` tras objcopy | **0** | **1** |
| ¿es `lib_client_exports`? | no existía | ✅ sí |
| `undefined reference to lib_*_exports` | 1 | **0** |

### Arreglo 2 — `vcs_info` embebido ✅

`dlls/wscript` y `cl_dll/wscript`: se compila `game_shared/vcs_info.c` dentro de cada
target y se elimina `use='vcs_info'` (§11.4).

| | antes | después |
|---|---|---|
| `undefined reference to g_VCSInfo_*` | 2 | **0** |

Los gamedlls **ya son autocontenidos**: el bug 2 de §13 deja de aplicarles.

### Lo que queda: 32 errores, todos del mismo tipo

```
`_ZTV11CBasePlayer' referenced in section `.text.startup' of cl_dll/client.o:
    defined in discarded section `.rodata._ZTV11CBasePlayer[_ZTV11CBasePlayer]'
    of cl_dll/client.o
```

Clasificación completa del log de enlazado tras los dos arreglos:

| Tipo de error | Nº |
|---|---|
| `g_VCSInfo_*` (arreglo 2) | **0** |
| `lib_*_exports` (arreglo 1) | **0** |
| `__x86.get_pc_thunk` (parche de §11.2) | **0** |
| otros `undefined reference` | **0** |
| **secciones COMDAT descartadas** | **32** |

Y un detalle que acota bien el problema:

| Objeto | Grupos COMDAT | Errores |
|---|---|---|
| `dlls/opfor.o` (servidor) | 1557 | **0** — enlaza limpio |
| `cl_dll/client.o` (cliente) | 363 | **32** |

**El servidor entero, con 4× más grupos COMDAT, enlaza sin un solo error.** Solo falla
el cliente.

### Diagnóstico y capa

**Capa: `xshlib`** (mecanismo del motor). No es hlsdk ni waf.

`objcopy -G lib_client_exports` deja global ese único símbolo y **localiza todos los
demás**, incluidos los símbolos de firma de los grupos COMDAT donde C++ coloca vtables
(`_ZTV*`) y funciones inline (`_ZN...UseDecrementEv`). Al quedarse locales, el
enlazador descarta el grupo, pero las referencias desde `.text` y `.text.startup` del
mismo objeto siguen ahí.

Por qué solo el cliente: `cl_dll` compila además las fuentes de armas de `dlls/`
(crossbow, egon, gauss, glock, m249, displacer, sporelauncher…) para la predicción.
Eso genera constructores globales en `.text.startup` que referencian las vtables de
esas clases; en el servidor esas mismas referencias se resuelven dentro de secciones
que sí se conservan.

Arreglarlo significa cambiar **cómo `xshlib` oculta símbolos**: en lugar de
`objcopy -G <uno>`, habría que preservar los símbolos de firma de todos los grupos
COMDAT (por ejemplo generando una lista con `--keep-global-symbols`, o sustituyendo
objcopy por un version script en el `ld -r`). Eso es rediseñar la ocultación de
símbolos del mecanismo.

**Por decisión tomada, no se sigue.** §13.4 ya documenta que upstream considera los
ports full-static una configuración no soportada, y aquí ya van tres bugs distintos
del mismo mecanismo.

### Verificaciones NO alcanzadas

Sin binario enlazado, **no se pudo probar nada de lo que se quería validar**:

- ❌ `game-static/` — no creado, no se copió ningún asset
- ❌ `map of0a0` — no ejecutado
- ❌ **save/cerrar/load por offsets — SIN VEREDICTO**

El save/restore por offsets (`XASH_ALLOW_SAVERESTORE_OFFSETS`, §13.2) **sigue sin
validarse empíricamente**. Es la pregunta abierta más relevante para el port a Xbox y
no ha podido responderse en este experimento.

### Estado del árbol

```
xash3d-fwgs/
  M wscript                      <- §11.2 (permanente)
  M scripts/waifulib/xshlib.py   <- §11.3 (dos arreglos, funcionan)
  ? stub/client/wscript          <- el envoltorio de §14.1
  ? build-hybrid/                <- build fallido, conservado

hlsdk-portable/
  M dlls/CMakeLists.txt          <- §11.1 (permanente)
  M dlls/wscript                 <- §11.4
  M cl_dll/wscript               <- §11.4
  ? dlls/exports.txt             <- §14.2
  ? cl_dll/exports.txt           <- §14.2
```

`game/` y `game-win32/` **intactos** (opfor.so 5-ago 14:57, opfor.dll 6-ago 03:04).

Los cuatro parches están en la raíz y se reaplican con §11.6.

---

### 14.8 Comandos del experimento (para reproducirlo)

```bash
cd ~/opposing-force-x/xash3d-fwgs
export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig

./waf configure -T release -o build-hybrid --static-linking=server,client

PYTHONPATH=~/opposing-force-x/hlsdk-portable/scripts/waifulib \
  ./waf build -o build-hybrid -j$(nproc)
```

El `configure` termina correctamente; el `build` falla en el enlazado de `xash` con
los errores de §14.4.

---

### 14.9 Build asimétrico (solo servidor estático)

**Estado: BLOQUEADO. Dos bloqueos nuevos, ninguno relacionado con el COMDAT de §14.7.**

**El servidor estático enlaza y produce binario** — eso funcionó. Lo que falla es que
el modo estático de Xash3D es **todo o nada**: no admite mezclar módulos estáticos y
dinámicos, que era exactamente la premisa de esta aproximación.

### 14.9.1 Intento A — `--static-linking=server` (build de cliente)

**Compila y enlaza.** ✅

```
[749/749] Linking build-static-sv/engine/xash
'build' finished successfully (40.420s)
```

```
extern table_t lib_server_exports[];
struct {const char *name;void *func;} libs[] = {
{ "server", &lib_server_exports },
{0,0}
};
```

`xash`: ELF 32-bit i386, 24.418.232 bytes. Se montó `game-static/` con los assets
copiados desde `game/`, `gearbox/dlls/` **vacío** y `gearbox/cl_dlls/client.so`
presente, como estaba previsto.

**No arranca.** ❌

```
Xash3D FWGS 0.21 (4144, 46add47-dirty, master, linux-i386)
Console initialized.
FS_LoadProgs: can't load filesystem library filesystem_stdio:
Note: Issuing host shutdown due to reason "caught an error"
```

**Causa — el cargador de librerías es excluyente.** `common/defaults.h:100-110` es una
cadena `#elif`:

```c
#elif defined( XASH_STATIC_LIBS )
	#define XASH_LIB LIB_STATIC
	#define XASH_INTERNAL_GAMELIBS
	#define XASH_ALLOW_SAVERESTORE_OFFSETS
#elif XASH_WIN32
	#define XASH_LIB LIB_WIN32
#elif XASH_POSIX
	#define XASH_LIB LIB_POSIX
#endif
```

Activar `--static-linking` con **un solo** módulo pone `XASH_LIB = LIB_STATIC`, lo que
**excluye** `LIB_POSIX`. Y `engine/platform/posix/lib_posix.c` está entero bajo
`#if XASH_LIB == LIB_POSIX`, así que **no queda ni una línea de `dlopen` en el
binario**. `COM_LoadLibrary` pasa a ser la de `lib_static.c`, que solo busca en la
tabla generada.

Se confirma en `filesystem_engine.c:218`, donde el propio motor deja claro que en este
modo espera el filesystem *dentro*:

```c
#ifdef XASH_INTERNAL_GAMELIBS
#define FILESYSTEM_STDIO_DLL "filesystem_stdio"      /* nombre pelado: de la tabla */
#else
#define FILESYSTEM_STDIO_DLL "filesystem_stdio." OS_LIB_EXT
#endif
```

`filesystem_stdio.so`, `libmenu.so` y `libref_gl.so` están en el directorio y son
perfectamente válidos, pero el binario ya no sabe abrir ficheros `.so`.

**Conclusión: el build asimétrico no es posible.** O todos los módulos que el motor
carga están en la tabla estática, o ninguno.

**Capa:** motor (`common/defaults.h` + `engine/platform/`). No es `xshlib`.

### 14.9.2 Intento B — servidor dedicado con `server,filesystem_stdio`

Si el problema es que faltan módulos en la tabla, la vía más corta es un **servidor
dedicado**, que no necesita cliente, menú ni renderer: solo `server` y
`filesystem_stdio` (y este último ya demostró enlazar limpio en §13.5.3). No requiere
arreglar nada, solo completar la lista.

```bash
./waf configure -T release -o build-ded -d --static-linking=server,filesystem_stdio
```

*(nota: en modo dedicado `env.CLIENT` es falso y `Subproject('stub/client')` está
condicionado a él, así que el envoltorio hay que ponerlo en `stub/server`, que no tiene
condición. La colisión de waifulib de §14.1b no aplica porque con `-d` no se construye
`3rdparty/vgui_support`.)*

Configure OK. **El enlazado falla:** 62 errores, todos del mismo tipo.

```
dlls/gamerules.cpp:470: undefined reference to `operator new(unsigned int)'
dlls/multiplay_gamerules.cpp:1211: undefined reference to `operator delete(void*, unsigned int)'
```

| Tipo de error | Nº |
|---|---|
| COMDAT descartada | **0** |
| `undefined reference` (`operator new`/`delete`) | **62** |

**Causa:** el target dedicado se crea con `bld.program(...)` en `engine/wscript:224`
sin `features = 'cxx c'` (el target de cliente, línea 301, sí lo lleva). Se enlaza como
programa **C**, así que no arrastra `libstdc++` — y el gamedll es C++.

**Capa:** motor / waf (`engine/wscript`). Tampoco es `xshlib`.

**No se arregla**, por la condición de parada.

### 14.9.3 Veredicto del save/restore por offsets: SIN VEREDICTO

**No se pudo obtener.** Ninguno de los dos intentos produjo un binario ejecutable con
el gamedll dentro, así que no hubo `map of0a0`, ni guardado, ni carga.

Sí quedó **establecido y validado el método de verificación**, que era la otra parte
del encargo. Es un discriminador directo sobre el fichero `.sav`.

**Qué escribe el motor en cada modo** (`engine/common/lib_common.c`):

```c
/* modo estatico: XASH_ALLOW_SAVERESTORE_OFFSETS */
const char *COM_OffsetNameForFunction( void *function )
{
	Q_snprintf( sname, MAX_STRING, "ofs:%zu",
		(size_t)((byte*)function - (byte*)svgame.dllFuncs.pfnGameInit ));
	...
}

/* al restaurar, se intercepta antes de cualquier busqueda por nombre */
void *COM_FunctionFromName_SR( void *hInstance, const char *pName )
{
#ifdef XASH_ALLOW_SAVERESTORE_OFFSETS
	if( !memcmp( pName, "ofs:", 4 ))
		return (byte*)svgame.dllFuncs.pfnGameInit + Q_atoi( pName + 4 );
#endif
```

**Test:**

```bash
strings <partida>.sav | grep -c '^ofs:'                              # estatico: > 0
strings <partida>.sav | grep -cE '^[A-Za-z_][A-Za-z0-9_]*@[A-Za-z_]' # estatico: 0
```

**Referencia medida sobre el build dinámico validado** (`game/gearbox/save/quick.sav`,
75.087 bytes):

| | build dinámico (medido) | build estático (esperado) |
|---|---|---|
| tokens `Metodo@Clase` | **86** | 0 |
| tokens `ofs:` | **0** | > 0 |

Muestra real de los 86 tokens del save dinámico:

```
CallMonsterThink@CBaseMonster
CineThink@CCineMonster
DoorGoDown@CBaseDoor
DoorTouch@CBaseDoor
FindThink@CScriptedSentence
FollowerUse@CTalkMonster
ManagerUse@CMultiManager
MultiTouch@CBaseTrigger
```

Aparece `DoorTouch@CBaseDoor`, el mismo símbolo de la falsa alarma de §12.6.5 — lo que
confirma de paso que ese formato neutro es en efecto lo que el motor guarda en modo
dinámico.

**El discriminador está listo y validado contra la referencia.** En cuanto exista un
binario estático que arranque, la prueba son dos comandos.

### 14.9.4 Estado del árbol

`game-static/` **existe pero no es funcional**: contiene el binario `xash` (24 MB, con
el servidor de Op4 dentro), los cinco `.so` de módulos, y los assets con
`gearbox/dlls/` vacío y `client.so` presente. Ocupa 1,1 GB y se puede borrar sin
pérdida (`rm -rf ~/opposing-force-x/game-static`).

```
xash3d-fwgs/
  M wscript                      <- §11.2
  M scripts/waifulib/xshlib.py   <- §11.3
  ? stub/client/wscript          <- envoltorio de §14.1 (restaurado tras el intento B)
  ? build-static-sv/             <- build del intento A (enlaza)

hlsdk-portable/
  M dlls/CMakeLists.txt          <- §11.1
  M dlls/wscript                 <- §11.4
  M cl_dll/wscript               <- §11.4
  ? dlls/exports.txt             <- §14.2
  ? cl_dll/exports.txt           <- §14.2
```

`game/` y `game-win32/` **intactos** (opfor.so 5-ago 14:57, opfor.dll 6-ago 03:04).
No se añadieron parches nuevos en esta sesión: los cuatro de §11 siguen siendo todos.

### 14.9.5 Resumen de bloqueos acumulados

Para que exista un Xash3D con el gamedll dentro y que arranque, hacen falta **a la vez**:

| # | Bloqueo | Capa | Estado |
|---|---|---|---|
| 1 | PIE por defecto rompe los grupos COMDAT de los thunks i386 | motor (`wscript`) | ✅ resuelto (§11.2) |
| 2 | `objcopy -G` usaba el nombre del target en vez del taskgen | `xshlib` | ✅ resuelto (§11.3) |
| 3 | `use=` no se propaga en `ld -r` (`vcs_info`) | `xshlib` / hlsdk | ✅ esquivado (§11.4) |
| 4 | `objcopy -G` descarta grupos COMDAT de C++ (vtables del cliente) | `xshlib` | ❌ abierto (§14.7) |
| 5 | El modo estático es todo-o-nada: sin `dlopen` no hay módulos dinámicos | motor | ❌ abierto (§14.9.1) |
| 6 | El target dedicado no enlaza `libstdc++` | motor (`engine/wscript`) | ❌ abierto (§14.9.2) |

El 5 es el que cierra la puerta a cualquier variante asimétrica: obliga a que **todos**
los módulos sean estáticos, lo que reabre el 4 (cliente) y el bug 2 de §13.5.2 (menú y
renderers, con sus uselibs externos).

---

## 15. Validación del save/restore por offsets — CIERRE DEL PASO 1.5

### 15.0 Veredicto: **FUNCIONA**

El save/restore por desplazamientos (`XASH_ALLOW_SAVERESTORE_OFFSETS`) **está validado
empíricamente** sobre el build dinámico i386, con la campaña de Opposing Force.

| Comprobación | Resultado |
|---|---|
| El save contiene tokens `ofs:` | ✅ **247** |
| El save NO contiene nombres `Metodo@Clase` | ✅ **0** |
| Carga en un proceso nuevo tras cerrar el motor | ✅ correcta |
| Fallos de resolución de punteros (`Can't find proc`) | ✅ **0** |
| Fallos inversos (`Can't find address`) | ✅ **0** |
| Crashes / `Host_Error` | ✅ **0** |
| Entidades y scripts activos tras restaurar | ✅ **1075** líneas de *firing* |
| Round-trip: re-guardar tras cargar da los mismos punteros | ✅ mismos **29** offsets distintos |
| Prueba cruzada: save en formato nombres en el motor de offsets | ✅ carga sin errores |

**Esto cierra el paso 1.5.** El mecanismo que tendrá que usar el port a Xbox —donde no
habrá tabla de exports— funciona de verdad, no solo sobre el papel.

### 15.1 Qué activa realmente el camino `ofs:` (investigación)

La pregunta del encargo era si basta el define. **No basta: hay una condición de
runtime que lo pre-empta.**

`engine/platform/posix/lib_posix.c`, tal cual viene upstream:

```c
const char *COM_NameForFunction( void *hInstance, void *function )
{
	// NOTE: dladdr() is a glibc extension
	Dl_info info = {0};
	int ret = dladdr( (void*)function, &info );
	if( ret && info.dli_sname )
		return COM_GetPlatformNeutralName( info.dli_sname );   // <-- gana SIEMPRE

#ifdef XASH_ALLOW_SAVERESTORE_OFFSETS
	return COM_OffsetNameForFunction( function );              // <-- solo si dladdr falla
#else
	return NULL;
#endif
}
```

**El camino de offsets es un fallback**, no un modo. Con un gamedll dinámico `dladdr()`
siempre resuelve —hlsdk marca los handlers con `EXPORT`, que es
`__attribute__((visibility("default")))`, así que están en la tabla dinámica— y el
bloque del `#ifdef` es inalcanzable en la práctica.

Resumen de las tres condiciones:

| Elemento | Condición |
|---|---|
| Definición de la macro | Solo en `common/defaults.h:105`, bajo `XASH_STATIC_LIBS`. Se puede forzar con `-D` |
| Camino de **escritura** (`COM_OffsetNameForFunction`) | Compilado por la macro, **pero además** requiere que `dladdr()` falle |
| Camino de **lectura** (`COM_FunctionFromName_SR`) | Compilado por la macro. **Sin condición de runtime**: comprueba el prefijo `ofs:` y, si no lo es, cae al camino normal de nombres |

Esa asimetría tiene una consecuencia útil: **el lector acepta los dos formatos** (§15.5).

Para activarlo de verdad hizo falta invertir la prioridad — 3 líneas, documentadas
como parche §11.5. Es un cambio de comportamiento intencionado, no la corrección de un
bug.

### 15.2 El build de validación

```bash
cd ~/opposing-force-x/xash3d-fwgs
git apply ~/opposing-force-x/xash-saverestore-offsets-priority.patch

export PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
export CFLAGS="-DXASH_ALLOW_SAVERESTORE_OFFSETS"
export CXXFLAGS="-DXASH_ALLOW_SAVERESTORE_OFFSETS"

./waf configure -T release -o build-offsets
./waf build -o build-offsets -j$(nproc)
./waf install -o build-offsets --destdir=$HOME/opposing-force-x/game-offsets
```

Todo dinámico: motor + `filesystem_stdio.so` + `libmenu.so` + renderers, y **los dos
gamedlls `.so` copiados desde `game/`**. Assets desde `game/`, con los `SAVE/`
heredados eliminados (los saves no son portables entre builds, §12.6.5, y habrían
contaminado justo esta prueba).

> Nota: el envoltorio `stub/` de §14 se aparta durante este build, porque aquí no
> queremos que hlsdk se construya dentro del motor.

### 15.3 Cómo se provocó el guardado sin teclado

`Posix_Input` (`engine/platform/posix/con_posix.c:33`) empieza con
`if( !Host_IsDedicated( )) return NULL;`, así que **la consola por stdin solo existe
para el servidor dedicado**; pipear comandos al cliente no hace nada.

El hook que sí funciona es `sv_init.c:962`, que al arrancar el servidor ejecuta
`maps/<mapa>_load.cfg`. Se generó `gearbox/maps/of0a0_load.cfg` con 400 `wait`
(≈ 400 frames) seguidos de `save offtest`, para guardar con el jugador ya en partida:

```
echo XTEST: of0a0_load.cfg ejecutado
wait          (x400)
save offtest
echo XTEST: save emitido
```

Traza en el log:

```
execing maps/of0a0_load.cfg
XTEST: of0a0_load.cfg ejecutado
XTEST: guardando
Saving game to save/offtest.sav...
Write save/offtest.bmp
```

### 15.4 El discriminador

| Fichero | Tamaño | tokens `ofs:` | tokens `Metodo@Clase` |
|---|---|---|---|
| `game-offsets` `offtest.sav` | 159.918 B | **247** | **0** |
| `game-offsets` `of0a0.HL1` | 150.178 B | **247** | **0** |
| **referencia** `game` `quick.sav` | 75.087 B | 0 | 86 |

Muestra real de los offsets escritos:

```
ofs:4294783952
ofs:4294784368
ofs:4294800208
...
29 tokens ofs: distintos
```

**Y la carga, en un proceso nuevo:**

```
Loading game from save/offtest.sav...
Spawn Server: of0a0
Loading game from save/of0a0.HL1...
```

```
Can't find proc    : 0
Can't find address : 0
Host_Error/crash   : 0
lineas 'firing'    : 1075
```

**Round-trip.** Se cargó `offtest.sav` y, con el mismo mecanismo de `wait`, se volvió
a guardar como `offtest2.sav`. El re-save produce **los mismos 29 offsets distintos,
con los mismos valores**, y 0 nombres. Es decir: los punteros reconstruidos al
restaurar vuelven a serializarse exactamente igual, luego apuntan a las mismas
funciones. No es una carga que "no falla": es una carga verificablemente correcta.

### 15.5 Prueba cruzada: ¿el lector acepta ambos formatos?

**Sí.** Se copió `quick.sav` de `game/` (formato nombres, 86 tokens `Metodo@Clase`) al
árbol de `game-offsets/` y se cargó con el motor parcheado:

```
Loading game from save/crossnames.sav...
Spawn Server: ofboot0
Loading game from save/ofboot0.HL1...
Can't find proc    : 0
Can't find address : 0
```

Coincide con lo que dice el código (§15.1): `COM_FunctionFromName_SR` solo intercepta
el prefijo `ofs:`; cualquier otra cosa cae al camino de nombres de siempre.

**Relevante para migraciones:** un motor con offsets activados puede leer partidas
antiguas guardadas con nombres. La dirección contraria **no** funciona: un motor sin
la macro no entiende `ofs:` y trataría el token como un nombre de función.

### 15.6 Hallazgo colateral: los offsets son negativos y eso importa en 64-bit

Los valores rondan los 4.294.xxx.xxx, sospechosamente cerca de 2³². No son
desplazamientos gigantes: son **negativos envueltos**.

```c
/* escritura: ptrdiff negativo -> size_t -> se imprime como unsigned */
Q_snprintf( sname, MAX_STRING, "ofs:%zu",
	(size_t)((byte*)function - (byte*)svgame.dllFuncs.pfnGameInit ));

/* lectura: acumula en un int con signo */
return (byte*)svgame.dllFuncs.pfnGameInit + Q_atoi( pName + 4 );
```

Comprobado numéricamente:

| token | como `int32` | significado |
|---|---|---|
| `ofs:4294783952` | −183.344 | función 183 KB **antes** de `pfnGameInit` |
| `ofs:4294784368` | −182.928 | ídem |
| `ofs:4294924144` | −43.152 | ídem |

Funciona **por simetría de complemento a dos en 32 bits**: el escritor imprime el
patrón de bits como unsigned y `Q_atoi` lo vuelve a acumular en un `int` de 32 bits,
recuperando el negativo. Es correcto, pero frágil por accidente, no por diseño.

**Implicación:** en 64 bits `size_t` son 8 bytes, así que el escritor imprimiría un
número de hasta 20 dígitos que `Q_atoi` (int de 32 bits) **no** puede reconstruir.
El mecanismo, tal cual, solo es correcto en plataformas de 32 bits.

Para el objetivo del proyecto esto es **irrelevante y hasta favorable**: el Xbox es
x86 de 32 bits. Pero conviene tenerlo fichado por si alguna vez se prueba en amd64.

### 15.7 Límite de la verificación

**No se jugó a mano.** No puedo pilotar el juego interactivamente (puertas, NPCs,
scripts) desde esta sesión. Lo que sí se verificó, de forma automática y objetiva:

- la carga completa sin un solo fallo de resolución de punteros,
- 1075 activaciones de entidades/triggers tras restaurar (el mundo sigue vivo),
- ningún crash en ~60 s de ejecución post-carga,
- el round-trip de punteros de §15.4, que es la prueba más fuerte de que el estado
  se reconstruyó bien.

Queda pendiente, si lo quieres cerrar del todo, una sesión de juego manual en
`game-offsets/` de unos minutos tras cargar. Los indicios automáticos son buenos, pero
no sustituyen a jugar.

```bash
cd ~/opposing-force-x/game-offsets && ./xash3d -game gearbox +load offtest
```

> Antes de jugar, borra `gearbox/maps/of0a0_load.cfg`: es el cfg de automatización de
> §15.3 y volvería a guardar encima.

### 15.8 Qué significa para el port a Xbox

- **La duda de fondo del paso 1.5 queda resuelta.** El save/restore sin tabla de
  símbolos funciona con el juego real, no solo en teoría.
- El mecanismo **existe y está probado**, así que el port no tiene que inventar su
  propio sistema de serialización de punteros a función.
- Es **32-bit-only** por lo de §15.6 — que coincide con el target.
- Sigue en pie la contrapartida de §13.2: **los saves no sobreviven a una
  recompilación**, porque los offsets se mueven. Habrá que versionar los saves o
  asumirlo.
- El camino de offsets **no se activa solo** al enlazar estáticamente en POSIX: en
  `lib_static.c` sí es la única opción (no hay `dladdr`), pero cualquier build que
  conserve carga dinámica necesita el parche de §11.5 para forzarlo.

### 15.9 Estado del árbol

```
~/opposing-force-x/
├── game/           1,1 GB  build dinámico validado (§3)      — INTACTO
├── game-win32/     143 MB  build win32 (§12)                 — INTACTO
├── game-static/    1,1 GB  intento estático (§14.9), no funcional
└── game-offsets/   1,1 GB  build de validación de §15        — FUNCIONA
```

Parches locales (cinco, todos en §11):

```
  M engine/platform/posix/lib_posix.c   <- §11.5  (nuevo en esta sesión)
  M scripts/waifulib/xshlib.py          <- §11.3
  M wscript                             <- §11.2
  ? stub/client/wscript                 <- envoltorio de §14.1
```

`game/` verificado intacto: `opfor.so` 5-ago 14:57, `quick.sav` 15-dic-2025 11:46.

---

## 16. Paso 2 — Mapa de la decisión de toolchain (investigación)

Sesión de solo investigación. No se ha compilado nada para Xbox.

### 16.0 Recomendación, sin rodeos

**Vía nxdk, partiendo de [`maximqaxd/xash3d-fwgs_xbox`](https://github.com/maximqaxd/xash3d-fwgs_xbox).**

Durante la investigación apareció un tercer camino que no estaba en el enunciado y que
domina a los otros dos: **ya existe un port de Xash3D FWGS al Xbox original sobre
nxdk**, en curso, sobre exactamente el mismo código y el mismo sistema de build que
llevamos usando desde §2. Último commit: 3-mayo-2026.

Lo defiendo en §16.5. Antes, los datos.

### 16.1 Half-LifeX — disección

Clonado en `~/opposing-force-x/research/Half-LifeX` (18 MB, HEAD `9cf45b2`, 20-mar-2026).

#### Toolchain exigido

| | |
|---|---|
| IDE | **Visual Studio .NET 2003** (`.vcproj` `Version="7.10"`, `.sln` Format 8.00) |
| SDK | **Microsoft Xbox XDK oficial** (`Keyword="XboxProj"`, `Platform Name="Xbox"`) |
| Build system | Proyectos `.vcproj` de VS — sin make, sin CMake, sin waf |
| Librerías | `xapilib.lib d3d8i.lib d3dx8.lib xgraphics.lib dsound.lib dmusici.lib xactengi.lib xsndtrk.lib xvoice.lib xonlines.lib xboxkrnl.lib xbdm.lib xperf.lib` |
| Defines | `_XBOX`, `_HARDLINKED`, `_USEFAKEGL01` |

El XDK de Microsoft **no es distribuible legalmente**. Existen alternativas
([`MrMilenko/OXDK`](https://github.com/MrMilenko/OXDK), `Team-Resurgent/RXDK`) para
cross-compilar proyectos XDK con clang, pero añaden su propia capa de incertidumbre.

#### Qué versión de Xash3D lleva dentro

**`XASH_VERSION "0.99"`** (`Engine/common/common.h:138`). Es el linaje **original de
Unkle Mike**, no el FWGS moderno. La divergencia es arquitectónica, no cosmética:

| | Half-LifeX (Xash 0.99) | FWGS actual (el nuestro) |
|---|---|---|
| Renderer | **dentro del motor** (`Engine/client/gl_*.c`) | módulo aparte `ref/gl` con `ref_api` |
| Filesystem | dentro del motor | módulo `filesystem_stdio` |
| Menú | `.lib` aparte (MenuUI propio) | submódulo `mainui_cpp` |
| Capa de plataforma | `sys_xbox.c` suelto en `common/` | `engine/platform/{posix,win32,sdl2,…}` |
| Total motor | 139.732 líneas | — |

#### Inventario de su capa Xbox

| Pieza | Cómo lo resuelve | Tamaño |
|---|---|---|
| **Renderer** | **FakeGLx** — subset de OpenGL sobre D3D8 fixed-function. Dos variantes (`fakeglx01.cpp` C++ / `fakeglx09.c` C). Derivado del FakeGL de Jack Palevich (2000). ~65 entradas GL distintas. D3D8 usado: `SetRenderState` (39 sitios), `SetTextureStageState` (14), `LockRect`, `SetTransform`, `SetTexture`, `SetVertexShader`, `DrawPrimitiveUP`, `CreateTexture`, `Present`, `Clear`, `Begin/EndScene` | **11.135 líneas** (`Engine/xbox/`) |
| **Sistema** | `Engine/common/sys_xbox.c` — reloj, clipboard, directorios, línea de comandos, crash handler, `Sys_LoadLibrary`/`GetProcAddress` (stubs) | 707 líneas |
| **Memoria** | Truco de MTRR para Xbox de 128 MB *"taken from XBMC"* (`host.c:648`). **Solo corre en Xbox modeados a 128 MB**; el ReadMe admite que la versión de 64 MB necesitó desactivar texturas de modelos | — |
| **Audio** | DirectSound (`Engine/client/s_backend.c`, `IDirectSoundBuffer8`), cargado por tabla de procs | — |
| **Input** | XDK puro: `Client/client/xbox/xbinput.cpp`, `XInputGetState`, `XBGAMEPAD g_Gamepads[4]` (código de ejemplo del XDK) | — |
| **Filesystem** | Rutas `D:\` **hardcodeadas** en `filesystem.c` (6 sitios) | — |
| **Carga del gamedll** | `_HARDLINKED`: `COM_LoadLibrary` devuelve un `&FakeDLL` stub; el juego se enlaza como `hl.lib hlcl.lib menuui.lib` | — |
| **Save/restore** | ⚠️ **Portaron el sistema datamap de Source al SDK del juego.** `COM_FunctionFromName`/`NameForFunction` están desactivadas en el motor (*"Not used, done in the dll for HLx"*); en su lugar `UTIL_FunctionFromName( datamap_t*, name )` recorre una tabla declarada a mano con `BEGIN_DATADESC`/`DEFINE_FUNCTION`. **252 declaraciones** repartidas por el código de entidades | `Server/dlls/xbox/datamap.h` + 252 sitios |

Ese último punto es el coste oculto grande: **es trabajo en el código del juego, por
clase de entidad**. Para Op4 habría que repetirlo en los ~40 ficheros de
`dlls/gearbox/`.

### 16.2 nxdk — estado hoy

[`XboxDev/nxdk`](https://github.com/XboxDev/nxdk): 572 estrellas, 93 forks, 115 issues
abiertos, creado 2015, **último push 5-jun-2026**. Vivo y mantenido.

| Pieza | Qué ofrece nxdk | Estado |
|---|---|---|
| **Toolchain** | clang + lld, `nxdk-cc`, cross-compila desde Linux/macOS/Windows con `make`. **Gratuito y legal** | ✅ activo |
| **CRT / libc** | [`nxdk-pdclib`](https://github.com/XboxDev/nxdk-pdclib) (PDCLib) + `libwinapi` con una capa Win32 parcial. Incluye **`LoadLibraryA`** | ✅ mantenido (jun-2025) |
| **C++** | [`nxdk-libcxx`](https://github.com/XboxDev/nxdk-libcxx) | ⚠️ último push 2022 |
| **SDL2** | [`nxdk-sdl`](https://github.com/XboxDev/nxdk-sdl), push jul-2025; nxdk lo empaqueta. Commit de abr-2026 en nxdk: *"sdl: Add support for SVG image loading"* | ✅ vivo |
| **OpenGL** | [`fgsfdsfgs/pbgl`](https://github.com/fgsfdsfgs/pbgl) sobre pbkit — push may-2026 | ✅ vivo |
| **Red** | `nvnetdrv` + lwip integrados (`libnxdk_net`) | ✅ activo |
| **USB / mandos** | `libusbohci`, commits may-2026 (*"usb: Use DPC for IRQ handling"*) | ✅ activo |

**pbgl cubre lo que FakeGLx cubre, y por el camino correcto.** Implementa *"un subset
de GL1.2 con algunas extensiones posteriores"*:

- ✅ 4 unidades de textura (`GL_ARB_multitexture`), `texture_env_combine`/`add`
- ✅ **texturas paletizadas con paleta compartida** — justo lo que GoldSrc necesita
- ❌ texturas 1D/3D/cubemap, NPOT, casi todas las conversiones de formato
- ❌ lighting/materiales incompletos
- ⚠️ formatos fijos RGBA8888 / D24S8, máximo 4096×4096

Que `fgsfdsfgs` sea el autor no es casual: es el mantenedor de los ports de **PSVita y
Switch** de Xash3D FWGS (`Documentation/ports.md`). pbgl está escrito por alguien que
sabe exactamente qué le pide Xash a GL.

#### ¿Alguien ha intentado Xash3D sobre nxdk?

**Sí.** Búsqueda en la API de GitHub (`q=xash xbox`): exactamente 2 repositorios, ambos
de `maximqaxd`:

| Repo | Creado | Último push |
|---|---|---|
| [`maximqaxd/xash3d-fwgs_xbox`](https://github.com/maximqaxd/xash3d-fwgs_xbox) | 13-abr-2026 | **3-may-2026** |
| [`maximqaxd/xash3d-fwgs-og_xbox`](https://github.com/maximqaxd/xash3d-fwgs-og_xbox) | 5-jun-2025 | 5-jun-2025 (parece un intento previo abandonado) |

`maximqaxd` es el mantenedor del port de **Dreamcast** que figura en el `ports.md` de
FWGS. En los issues de nxdk no hay nada sobre Xash (solo 2 PRs no relacionados).

### 16.3 El hallazgo: el fork FWGS-Xbox

Clonado en `~/opposing-force-x/research/xash3d-fwgs_xbox`. **Es un fork del FWGS
moderno** — mismo árbol (`engine/`, `ref/`, `filesystem/`, `wscript`, waf) con 6
commits Xbox encima de un upstream reciente.

Delta total contra upstream: **64 ficheros, 1.991 inserciones**. Reparto:

```
filesystem/filesystem.c    +262   FAT-X, montaje de unidades, lanzar desde HDD
engine/platform/xbox/      +666   vid_xbox.c (413), sys_xbox.c (150), net_xbox.h (103)
scripts/waifulib/xcompile.py +144 toolchain nxdk integrado en waf
ref/gl/*                   ~200   condicionales XASH_XBOX en el ref_gl REAL
wscript + engine/wscript   +102   DEST_OS == 'xbox'
engine/platform/win32/lib_win.c +51  carga de librerías adaptada
engine/server/sv_save.c      +4
public/crtlib.*             +26
```

Los puntos que deciden la comparación:

**a) La carga dinámica de gamedlls FUNCIONA en Xbox.** El `wscript` dice:

```python
elif conf.env.DEST_OS == 'xbox':
    # nxdk: no libdl (LoadLibraryA is in libwinapi.lib, already linked)
```

Es decir: `XASH_LIB = LIB_WIN32`, `LoadLibraryA` de nxdk. **No hace falta enlazado
estático.** Retrospectivamente, toda la odisea de §13/§14 perseguía un requisito que
no existe por esta vía — y el save/restore por offsets de §15 pasa de obligatorio a
opcional.

**b) Usa el `ref/gl` de verdad, no un FakeGL.** Enlaza
`/opt/toolchains/xbox/pbgl/libpbgl.lib` + `libpbkit.lib`, con condicionales
`XASH_XBOX` dentro de `gl_image.c`, `gl_backend.c`, `gl_rsurf.c`… Un commit concreto:
*"ref: gl: ogxbox: use GL_COLOR_INDEX8_EXT for indexed textures"*.

**c) SDL2 para vídeo/audio/input.** `# nxdk ships SDL2 -- find it via pkg-config` y
`find_sdl(conf)`. Es el mismo camino que ya usa nuestro build de escritorio.

**d) `sys_xbox.c` es sorprendentemente pequeño** (150 líneas) y casi todo es una
consola serie por UART 16550 para depuración. La razón de que sea tan corto es que
nxdk + SDL2 cubren el resto.

**Madurez real — y aquí toca ser honesto:**

- ⚠️ **1 estrella, un solo autor, 6 commits, sin CI de Xbox.**
- ⚠️ **No lo he compilado ni ejecutado.** No sé si arranca en hardware ni en emulador.
- ⚠️ Xbox **no figura** en `Documentation/ports.md` — no está upstreameado.
- ✅ Solo 4 marcas de trabajo pendiente, todas menores y en `vid_xbox.c` (720p,
  minimización de ventana, resolución fija).
- ✅ Toca `sv_save.c`, `filesystem.c` y `lib_win.c` — señal de que llegó a problemas
  reales de ejecución, no es un esqueleto de compilación.

### 16.4 Tabla comparativa por componente

| Componente | Vía Half-LifeX | Vía nxdk (partiendo del fork) | Trabajo estimado |
|---|---|---|---|
| **Toolchain** | VS.NET 2003 + XDK de Microsoft (no distribuible). Build por `.vcproj` | clang/lld vía nxdk, `NXDK_DIR`, **integrado en el waf que ya usamos** | **HLX: alto** (conseguir y montar XDK + VS2003, o apostar por OXDK/RXDK). **nxdk: bajo** — instalar nxdk y compilar |
| **Renderer** | FakeGLx: 11.135 líneas propias, GL→D3D8 fixed-function, ~65 entradas | pbgl (GL1.2 subset sobre pbkit) + ~200 líneas de `XASH_XBOX` en el `ref/gl` real | **HLX: mantener un GL propio.** **nxdk: bajo-medio** — pbgl es externo y mantenido; el trabajo es tapar sus huecos (NPOT, conversiones de formato) |
| **Sistema** | `sys_xbox.c`, 707 líneas | `sys_xbox.c`, 150 líneas (+ nxdk hal/xboxkrnl) | **nxdk claramente menor** |
| **Memoria** | Truco MTRR de XBMC; **requiere Xbox de 128 MB**. La build de 64 MB sacrifica texturas | Sin resolver por nadie. El motor moderno es más pesado que el 0.99 | **Empate, y es el riesgo nº1 de las dos vías** |
| **Audio** | DirectSound directo | SDL2 (nxdk lo empaqueta) | **nxdk menor**: reutiliza el backend SDL del motor |
| **Input** | XDK `XInputGetState` + `XBGAMEPAD` | SDL2 joystick; `joy_sdl2.c` ya tiene un `#if !XASH_XBOX` | **nxdk menor** |
| **Filesystem** | `D:\` hardcodeado en 6 sitios | +262 líneas en `filesystem.c`: montaje de unidades, FAT-X, lanzar desde HDD | **nxdk mejor hecho** (soporta más de un origen) |
| **Enlazado del gamedll** | `_HARDLINKED`: game como `.lib` estática, `COM_LoadLibrary` devuelve stub | **`LoadLibraryA` de nxdk — carga dinámica normal** | **nxdk drásticamente menor.** Evita todo §13/§14 |
| **Save/restore** | **Datamap de Source portado al juego: 252 `BEGIN_DATADESC`/`DEFINE_FUNCTION`**, y habría que repetirlo para las entidades de Op4 | Con carga dinámica, la ruta `LIB_WIN32` por tabla de exports funciona igual que en el win32 de §12. Y si hiciera falta, el `ofs:` validado en §15 está disponible | **HLX: muy alto.** **nxdk: casi nulo** |
| **Base de motor** | Xash3D **0.99** (Unkle Mike), monolítico | **FWGS master** — el mismo que compilamos en §2 | **nxdk: todo nuestro §1-§15 se conserva** |
| **Op4** | HL1 únicamente. Añadir gearbox = trabajo nuevo en código antiguo | `hlsdk-portable` rama `opfor`, la que ya compilamos en §3 | **nxdk drásticamente menor** |

### 16.5 Recomendación razonada

**Vía nxdk, partiendo del fork de `maximqaxd`. Sin medias tintas.**

Los argumentos, por orden de peso:

1. **Continuidad total con lo hecho.** El fork es FWGS master + waf + i386. Nuestros
   §1-§15 —dependencias, build i386, hlsdk `opfor`, los cuatro parches, la validación
   del save/restore— siguen siendo válidos. La vía Half-LifeX significa tirar eso y
   empezar sobre un motor de 2011 al que habría que añadirle Op4.

2. **El bloqueo que más nos costó no existe por esta vía.** nxdk da `LoadLibraryA`.
   El gamedll de Op4 se carga dinámicamente, como en win32 (§12). Ni datamaps, ni
   enlazado estático, ni los tres bugs de `xshlib`. §15 pasa de requisito a red de
   seguridad.

3. **Legalidad y reproducibilidad del toolchain.** nxdk se instala con un `git clone`
   y compila desde nuestra WSL. El XDK de Microsoft no es distribuible; montar VS.NET
   2003 en 2026 es un proyecto en sí mismo.

4. **El renderer es de otro.** pbgl está mantenido por el autor de los ports de Vita y
   Switch de Xash. FakeGLx serían 11.000 líneas nuestras a perpetuidad.

5. **El coste de Op4 es casi cero por esta vía y alto por la otra.** Half-LifeX es
   HL1; su SDK de servidor es la 2.3 de Valve con datamaps a mano.

**Lo que Half-LifeX sí tiene y el fork no:** es un producto **terminado que funciona en
hardware real** (Beta 0.90, builds públicas). El fork es una promesa sin verificar.
Ese es el argumento fuerte del otro lado y no lo minimizo. Pero lo que aporta
Half-LifeX es sobre todo **conocimiento** —cómo se resuelve memoria en 64/128 MB, qué
recorta, cómo mapea el mando—, y ese conocimiento se puede leer y aprovechar sin
adoptar su toolchain. Recomiendo tratarlo como **documentación de referencia, no como
base de código**.

### 16.6 Riesgos, y qué NO he verificado

| Riesgo | Comentario |
|---|---|
| **El fork puede no compilar ni arrancar** | **No lo he probado.** Es lo primero que hay que hacer y es barato: instalar nxdk y lanzar el build |
| **Memoria en 64 MB** | Nadie lo ha resuelto para el motor moderno. Half-LifeX necesitó 128 MB con Xash 0.99, que es más ligero. **Es el riesgo real del proyecto**, y es independiente de la vía |
| **Bus factor 1** | Un autor, 1 estrella, sin CI. Si abandona, nos quedamos con el fork tal cual (aunque el código es nuestro a partir de ahí) |
| **Huecos de pbgl** | NPOT, conversiones de formato, lighting incompleto. Puede que Xash pida cosas que falten |
| **`nxdk-libcxx` parado desde 2022** | Relevante si algo del árbol necesita C++ (mainui_cpp es C++). Habría que ver cómo lo resuelve el fork |
| **Op4 sobre el fork** | El fork compila el motor; que el gamedll `opfor` cargue en Xbox no está demostrado por nadie |

### 16.7 Siguiente paso que propongo

Barato y decisivo, en este orden:

1. **Instalar nxdk en la WSL y compilar el fork tal cual**, sin tocar nada. Responde de
   golpe a "¿esto es real?".
2. Si compila: probar en **xemu** (emulador de Xbox) antes que en hardware.
3. Solo entonces: cross-compilar `hlsdk-portable` rama `opfor` para Xbox y ver si
   `LoadLibraryA` lo carga.
4. Y ya con eso en pie, atacar el problema de memoria, que es el que decide si esto
   llega a puerto o no.

Half-LifeX se queda clonado en `~/opposing-force-x/research/Half-LifeX` como
referencia para el punto 4 — su gestión de memoria y sus recortes son el mejor material
que hay sobre el tema.

---

## 17. Paso 2 — nxdk instalado, fork bloqueado

**Resumen: el toolchain funciona; el fork no compila porque está publicado incompleto.**

| Pieza | Estado |
|---|---|
| nxdk instalado y verificado | ✅ **funciona** — sample compilado a `.xbe` válido + ISO |
| pbgl compilado | ✅ **funciona** — `libpbgl.lib`, 157 símbolos GL |
| Fork `maximqaxd/xash3d-fwgs_xbox` | ❌ **no compila**: le faltan 2 ficheros que su propio build referencia |
| xemu | ⏸️ no instalado — decisión y requisitos en §17.4 |
| Op4 (gearbox) | ⏸️ no alcanzado |

### 17.1 nxdk — instalado y verificado

Prerrequisitos (Ubuntu 22.04, coinciden con el wiki oficial):

```bash
apt install build-essential cmake flex bison clang lld git llvm
```

Instalados: clang 14.0.0, LLD 14.0.0, bison 3.8.2, flex 2.6.4, cmake 3.22.1.
El `bin/activate` de nxdk avisa si clang < 10 o si está en el rango 19.x–20.1.2
(bug conocido de LLVM con optimizaciones). La 14 está fuera de ambos.

```bash
mkdir -p /opt/toolchains/xbox            # ruta que el fork espera hardcodeada
cd /opt/toolchains/xbox
git clone --recursive https://github.com/XboxDev/nxdk.git
```

HEAD: `29638d0` (5-jun-2026). Submódulos: SDL2, SDL2_image, SDL_ttf, lwip, pdclib,
libcxx, zlib, libjpeg-turbo, libpng, libusbohci, extract-xiso.

**Verificación con un sample:**

```bash
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"       # imprescindible: es lo que hace bin/activate
cd $NXDK_DIR/samples/hello && make
```

Resultado:

```
bin/default.xbe        110.592 bytes
magic                  XBEH            <- cabecera XBE valida
nxdk sample - hello.iso 655.360 bytes  <- ISO Xbox conteniendo /default.xbe
```

**El toolchain está sano.**

> Nota: la primera vez falló con `nxdk-cc: No such file or directory`. Definir
> `NXDK_DIR` no basta; hay que añadir `$NXDK_DIR/bin` al `PATH`.

### 17.2 pbgl — compilado

```bash
cd /opt/toolchains/xbox
git clone https://github.com/fgsfdsfgs/pbgl.git
```

HEAD: `017ab17` (20-may-2026).

pbgl **no trae Makefile propio**: se integra en el Makefile nxdk del proyecto que lo
usa (vía `config_pbgl.make`) o se compila con CMake. Pero el fork enlaza
`/opt/toolchains/xbox/pbgl/libpbgl.lib` **directamente**, así que hubo que producir la
librería suelta. Escribí este envoltorio mínimo:

```make
# /opt/toolchains/xbox/pbgl/Makefile.standalone
NXDK_DIR ?= /opt/toolchains/xbox/nxdk

include $(NXDK_DIR)/Makefile
include /opt/toolchains/xbox/pbgl/config_pbgl.make

.PHONY: pbgl
pbgl: $(PBGL_LIB)
```

```bash
make -f Makefile.standalone pbgl
```

Resultado: `libpbgl.lib`, **130.050 bytes**, 16 objetos, **157 símbolos GL**.

> Dos trampas: (a) el `include` de `config_pbgl.make` debe llevar **ruta absoluta**,
> porque tras incluir el Makefile de nxdk `$(lastword $(MAKEFILE_LIST))` ya apunta a
> otro sitio; (b) hay que pedir el target `$(PBGL_LIB)` por su nombre real
> (`.../pbgl//libpbgl.lib`, con doble barra) o vía un target `.PHONY`, porque si
> escribes la ruta "limpia" a mano, make casa con una regla genérica `%.lib` y
> produce una librería vacía sin quejarse.

### 17.3 El fork — BLOQUEADO: faltan ficheros en el repositorio

```bash
cd ~/opposing-force-x/research/xash3d-fwgs_xbox
git submodule update --init --recursive
NXDK_DIR=/opt/toolchains/xbox/nxdk ./waf configure --xbox
```

```
ModuleNotFoundError: No module named 'xbox'
  File ".../wscript", line 131, in options
    opt.load('... subproject ninja xbox')
```

**Faltan dos ficheros que el propio build referencia:**

| Fichero | Lo referencia | ¿Existe? | ¿Estuvo alguna vez? |
|---|---|---|---|
| `scripts/waifulib/xbox.py` | `wscript:131` (`opt.load`) y `wscript:237` (`conf.load`) | **NO** | **No.** `git log --all --diff-filter=A -- '*xbox.py'` no devuelve nada |
| `engine/platform/xbox/xbox_sbrk.c` | `engine/wscript:232` | **NO** | No |

Lo que sí está en `engine/platform/xbox/`: `sys_xbox.c` (3.858 B), `vid_xbox.c`
(11.355 B), `net_xbox.h` (2.758 B). Nada más.

Comprobé además que **no hay otra rama ni tag del fork con trabajo Xbox**: el listado
de refs remotas es el heredado de FWGS upstream, y solo `master` lleva los 6 commits
de Xbox.

**Diagnóstico:** el fork está publicado incompleto. No es que falle a mitad de
compilación por un bug: es que **el primer paso, el `configure`, muere antes de
empezar** porque el autor nunca commiteó el módulo waf de su propia plataforma.

**Qué sería `xbox.py`** — solo se puede inferir. No aparece llamado por nombre en
ningún sitio (`conf.xbox_*`, `bld.xbox_*`: cero coincidencias), así que su API no se
deduce del resto del árbol. Por analogía con `psvita.py` y `nswitch.py`, que sí están
en el repo y hacen exactamente eso para sus plataformas, lo más probable es que
proporcione **la tarea de empaquetado del ejecutable final** (PE → `.xbe` con `cxbe`,
y opcionalmente el ISO con `extract-xiso`). Pero es una inferencia, no un hecho.

**No lo escribo yo**, por la condición de parada: reconstruir el módulo de plataforma
que falta es escribir la pieza, no compilar el fork tal cual.

### 17.4 xemu — dónde instalarlo y qué ficheros pide

**Recomendación: instálalo en Windows, no en WSL.**

Motivos: xemu necesita aceleración gráfica real (OpenGL/Vulkan) contra la GPU; bajo
WSLg irías a través de una capa de traducción que no está soportada ni probada por el
proyecto. Además el mando se pasa mucho más limpio en Windows, y ya tienes ahí el
entorno de §12 para lanzar cosas. En WSL solo tiene sentido si algún día quieres
automatizar pruebas sin ventana, que no es el caso ahora.

**Ficheros que exige xemu** (tres, y no los genera):

| Fichero | Qué es |
|---|---|
| **MCPX Boot ROM** (`mcpx_1.0.bin`) | El bootloader del sistema |
| **Flash ROM / BIOS** | Firmware del sistema. Debe ser una versión *debug* o una retail modificada |
| **Imagen de disco duro** | Almacenamiento del dashboard y los juegos |

**Cómo obtenerlos legalmente.** La documentación de xemu es explícita:

> *"The only legal way to acquire these files is to dump them from your real, physical
> Xbox."*

Es decir: **hay que volcarlos de tu propia consola física.** El proyecto declara que no
respalda la piratería y no enlaza material con copyright. **No he descargado ninguno de
estos ficheros ni voy a hacerlo**, y no debemos usar sitios de dumps.

Excepción parcial: para el **disco duro**, el propio proyecto xemu ofrece una imagen de
8 GB pre-construida y descargable, que contiene únicamente un dashboard sin firmar con
funcionalidad básica — no el dashboard oficial de Xbox. Esa sí es de origen legítimo.
También existen herramientas de terceros (XboxHDM, FATXplorer) para crear imágenes
propias, no mantenidas por xemu.

Referencia: <https://xemu.app/docs/required-files/>

### 17.5 Qué falta para intentar gearbox

En orden, y el primero es el que bloquea todo:

1. **Resolver los ficheros que faltan en el fork.** Tres caminos, de mejor a peor:
   - **Preguntar al autor.** Abrir un issue en el repo pidiendo `xbox.py` y
     `xbox_sbrk.c`. Es un fork de 1 estrella y un autor; lo más probable es que sea un
     descuido al commitear y que los tenga en local. **Es la vía barata y la que
     recomiendo.**
   - Escribir `xbox.py` nosotros tomando `psvita.py`/`nswitch.py` como plantilla, y
     `xbox_sbrk.c` como un `sbrk` mínimo sobre el heap de nxdk. Factible, pero es
     asumir el mantenimiento de la capa de plataforma de otro.
   - Buscar si el autor lo publicó en otro sitio (su port de Dreamcast, algún gist).
2. Con el fork compilando: producir el `.xbe` y montar la estructura de disco.
3. Probar en xemu con los ficheros de §17.4, que tú tienes que volcar de tu consola.
4. Solo entonces: cross-compilar `hlsdk-portable` rama `opfor` con nxdk y ver si
   `LoadLibraryA` carga `opfor.dll` en Xbox — la hipótesis central de §16.

### 17.6 Estado real del fork, sin adornos

En §16.3 escribí que el fork *"toca `sv_save.c`, `filesystem.c` y `lib_win.c`, señal de
que llegó a problemas reales de ejecución, no es un esqueleto de compilación"*. Eso
sigue siendo cierto del **código**, y también lo es que **el repositorio publicado no
compila**. Las dos cosas conviven: el autor tiene trabajo real hecho, pero lo que subió
no basta para reproducirlo.

Corrijo por tanto la valoración de §16.6, donde puse *"El fork puede no compilar ni
arrancar — no lo he probado"* como riesgo: **ya está comprobado y no compila.** Lo que
no sabemos todavía es si arrancaría una vez completado.

Esto **no invalida la decisión de §16** de ir por nxdk: el toolchain funciona, pbgl
funciona, y la arquitectura (FWGS moderno + waf + carga dinámica) sigue siendo la
correcta. Lo que cambia es que la vía no es "clonar y compilar", sino "clonar,
conseguir las dos piezas que faltan, y compilar".

### 17.7 Estado del entorno tras esta sesión

```
/opt/toolchains/xbox/
├── nxdk/     29638d0, submódulos completos, sample verificado -> .xbe valido
└── pbgl/     017ab17, libpbgl.lib compilada (130 KB, 157 simbolos GL)
                 + Makefile.standalone (nuestro, para la build suelta)

~/opposing-force-x/research/
├── Half-LifeX/          referencia de §16
└── xash3d-fwgs_xbox/    submódulos inicializados; NO configura
```

Paquetes añadidos al sistema: `clang lld llvm bison flex`.
`game/`, `game-win32/` y `game-offsets/` intactos.

---

## 18. Paso 2b — El gamedll de Op4 compilado con nxdk (sesión paralela)

**Resumen: el gamedll compila y enlaza al 100 % con nxdk. Salen `opfor.dll` (1,52 MB)
y `client.dll` (427 KB), PE32 i386 válidos, con la tabla de exports que el motor
necesita. Y en el camino aparece el problema serio del port, que no está en el juego
sino en el motor: nxdk no tiene cargador de DLLs.**

Esta sesión no dependía del fork bloqueado de §17. Objetivo: censo de problemas al
cross-compilar `hlsdk-portable` rama `opfor` con el toolchain nxdk verificado en §17.1.

| | Resultado |
|---|---|
| Fuentes que compilan, sin tocar nada | **222/231 (96 %)** |
| Fuentes que compilan con la capa de compatibilidad | **231/231 (100 %)**, a `-O0` y a `-O2` |
| Símbolos sin resolver al enlazar, antes del shim | 5 |
| Símbolos sin resolver, después | **0** |
| `./waf configure --nxdk && ./waf build` | ✅ funciona de punta a punta |
| Cargar ese `.dll` en la Xbox | ❌ **imposible hoy** — §18.6 |

### 18.1 Precedente en hlsdk: sí lo hay, y es exactamente el patrón a imitar

`hlsdk-portable` ya soporta **dos consolas con toolchain propio**, y las dos siguen la
misma estructura:

| Plataforma | Variable de entorno | Clase en `xcompile.py` | `DEST_OS` |
|---|---|---|---|
| Nintendo Switch | `DEVKITPRO` | `NintendoSwitch` | `nswitch` |
| PlayStation Vita | `VITASDK` | `PSVita` | `psvita` |

El patrón, en `scripts/waifulib/xcompile.py`:

1. Una constante `<PLAT>_ENVVARS` con la variable que apunta al SDK.
2. Una clase que valida que el SDK existe y expone `cc() cxx() ar() strip()
   cflags() linkflags() ldflags()`.
3. Una opción `--<plat>` en `options()`.
4. Una rama en `configure()` que mete esos comandos en `conf.environ` y fija
   `conf.env.DEST_OS`.
5. Si el compilador define una macro que delata la plataforma, se registra en
   `MACRO_TO_DESTOS` **antes** de la tabla estándar de waf.

Y en el `wscript` raíz, ramas por `DEST_OS` para los detalles (líneas 128-137: la Switch
quita `-Wl,--no-undefined` y enlaza sin libc estándar; la Vita añade `-fPIC` y
`--unresolved-symbols=ignore-all`).

No hay precedente de Dreamcast ni de Xbox. El CI (`.github/workflows/build.yml`) solo
cubre Linux y Windows vía CMake: **el soporte de consolas es solo-wscript y no está
probado por CI**, ni el de ellos ni el nuestro.

He seguido ese patrón al pie de la letra. Ver §18.4.

### 18.2 Qué es realmente el toolchain nxdk, y por qué eso ayuda

```
$ nxdk-cxx --version
Ubuntu clang version 14.0.0-1ubuntu1.1
Target: i386-pc-windows-msvc
```

Esto es **la mejor noticia de la sesión**. `nxdk-cc`/`nxdk-cxx` no son un cross-GCC:
son clang apuntando a `i386-pc-win32`, con `lld-link` de enlazador. Consecuencias:

- **La ABI de C++ es la de MSVC** — la misma con la que se construyeron los gamedll
  originales de GoldSrc. El *name mangling*, el `this` en `ecx`, el layout de vtables:
  todo coincide con el `opfor.dll` retail.
- El código toma las rutas `_WIN32` de hlsdk, no las de Linux. Que es lo correcto.
- La libc es **pdclib** (C99/C11 estricta) más un shim Win32 pequeño, ambos
  *freestanding*: `-ffreestanding -nostdlib -fno-builtin`.

### 18.3 El censo, sin tocar una sola línea de hlsdk

Compilación fichero a fichero (`nxdk-census/census.sh`), replicando los `ant_glob` de
`dlls/wscript` y `cl_dll/wscript`:

| | Compilan | Total | % |
|---|---|---|---|
| server (opfor) | 149 | 153 | 97 % |
| client | 73 | 78 | 93 % |
| **total** | **222** | **231** | **96 %** |

**Los nueve fallos, clasificados:**

| # | Fallo | Ficheros | Clase | Por qué |
|---|---|---|---|---|
| 1 | `unknown type name 'va_list'` | `dlls/util.cpp`, `cl_dll/hl/hl_weapons.cpp` | **trivial** | En MSVC `va_list` llega por `<vadefs.h>` arrastrado por `<windows.h>`; el `windows.h` reducido de nxdk no lo hace. pdclib sí trae `<stdarg.h>` completo |
| 2 | `DLL_PROCESS_ATTACH` / `_DETACH` no declarados | `dlls/h_export.cpp` | **trivial** | El `winnt.h` de nxdk no los define porque nxdk no tiene cargador: nadie llama nunca a `DllMain` |
| 3 | `'sys/types.h' file not found` | `external/openbsd/strlcpy.c`, `strlcat.c` (×2, server y client) | **trivial** | pdclib no implementa la jerarquía `<sys/*>` de POSIX. De ella estos ficheros solo usan `size_t` |
| 4 | `'memory.h' file not found` | `cl_dll/entity.cpp` | **trivial** | Cabecera histórica de MSVC; C89 movió su contenido a `<string.h>` |
| 5 | `POINT`, `SetCursorPos`, `GetCursorPos` no declarados | `cl_dll/in_camera.cpp` | **mediano** | API de cursor de escritorio. En Xbox no hay puntero que leer ni recentrar |

**Ningún fallo serio en la compilación.** No hay una sola asunción estructural del
código del juego que rompa contra nxdk. Los 231 ficheros son, salvo esas cinco cosas,
portables tal cual.

> Verificado también a `-O2`: 231/231. El aviso de §17.1 sobre bugs de optimización de
> LLVM aplica al rango 19.x–20.1.2; estamos en clang 14.

### 18.4 Los parches (todos triviales y medianos; ninguno serio)

Dos bloques. **Ninguno toca un fuente de hlsdk** — el árbol sigue produciendo los
builds i386 y win32 de §12/§15 sin cambios (comprobado, §18.10).

**(a) Capa de compatibilidad — `external/nxdk/`, cuatro ficheros nuevos.**
Sigue el idiom que hlsdk ya usa en `external/openbsd/` para suplir funciones de libc
ausentes. Se inyecta con `-include`, así que ningún `.cpp` del juego se modifica.

| Fichero | Cubre |
|---|---|
| `nxdk_compat.h` | fallos 1, 2 y 5 del censo |
| `sys/types.h` | fallo 3 |
| `memory.h` | fallo 4 |
| `nxdk_runtime.cpp` | los dos huecos de enlazado de §18.5 |

**(b) Integración en waf — parche de 169 líneas**, en
`~/opposing-force-x/hlsdk-nxdk-gamedll.patch`:

| Fichero | Cambio |
|---|---|
| `scripts/waifulib/xcompile.py` | `NXDK_ENVVARS`, clase `Nxdk`, opción `--nxdk`, rama en `configure()`, y `'NXDK'` en `MACRO_TO_DESTOS`. Calcado de `PSVita`/`NintendoSwitch` |
| `wscript` | Rama `DEST_OS == 'nxdk'`: patrones de librería PE, `IMPLIB_ST`, flags de `llvm-lib`, `-I`/`-include` de la capa compat, sin `-fPIC`, sin `libm`/`user32`, y sin definir `_LINUX` |
| `dlls/wscript` | Añade `nxdk_runtime.cpp` y exporta `GiveFnptrsToDll` sin decorar |
| `cl_dll/wscript` | Añade `nxdk_runtime.cpp` |

Cinco detalles que costaron una iteración cada uno, por si vuelven a aparecer:

1. **`DEST_OS` se recalcula.** Fijar `conf.env.DEST_OS = 'nxdk'` no basta: waf lo
   redetecta desde las macros del compilador, ve `_WIN32` y decide `win32`, y luego
   busca `user32`. La solución es la que ya usan Switch y Vita: registrar `'NXDK'` en
   `MACRO_TO_DESTOS` delante de la tabla estándar.
2. **`-fPIC` no existe para `i386-pc-windows-msvc`** y clang lo rechaza como error. Un
   PE se reubica por relocaciones. Mismo caso que el fork excluye para `psvita` y `xbox`.
3. **`nxdk-lib` es `llvm-lib`, no `ar`**: sintaxis MSVC. Hay que vaciar `ARFLAGS`
   (`rcs` no existe) y poner `AR_TGT_F = '/out:'`.
4. **`bin/nxdk-link` fuerza `-fixed -base:0x00010000`**, correcto para un XBE y
   equivocado para una DLL: `-fixed` borraría `.reloc` y esa base pisaría la imagen del
   motor. Hay que deshacerlo con `/fixed:no` y `/base:0x10000000`.
5. **`GiveFnptrsToDll` es `WINAPI`**, así que `dllexport` la publica decorada como
   `_GiveFnptrsToDll@8` y el motor, que la busca por el nombre pelado, no la
   encontraría. En MSVC lo arregla `dlls/hl.def`; aquí `lld-link` rechaza su bloque
   `SECTIONS`, así que se pide el alias con `/export:GiveFnptrsToDll`.

Con los parches: **231/231, 100 %**.

> Nota de honestidad sobre el censo: la clase `Nxdk` añade `-Wno-deprecated-declarations`
> porque nxdk marca `stricmp`/`strnicmp` como *deprecated* y hlsdk las usa por todas
> partes. Es silenciar un aviso, no ocultar un error. El resto de la política de avisos
> de hlsdk sí se aplica: waf comprobó los `-Werror=` soportados por clang durante el
> `configure` y el build pasó con ellos activos.

### 18.5 El enlazado: dos huecos, ambos medianos

Enlazando contra las librerías de nxdk quedaron **5 símbolos sin resolver** (los ~60
`Nt*`/`Ke*`/`Mm*` que aparecían al principio eran mi error: faltaba
`lib/xboxkrnl/libxboxkrnl.lib` en la línea de enlace, no un hueco real):

| Símbolo | Clase | Diagnóstico |
|---|---|---|
| `operator new`, `new[]`, `delete`, `delete[]` | **mediano** | nxdk trae el submódulo `libcxx` con todas las cabeceras, pero **no construye ninguna `libcxx.lib`**. Como nxdk-cxx fuerza `-fno-exceptions` y hlsdk no usa la STL en el camino caliente, esos cuatro operadores son lo único que faltaba. Se implementan sobre `malloc`/`free` |
| `atof` | **mediano** | pdclib **declara** `atof()` y `strtod()` en `<stdlib.h>` pero no las implementa: el propio fichero lo dice, `/* TODO: atof(), strtof(), strtod(), strtold() */`. El enlazador la reporta referenciada **100 veces**, desde los `KeyValue()` de las entidades: es con lo que el juego parsea cada propiedad flotante de cada entidad del `.bsp`. Sin ella no carga ni un mapa |

Ambos resueltos en `external/nxdk/nxdk_runtime.cpp`. El `atof` es un parser decimal con
signo y exponente, suficiente para lo que Hammer escribe en un `.bsp`; **es un tapón,
no la solución**: lo correcto sería implementar `strtod()` en pdclib y mandarlo aguas
arriba a nxdk.

**Resultado del enlazado:**

```
build-nxdk/dlls/opfor.dll     1.594.880 bytes   PE32 (DLL) Intel 80386
build-nxdk/cl_dll/client.dll    437.248 bytes   PE32 (DLL) Intel 80386
```

| | `opfor.dll` | `client.dll` |
|---|---|---|
| Exports | **660** | **71** |
| Imports | 60, todos por ordinal, de `xboxkrnl.exe` | ídem |
| `.reloc` | sí | sí |
| `GiveFnptrsToDll` sin decorar | sí | — |
| `GetEntityAPI` / `GetEntityAPI2` | sí | — |
| `HUD_Init` … `V_CalcRefdef` | — | sí (los 42 de `exports.txt`) |

**Los 660 exports del servidor importan más de lo que parece.** No son los cuatro de
`exports.txt`: son las funciones `Think`/`Touch`/`Use` que hlsdk marca con la macro
`EXPORT` para que el save/restore las pueda resolver por nombre. Que estén ahí, con el
mangling de MSVC, significa que **`COM_FunctionFromName` / `NameForFunction` funcionan
igual que en Windows** — y por tanto que en esta vía **no hacen falta ni los datamaps
de estilo Source que exigía HLX (§16), ni siquiera el `ofs:` por offsets validado en
§15**. §15 se confirma como red de seguridad, no como requisito.

### 18.6 EL PROBLEMA SERIO: nxdk no tiene cargador de DLLs

No lo arreglo, por la condición de parada. Pero hay que dejarlo escrito porque
**corrige la premisa central de §16**.

En §16.4 escribí, como argumento decisivo para elegir nxdk sobre HLX, que nxdk ofrecía
*"`LoadLibraryA` de nxdk — carga dinámica normal"*, y que eso *"evita todo §13/§14"*.
**Eso es falso.** El código de `nxdk/lib/winapi/libloaderapi.c` en HEAD `29638d0`:

```c
HMODULE LoadLibraryExA (LPCSTR lpLibFileName, HANDLE hFile, DWORD dwFlags)
{
    // Always fail with not having found the library
    SetLastError(ERROR_MOD_NOT_FOUND);
    return NULL;
}
```

`LoadLibraryA` la llama y devuelve `NULL` siempre. No abre el fichero. No hay ningún
cargador PE en `nxdk/lib/` — lo comprobé buscando por `IMAGE_NT_HEADERS` y `LdrLoad`.

`GetProcAddress` **sí** funciona, pero **solo con `hModule == NULL`**: resuelve nombres
contra la sección `.edataxb` del propio XBE (eso es lo que produce el
`-merge:.edata=.edataxb` de `bin/nxdk-link`). Con un handle de módulo devuelve
`ERROR_PROC_NOT_FOUND` sin mirar.

**Y el fork de §17 depende justo de eso.** Su `lib_win.c` hace, para Xbox:

```c
#if XASH_X86 && !XASH_XBOX      // <- desactiva el cargador propio de Xash
	if( hInst->custom_loader ) hInst->hInstance = MemoryLoadLibrary( ... );
	else
#endif
	{ hInst->hInstance = LoadLibraryA( hInst->fullPath ); }   // <- stub que falla
```

Es decir: el autor **apagó el cargador PE propio de Xash** y se apoyó en el
`LoadLibraryA` de nxdk, que no carga nada.

> **Inferencia, no hecho comprobado:** en su `GetLastErrorAsString` el autor escribió el
> comentario *"nxdk: LoadLibraryExA maps any read/open failure to ERROR_MOD_NOT_FOUND"*
> y el mensaje *"open/read/size failed; not only missing path"*. Eso describe un
> `LoadLibraryExA` que **sí abre y lee el fichero**, que no es el de nxdk upstream. Lo
> más probable es que esté trabajando contra un nxdk parcheado en local. Si es así, ese
> parche es **una tercera pieza que falta**, además de `xbox.py` y `xbox_sbrk.c` de
> §17.3 — y la más grande de las tres. Refuerza la recomendación de §17.5: preguntar al
> autor, y preguntarle también por esto.

### 18.7 La salida probable: Xash ya trae su propio cargador PE

`engine/platform/win32/lib_custom_win.c`, **464 líneas**, presente tanto en nuestro
`xash3d-fwgs` como en el fork. Es `MemoryLoadLibrary`, el cargador en memoria que FWGS
usa para tragarse DLLs en formato GoldSrc. Qué APIs de Win32 necesita, y qué tiene nxdk:

| API | ¿En nxdk? | Si no |
|---|---|---|
| `VirtualAlloc` | ✅ | |
| `VirtualFree` | ✅ | |
| `VirtualProtect` | ❌ | `NtProtectVirtualMemory` está en `xboxkrnl`; o un stub que devuelva TRUE |
| `HeapAlloc` | ❌ | `malloc` |
| `IsBadReadPtr` | ❌ | el fork ya lo stubea a FALSE |
| `GetProcAddress` | ✅ | |

Y la resolución de imports del gamedll cargado es más fácil de lo habitual: **importa de
un solo módulo, `xboxkrnl.exe`, y todo por ordinal** — que es exactamente lo que el
cargador de XBE de la consola ya hace al arrancar.

Otra opción, más limpia y más trabajo: enlazar el gamedll **sin ninguna librería de
nxdk**, de modo que resuelva su libc contra el XBE del motor vía `.edataxb`. Medí esa
superficie: **55 símbolos para el servidor, 52 para el cliente**, y son todos libc plana
(`malloc`, `memcpy`, `sinf`, `sprintf`, `fopen`…) más media docena de helpers de MSVC
(`__chkstk`, `__fltused`, `__purecall`). Esto resolvería de paso el riesgo de §18.9.

Ninguna de las dos está escrita. Lo que sí está medido es que **son un problema de
cientos de líneas, no del orden de la odisea de enlazado estático de §13/§14**.

### 18.8 Qué falta para el gamedll Xbox completo — estimación honesta

Lo que hay hoy: **una DLL que compila, enlaza, exporta lo correcto y nadie ha
ejecutado nunca.** Cero validación en runtime. Ni en xemu ni en consola.

| # | Falta | Tamaño | Bloquea |
|---|---|---|---|
| 1 | **Que el motor arranque en Xbox** (los ficheros que faltan de §17.3 + el nxdk parcheado de §18.6) | grande, y no es nuestro | todo |
| 2 | **Un camino para cargar la DLL**: reactivar `MemoryLoadLibrary` con 3 stubs, o implementar `LoadLibraryEx` en nxdk | mediano (§18.7) | todo |
| 3 | `atof` de verdad (`strtod` en pdclib) en vez de nuestro tapón | pequeño | precisión de las entidades |
| 4 | Decidir qué hacer con las dos instancias de libc (§18.9) | por determinar | estabilidad |
| 5 | Entrada de mando en lugar del ratón neutralizado en `in_camera.cpp` | pequeño | cámara en 3ª persona |
| 6 | Presupuesto de memoria: 64 MB en Xbox retail, y `opfor.dll` son 1,52 MB sin contar el heap del juego | por medir | jugabilidad |
| 7 | Validar en xemu (necesita los volcados de §17.4) | — | todo lo anterior |

**Lo que esta sesión sí zanja:** el gamedll de Opposing Force **no es el problema del
port**. Compila entero, enlaza limpio y con la ABI correcta. El problema está aguas
arriba, en el motor y en el cargador — donde §16 daba por resuelto que no lo estaba.

### 18.9 Riesgo abierto, no verificado: dos libc en el proceso

El mapa del enlazador confirma que pdclib entra **estáticamente dentro del gamedll**:

```
_malloc    libpdclib:malloc.obj
_free      libpdclib:malloc.obj
_strcmp    libpdclib:strcmp.obj
```

Y el XBE del motor llevará la suya. Eso son **dos heaps y dos estados de libc en el
mismo proceso**. En Windows no pasa porque ambos módulos importan el mismo CRT del
sistema. Si el motor y el juego se pasan punteros que uno reserva y el otro libera, eso
revienta.

**No lo he verificado** — habría que auditar el interfaz `enginefuncs_t`. Lo apunto
porque la opción "resolver la libc del gamedll contra el XBE" de §18.7 lo elimina de
raíz, lo que es un argumento a su favor más allá del tamaño.

### 18.10 Estado del entorno tras esta sesión

```
~/opposing-force-x/
├── hlsdk-portable/
│   ├── build-nxdk/          NUEVO — opfor.dll 1,52 MB + client.dll 427 KB
│   ├── external/nxdk/       NUEVO — capa de compatibilidad (4 ficheros)
│   ├── wscript                       M  <- §18.4b
│   ├── dlls/wscript                  M  <- §18.4b (ya lo estaba por §11.4)
│   ├── cl_dll/wscript                M  <- §18.4b (ya lo estaba por §11.4)
│   └── scripts/waifulib/xcompile.py  M  <- §18.4b
├── nxdk-census/             NUEVO — scripts del censo, logs y objetos
└── hlsdk-nxdk-gamedll.patch NUEVO — el parche de 169 líneas
```

`game/`, `game-win32/`, `game-offsets/` y el toolchain de §17 intactos. Comprobada la
no-regresión: `./waf configure -4 && ./waf build` sigue produciendo `opfor.so` y
`client.so`.

Reproducir el build de Xbox:

```bash
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"
cd ~/opposing-force-x/hlsdk-portable
./waf configure --nxdk -T release -o build-nxdk && ./waf build
```

**Aviso sobre las cifras de este apartado:** todas miden compilación y enlazado
estático. La única afirmación de runtime que contiene §18 es que **no ha habido
ninguna**.

## 19. Paso 2c — El fork Xbox compila: escribimos nosotros las piezas que faltaban

**Resumen: `./waf configure --xbox && ./waf build` terminan y producen
`build/engine/default.xbe`, 2.805.760 bytes, cabecera `XBEH` válida. Para llegar ahí
no bastaron las dos piezas que §17.3 identificó: hicieron falta 18 ficheros nuestros,
1.799 líneas. Nadie ha ejecutado ese binario.**

Contexto: el autor del fork (`maximqad/xash3d-fwgs_xbox`) no responde desde hace nueve
días. §17.5 recomendaba preguntarle por `xbox.py` y `xbox_sbrk.c`; sin respuesta, se
toma la segunda vía de esa misma lista — escribirlas nosotros.

| | Resultado |
|---|---|
| `./waf configure --xbox` | ✅ termina |
| `./waf build` | ✅ termina, 430 tareas |
| `build/engine/default.xbe` | ✅ **2.805.760 B**, magic `XBEH` |
| Ficheros nuestros dentro del fork ajeno | 18, **1.799 líneas**, 0 ficheros trackeados del fork modificados |
| ¿Arranca en la consola? | ❌ **no probado** — §19.9 y §19.10 |

### 19.0 Regla que se ha seguido con el código ajeno

**No se ha modificado ni un fichero trackeado del fork.** `git status` sale limpio salvo
por los cinco caminos nuevos. Todo lo nuestro va en ficheros nuevos, y todo lo que había
que forzar sobre el árbol ajeno se hace desde `scripts/waifulib/xbox.py` (nuestro) con
`-D`, `-I` e `-include`.

**Cada fichero nuestro lleva en su cabecera un bloque de comentario que dice, con esas
palabras, que no forma parte del fork upstream, quién lo escribió y por qué.** Ese es el
marcador; no hay otro. Ejemplo, en `scripts/waifulib/xbox.py`:

```
############################################################################
NOT PART OF THE UPSTREAM FORK.  Written for the Opposing Force X port
because maximqad/xash3d-fwgs_xbox references this module from
wscript:131 (opt.load) and wscript:237 (conf.load) but never committed it.
See BUILD.md section 19.  The API implemented here is *inferred* from those
two call sites and from the psvita.py / nswitch.py modules that ship in the
same directory; it is not the original author's code.
############################################################################
```

### 19.1 Lo que faltaba de verdad

§17.3 contó dos ficheros. Eran dos ficheros *referenciados por nombre*; el agujero real
es mayor, y tiene una causa única que §17 no vio:

> **El submódulo `3rdparty/library_suffix` apunta a FWGS upstream, y ahí vive `build.h`.**
> El fork consulta `XASH_XBOX` en unos cincuenta sitios y **nadie lo define**. El autor
> escribió las guardas (`#if XASH_WIN32 && !XASH_XBOX`, `#elif XASH_XBOX`, …) pero nunca
> movió el puntero del submódulo a un `build.h` que conociera la plataforma. Sin eso,
> todas esas guardas caen por la rama Win32 de escritorio y el motor pide `GetUserNameW`,
> `PeekMessage`, `_wstat`…

Inventario completo de lo que hubo que escribir:

| Pieza | Quién la pedía | Qué es |
|---|---|---|
| `scripts/waifulib/xbox.py` | `wscript:131`, `wscript:237` | 346 líneas. Empaquetado PE→XBE y toda la configuración de plataforma. §19.2 |
| `engine/platform/xbox/xbox_sbrk.c` | `engine/wscript:232` | 154 líneas. `sbrk` sobre el heap de nxdk. §19.3 |
| `engine/platform/xbox/nxdk-compat/` | pdclib | 13 ficheros. Cabeceras POSIX/MSVC que nxdk no trae + runtime. §19.4 |
| `public/xbox_fd.c` | `filesystem/filesystem.c` | 403 líneas. Descriptores POSIX **de verdad** sobre la API Win32 de nxdk. §19.5 |
| `engine/platform/xbox/xbox_winapi.c` | `engine/common/whereami.c` | 90 líneas. `GetModuleFileNameA`. §19.4 |

Las dos últimas filas confirman la inferencia de §18.6: **el autor trabajaba contra un
nxdk parcheado en local.** Su `filesystem.c` llama a `_stat()` y `_open()` en ramas
`#elif XASH_XBOX` escritas a mano; nxdk upstream no tiene ninguna de las dos. Ese nxdk
parcheado es la tercera pieza que falta, y sigue faltando: lo que hay aquí es una
reimplementación nuestra, no la suya.

### 19.2 `scripts/waifulib/xbox.py` — la API, deducida

No hay documentación: la API se deduce de los dos sitios que lo cargan y de los módulos
hermanos que sí están en el repo (`psvita.py`, `nswitch.py`).

**Restricción que no tienen los otros dos y que condiciona el diseño entero:**
`nswitch.py` y `psvita.py` se cargan **dentro de un `if conf.options.NSWITCH:`**
(`wscript:224` y `:227`). `xbox` no: está en la lista plana de `conf.load(...)` de
`wscript:237` y de `opt.load(...)` de `wscript:131`, así que **se carga siempre, para
todos los `DEST_OS`**. Por tanto:

- `configure()` sale inmediatamente si `conf.env.DEST_OS != 'xbox'`.
- Los métodos `@TaskGen.feature('cprogram','cxxprogram')` — que en `psvita.py` se
  registran globalmente sin miedo — aquí **tienen que comprobar `DEST_OS` y volver**,
  o le añadirían tareas XBE a la build de Linux.
- Salen también en las builds de prueba del `configure`: waf marca esos contextos con
  `bld.conf`, y sin ese filtro el `config.log` se llena de volcados de `cxbe` (pasó).

Comprobado: `./waf configure -o build-host --dedicated` sigue funcionando y `xbox.py`
no imprime ni una línea.

**Lo que hace, por bloques:**

| Bloque | Qué |
|---|---|
| `options()` | `--xbe-title`, `--xbe-mode {retail,debug}`, `--xbe-dumpinfo`, `--xbe-iso`. `--xbox` **no** está aquí: vive en `xcompile.py`, que es del autor |
| `_add_platform_flags()` | `-DXASH_XBOX=1` y `-DMAINUI_USE_STB=1`; `-Wno-error=strict-prototypes` |
| `_add_missing_nxdk_libs()` | añade `nxdk_usb.lib`, recolocando `libxboxkrnl.lib` al final |
| `_add_compat_includes()` | `-I` a `nxdk-compat/` y `-include nxdk_compat.h` |
| `configure()` | construye `cxbe` si hace falta, lo localiza, fija `XBE_TITLE`/`XBE_MODE` |
| `apply_nxdk_runtime()` | inyecta las fuentes de runtime en cada módulo enlazado |
| `apply_xbe()` | tarea `cxbe`: `xash.exe` → `default.xbe` |
| `apply_xiso()` | tarea opcional `extract-xiso` → `.iso` |

**Tres detalles que costaron una iteración cada uno:**

1. **`-DXASH_XBOX=1` sobrevive.** `build.h` hace `#undef` de `XASH_WIN32`, `XASH_NSWITCH`,
   `XASH_PSVITA` y compañía antes de detectar plataforma, **pero no de `XASH_XBOX`** —
   no lo conoce. Así que un `-D` en línea de órdenes llega intacto al preprocesador y
   arregla las cincuenta guardas sin tocar el submódulo. Si algún día FWGS añade
   `XASH_XBOX` a esa lista de `#undef`, esto deja de funcionar en silencio.

2. **`-Wno-error=...` tiene que ir en `CPPFLAGS`, no en `CFLAGS`.** `wscript:429` mete
   toda la política `-Werror=` en la uselib `werror`, y waf empalma las flags de uselib
   **dentro de `CFLAGS`**, que la orden de compilación emite *antes* de `CPPFLAGS`. Gana
   la última de la línea. Esto explica de paso por qué el
   `-Wno-error=implicit-function-declaration` que el autor puso en `Xbox.cflags()` de
   `xcompile.py` **no hace nada**: queda enterrado bajo el `-Werror=` de la uselib.

3. **`${CXBE_OUT_F}${TGT[0].abspath()}` en un `run_str` funciona** (waf fusiona tokens
   adyacentes sin espacio, es el idiom de `msvc.py`), pero `-DUMPINFO:` es opcional y un
   `${VAR}${TGT[1]}` con `TGT[1]` inexistente dejaría un `-DUMPINFO:` suelto. La tarea
   `cxbe` construye la línea a mano en `run()` por eso.

**Nombre de salida.** `default.xbe`, porque es lo único que el cargador de la consola
arranca. Si un día hubiera dos programas en la misma build (el dedicado y el cliente se
llaman los dos `xash`), el segundo cae a `<target>.xbe` con un aviso, en vez de pisar al
primero en silencio.

Verificado con `--xbe-title 'Opposing Force X' --xbe-dumpinfo`:

```
Title identified as "Opposing Force X"
Base Address        : 0x00010000
Size of Image       : 0x0061A114
Entry Point         : 0xA8D99BDB (Retail: 0x0025CC70)
Init Flags          : 0x00000005 [Mount Utility Drive] [Limit Devkit Run Time Memory to 64MB] [Setup Harddisk]
Secciones           : .text  .rdata  .data  .edataxb  .tls
```

Y con `--xbe-iso`: `default.iso`, 14.876.672 bytes. (Empaqueta el directorio entero, así
que se lleva dentro también el `xash.exe` intermedio; irrelevante para el despliegue por
FTP de §19.9, y por eso la opción viene apagada por defecto.)

### 19.3 `engine/platform/xbox/xbox_sbrk.c` — y la verdad incómoda sobre él

154 líneas. Reserva espacio de direcciones una vez con
`VirtualAlloc(NULL, size, MEM_RESERVE, PAGE_READWRITE)` y va confirmando páginas con
`MEM_COMMIT` a medida que el break sube, en trozos de 64 KB. Semántica de `sbrk` tal y
como la espera `platform/misc/kmalloc.c`: `0` devuelve el break actual, positivo crece y
devuelve el break *anterior*, negativo encoge, `(void*)-1` si falla. Arena de 64 MiB por
defecto, con caída a la mitad hasta un suelo de 8 MiB si la reserva no entra.

Reservar no cuesta memoria física, así que el tamaño por defecto puede exceder los 64 MB
de una consola retail: lo que no quepa falla en el `MEM_COMMIT`, es decir en el momento
en que hace falta de verdad, no al arrancar.

**Y ahora lo incómodo: este fichero no entra en el build.** `engine/wscript:232` lo mete
sólo dentro de `if bld.get_define('XASH_CUSTOM_SWAP')`, y `engine/wscript:108` hace
`conf.options.CUSTOM_SWAP = False` para `xbox`, sin dejar forma de reactivarlo desde la
línea de órdenes. Es decir: **el propio fork apaga el único camino que llega al fichero
que él mismo declara que le falta.**

Peor: ese camino está roto río arriba para *todas* las plataformas.
`platform/misc/kmalloc.c:58`, `platform/misc/sbrk.c:27` y `common/zone.c:25` incluyen
`"platform/swap/swap.h"`, y el directorio `engine/platform/swap/` **no existe** — la
cabecera está en `engine/platform/misc/swap.h`. `XASH_CUSTOM_SWAP` no compila en ningún
sitio.

Así que se ha escrito, y se ha verificado que compila suelto:

```bash
nxdk-cc -c -DXASH_XBOX=1 -Iengine -Icommon -Ipublic \
  -I3rdparty/library_suffix/include -Iengine/platform \
  -Iengine/platform/xbox/nxdk-compat \
  engine/platform/xbox/xbox_sbrk.c -o /tmp/xbox_sbrk.obj
# rc=0, objeto de 1.454 bytes
```

**No se ha ejecutado, y no lo ejecuta el `default.xbe` de esta sesión.** Se incluye
porque el enunciado lo pedía y porque es la pieza que el fork nombra; el día que alguien
arregle `XASH_CUSTOM_SWAP` ya estará aquí. Nótese que incluye `"platform/misc/swap.h"`,
la ruta correcta, no la rota.

### 19.4 `nxdk-compat/` — la capa que no estaba prevista

13 ficheros en `engine/platform/xbox/nxdk-compat/`. Mismo idiom que ya se usó para el
gamedll en `hlsdk-portable/external/nxdk/` (§18.4a): se inyecta con `-I` y `-include`
desde `xbox.py`, y **ningún fuente del fork se modifica**.

La libc de nxdk es **pdclib**, C99/C11 estricta, más un shim Win32 pequeño. No trae
`<sys/*>`, ni `<io.h>`, ni GDI, ni cargador de módulos. Y como el clang de nxdk apunta a
`i386-pc-windows-msvc`, define `_MSC_VER` y `_WIN32`, así que medio árbol toma sus ramas
Microsoft y pide el CRT de Microsoft, que no está.

| Fichero | Qué cubre | Relleno o de verdad |
|---|---|---|
| `sys/types.h` | `off_t` para `common/xash3d_types.h:6` | typedefs |
| `sys/stat.h` | `struct _stat` y `_stat()` sobre `GetFileAttributesExA` | **de verdad** |
| `io.h` | declara la capa de descriptores de §19.5 | declaraciones |
| `fcntl.h` | las `O_*` que `identification.c` incluye sin usar | constantes |
| `intrin0.h` | reenvía a `<intrin.h>`; opus lo pide porque clang dice ser MSVC 2017+ | 1 línea |
| `new.h` | reenvía a `<new>` de libcxx; lo pide `mainui/miniutl` | 1 línea |
| `psapi.h` | `lib_win.h` lo incluye y no usa nada de él | vacío |
| `winsock2.h` | `mainui/miniutl/winlite.h` lo incluye y no usa nada de él | vacío |
| `dbghelp.h` | ídem | vacío |
| `nxdk_compat.h` | `-include` en cada TU: `_set_abort_behavior` de opus, `GetModuleFileName` | mixto |
| `nxdk_libc.c` | `atof()` | **de verdad** (tapón) |
| `nxdk_dllcrt.c` | `_DllMainCRTStartup` | stub consciente |
| `nxdk_cxxrt.cpp` | `operator new` / `delete` / `new[]` / `delete[]` | **de verdad** |

Más `engine/platform/xbox/xbox_winapi.c`, que no es una cabecera sino código: implementa
`GetModuleFileNameA`, que nxdk no tiene en absoluto. Devuelve `D:\<nombre del xbe>`,
porque `libnxdk_automount_d` — que el `-include:_automount_d_drive` de `xcompile.py` ya
arrastra — monta `D:` sobre el directorio del XBE en ejecución; el nombre real sale del
path NT del kernel vía `nxGetCurrentXbeNtPath()`. Va en `platform/xbox/` porque
`engine/wscript:207` hace `ant_glob('platform/%s/*.c' % DEST_OS)` y lo recoge solo.

**Sobre `atof`:** exactamente el mismo agujero que §18.5 encontró en el gamedll. pdclib
*declara* `atof()`, `strtof()`, `strtod()` y `strtold()` y en su propio fuente pone
`/* TODO: ... */`. Aquí lo pedía `mainui/CFGScript.cpp`. Sigue siendo **un tapón**: lo
correcto es implementar `strtod()` en pdclib y mandarlo aguas arriba a nxdk.

**Sobre `nxdk_dllcrt.c`, con todas las letras:** `xcompile.py` enlaza cada librería
compartida con `-Wl,/entry:_DllMainCRTStartup`, símbolo del CRT de Microsoft que nxdk no
tiene. Este fichero lo define para que el enlazado termine. **Ese punto de entrada no lo
va a llamar nadie**, porque nxdk no puede cargar una DLL: su `LoadLibraryExA` es un stub
que siempre devuelve `NULL` (§18.6). Los tres `.dll` que salen del build
(`menu.dll`, `filesystem_stdio.dll`, `ref_gl.dll`) son, hoy, ficheros que la consola no
puede abrir. Están porque el build los produce, no porque sirvan.

**Sobre `MAINUI_USE_STB`:** `3rdparty/mainui` elige backend de fuentes por plataforma y,
sin caso para ésta, cae a `#elif defined(_WIN32)` y al backend GDI —
`CreateFontIndirect`, `HDC`, `TEXTMETRIC`, API Unicode. Nada de eso existe en nxdk.
`-DMAINUI_USE_STB=1` selecciona el renderizador `stb_truetype` que ya viene dentro y que
usan las demás consolas, y de paso vacía `font/WinAPIFont.cpp` por su propia guarda. Se
pone desde `xbox.py` porque **mainui es un submódulo de git** y no toca editarlo.

### 19.5 `public/xbox_fd.c` — descriptores POSIX de verdad

403 líneas, y es la pieza de la que menos orgulloso hay que estar y más falta hacía.

`filesystem/filesystem.c` tiene ramas `#elif XASH_XBOX` escritas por el autor, y **todas**
llaman al IO de bajo nivel del CRT de Microsoft: `_open()`, `_read()`, `write()`,
`lseek()`, `close()`, `dup()`, `_commit()`, y `_findfirst()`/`_findnext()`/`_findclose()`
para listar directorios. pdclib no tiene nada de eso: ofrece `<stdio.h>` y por debajo,
nada.

**Esto no se podía rellenar con stubs.** `FS_SysOpen`, `FS_SysFileExists`,
`FS_SysFolderExists`, `FS_SysFileTime` y `listdirectory()` son cómo el motor encuentra
gamedirs, paks y configs; contestar «no existe» a todo compila y luego no encuentra nada.
Así que está implementado de verdad, sobre
`CreateFileA`/`ReadFile`/`WriteFile`/`SetFilePointer`/`FindFirstFileA`, que nxdk sí trae.

Decisiones que conviene tener escritas:

- **Los descriptores son índices de una tabla de 128, nunca `HANDLE`s.** Así un descriptor
  válido es siempre un `int` pequeño no negativo y el fallo siempre `-1`, que es lo que
  comprueban las llamadas. El índice 0 se deja libre: demasiado código trata el 0 como
  «sin abrir».
- **`dup()` reabre por ruta y se posiciona donde estaba el original**, porque nxdk no
  tiene `DuplicateHandle`. `FS_OpenHandle()` lo usa justamente para que un fichero dentro
  de un pak tenga su propia posición, así que la semántica que sale es la que quiere quien
  llama — pero no es un `dup()` general: los dos descriptores no comparten posición.
- **`_commit()` devuelve éxito sin hacer nada.** nxdk no expone `FlushFileBuffers` y no hay
  buffer en espacio de usuario que empujar. `filesystem.c` trata negativo como error de
  escritura, así que devolver `-1` sería mentir en la dirección peligrosa.
- **`errno`:** pdclib sólo garantiza `EDOM`, `EILSEQ` y `ERANGE`. `ENOENT`, `EBADF`,
  `EMFILE` y `EINVAL` se definen a mano. Importa: `FS_SysOpen` sólo se calla cuando el
  error es `ENOENT`, y sondear gamedirs opcionales es el caso común.

Está en `public/` y no en `engine/platform/xbox/` por una razón de enlazado, no de gusto:
`public/wscript` hace `ant_glob('*.c')` sobre `libpublic`, que es **la única librería
estática que enlazan a la vez el motor y el módulo `filesystem_stdio`**, que es una
librería compartida aparte.

### 19.6 Cambios en el entorno (fuera del repo)

Dos, y ninguno es código del port:

**(a) `pkgconf`.** `$NXDK_DIR/bin/nxdk-pkg-config` hace `exec pkgconf ... | sed ...`, y en
este sistema sólo está `pkg-config` 0.29.2 de freedesktop, no `pkgconf`. Y como el `exec`
va dentro de una tubería, **el script devuelve 0 con salida vacía**: waf preguntó por
SDL2, le contestaron «sí» y cero flags, y el chequeo de sanidad falló con `SDL.h: file not
found` sin ninguna pista. Sin `sudo` sin contraseña, la solución fue un shim en
`$NXDK_DIR/bin/pkgconf` que reenvía a `pkg-config` — ese directorio ya va al frente del
`PATH` porque lo pone `xcompile.py`. **Lo limpio es `apt install pkgconf`**; el shim lo
dice en su propio comentario.

**(b) `libSDL2.lib`.** nxdk trae SDL2 como submódulo y lo compila sólo dentro de un
proyecto que la use (`NXDK_SDL = y`). El fork enlaza contra
`$NXDK_DIR/lib/libSDL2.lib` vía `sdl2.pc`, así que hubo que producirla suelta, con el
mismo truco del `Makefile.standalone` de pbgl de §17.2:

```make
# /opt/toolchains/xbox/nxdk/lib/sdl/SDL2/Makefile.standalone
NXDK_DIR ?= /opt/toolchains/xbox/nxdk
include $(NXDK_DIR)/Makefile
include $(NXDK_DIR)/lib/sdl/SDL2/Makefile.xbox
.PHONY: sdl2
sdl2: $(NXDK_DIR)/lib/libSDL2.lib
```

```bash
make -f Makefile.standalone sdl2 -j8
```

Resultado: `libSDL2.lib`, **1.525.470 bytes**. Con ella aparece el primer síntoma real:
el backend de joystick Xbox de SDL2 referencia `usbh_*`, que vive en `nxdk_usb.lib`, y
**`sdl2.pc` no la declara** aunque nxdk la enlaza en todos sus títulos desde
`lib/usb/Makefile`. La clase `Xbox` de `xcompile.py` tampoco la lista. La añade
`xbox.py`, recolocando `libxboxkrnl.lib` al final como pide el comentario del autor.

### 19.7 El binario

```
build/engine/default.xbe            2.805.760   <- XBEH, retail, base 0x00010000
build/engine/xash.exe               2.787.840   <- PE intermedio de nxdk-link
build/3rdparty/mainui/menu.dll        454.656
build/ref/gl/ref_gl.dll               346.112
build/filesystem/filesystem_stdio.dll 192.512
```

Reproducir desde limpio:

```bash
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"
cd ~/opposing-force-x/research/xash3d-fwgs_xbox
./waf distclean && rm -rf build
./waf configure --xbox --xbe-title 'Opposing Force X'
./waf build -j8
```

18,4 s de build con `-j8`, 430 tareas, cero errores. Avisos: los normales de FWGS
(`-Wunused-but-set-variable`, `-Wswitch`) más `'strnicmp' is deprecated`, que sale porque
nxdk marca `stricmp`/`strnicmp` como *deprecated* y `public/crtlib.h` las usa — mismo
aviso que ya salía en el gamedll de §18.4.

### 19.8 Bugs del fork encontrados de paso (no arreglados)

Ninguno bloquea el objetivo, y son ficheros del autor:

1. **`MSGBOX_XBOX` no está definido en ningún sitio.** `common/defaults.h:71` hace
   `#define XASH_MESSAGEBOX MSGBOX_XBOX`, y `common/backends.h` — donde están
   `MSGBOX_STDERR 0`, `MSGBOX_SDL`, etc. — **no tiene la constante**. En `#if` un
   identificador desconocido vale 0, y `MSGBOX_STDERR` vale 0, así que
   `XASH_MESSAGEBOX == MSGBOX_XBOX` es **cierto** en cualquier build que caiga en
   `MSGBOX_STDERR`. Consecuencia: `./waf configure --dedicated && ./waf build` sobre Linux
   falla con `redefinition of 'Platform_MessageBox'` en `engine/common/sys_con.c:435`.
   **El fork no compila para Linux dedicado**, con o sin nosotros. La build Xbox se libra
   porque usa SDL2 y toma `MSGBOX_SDL`.
2. **`XASH_CUSTOM_SWAP` está roto para todas las plataformas** — la cabecera
   `platform/swap/swap.h` de §19.3 no existe.
3. **`-Wno-error=implicit-function-declaration` de `Xbox.cflags()` no surte efecto**, por
   el orden `CFLAGS`/uselib de §19.2, punto 2.
4. **Rutas absolutas a mano en `engine/wscript`**: `/opt/toolchains/xbox/pbgl` y
   `/opt/toolchains/xbox/nxdk` aparecen literales en las líneas 212-214 y 351. Funciona
   aquí porque §17 instaló justo ahí; no es portable.

### 19.9 Ciclo de despliegue a hardware real

**Decisión firme de esta sesión: se prueba contra la consola física, no en xemu.** Los
motivos de §17.4 siguen en pie (xemu exige volcados del MCPX y de la BIOS que sólo son
legales si salen de tu propia consola), y además el ciclo de FTP ya está validado.

| Elemento | Valor |
|---|---|
| Consola | Xbox original, ampliada a **128 MB** de RAM |
| Dashboard | **UnleashX** |
| IP | **<IP-consola>** (DHCP, ver aviso abajo), FTP en el 21 |
| Carpeta del título | **`E:\Apps\xash\`** |
| Validado con | el sample `hello` de nxdk (§17.1), que arrancó |

> **Corrección aplicada en §21.** Esta tabla decía `<IP-incorrecta>` y
> `E:\Games\OpposingForceX`, y las dos cosas eran falsas. Ver §21.1: la consola está en
> **DHCP** y su IP hay que leerla en el **dashboard**, no en el `config.xml` de UnleashX,
> que conserva valores viejos. Y el directorio que UnleashX escanea aquí es `E:\Apps`,
> que es donde arrancó el sample `hello`.

```bash
# 1. build
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"
cd ~/opposing-force-x/research/xash3d-fwgs_xbox
./waf build -j8

# 2. subir: una carpeta por título bajo E:\Apps\
curl --ftp-create-dirs -u xbox:xbox \
     -T build/engine/default.xbe \
     ftp://<IP-consola>/E:/Apps/xash/default.xbe

# 3. en la consola: UnleashX -> Explorador -> E:\Apps\xash -> default.xbe
```

Desde §20 esto está en `research/ofx-xbox-deploy.sh`, que además se trae el log de
vuelta; ver §20.8 y §21.

Notas del ciclo, para no repetir tropiezos:

- **Las credenciales por defecto de UnleashX son `xbox`/`xbox`.** Si se cambiaron, están
  en el `config.xml` de la consola — pero **no te fíes de ese fichero para la IP**
  (§21.1).
- **`default.xbe` es el nombre obligatorio** dentro de la carpeta del juego; por eso
  `xbox.py` lo genera así y no como `xash.xbe`.
- **`D:` es la carpeta del XBE en ejecución.** Es lo que monta `libnxdk_automount_d`, y es
  de donde `GetModuleFileNameA` (§19.4) dice que viene el ejecutable. Los datos del juego
  — `valve/`, `gearbox/` — van **al lado del `default.xbe`**, no en otra partición.
- **No hace falta ISO.** El `--xbe-iso` de §19.2 es para xemu o para grabar; por FTP
  sobra.
- **La salida de diagnóstico va por serie.** El fork trae `Xbox_SerialPrint()` en
  `platform/xbox/sys_xbox.c`, hablando directamente al UART 16550 en 0x3F8. Sin cable
  serie no hay traza de arranque: si el XBE se cuelga, no habrá nada que leer.

**Lo que NO se hizo en esta sesión:** subir este `default.xbe` a la consola. Se sondeó
la dirección que se creía buena y no respondió ni a ICMP ni en el 21. Se dio por hecho
que la consola estaba apagada; en §21.1 resultó que además **la dirección era la
equivocada**. El despliegue real está en §21.

### 19.10 Qué NO está verificado

Con las mismas letras que §18: **todo lo de §19 mide compilación y enlazado. La única
afirmación de ejecución que contiene es que no ha habido ninguna.**

En concreto, y por orden de probabilidad de estallar:

1. **El motor no puede cargar sus propios módulos.** `menu.dll`, `ref_gl.dll` y
   `filesystem_stdio.dll` son DLLs, y nxdk no carga DLLs (§18.6). Esto no se ha tocado.
   Es el mismo muro de §18.7, y sigue siendo el problema central del port.
2. **`public/xbox_fd.c` nunca ha abierto un fichero.** La traducción de banderas, el
   `dup()` por reapertura y la conversión `FILETIME`→`time_t` de `sys/stat.h` son código
   escrito y compilado, no probado.
3. **`GetModuleFileNameA` supone que `D:` está montado** cuando se le llama. Si el motor
   pregunta antes de que `libnxdk_automount_d` haya hecho su trabajo, devolverá una ruta
   que no existe.
4. **`xbox_sbrk.c` no se ejecuta** y ni siquiera se compila en la build por defecto
   (§19.3).
5. **El presupuesto de memoria no está medido.** La consola tiene 128 MB (una retail sin
   ampliar tendría 64), el XBE son 2,8 MB, y el `Init Flags` de la cabecera dice
   *Limit Devkit Run Time Memory to 64MB*. Eso último puede que haya que cambiarlo para
   aprovechar los 128; no se ha investigado.
6. **`atof` es un tapón** (§19.4), como en §18.5.

### 19.11 Estado del entorno tras esta sesión

```
/opt/toolchains/xbox/
├── nxdk/
│   ├── bin/pkgconf                        NUEVO (nuestro) — shim a pkg-config, §19.6a
│   ├── lib/libSDL2.lib                    NUEVO — 1.525.470 B, §19.6b
│   ├── lib/sdl/SDL2/Makefile.standalone   NUEVO (nuestro) — §19.6b
│   └── tools/cxbe/cxbe                    ya estaba, de §17.1
└── pbgl/                                  intacto, §17.2

~/opposing-force-x/research/xash3d-fwgs_xbox/
├── scripts/waifulib/xbox.py                    NUEVO (nuestro)   346 lineas
├── engine/platform/xbox/xbox_sbrk.c            NUEVO (nuestro)   154
├── engine/platform/xbox/xbox_winapi.c          NUEVO (nuestro)    90
├── engine/platform/xbox/nxdk-compat/           NUEVO (nuestro)  13 ficheros
├── public/xbox_fd.c                            NUEVO (nuestro)   403
└── build/engine/default.xbe                    2.805.760 B
```

`git status` del fork: **ningún fichero trackeado modificado.** Los cinco caminos de
arriba salen como `??`.

`game/`, `game-win32/`, `game-offsets/`, `hlsdk-portable/` y el `build-nxdk` del gamedll
de §18: intactos. No se ha tocado nada de los pasos 1 y 2b.

## 20. Paso 2d — Ver qué pasa: trazas de arranque y los tres módulos dentro del XBE

**Resumen: la pantalla estaba negra y sin log porque en este target el motor no
escribe nada en pantalla — todo su console va al puerto serie, y el único camino que
sí pinta (el message box de error) es código muerto por un `#define` que falta. Se ha
escrito un trazador que escribe a la vez en pantalla, en serie y en
`D:\xash-boot.log`, recuperable por FTP. Y se ha cerrado el bloqueo de §18.6:
`filesystem_stdio`, `ref_gl` y `menu` ya no son DLLs, están dentro del `default.xbe`.**

| | Antes | Ahora |
|---|---|---|
| Salida en pantalla durante el arranque | ninguna | cada línea de traza y todo el console |
| Log recuperable sin cable serie | no existía | `D:\xash-boot.log` por FTP |
| `Sys_Error` visible | no (§20.1b) | sí, en pantalla y en el log |
| `filesystem_stdio` / `ref_gl` / `menu` | 3 DLLs que nxdk no puede abrir | dentro del XBE |
| `default.xbe` | 2.805.760 B | **3.350.528 B** |
| Ficheros nuestros | 18 / 1.799 líneas | **19 / 2.475 líneas** |
| Ficheros del fork modificados | 0 | **5, +97/−1** (§20.3) |

**Nadie ha ejecutado este binario todavía.** La consola no respondía durante la sesión.
Todo lo de abajo es análisis estático y verificación de compilación y enlazado.

### 20.0 Cambio de regla respecto a §19

§19 presumía de no tocar ni un fichero trackeado del fork. Eso ya no se sostiene: el
encargo era instrumentar el arranque «desde la primera línea de `main`», y las trazas
tienen que ir donde está el arranque. Se han tocado **cinco ficheros, 97 líneas
añadidas y una borrada**, todas marcadas con el comentario `XTRACE` y el número de
sección, y el diff completo está en `~/opposing-force-x/xash-xbox-boot-trace.patch`
(298 líneas). Todo lo demás sigue en ficheros nuevos nuestros.

### 20.1 Por qué la pantalla estaba negra y sin log

Tres causas, las tres leídas en el código, ninguna adivinada:

**a) En este target el motor no escribe en pantalla. Nunca.**
`engine/common/sys_con.c:277-292` es la rama Xbox de `Sys_Print`, y lo único que hace
con cada línea del console es `Xbox_SerialPrint( line )`. No hay `debugPrint`, no hay
fichero. Es decir: **un arranque perfectamente sano, hasta el último `Con_Printf`,
produce exactamente lo que se vio — una pantalla negra.** El síntoma no dice que el
motor esté muerto; dice que no tiene por dónde hablar.

**b) `Sys_Error` era mudo.** `engine/common/system.c:425` sí llama a `debugPrint`, así
que un error fatal *debería* verse. Pero antes de eso, `Platform_MessageBox` acaba en
SDL2, que en nxdk no muestra nada. Y el message box de Xbox que el autor escribió
(`sys_con.c:438`, que pinta con `debugPrint` y manda por serie) **es código muerto**:
`common/defaults.h:71` selecciona `MSGBOX_XBOX` y `common/backends.h` **no define esa
constante** — el autor añadió el caso y se dejó el valor. Dentro de un `#if`, un
identificador desconocido vale 0, `MSGBOX_STDERR` vale 0, y `XASH_SDL >= 2` gana antes
en la cadena de `#elif`, así que la plataforma se queda con `MSGBOX_SDL`.

Ya se apuntó este bug en §19.8 como «rompe el build de Linux dedicado». Lo que no se
vio entonces es que **también silencia todos los errores fatales en la consola**.
Arreglado desde `xbox.py` con `-DMSGBOX_XBOX=5 -DXASH_MESSAGEBOX=5`: `backends.h` no
define la constante y `defaults.h` guarda `XASH_MESSAGEBOX` con `#ifndef`, así que dos
`-D` bastan sin tocar el fork.

**c) Riesgo que se ha quitado, sin poder confirmar que fuera la causa.**
`Xbox_SerialPutc()` en `platform/xbox/sys_xbox.c` gira hasta **100.000 veces** esperando
el bit *transmitter holding register empty* antes de cada carácter, y `sys_con.c` lo usa
para **cada** `Con_Printf`. En una consola retail el conector serie no está poblado.

Si el puerto 0x3F8 no responde y `inb` devuelve 0xFF —lo normal en un bus LPC sin
dispositivo— el bit THRE (0x20) sale puesto y no se gira nada: no habría problema. El
problema aparecería solo si algo contesta y nunca afirma THRE, y entonces cada línea
costaría segundos y **un motor sano parecería colgado**. No se puede saber cuál de los
dos casos es el de esta consola sin ejecutarla, así que **esto es una hipótesis, no un
diagnóstico**. Lo que sí se ha hecho es quitar el riesgo: se sondea el UART por su
registro *scratch* al arrancar y, si no contesta, no se manda nada por serie.

### 20.2 El trazador — `engine/platform/xbox/xbox_trace.c`

302 líneas. Escribe cada línea en tres sitios a la vez, y el orden importa:

| Destino | Cómo | Para qué |
|---|---|---|
| Pantalla | `debugPrint` de nxdk | se ve sin hardware extra, pero hace scroll y cualquier `XVideoSetMode` posterior la borra |
| Serie | `Xbox_SerialPrint` del fork, **solo si el sondeo encuentra UART** | sobrevive a todo, hace falta un cable que no hay |
| Disco | `D:\xash-boot.log`, junto al XBE | **es el que importa**: sobrevive al cuelgue, al reset y a la pantalla borrada, y se recoge por FTP con lo que ya hay montado |

Decisiones que conviene tener escritas:

- **El fichero se abre, se escribe y se cierra en cada línea.** Es absurdamente lento y
  completamente deliberado: la última línea antes de un cuelgue tiene que estar ya en
  disco, y nxdk no expone `FlushFileBuffers` para forzarlo de otra forma. Un arranque
  son unos cientos de líneas.
- **Número de secuencia y `GetTickCount()` en cada línea.** Si el log acaba en la 41 y
  la pantalla enseña la 43, las dos últimas murieron entre el pintado y la escritura.
- **Un constructor en `.CRT$XCU`** arranca el trazador antes de `main`. nxdk ejecuta los
  inicializadores CRT en `_PDCLIB_xbox_run_crt_initializers()` justo antes de llamar a
  `main` (`lib/pdclib/platform/xbox/crt0.c`), y el automontaje de `D:` va aún antes, en
  `.CRT$XIT`. Así que **si el log sale vacío, eso ya es el diagnóstico**: se muere en
  crt0, en el TLS o en el montaje de `D:`, antes que cualquier código del motor.
- **Levanta el framebuffer si nadie lo ha hecho** (`XVideoGetFB()` a NULL →
  `XVideoSetMode`). `launcher.c` lo hace también en su primera línea; la segunda llamada
  vuelve a poner el framebuffer a cero, así que **la pantalla solo conserva trazas a
  partir de ahí**. El log las tiene todas.
- **`Xbox_TraceRaw()`** imprime texto ya formado sin numerar, y es donde `sys_con.c`
  manda ahora el console entero. Eso convierte el log de una lista de hitos en **toda la
  salida de consola del arranque**, que es lo que de verdad hace falta en una máquina
  sin terminal.

Se declara en `nxdk-compat/nxdk_compat.h`, que `xbox.py` fuerza con `-include` en cada
unidad de traducción, así que una traza en cualquier fuente del motor es una línea sin
`#include` y sin pensar en el orden de enlazado.

### 20.3 Dónde se instrumenta

`engine/wscript:207` recoge solo `platform/<DEST_OS>/*.c`, así que el trazador entra sin
tocar ningún wscript. Las trazas sí van en ficheros del fork:

| Fichero | Qué se añade |
|---|---|
| `engine/common/launcher.c` | entrada a `main`, `argc`/`argv`, después de `XVideoSetMode`, después de `Xbox_GetArgv`, antes de `Host_Main` |
| `engine/common/host.c` | macro `XTRACE` + 15 hitos: `Host_InitCommon` entrada, línea de órdenes, nombre del ejecutable, `Memory_Init`, `Cmd`/`Cvar`, `Con_Init`, `Platform_Init`, `FS_Init`, `Image`/`Sound`, `FS_LoadGameInfo`, `gfx/conchars`, fin; y en `Host_Main`: entrada, `SV_Init`, `CL_Init`, entrada al bucle de frames |
| `engine/common/filesystem_engine.c` | `FS_DetermineRootDirectory`, `rootdir`, `rodir`/`gamedir`, `COM_LoadLibrary` del módulo y su handle, `InitStdio`, montaje completo |
| `engine/common/system.c` | el texto de `Sys_Error` también al log, que sobrevive al cuelgue que viene detrás |
| `engine/common/sys_con.c` | el console a pantalla y a disco; el serie solo si hay UART |

En `host.c` y `filesystem_engine.c` la traza es una macro que se compila a `((void)0)`
fuera de Xbox. **Comprobado**: `./waf configure -o build-host --enable-stbtt && ./waf
build -o build-host` sigue saliendo entero en Linux x86_64 — 498 tareas, `libxash.so`,
`libmenu.so`, `libref_gl.so`, `filesystem_stdio.so`, cero `.xbe`.

### 20.4 Los módulos, dentro del XBE

Ese era el bloqueo de §18.6: nxdk no puede cargar una DLL, su `LoadLibraryExA` es un
stub que devuelve `NULL` siempre, y el motor pide `filesystem_stdio` antes de nada. En
una máquina sin cargador dinámico, meterlos dentro no es un apaño: **es la única forma
que puede tener el binario.**

**Qué de FWGS sirve y qué no.** §13 dejó el mecanismo mapeado. Tiene dos mitades:

- **La mitad de ejecución sirve tal cual.** `XASH_STATIC_LIBS` conmuta
  `common/defaults.h` a `XASH_LIB == LIB_STATIC`, y `platform/misc/lib_static.c`
  sustituye `LoadLibrary`/`GetProcAddress` por una búsqueda lineal sobre unas tablas que
  vienen de `generated_library_tables.h`. Es C plano y portable.
- **La mitad de build no sirve.** `scripts/waifulib/xshlib.py` hace cada módulo
  relocalizable con `ld -r` y le esconde los símbolos con `objcopy -G`. Las dos son
  herramientas **solo-ELF**; para COFF no hay enlazado parcial equivalente en lld. Y §13.5
  ya había dejado escrito que ni siquiera en Linux funciona para módulos con
  dependencias externas.

Así que se ha reimplementado la mitad de build en `xbox.py`, sin `ld -r` y sin
`objcopy -G`:

1. Los tres task generators cambian de `cxxshlib` a `cxxstlib`, y salen librerías
   estáticas normales.
2. Se genera el `link_helper.c` de cada módulo a partir de su `exports.txt` — el mismo
   fichero que lee `xshlib.py` — más los puntos de entrada que el motor pide y que ahí
   no están (`CreateInterface` para el filesystem, que sale de
   `filesystem/VFileSystem009.cpp`).
3. Se genera `generated_library_tables.h` en el directorio de build del motor. Lo
   encuentra porque el target `engine_includes` exporta `.`, que waf traduce a la vez al
   directorio fuente y al de build.
4. Se añade `platform/misc/lib_static.c` a las fuentes del motor y los tres módulos a su
   `use`.

**Una trampa de waf que costó una iteración.** waf calcula la lista de métodos a
ejecutar *al empezar* a postear el task generator, a partir de las features de ese
momento. Cambiar `self.features` después no desprograma nada, y `apply_implib` — que en
un target PE añade una import library llamada `<target>.lib` a las salidas del enlace —
seguía corriendo. Ese es exactamente el nombre que ahora tiene la librería estática, así
que waf veía el mismo nodo creado dos veces y `llvm-ar` recibía su propia salida como
entrada. Se resuelve tapando el método con un atributo de instancia (`post()` lo resuelve
con `getattr(self, nombre)`, y una instancia gana a la clase).

### 20.5 Los choques de símbolos, medidos

Sin `objcopy -G` todos los símbolos del módulo siguen siendo globales en la imagen
final. Medido con `llvm-nm` sobre los objetos ya compilados:

| Módulo | Símbolos globales | Coinciden con el motor | **Choques reales** |
|---|---|---|---|
| `filesystem_stdio` | 447 | 39 | **8** |
| `ref_gl` | 1.750 | 293 | **21** |
| `menu` | 2.767 | 247 | **0** |

La diferencia entre «coinciden» y «choques reales» son literales de cadena COMDAT
(`??_C@...`) y constantes SSE (`__xmm@...`): el enlazador los funde, no chocan. Los
reales son de una sola clase — el motor guarda envoltorios con el mismo nombre que la
implementación de verdad: `FS_Open`, `FS_Close`, `FI`, `_Mem_Alloc` en el filesystem;
`R_Init`, `R_Shutdown`, el TriAPI entero y unos cuantos cvars en el renderer. `menu`
habla con el motor solo por `GetMenuAPI`/`GetExtAPI` y no choca en nada.

Con 29 símbolos no hacía falta enlazado parcial: se renombra la copia del módulo con
`llvm-objcopy --redefine-syms`, objeto a objeto, antes de archivar. Renombrar por
nombre alcanza a la definición y a las referencias a la vez, así que aplicado a todo el
módulo es equivalente a localizar el símbolo. `_FS_Open` pasa a `_xs_filesystem_FS_Open`
dentro del módulo, y el motor se queda con el suyo — que es exactamente el reparto que
había con DLLs separadas.

> Comprobado que `llvm-objcopy` 14 hace `--redefine-syms` sobre COFF, y que renombra
> también las referencias indefinidas. La lista está a mano en `STATIC_MODULE_RENAMES`,
> con `research/ofx-xbox-symbols.sh` para recalcularla; se ha verificado que el script
> reproduce exactamente las dos listas. Si upstream añade un choque nuevo, el enlace
> falla con un `duplicate symbol` que lo nombra: es un error perfectamente bueno.
>
> No se ha renombrado a ciegas todo el módulo, que habría evitado la lista: los puntos
> de entrada llevan `__declspec(dllexport)` y eso deja directivas `/EXPORT:` en
> `.drectve` que `objcopy` no reescribe. Renombrar el símbolo y dejar la directiva
> apuntando al nombre viejo rompería el enlace.

### 20.6 Un agujero de FWGS que sale al usar su propio modo estático

`engine/common/library.h` declara `COM_CheckLibraryDirectDependency`, `lib_win.c` y
`lib_posix.c` la definen, y **`platform/misc/lib_static.c` —que sustituye a las dos
cuando `XASH_LIB` es `LIB_STATIC`— no**. `engine/client/cl_game.c:3953` la llama sin
condición. Es decir: **el modo estático de FWGS no enlaza en ninguna plataforma**, no
solo en ésta.

Implementada en `engine/platform/xbox/xbox_winapi.c`. Contesta a una sola pregunta: si
la librería de cliente importa SDL2 directamente, lo que decide si el motor le pasa
input crudo. Metido un módulo dentro del ejecutable ya no hay tabla de imports que
mirar, y aquí no hay módulo de cliente en absoluto, así que devuelve `false`. **Habrá
que revisarlo cuando entre el gamedll (§13.6): un cliente enlazado estáticamente
comparte el SDL2 del motor lo pida o no.**

### 20.7 Efectos secundarios de `XASH_STATIC_LIBS` que hay que tener presentes

`common/defaults.h:107-110` activa tres cosas de golpe, no una:

```c
#elif defined( XASH_STATIC_LIBS )
	#define XASH_LIB LIB_STATIC
	#define XASH_INTERNAL_GAMELIBS
	#define XASH_ALLOW_SAVERESTORE_OFFSETS
```

- `XASH_INTERNAL_GAMELIBS` hace que `COM_GenerateServerLibraryPath()` devuelva
  literalmente `"server"` y el cliente y el menú se busquen como `"client"` y `"menu"`
  (§13.2). El menú está en la tabla; **`server` y `client` no existen todavía**, así que
  la carga del gamedll fallará. Es lo esperado en este punto y saldrá en el log.
- También desactiva `Host_CheckGameLibraries()` (`host.c:914`), que buscaba `.dll`/`.so`
  sueltos en el disco. Bien: aquí no los hay.
- `XASH_ALLOW_SAVERESTORE_OFFSETS` se activa gratis. Es el mecanismo de §15, validado
  entonces como red de seguridad. Ahora está puesto de verdad.

### 20.8 El binario, y cómo usarlo

```
build/engine/default.xbe   3.350.528 B   XBEH, retail, base 0x00010000
build/engine/xash.exe      3.335.168 B   PE intermedio
build/engine/xash.map      2.196.824 B   mapa del enlazador, para resolver direcciones
```

Reproducir:

```bash
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"
cd ~/opposing-force-x/research/xash3d-fwgs_xbox
./waf distclean && rm -rf build
./waf configure --xbox --xbe-title 'Opposing Force X'
./waf build -j8
```

Verificado sobre `xash.map` que los tres módulos están dentro:

```
_GetRefAPI    ref_gl:gl_context.c.1.rn.o
_GetFSAPI     filesystem_stdio:filesystem.c.2.rn.o
_GetMenuAPI   menu:udll_int.cpp.1.o
```
más `_lib_*_exports`, `_COM_LoadLibrary`, `_Lib_Find` y los 29 símbolos `_xs_*`.

Para volver al comportamiento anterior (tres DLLs, que la consola no puede abrir):
`./waf configure --xbox --xbox-shared-modules`.

**El ciclo con la consola**, con `research/ofx-xbox-deploy.sh`:

```bash
# sube el xbe y, antes de pisarlo, se trae el log de la ejecucion anterior
~/opposing-force-x/research/ofx-xbox-deploy.sh

# solo recoger el log
~/opposing-force-x/research/ofx-xbox-deploy.sh log
```

Ajustable por entorno: `XBOX_IP` (**<IP-consola>**), `XBOX_USER`/`XBOX_PASS`
(xbox/xbox), `XBOX_DIR` (**`E:/Apps/xash`**). Los logs recogidos se guardan con marca
de tiempo en `research/logs/`.

> **Corrección aplicada en §21.** Cuando se escribió §20 estos dos valores eran
> `<IP-incorrecta>` y `E:/Games/OpposingForceX`, tomados del `config.xml` del dashboard y
> de la convención habitual de UnleashX. Los dos estaban mal. La consola está en
> **DHCP** y la IP se lee en el dashboard; la carpeta que escanea es `E:\Apps`. Ver
> §21.1.

### 20.9 Cómo leer el log

Esta es la secuencia que debería salir. Donde se corte es donde muere:

```
[0000 ...] trace up: log=D:\xash-boot.log fb=0x... serial=no uart at 0x3F8
[0001 ...] crt initializers: alive before main()
[0002 ...] main: entered, argc=0 argv=0x... argv0=(null)
[0003 ...] main: XVideoSetMode 640x480x32 done
[0004 ...] main: Xbox_GetArgv -> argc=0 argv=0x...
[0005 ...] main: calling Host_Main, gamedir=valve
[0006 ...] Host_Main: enter
[0007 ...] Host_InitCommon: enter, argc=0 basedir=valve
...        exename, Memory_Init, Cmd/Cvar, Con_Init
[....] Host_InitCommon: Platform_Init (SDL2 on this target)
[....] Host_InitCommon: FS_Init
[....] FS_Init: FS_DetermineRootDirectory (mounts D: on this target)
[....] FS_Init: rootdir=D:\
[....] FS_LoadProgs: COM_LoadLibrary(filesystem_stdio)
[....] FS_LoadProgs: handle=0x...        <- distinto de 0 = el modo estatico funciona
[....] FS_LoadProgs: filesystem_stdio ready
[....] FS_Init: InitStdio
[....] FS_Init: filesystem mounted       <- objetivo ideal de la sesion alcanzado
```

Tres lecturas rápidas:

- **Log vacío o inexistente**: muere antes de los inicializadores CRT. Mira crt0, el
  TLS y el automontaje de `D:`. También puede ser que `D:` no sea escribible, en cuyo
  caso el trazador cae a `E:\xash-boot.log`; la primera línea dice cuál eligió.
- **Llega a `main: entered` con `argc=0` y `argv0=(null)`**: es lo normal aquí, no un
  fallo. nxdk llama a `main(0, &argv)` con `argv[0]` a NULL (`crt0.c`), así que
  cualquier código que dé por hecho un nombre de ejecutable es sospechoso.
  `Sys_SetupCrashHandler( argv[0] )` recibe ese NULL, aunque en este target el crash
  handler está desactivado (`engine/wscript:105`) y lo que se llama es el stub
  `static inline` de `platform/platform.h:208`, que no lo mira.
- **`FS_LoadProgs: handle=0x0`**: el modo estático no encontró el módulo. Sería un fallo
  de las tablas generadas, no del cargador.

Las direcciones que salgan en un `Sys_Error` se resuelven contra
`build/engine/xash.map`, que el `-Wl,/MAP` de `wscript:280` ya genera en cada build.

### 20.10 Qué NO está verificado

Igual que §18 y §19: **todo esto mide compilación y enlazado. Sigue sin haber ninguna
ejecución.** En concreto:

1. **El trazador nunca ha escrito una línea.** El camino de disco
   (`CreateFileA`/`SetFilePointer`/`WriteFile` sobre FATX), el sondeo del UART y
   `debugPrint` sobre un framebuffer levantado por nosotros son código escrito y
   compilado, nada más.
2. **La hipótesis del serie (§20.1c) sigue siendo una hipótesis.** Se ha quitado el
   riesgo, no se ha demostrado que fuera la causa.
3. **El modo estático no se ha ejercitado.** Que `GetFSAPI` esté en el mapa dice que el
   enlazador lo metió, no que `Lib_Find` lo encuentre ni que `InitStdio` funcione.
4. **Los 29 renombrados no se han ejecutado.** Renombrar definiciones y referencias a la
   vez es correcto por construcción, pero nadie ha llamado a ninguna de esas funciones.
5. **La carga del gamedll va a fallar** (§20.7), y es lo esperado: no hay módulo
   `server` ni `client`.
6. **El presupuesto de memoria empeora.** El `Size of Image` del XBE pasa de 0x0061A114
   a **0x00A4B114**, unos 10,8 MB de imagen virtual, y la cabecera sigue diciendo *Limit
   Devkit Run Time Memory to 64MB* pese a que la consola tiene 128 MB. Sigue sin medirse.
7. **`public/xbox_fd.c` de §19.5 sigue sin abrir un fichero.** Ahora tiene por fin un
   camino que lo va a ejercitar de verdad: `InitStdio` escanea directorios en cuanto
   arranca.

### 20.11 Estado del entorno tras esta sesión

```
~/opposing-force-x/
├── xash-xbox-boot-trace.patch          NUEVO — 5 ficheros del fork, +97/-1
└── research/
    ├── ofx-xbox-verify.sh              build desde limpio + comprobaciones
    ├── ofx-xbox-symbols.sh             recalcula STATIC_MODULE_RENAMES
    ├── ofx-xbox-deploy.sh              NUEVO — sube el xbe, baja el log
    ├── ofx-pkgconf-shim.sh             el shim de §19.6a
    └── xash3d-fwgs_xbox/
        ├── scripts/waifulib/xbox.py            678 lineas (era 346)
        ├── engine/platform/xbox/xbox_trace.c   NUEVO, 302 lineas
        ├── engine/platform/xbox/xbox_winapi.c  121 (era 90)
        ├── engine/platform/xbox/xbox_sbrk.c    154, sigue sin entrar en el build
        ├── engine/platform/xbox/nxdk-compat/   14 ficheros
        ├── public/xbox_fd.c                    403
        └── build/engine/default.xbe            3.350.528 B
```

19 ficheros nuestros, **2.475 líneas**. Cinco ficheros del fork modificados, con el
diff aparte. `game/`, `game-win32/`, `game-offsets/`, `hlsdk-portable/` y el toolchain
de §17: intactos.

## 21. Paso 2e — Primer arranque instrumentado en hardware, y por qué moría

**Resumen: el trazador funciona. El primer log real de la consola tiene siete líneas y
se corta justo después de `Host_Main: enter`. El desensamblado dice que en ese hueco
sólo hay una llamada, `Platform_DoubleTime()`, y que compilaba a **SSE2** — que el
Pentium III del Xbox no tiene. Era un opcode inválido en la primera operación en coma
flotante de todo el arranque. Arreglado en el origen y verificado: 0 instrucciones SSE2
en los 418 objetos del build. Y el XBE ya vuelve solo al dashboard, así que se acabó el
botón de encendido.**

| | |
|---|---|
| Primer log en hardware | **7 líneas, 407 bytes**, recuperado por FTP |
| Dónde muere | entre `Host_Main: enter` y `Host_InitCommon: enter` |
| Qué hay en ese hueco | **una sola llamada**: `Platform_DoubleTime()` |
| Causa | `-msse2` en un CPU con SSE1; §21.2 |
| Objetos del motor afectados | **107 de 418**, `sv_client.c` con 554 instrucciones |
| Tras el arreglo | **0 en 418** |
| Salida al dashboard | `Sys_Quit`, vigilante de 30 s y START+BACK |

### 21.1 Corrección de los datos de la consola

Los dos datos con los que se trabajó en §19 y §20 eran falsos, y conviene dejar escrito
por qué, porque el modo de fallar se repetirá:

| | §19/§20 decía | Real |
|---|---|---|
| IP | `<IP-incorrecta>` | **<IP-consola>** |
| Carpeta | `E:\Games\OpposingForceX` | **`E:\Apps\xash\`** |

- **La consola está en DHCP.** La IP hay que leerla **en el dashboard** (pantalla de
  estado / info de red de UnleashX), **no en su `config.xml`**, que conserva valores
  viejos: de ahí salió `<IP-incorrecta>`. Que además no respondiera en §19 se atribuyó a
  «la consola estaba apagada», y esa explicación tapó el error durante dos sesiones.
- **UnleashX escanea `E:\Apps`** en esta instalación, no `E:\Games`. Se confirma en el
  listado FTP: junto a `xash` está `hello`, la carpeta del sample de nxdk de §17.1 que
  sí arrancó.

Corregidos §19.9, §20.8 y `research/ofx-xbox-deploy.sh`, que ahora además avisa de lo
del DHCP en el mensaje de error del ping.

### 21.2 El log, íntegro

```
[0000     2519] trace up: log=D:\xash-boot.log fb=0x83eb4000 serial=no uart at 0x3F8
[0001     2520] crt initializers: alive before main()
[0002     2521] main: entered, argc=0 argv=0xd0061c84 argv0=(null)
[0003     2597] main: XVideoSetMode 640x480x32 done
[0004     2598] main: Xbox_GetArgv -> argc=0 argv=0xd0061c84
[0005     2599] main: calling Host_Main, gamedir=valve
[0006     2600] Host_Main: enter
```

Lo que confirma, antes de mirar el fallo:

- **El trazador funciona entero.** Escribió en pantalla y en `D:\xash-boot.log`, y el
  fichero sobrevivió al cuelgue y al reset — que era exactamente el objetivo de §20.
- **`serial=no uart at 0x3F8`**: el sondeo del registro *scratch* dice que esta consola
  no tiene UART poblado. La precaución de §20.1c estaba justificada; en esta corrida no
  llegó a importar porque no hubo ni un `Con_Printf`.
- **`argc=0`, `argv0=(null)`**: como se predijo en §20.9, nxdk llama a `main(0, &argv)`
  con `argv[0]` a NULL. No es el fallo, pero queda confirmado.
- **`XVideoSetMode` tarda 76 ms** (2521→2597). Todo lo demás va en 1 ms.
- Los inicializadores CRT pasaron, así que los constructores de C++ que ahora entran en
  el XBE (mainui, §20.4) se ejecutaron bien.

**El corte es en el tick 2600 y no hay una línea más.**

#### El hueco, medido en vez de adivinado

Entre la traza que sí sale y la siguiente, el fuente tiene esto:

```c
	XTRACE( "Host_Main: enter" );

	host.starttime = Platform_DoubleTime();

	Host_InitCommon( argc, argv, progname, bChangeGame, exename, sizeof( exename ));
```

y `Host_InitCommon` empieza declarando `basedir` y trazando. Desensamblando el objeto
(`llvm-objdump -d engine/common/host.c.2.o`) se ve que **clang inlinea `Host_InitCommon`
dentro de `Host_Main`**, así que el hueco real es aún más corto de lo que parece:

```
 844:  pushl $0 ; calll        <- Xbox_Trace( "Host_Main: enter" )
 851:  calll                   <- Platform_DoubleTime()
 856:  fstpl 616               <- host.starttime =
 85e:  cmpb  $35, (%ebx)       <- progname[0] == '#'   (Host_InitCommon, inlineado)
 86f:  pushl %eax ; pushl %esi ; pushl $0 ; calll   <- Xbox_Trace( "Host_InitCommon: enter, ..." )
```

**Una única llamada en todo el hueco.** No hay margen para dudar de dónde muere.

#### Y `Platform_DoubleTime` compilaba a SSE2

`engine/platform/sdl2/sys_sdl2.c`, tal y como salía del compilador:

```
 47:  movd      %edx, %xmm0
 4f:  punpckldq %xmm0, %xmm1     <- SSE2
 53:  movdqa    0, %xmm0         <- SSE2
 5f:  movapd    0, %xmm2         <- SSE2
 67:  subpd     %xmm2, %xmm1     <- SSE2
 6f:  unpckhpd  %xmm1, %xmm3     <- SSE2
 73:  addsd     %xmm1, %xmm3     <- SSE2
 93:  divsd     %xmm0, %xmm3     <- SSE2
 97:  movsd     %xmm3, (%esp)    <- SSE2
```

El CPU del Xbox es un **Pentium III Coppermine a 733 MHz: MMX y SSE, sin SSE2.**
`punpckldq %xmm0,%xmm1` ahí es un **opcode inválido**, `#UD`.

Y encaja con el síntoma hasta el último detalle: `Platform_DoubleTime()` es
**la primera aritmética en coma flotante que ejecuta el arranque**. Todo lo anterior
—las trazas, el montaje de `D:`, `XVideoSetMode`, `Xbox_GetArgv`— es enteros y cadenas.
El motor moría en la primera instrucción que el hardware no sabe ejecutar.

Esto también explica, retroactivamente, el «cuelga en negro» original de antes de §20:
no era el cargador de DLLs, ni el filesystem, ni los assets. **Nunca llegó tan lejos.**

### 21.3 La cadena completa, que es lo interesante

El bug no está donde duele, y tiene cuatro eslabones:

1. **`3rdparty/opus/wscript:74`** comprueba `conf.check(header_name='emmintrin.h')` y,
   si existe, hace `conf.env.CFLAGS += ['-msse2']`.
2. Esa comprobación **contesta a la pregunta equivocada**: clang trae `emmintrin.h`
   para cualquier target x86, tenga el CPU SSE2 o no. Nunca se pregunta por el CPU.
3. **`conf.env.CFLAGS` es compartido.** Los subproyectos se configuran en el orden de
   `SUBDIRS`, `3rdparty/opus` está en la línea 98 y **`engine` es el último, línea 109,
   con el comentario `# keep latest for static linking`**. Así que el `-msse2` de opus
   se hereda en mainui, MultiEmulator y el motor.
4. **El orden de las flags.** `wscript:278` sí hace
   `conf.env.append_unique('CFLAGS', ['-mno-sse2'])` para xbox — el autor **sabía** que
   este CPU no tiene SSE2. Pero eso ocurre en la línea 278 y los flags de optimización
   se añaden en la 345, así que la línea de órdenes acaba en
   `... -mno-sse2 -g ... -Os -msse -msse2`. **Gana el último.**

El `CFLAGS` real del motor, del `c4che`, con el `-mno-sse2` muerto en mitad de la lista:

```
'-mno-sse2', '-g', '-gdwarf-2', '-fvisibility=hidden', '-fno-threadsafe-statics',
'-fasynchronous-unwind-tables', '-Os', '-msse', '-msse2'
```

Es **el tercer bug de la misma familia** en este port: una flag correcta anulada por
otra posterior. Los otros dos están en §20.2 (`-Wno-error=strict-prototypes`) y en
§19.8 (`-Wno-error=implicit-function-declaration` de `xcompile.py`, que tampoco hace
nada). En un build con varios subproyectos que comparten `conf.env`, **el orden es
parte de la semántica**, y ni el fork ni FWGS lo tratan como tal.

### 21.4 El arreglo, en dos sitios, y su verificación

**a) En el origen — `3rdparty/opus/wscript`.** En xbox no se define
`OPUS_X86_MAY_HAVE_SSE2`, no se añade `-msse2` y no se compilan
`opus/celt/x86/pitch_sse2.c` ni `vq_sse2.c`. Hicieron falta las dos cosas: esos dos
ficheros incluyen `<emmintrin.h>` y usan los intrínsecos **sin ninguna guarda dentro**,
así que quitar el define no basta, hay que quitar los ficheros.

**No se pierde nada.** En 32 bits opus no define `OPUS_X86_PRESUME_SSE2`: usa despacho
en tiempo de ejecución (`OPUS_HAVE_RTCD` + `CPU_INFO_BY_C`), así que en un Pentium III
esos caminos eran código muerto que el `cpuid` nunca habría elegido. `-msse` se queda:
SSE1 sí lo tiene esta máquina.

**b) Como red — `scripts/waifulib/xbox.py`.** `-mno-sse2` en **`CPPFLAGS`**, no en
`CFLAGS`, por la misma razón de orden de §20.2: la orden de compilación emite `CPPFLAGS`
*después* de `CFLAGS`, así que es la última palabra diga lo que diga cualquiera más
tarde. Si alguien vuelve a colar un `-msse2`, esto lo gana; y si además vuelve a colar
un fichero de intrínsecos SSE2, el build fallará **ruidosamente** en vez de producir un
binario que se cae en la consola.

**Verificación.** `research/ofx-xbox-nosse2.sh` desensambla cada objeto que acaba dentro
del XBE y busca mnemónicos SSE2 (excluyendo a propósito `movq`/`movd`, que ya existen en
MMX/SSE1):

```
antes:   107 objetos con SSE2 (sv_client.c 554, snd_utils.c 486, cl_main.c 418, host.c 107...)
después: OK: 0 instrucciones SSE2 en 418 objetos
```

Y `Platform_DoubleTime` ahora es x87 puro, que es lo que un Pentium III ejecuta:

```
 51:  fildll (%esp)
 54:  fadds  (,%edx,4)
 5b:  fstpl  16(%esp)
 88:  fdivl  24(%esp)
```

> Comprobado que el arreglo está acotado: en un build de Linux x86_64 opus **sigue**
> con `-msse2` y con sus dos objetos SSE2. Sólo cambia el target xbox.

### 21.5 Salida limpia al dashboard

El otro encargo de la sesión, y el que quita el botón de encendido del ciclo.

**El cuelgue era intencionado.** `engine/common/system.c`, `Sys_Quit()`:

```c
Host_ShutdownWithReason( reason );
#if XASH_XBOX
	for( ;; ) Platform_Sleep( 1000 );
```

Una admisión honesta: un título que termina en una consola no tiene a dónde volver. Pero
tiene un coste que en §20 no se había medido — **mientras corre el título la consola
desaparece de la red**, porque el servidor FTP es del dashboard. Sin reset no hay forma
de recoger el log, y con reset se pierde el tiempo de cada iteración.

`XLaunchXBE(NULL)` es la salida. En `nxdk/lib/hal/xbox.c` rellena la *launch data page*
con `LDT_LAUNCH_DASHBOARD` y pide `HalReturnToFirmware(HalQuickRebootRoutine)`: la
consola vuelve al dashboard —UnleashX aquí— con la red levantada. No retorna;
`XReboot()` queda de reserva para el único camino en que puede (fallo al reservar la
página), que es un reinicio en frío al mismo sitio.

Tres mecanismos, para tres formas distintas de acabar:

| Mecanismo | Cubre | Dónde |
|---|---|---|
| `Sys_Quit` → `Xbox_ReturnToDashboard(reason)` | cualquier `Sys_Error` y la salida normal | `system.c` |
| **Vigilante**: 30 s sin imprimir nada → dashboard | el motor colgado, que es el caso de hoy | hilo en `xbox_trace.c` |
| **START+BACK** en el mando | el motor vivo, cuando aún no hay menú | `platform/sdl2/host_sdl2.c` |

Detalles que importan:

- El vigilante se alimenta de **cualquier** salida: tanto `Xbox_Trace()` como el console
  entero, que desde §20 pasa por `Xbox_TraceRaw()`. Así que sólo salta con silencio real.
  Se desarma al entrar en el bucle de frames (`host.c`), porque a partir de ahí callar es
  lo normal: un juego corriendo no imprime.
- Es un **hilo**, no un temporizador, precisamente para poder correr mientras el hilo
  principal está atascado.
- Antes de reiniciar deja **5 s** con el motivo en pantalla y en el log.
- **Y de paso distingue cuelgue de fallo duro**: si el vigilante salta y la consola
  vuelve, el hilo seguía vivo y era un bloqueo; si no vuelve, ni el vigilante sobrevivió
  y fue una excepción que se llevó el título entero. Con el `#UD` de §21.2 lo esperable
  habría sido lo segundo.

### 21.6 Assets: qué subir, y cuánto pesa

La pregunta era si el fallo era por falta de assets. **No lo era**: moría sin haber
tocado el filesystem. Pero hará falta en cuanto pase de `FS_Init`, así que queda medido.

La estructura va **al lado del XBE**, porque `D:` se monta sobre el directorio del XBE:

```
E:\Apps\xash\
├── default.xbe
├── valve\          <- obligatorio, es el gamedir base
└── gearbox\        <- después, para Opposing Force
```

| Tier | Contenido | Tamaño |
|---|---|---|
| **Mínimo para llegar al menú** | `valve/` con `liblist.gam`, `gameinfo.txt`, `gfx.wad`, `fonts.wad`, `cached.wad`, `decals.wad`, `delta.lst`, `gfx/`, `events/`, `sprites/` | **28 MB** |
| `valve/` sin `media/` ni `overviews/` (vídeos de intro, mapas de radar) | | 453 MB |
| `valve/` completo | | 544 MB |
| `gearbox/` completo | | 470 MB |

Empezar por los 28 MB: es lo justo para pasar el
`FS_FileExists( "gfx/conchars" )` de `Host_InitCommon` y ver si el menú levanta, y por
FTP a un Xbox se sube en un par de minutos en vez de veinte.

### 21.7 Qué NO está verificado

- **El arreglo del SSE2 no se ha ejecutado.** Lo que está demostrado es que el binario ya
  no contiene instrucciones que el CPU no entiende; que el arranque pase de ahí es la
  predicción, no un hecho. El siguiente log lo dirá.
- **Hay 418 objetos limpios de SSE2, no de todo lo demás.** El chequeo busca una lista de
  mnemónicos concreta. SSE1, MMX y x87 sí están, y son correctos para un Pentium III,
  pero nadie ha auditado el binario entero contra el manual del CPU.
- **La salida al dashboard no se ha probado.** `XLaunchXBE(NULL)`, el vigilante y
  START+BACK son código escrito y compilado. La primera vez que el XBE vuelva solo será
  la primera vez que se sepa.
- **Sigue sin ejecutarse nada del modo estático de §20.4** ni de `public/xbox_fd.c`
  (§19.5): el arranque no llegó a `FS_Init`.
- **La carga del gamedll va a fallar igual** (§20.7): no hay módulos `server` ni `client`.

### 21.8 Estado del entorno tras esta sesión

```
~/opposing-force-x/
├── xash-xbox-boot-trace.patch      7 ficheros del fork, +159/-5 (402 lineas)
└── research/
    ├── logs/xash-boot-*.log        NUEVO — los logs recogidos, con marca de tiempo
    ├── ofx-xbox-deploy.sh          IP y carpeta corregidas, aviso de DHCP
    ├── ofx-xbox-nosse2.sh          NUEVO — verifica 0 instrucciones SSE2
    ├── ofx-xbox-symbols.sh         recalcula STATIC_MODULE_RENAMES
    ├── ofx-xbox-verify.sh          build desde limpio + comprobaciones
    ├── ofx-pkgconf-shim.sh         el shim de 19.6a
    └── xash3d-fwgs_xbox/
        ├── engine/platform/xbox/xbox_trace.c   +vigilante y salida al dashboard
        ├── scripts/waifulib/xbox.py            +-mno-sse2 en CPPFLAGS
        ├── 3rdparty/opus/wscript               M — la fuga de -msse2
        └── build/engine/default.xbe            3.346.432 B, ya en la consola
```

19 ficheros nuestros, **2.625 líneas**. Siete ficheros del fork modificados, con el diff
aparte. `game/`, `game-win32/`, `game-offsets/`, `hlsdk-portable/` y el toolchain de §17:
intactos.

## 22. Paso 2f — El motor monta el filesystem en la consola, y los assets

**Resumen: el arreglo del SSE2 de §21 era el correcto. El arranque atraviesa entero
`Host_InitCommon` —memoria, consola, SDL2, filesystem— y muere en `FS_LoadGameInfo` con
`Couldn't find game directory 'valve'`, que es exactamente donde tenía que morir sin
assets. Y vuelve solo al dashboard. Subidos 262 ficheros, 30.984.611 bytes, verificados
uno a uno en destino.**

| | |
|---|---|
| Log | **31 líneas**, de 7 en §21 |
| Última fase alcanzada | `FS_LoadGameInfo` |
| Fallo | falta de assets, esperado |
| Vuelta al dashboard | ✅ automática, `caught an error` |
| Assets subidos | **262 ficheros, 30.984.611 B**, verificados |
| Instrumentación nueva | renderer (`ref_common.c`, `vid_xbox.c`) y menú (`cl_gameui.c`) |

### 22.1 El log, íntegro

```
[0000      447] trace up: log=D:\xash-boot.log fb=0x83eb4000 serial=no uart at 0x3F8
[0001      449] watchdog armed: back to the dashboard after 30 s of silence
[0002      450] crt initializers: alive before main()
[0003      451] main: entered, argc=0 argv=0xd0061c84 argv0=(null)
[0004      527] main: XVideoSetMode 640x480x32 done
[0005      528] main: Xbox_GetArgv -> argc=0 argv=0xd0061c84
[0006      529] main: calling Host_Main, gamedir=valve
[0007      530] Host_Main: enter
[0008      530] Host_InitCommon: enter, argc=0 basedir=valve
[0009      531] Host_InitCommon: Sys_ParseCommandLine ok
[0010      532] Host_InitCommon: exename=default
[0011      532] Host_InitCommon: Memory_Init
[0012      533] Host_InitCommon: Zone Engine pool ok
[0013      534] Host_InitCommon: Cmd/Cvar ok, Sys_InitLog
[0014      536] Host_InitCommon: Con_Init ok
[0015      536] Host_InitCommon: Platform_Init (SDL2 on this target)
[0016      537] Host_InitCommon: Platform_Init ok
[0017      538] Host_InitCommon: FS_Init
[0018      539] FS_Init: FS_DetermineRootDirectory (mounts D: on this target)
[0019      540] FS_Init: rootdir=D:\
[0020      554] FS_Init: rodir= gamedir=valve
[0021      554] FS_LoadProgs: COM_LoadLibrary(filesystem_stdio)
[0022      555] FS_LoadProgs: handle=0x339be8
[0023      556] FS_LoadProgs: filesystem_stdio ready
[0024      557] FS_Init: InitStdio
[0025      558] FS_Init: filesystem mounted
[0026      559] Host_InitCommon: FS_Init ok
[0027      559] Host_InitCommon: Image_Init / Sound_Init
[0028      560] Host_InitCommon: Image/Sound ok
[0029      561] Host_InitCommon: FS_LoadGameInfo
[0030      562] *** Sys_Error: Couldn't find game directory 'valve'

Couldn't find game directory 'valve'
[0031      567] returning to dashboard in 5 s: caught an error
```

**115 ms** desde `main` hasta el error. El arranque entero es instantáneo; lo que costaba
tiempo era el botón de encendido.

### 22.2 Lo que este log convierte en hecho

Hasta ahora §19, §20 y §21 acumulaban piezas escritas, compiladas y **nunca ejecutadas**.
Este log ejercita cuatro de ellas de golpe:

| Pieza | Línea | Qué demuestra |
|---|---|---|
| **Modo estático** (§20.4) | `FS_LoadProgs: handle=0x339be8` | `COM_LoadLibrary("filesystem_stdio")` devuelve un handle **no nulo**. `platform/misc/lib_static.c` encontró el módulo en `generated_library_tables.h`, y `COM_GetProcAddress` sacó `GetFSAPI` y `CreateInterface` de la tabla que genera `xbox.py`. El mecanismo funciona en la consola. |
| **`public/xbox_fd.c`** (§19.5) | `FS_Init: filesystem mounted` | `InitStdio` escanea directorios nada más arrancar: `_findfirst`/`_findnext`, `_open`, `_stat`. La capa de descriptores sobre la API Win32 de nxdk **abre ficheros de verdad**. |
| **`nxdk-compat/sys/stat.h`** (§19.4) | ídem | `_stat()` sobre `GetFileAttributesExA` responde bien; si no, `FS_SysFolderExists` habría dicho que no existe nada y el error habría sido otro. |
| **Salida al dashboard** (§21.5) | `returning to dashboard in 5 s: caught an error` | `Sys_Quit` → `Xbox_ReturnToDashboard` → `XLaunchXBE(NULL)`. La consola volvió sola, con la red en pie, y el log se recogió **sin tocar el botón**. |

También queda confirmado que el **vigilante se arma** (línea 0001) y que no llegó a
saltar, porque el error fue limpio y llegó antes de los 30 s.

Y una lectura por descarte que conviene dejar escrita: el `#UD` de §21.2 se llevaba el
título tan pronto que **todo lo demás estaba sin probar**. La primera ejecución que pasa
de la línea 7 valida de golpe tres sesiones de trabajo a ciegas.

### 22.3 Los assets

`FS_LoadGameInfo` recorre el directorio raíz buscando subdirectorios con `gameinfo.txt`
o `liblist.gam`. En `D:\` sólo estaba el XBE, así que no encontró ningún juego. Nada roto:
faltaba el contenido.

**Fuente: `~/opposing-force-x/game/valve`.** `game-win32/valve` sólo contiene
`extras.pk3` —el build de Windows apunta a los datos de otro sitio— y su directorio está
además lleno de ficheros `:Zone.Identifier`, los flujos alternativos de NTFS que Windows
pega a todo lo descargado. La copia de `game/` es la retail completa que validó §7.

**Subido** (`research/ofx-xbox-assets.sh`), 262 ficheros, **30.984.611 bytes**:

| | Tamaño | Por qué |
|---|---|---|
| `liblist.gam`, `gameinfo.txt` | 717 B | lo que `FS_LoadGameInfo` busca |
| `gfx.wad` | 103 KB | de aquí sale `gfx/conchars`, el recurso sin el cual el motor no sigue |
| `gfx/` | 27 MB | fondos del menú (`shell/`, 23 MB), `env/`, `vgui/` |
| `fonts.wad`, `cached.wad`, `decals.wad` | 1,1 MB | |
| `extras.pk3` | 1,6 MB | el paquete de extras de FWGS |
| `sound/`, `logos/`, `sprites/`, `events/`, `scripts/` | 840 KB | |
| `delta.lst`, `maps.lst`, `titles.txt`, `mapcycle.txt`, `profile.lst`, `rooms.lst` | 30 KB | |

**Y lo que se dejó fuera a propósito**, que importa tanto como lo que entra:

- **`config.cfg`, `autoexec.cfg`, `opengl.cfg`, `listenserver.cfg`, `server.cfg` y sus
  `.bak`.** Son los ajustes de la máquina de desarrollo: resolución, renderer,
  anisotropía. En una consola de 640x480 son mentira, y `Host_Main` ejecuta `config.cfg`
  durante el arranque. Que el motor escriba los suyos.
- **`.xash_id`, `.fontcache`, `SAVE/`, `custom.hpk`, `console_history.txt`**: basura de
  ejecución del PC.
- **`dlls/`, `cl_dlls/`**: los gamedlls de PC. Con `XASH_INTERNAL_GAMELIBS` el motor
  busca módulos `server`/`client` **enlazados**, no ficheros (§20.7); subirlos no serviría
  de nada.
- **`game.ico`** (404 KB) y su `game.ico:Zone.Identifier`.
- **`pak0.pak`** (299 MB), `halflife.wad` (37 MB), `maps/`, `models/`, `media/`,
  `overviews/`: el grueso. Para el menú no hace falta, y por FTP son veinte minutos.

**Verificación en destino**: el script lista recursivamente por FTP y compara contra el
árbol local, fichero a fichero.

```
local:    262 ficheros    30984611 bytes
remoto:   262 ficheros    30984611 bytes
OK: coinciden
```

### 22.4 Dos trampas del despliegue por FTP

Ninguna es del port, las dos costaron un intento y las dos volverán:

1. **`curl` se come el stdin del bucle.** La primera versión leía `find` en streaming y
   llamaba a `curl` dentro del `while read`. `curl` hereda ese stdin y se traga lo que
   queda: subió 67 de 262 ficheros y murió. Se arregla leyendo la lista entera a un array
   antes de empezar, y con `< /dev/null` en cada invocación.
2. **Half-Life tiene ficheros con espacios en el nombre.** `gfx/vgui/fonts/` está lleno de
   cosas como `320_Team Info Text.tga`, y un espacio crudo en una URL hace que `curl`
   conteste `URL using bad/illegal format or missing URL` — un mensaje que no menciona el
   fichero ni el espacio. FATX los admite sin problema; sólo hay que codificar la URL.
   El script lleva ahora su propio `urlenc`.

Se suben en lotes de 100 ficheros por invocación de `curl`, que reutiliza la conexión
dentro de cada una: los 30 MB tardan menos de un minuto.

### 22.5 Instrumentación de la fase siguiente

Si el motor pasa de `FS_LoadGameInfo`, lo que viene es `SV_Init` y `CL_Init`, y dentro de
`CL_Init` está lo interesante: cargar el renderer, levantar **pbgl** contra la GPU y fijar
el modo de vídeo. Trazas nuevas, para no tener que adivinar otra vez:

| Dónde | Traza |
|---|---|
| `engine/client/ref_common.c` | `R_LoadProgs`: nombre pedido, handle devuelto, antes y después del `R_Init` del renderer |
| `engine/platform/xbox/vid_xbox.c` | `R_Init_Video`: antes de `pbgl_init`, su código de error, `VID_SetMode`, `GL_InitExtensions` |
| `engine/client/cl_gameui.c` | `UI_LoadProgs`: nombre del módulo de menú y handle |

La de `vid_xbox.c` es la que más falta hace: pbgl habla con la GPU a través de pbkit, y
si el título muere ahí lo hace con el framebuffer en un estado desconocido, donde
`debugPrint` puede no sobrevivir. **El log en disco es el único testigo fiable a partir de
ese punto.**

### 22.6 Qué esperar en el próximo log

```
Host_InitCommon: FS_LoadGameInfo
Host_InitCommon: gamedir=valve dll_path=cl_dlls    <- ya encuentra el juego
Host_InitCommon: gfx/conchars found                <- gfx.wad resuelto desde el wad
Host_InitCommon: done
Host_Main: SV_Init
Host_Main: CL_Init (loads ref_gl and menu)
R_LoadProgs: COM_LoadLibrary(ref_gl)
R_LoadProgs: handle=0x...                          <- modo estatico otra vez
R_LoadProgs: GetRefAPI ok, calling the renderer's R_Init
R_Init_Video: pb_size 2 MB, pbgl_init              <- la GPU, por primera vez
R_Init_Video: pbgl up, VID_SetMode
R_Init_Video: done
UI_LoadProgs: COM_LoadLibrary(menu)
Host_Main: entering frame loop -- the engine is up  <- menu principal
```

Tres cosas concretas que mirar:

- **`gfx/conchars found`**: `FS_AddGameDirectory` mete los `.wad` del gamedir en la ruta
  de búsqueda, así que el lump debería salir de `gfx.wad` sin necesitar `pak0.pak`. Si
  falla ahí, la respuesta es subir `pak0.pak` (299 MB).
- **`R_Init_Video`**: es lo primero que toca la GPU en todo el port. `libpbgl.lib` se
  compiló en §17.2 y **nunca se ha ejecutado**.
- **Si el log se corta sin `Sys_Error`**, saltará el vigilante a los 30 s y la consola
  volverá sola; el log tendrá la última línea antes del cuelgue.

### 22.7 Qué NO está verificado

- **El renderer no se ha ejecutado nunca.** pbgl, `vid_xbox.c` y `ref_gl` enlazado
  estáticamente son, hoy, exactamente lo que era el filesystem antes de este log.
- **El menú tampoco.** mainui con `stb_truetype` (§19.4) sigue sin dibujar un píxel.
- **El conjunto de assets es una apuesta informada, no una lista comprobada.** Está
  medido para pasar `gfx/conchars` y alimentar al menú; si falta algo, saldrá en el log.
- **La carga del gamedll fallará igual** (§20.7): no hay módulos `server` ni `client`.
  Eso llega después del menú.
- **`xbox_sbrk.c` sigue sin entrar en el build** (§19.3).
- **START+BACK sigue sin probarse**: hasta ahora todas las salidas han sido por
  `Sys_Error`.

### 22.8 Estado del entorno tras esta sesión

```
~/opposing-force-x/
├── xash-xbox-boot-trace.patch      10 ficheros del fork, +209/-6 (539 lineas)
└── research/
    ├── stage/valve/                NUEVO — el arbol exacto que se subio, 262 ficheros
    ├── logs/                       los logs recogidos, con marca de tiempo
    ├── ofx-xbox-assets.sh          NUEVO — prepara, sube y verifica los assets
    ├── ofx-xbox-deploy.sh          sube el xbe, baja el log
    ├── ofx-xbox-nosse2.sh          verifica 0 instrucciones SSE2
    ├── ofx-xbox-symbols.sh         recalcula STATIC_MODULE_RENAMES
    ├── ofx-xbox-verify.sh          build desde limpio + comprobaciones
    └── ofx-pkgconf-shim.sh         el shim de 19.6a

En la consola, E:\Apps\xash\
├── default.xbe        3.346.432 B
├── xash-boot.log      el de arriba
└── valve\             262 ficheros, 30.984.611 B
```

19 ficheros nuestros, 2.625 líneas. Diez ficheros del fork modificados, con el diff
aparte. `game/`, `game-win32/`, `game-offsets/` y `hlsdk-portable/`: intactos — de
`game/valve` sólo se ha **copiado**, no se ha tocado nada.

## 23. Paso 2g — pbgl funciona: el renderer arranca en la consola

**Resumen: las cuatro trazas de `R_Init_Video` se escribieron. Todas. `pbgl_init` en
54 ms, `VID_SetMode` en 14, `GL_InitExtensions`, y `R_LoadProgs: renderer up`. La GPU
del Xbox está renderizando por primera vez en este port. El menú también carga. Lo que
falta ahora es la única pieza que siempre supimos que faltaba: el gamedll de cliente.**

| | |
|---|---|
| `pbgl_init` | ✅ **54 ms**, sin error |
| `VID_SetMode` | ✅ 14 ms |
| `GL_InitExtensions` | ✅ |
| `ref_gl` estático | ✅ `handle=0x339bd8` |
| `menu` estático | ✅ `handle=0x339c0c` |
| Fallo | `Host_ErrorInit: can't initialize client:` — **no existe el módulo `client`** |
| Arranque completo | 12,0 s, de los cuales **9,0 s son `NET_Init`** (§23.4) |

### 23.1 El log, de la línea 0026 en adelante

```
[0026      594] Host_InitCommon: FS_Init ok
[0027      595] Host_InitCommon: Image_Init / Sound_Init
[0028      597] Host_InitCommon: Image/Sound ok
[0029      597] Host_InitCommon: FS_LoadGameInfo
[0030      654] Host_InitCommon: gamedir=valve dll_path=cl_dlls
[0031      681] Host_InitCommon: gfx/conchars found
[0032     2665] Host_InitCommon: done
[0033     2666] Host_Main: Host_InitCommon returned
[0034    11680] Host_Main: SV_Init
[0035    11683] Host_Main: CL_Init (loads ref_gl and menu)
[0036    11688] R_LoadProgs: COM_LoadLibrary(ref_gl)
[0037    11688] R_LoadProgs: handle=0x339bd8
[0038    11689] R_LoadProgs: GetRefAPI ok, calling the renderer's R_Init
[0039    11691] R_Init_Video: pb_size 2 MB, pbgl_init
[0040    11745] R_Init_Video: pbgl up, VID_SetMode
[0041    11759] R_Init_Video: mode set, GL_InitExtensions
[0042    11762] R_Init_Video: done
[0043    11773] R_LoadProgs: renderer up
[0044    11774] UI_LoadProgs: COM_LoadLibrary(menu)
[0045    11775] UI_LoadProgs: handle=0x339c0c
[0046    11977] *** Sys_Error: Host_ErrorInit: can't initialize client:

Host_ErrorInit: can't initialize client:

[0047    12000] returning to dashboard in 5 s: caught an error
```

**Respuesta directa a la pregunta: se escribieron las cuatro trazas de `R_Init_Video`.**
No hubo código de error de `pbgl_init` porque no hubo error — esa traza sólo se emite en
el camino de fallo. Los assets también funcionaron: `gfx/conchars found` confirma que
`FS_AddGameDirectory` mete los `.wad` del gamedir en la ruta de búsqueda y que el lump
sale de `gfx.wad` **sin necesitar `pak0.pak`**, que era la duda de §22.6.

### 23.2 pbgl, que llevaba desde §17.2 sin ejecutarse

`libpbgl.lib` se compiló en §17.2 —con su `Makefile.standalone` y sus dos trampas de
`make`— y desde entonces habían pasado seis apartados sin que nadie la ejecutara. Ahora
sí:

```
[0039    11691] R_Init_Video: pb_size 2 MB, pbgl_init
[0040    11745] R_Init_Video: pbgl up, VID_SetMode      <- 54 ms
[0041    11759] R_Init_Video: mode set, GL_InitExtensions
```

`pbgl_init( GL_TRUE )` devolvió >= 0, `pb_size(2 MB)` para el pushbuffer se aceptó, el
modo se fijó y `GL_InitExtensions` —que interroga a la implementación GL por sus
extensiones— pasó sin caerse. Los 157 símbolos GL que §17.2 contó en `libpbgl.lib`
responden.

**Y esto explica el "amago de imagen y luego negro" que se vio en pantalla**, que era la
otra cosa que había que entender. `debugPrint` de nxdk dibuja sobre el framebuffer que
devuelve `XVideoGetFB()`. pbgl **no usa ese framebuffer**: reserva el suyo y programa el
CRTC directamente a través de pbkit, sin pasar por nxdk. Así que en el instante en que
`pbgl_init` toma la GPU:

- lo que se ve pasa a ser el framebuffer de pbgl, que está vacío — el destello;
- y **todas las trazas posteriores se escriben en un framebuffer que ya no se muestra**.

Era exactamente el riesgo que se anticipó en §22.5, y se confirma: **a partir de
`R_Init_Video`, la pantalla deja de ser un testigo y el log en disco es el único que
queda.** Sin `D:\xash-boot.log` esta sesión habría vuelto a ser una pantalla negra sin
explicación.

### 23.3 Dónde muere ahora, y por qué era inevitable

`engine/client/cl_main.c:3691`, dentro de `CL_Init`:

```c
COM_GetCommonLibraryPath( LIBRARY_CLIENT, libpath, sizeof( libpath ));

if( !CL_LoadProgs( libpath ))
	Host_Error( "can't initialize %s: %s\n", libpath, COM_GetLibraryError( ));
```

`COM_GetCommonLibraryPath( LIBRARY_CLIENT, ... )` acaba en
`COM_GenerateClientLibraryPath( "client", ... )` y, con `XASH_INTERNAL_GAMELIBS` puesto,
ese camino es literalmente `Q_strncpy( out, name, size )` (`lib_common.c:145`). Así que
`libpath` vale **`"client"`**, y el motor pide a `lib_static.c` un módulo llamado
`client` que **no está en `generated_library_tables.h`**: ahí sólo hay
`filesystem_stdio`, `ref_gl` y `menu` (§20.4).

El mensaje sale con la razón vacía —`can't initialize client:` y nada detrás— porque
`COM_GetLibraryError()` en modo estático no tiene nada que contar: `lib_static.c` no
registra errores, sólo busca en una tabla y devuelve `NULL`.

**Esto es exactamente lo que §20.7 anunció** cuando se activó `XASH_STATIC_LIBS`:

> `XASH_INTERNAL_GAMELIBS` hace que (…) el cliente y el menú se busquen como `"client"` y
> `"menu"`. El menú está en la tabla; **`server` y `client` no existen todavía**, así que
> la carga del gamedll fallará. Es lo esperado en este punto y saldrá en el log.

Salió en el log. No es un bug nuevo: es la frontera del trabajo hecho.

**Y no hay atajo.** En FWGS, `CL_LoadProgs` fallando es fatal: no existe un modo
«sólo menú». Para ver el menú principal hace falta que el módulo `client` exista, y eso
es el trabajo de §13.6 — meter `hlsdk-portable` dentro del árbol del motor —, que aquel
apartado estimó y dejó sin hacer a propósito. Con una diferencia importante a favor:
§18 ya demostró que **el gamedll de Op4 compila y enlaza entero con nxdk** (231/231
ficheros, `opfor.dll` y `client.dll` con la tabla de exports correcta), y §20.4 demostró
que el mecanismo de módulos estáticos funciona en la consola. Las dos mitades existen;
falta unirlas.

### 23.4 Nueve de los doce segundos son `NET_Init`

Un hallazgo colateral que importa para la velocidad de iteración, y que el log regala:

| Tramo | Tiempo |
|---|---|
| `main` → `Host_InitCommon: gfx/conchars found` | 0,68 s |
| `gfx/conchars` → `Host_InitCommon: done` | **1,98 s** |
| `Host_InitCommon returned` → `Host_Main: SV_Init` | **9,01 s** |
| `SV_Init` → error | 0,30 s |

Entre `Host_InitCommon returned` y `SV_Init`, `Host_Main` sólo hace registrar unos veinte
cvars y llamar a `Mod_Init()`, `NET_Init()`, `NET_InitMasters()` y `Netchan_Init()`.
Nueve segundos ahí son, casi con seguridad, **lwip levantando la interfaz `nforceif` y
negociando DHCP** — la consola está en DHCP, como quedó escrito en §21.1. Los 1,98 s del
tramo anterior son `Host_InitDecals`, `HPAK_Init`, `IN_Init` y `Key_Init`, con `IN_Init`
enumerando USB como sospechoso.

**No está medido, es inferencia por descarte**: no hay trazas dentro de ese tramo. Si en
algún momento el ciclo de prueba se vuelve molesto, ahí están los nueve segundos, y
`-noip` o equivalente sería lo primero que probar.

### 23.5 Lo que este log convierte en hecho

Tercera sesión seguida en que una ejecución valida piezas escritas a ciegas:

| Pieza | Desde | Ahora |
|---|---|---|
| `libpbgl.lib` | §17.2, compilada y nunca ejecutada | **la GPU responde** |
| `ref_gl` estático | §20.4 | `handle=0x339bd8`, `GetRefAPI` resuelto desde la tabla |
| `menu` estático | §20.4 | `handle=0x339c0c` |
| `vid_xbox.c` del autor del fork | heredado, nunca ejecutado | `R_Init_Video` entero |
| Los assets de §22.3 | subidos a ciegas | `gfx/conchars` sale de `gfx.wad`, sin `pak0.pak` |

### 23.6 Qué NO está verificado

- **Que pbgl *dibuje*.** Lo demostrado es que inicializa, fija modo y publica
  extensiones. Nadie ha visto un triángulo: el motor muere antes del primer frame.
- **Que el menú funcione.** `menu` carga; `GetMenuAPI` se resuelve. Pero `UI_LoadProgs`
  no llega a terminar su trabajo porque `CL_Init` aborta justo después.
- **Los nueve segundos de `NET_Init`** son inferencia por descarte (§23.4).
- **`xbox_sbrk.c`** sigue sin entrar en el build (§19.3).
- **START+BACK** sigue sin probarse: todas las salidas han sido por `Sys_Error`.
- **El presupuesto de memoria** sigue sin medirse, y ahora hay un pushbuffer de 2 MB y
  un framebuffer de pbgl encima del XBE de 3,3 MB.

### 23.7 Estado

Nada que cambiar en la consola: el `default.xbe` y los 262 ficheros de `valve/` de §22
siguen siendo los correctos. Esta sesión no ha tocado código; sólo ha leído el log que
la instrumentación de §22.5 puso ahí.

El siguiente bloqueo tiene nombre y apartado propio desde hace tiempo: **§13.6**, meter
`client` (y después `server`) dentro del XBE. Es lo único que separa este arranque del
menú principal.

## 24. Paso 2h — El gamedll de Half-Life dentro del XBE

**Resumen: `server` y `client` de `hlsdk-portable` están enlazados dentro del
`default.xbe`, con sus 499 y 63 puntos de entrada en las tablas del cargador estático.
El binario pasa de 3.346.432 a 4.608.000 bytes. Y por el camino: una línea de órdenes
para una máquina que no tenía ninguna, y el mismo bug de SSE2 de §21 esperando en el
otro proyecto.**

| | |
|---|---|
| `server` | 107 objetos, **499 puntos de entrada** |
| `client` | 67 objetos, **63 puntos de entrada** |
| Choques renombrados | 1 en el servidor, **838** en el cliente |
| Duplicados al enlazar | **0** |
| SSE2 en el gamedll | **2892 instrucciones en 118 objetos** → 0 (§24.7) |
| `default.xbe` | 3.346.432 → **4.608.000 B** |

**Nadie lo ha ejecutado todavía.** Está subido a la consola junto con `xash.cmd`.

### 24.0 Por qué Half-Life base y no Opposing Force

El plan es el propuesto, y por la razón correcta: los assets de `valve/` ya están en la
consola y validados desde §22, así que si algo falla **es del mecanismo**, no de Op4 ni
de sus datos. Añade dos cosas más a favor: §18 midió el censo de compilación sobre la
rama `opfor`, que es el caso *más* difícil de los dos, y `hlsdk-portable` master es la
rama que upstream mantiene. Cambiar a `opfor` después es recorrer el mismo camino con
otra rama.

El trabajo se hizo en un **worktree de git** (`~/opposing-force-x/hlsdk-hl`, rama
`master`), no cambiando de rama en el árbol existente: así el `opfor` de §18 queda
intacto y los dos comparten almacén de objetos.

### 24.1 El muro de §13.6, y por dónde se rodea

§13.6 dejó esto escrito como el bloqueo estructural:

> `xshlib.py` solo puede convertir targets que sean *task generators de la misma
> ejecución de waf*. hlsdk-portable es un proyecto waf **separado**, así que sus targets
> `server` y `client` no existen dentro del build del motor.

Y estimó el trabajo en «reimplementar el sistema de build de hlsdk dentro del motor»,
60-100 líneas de wscript por módulo más mantenerlo sincronizado.

**No hace falta.** Esa conclusión venía de asumir el mecanismo de FWGS, que necesita
task generators porque hace `ld -r` sobre ellos. El de §20.4 no: lo único que el motor
necesita de un módulo estático es

1. sus objetos, en algún sitio del enlace, y
2. una tabla `{nombre, puntero}` que `platform/misc/lib_static.c` pueda recorrer.

Ninguna de las dos exige que el módulo sea un target de este waf. Así que el gamedll se
compila **con su propio waf**, como en §18, y llega aquí ya archivado:

```
hlsdk-hl/  --waf-->  build-nxdk/**/*.o
                          |
      research/ofx-hlsdk-package.sh
                          |
          gamelibs-hl/{server,client}.lib
          gamelibs-hl/{server,client}.exports
                          |
   ./waf configure --xbox --xbox-gamelibs=gamelibs-hl
                          |
                    default.xbe
```

`xbox.py` solo tiene que poner los `.lib` en la línea de enlace y generar las tablas.
Son unas 60 líneas de Python, no dos wscripts que mantener sincronizados con hlsdk.

### 24.2 El manifiesto: el nombre que se pide no es el símbolo que hay

Ésta es la pieza que hacía falta pensar. El motor busca sus puntos de entrada **por
nombre de export**:

```c
COM_GetProcAddress( svgame.hInstance, "GetEntityAPI" );
COM_GetProcAddress( clgame.hInstance, cdll_exports[i].name );   // "HUD_Init", ...
```

y ese nombre **no es** el símbolo del objeto. En un PE i386 hay tres decoraciones
distintas conviviendo en el mismo gamedll:

| Export | Símbolo | Por qué |
|---|---|---|
| `GetEntityAPI` | `_GetEntityAPI` | C, cdecl |
| `GiveFnptrsToDll` | `_GiveFnptrsToDll@8` | C, `__stdcall` (WINAPI) |
| `ActiveThink` | `?ActiveThink@CBaseTurret@@QAEXXZ` | método C++, mangling de MSVC |

Con DLLs eso no importa: la tabla de exports del PE hace de puente. Enlazando estático
el puente hay que construirlo, y desde C no se puede escribir `?ActiveThink@...` como
identificador.

La salida son las **etiquetas asm**, que clang emite tal cual, sin decoración propia:

```c
extern void s0_( void ) __asm__( "?ActiveThink@CBaseTurret@@QAEXXZ" );
extern void s1_( void ) __asm__( "_GiveFnptrsToDll@8" );

struct { const char *name; void *func; } lib_server_exports[] = {
	{ "ActiveThink",     (void *)&s0_ },
	{ "GiveFnptrsToDll", (void *)&s1_ },
	...
};
```

Comprobado antes de construir nada encima: los tres casos generan exactamente el
símbolo indefinido esperado y los tres resuelven contra `server.lib`.

El emparejamiento nombre→símbolo lo hace `ofx-hlsdk-package.sh`, leyendo la **tabla de
exports de la DLL** que ya produce el build de §18 —`llvm-readobj --coff-exports`— y
resolviendo cada nombre contra los símbolos realmente definidos: si empieza por `?` es
C++ y el nombre ya es el símbolo; si no, prueba `_nombre` y luego `_nombre@0`, `@4`,
`@8`… Cada candidato se verifica, y lo que no resuelve se dice en voz alta.

Resultado: **499 de 501** en el servidor y **63 de 64** en el cliente. Lo que falta es
un alias duplicado (`_GiveFnptrsToDll@8` aparece además como nombre de export por sí
mismo, cortesía del `/export:` de §18.4) y una entrada del cliente. Ninguno es un punto
de entrada del motor.

> Los 499 no son adorno. `GetEntityAPI` y compañía son cuatro; el resto son las
> funciones marcadas con `EXPORT` en hlsdk, y `sv_game.c:1080` las usa para resolver
> **el nombre de clase de cada entidad** del `.bsp` con `COM_GetProcAddress`. Sin ellas
> no se instancia ni un `func_door`.

### 24.3 Los choques, medidos

Igual que en §20.5, pero con tres módulos en juego en vez de dos:

| | Símbolos globales | Choques reales |
|---|---|---|
| `server` vs el XBE | 9.367 | **1** (`_VectorAngles`) |
| `client` vs XBE + `server` | 2.244 | **838** |
| `server` vs `client`, en bruto | | 582 «coincidencias», de las que lld sólo protesta por ~20 |

Lo interesante es la última fila. `comm` decía 582 coincidencias entre servidor y
cliente, pero el enlazador sólo consideró error una veintena: el resto son **vtables de
C++ y literales COMDAT**, que funde en silencio. Igual que en §20.5, la diferencia entre
«coincide» y «choca de verdad» es enorme, y quien la decide es el enlazador, no `nm`.

Se renombra en el **cliente**, que es el que compila una copia del código de armas del
servidor para la predicción — con DLLs separadas cada uno llamaba a la suya, y renombrar
conserva exactamente ese reparto. Y se renombra el conjunto entero, COMDAT incluidos: el
enlazador los habría fundido, así que renombrarlos sólo cuesta unos kilobytes de
duplicado, y a cambio no hay que averiguar cuáles lo son — que en COFF `llvm-nm` no lo
dice.

Duplicados al enlazar tras el renombrado: **0**.

### 24.4 Una línea de órdenes para una máquina que no tiene ninguna

Todos los logs desde §21 dicen lo mismo: `argc=0 argv0=(null)`. nxdk llama a
`main(0, &argv)`; un título lanzado desde un dashboard no recibe argumentos, nunca. Lo
que significa que **ninguna opción del motor era alcanzable**: ni `-dev`, ni `-console`,
ni `-noip`, ni `-game`. Cada una costaba recompilar y subir 4 MB.

`engine/platform/xbox/xbox_cmdline.c` (nuestro) lee **`D:\xash.cmd`** antes de
`Host_Main` y lo añade a `argv`. Una línea, separada por espacios, comillas dobles para
agrupar, sin escapes: existe para escribir `-noip -dev 2`, no para ser un shell. `D:` es
el directorio del XBE, así que el fichero se edita por FTP en un segundo, junto al
`default.xbe`.

Las trazas dicen qué se leyó, para que no haya dudas de si el fichero llegó:

```
cmdline: D:\xash.cmd gave 1 argument
cmdline:   argv[1] = -noip
```

Es la respuesta al punto 3 del encargo —`-noip` configurable— pero generaliza a
cualquier switch, que es lo que hace falta si se va a iterar mucho.

### 24.5 Los nueve segundos, ahora medibles

§23.4 dedujo por descarte que 9 de los 12 segundos de arranque estaban en `NET_Init`,
sin trazas dentro del tramo. Ahora las hay: `Mod_Init`, `NET_Init` con entrada y salida,
`NET_InitMasters`, `Netchan_Init`, más un corte extra en `Host_InitCommon` antes de
`IN_Init`. Con `-noip` en `xash.cmd` el próximo log dice las dos cosas a la vez: dónde
están los segundos y si quitarlos los devuelve.

**Sigue sin medirse.** Está subido, no ejecutado.

### 24.6 Tres trampas del empaquetado

Ninguna es del port; las tres son de cómo waf deja los objetos, y las tres darían un
binario silenciosamente equivocado en vez de un error:

1. **waf compila una docena de fuentes de `dlls/` dos veces.** Una para el servidor y
   otra para el cliente, que necesita el código de armas para la predicción. Los dos
   juegos conviven en `build-nxdk/dlls/` y sólo los distingue el índice del task
   generator (`crowbar.cpp.1.o` y `crowbar.cpp.2.o`). El primer archivo del servidor se
   llevó los dos y el enlazador reportó `CCrowbar::Spawn` duplicado **consigo mismo**.
   El índice no se supone: `cl_dll/` sólo lo compila el cliente, así que el índice que
   domina ahí es el suyo.
2. **Faltaban directorios enteros.** `pm_shared/`, `external/openbsd/` y `public/`
   también entran en el enlace de la DLL. Dejarlos fuera **no da un error**: sus
   símbolos se resuelven en silencio contra los del motor, que tiene su propia copia de
   `pm_shared`. El gamedll habría ejecutado código que no es el suyo, y nadie se habría
   enterado hasta ver físicas raras.
3. **El conjunto «lo que ya está en el XBE» no son sólo los objetos del motor.** Son
   también sus librerías estáticas. `_VectorAngles` vive en `public.lib`, y mirando sólo
   `engine/*.o` el choque no aparecía hasta el enlace.

### 24.7 SSE2 otra vez, en el otro proyecto

Antes de empaquetar nada, la comprobación de §21 sobre el gamedll recién compilado:

```
objetos con SSE2: 118 de 176
instrucciones:    2892
```

`scripts/waifulib/compiler_optimizations.py` de hlsdk añade `-march=pentium-m
-mtune=core2` para cualquier x86 de 32 bits, «porque HLSDK compila con estas opciones
bajo Linux». **Un Pentium M sí tiene SSE2**; el Coppermine de la consola no. Es
exactamente el bug de §21.2 esperando en el segundo proyecto, y habría matado el
arranque en la primera operación en coma flotante del código de juego — probablemente
en cuanto se cargara un mapa, con un log que apuntaría a cualquier otro sitio.

Arreglado igual: `-march=pentium3 -mno-sse2` en **`CPPFLAGS`**, que la orden de
compilación emite después de `CFLAGS`. Verificado: **0 instrucciones SSE2 en 176
objetos**, y 0 en los 420 del motor.

La lección, tercera vez que aparece: **en un build donde varios sitios tocan `CFLAGS`,
el orden es parte de la semántica**, y ni FWGS ni hlsdk lo tratan como tal.

### 24.8 El binario

```
build/engine/default.xbe   4.608.000 B   (era 3.346.432)
gamelibs-hl/server.lib    13.705.122 B   107 objetos, 499 entradas
gamelibs-hl/client.lib     2.652.756 B    67 objetos,  63 entradas
```

Verificado sobre `xash.map` que los dos módulos están dentro y de dónde vienen:

```
_GetEntityAPI          server:cbase.cpp.1.o
_HUD_Init              client:cdll_int.cpp.2.o
_lib_server_exports    xbox_gamelib_tables.c.2.o
_lib_client_exports    xbox_gamelib_tables.c.2.o
```

y que `generated_library_tables.h` lista los cinco módulos: `filesystem_stdio`,
`ref_gl`, `menu`, `server`, `client`.

Reproducir de cero:

```bash
# 1. el gamedll de HL base
bash ~/opposing-force-x/research/ofx-hlsdk-setup.sh      # worktree de master + parche nxdk
bash ~/opposing-force-x/research/ofx-hlsdk-build.sh reconf
bash ~/opposing-force-x/research/ofx-hlsdk-package.sh    # .lib + manifiestos + renombrados

# 2. el motor
export NXDK_DIR=/opt/toolchains/xbox/nxdk
export PATH="$NXDK_DIR/bin:$PATH"
cd ~/opposing-force-x/research/xash3d-fwgs_xbox
./waf configure --xbox --xbe-title 'Opposing Force X' \
                --xbox-gamelibs=$HOME/opposing-force-x/gamelibs-hl
./waf build -j8
```

> `ofx-hlsdk-package.sh` lee los símbolos del build del motor para calcular los choques,
> así que el orden es: construir el motor una vez (aunque falle el enlace), empaquetar, y
> volver a construir. Se avisa en el propio script.

Sin `--xbox-gamelibs` el motor sigue construyéndose igual que en §23: arranca, monta el
filesystem, levanta pbgl y muere en `CL_Init`.

### 24.9 Qué NO está verificado

- **Nada de esto se ha ejecutado.** Lo demostrado es que enlaza, que los símbolos están
  donde deben y que no hay instrucciones que el CPU no entienda. El menú principal sigue
  siendo la hipótesis de la ronda.
- **Los 838 renombrados del cliente no se han ejercitado.** Renombrar definiciones y
  referencias a la vez es correcto por construcción, pero la predicción del cliente es
  justo el código que se ha tocado, y no se prueba hasta que haya un mapa cargado.
- **`-noip` está subido, no medido** (§24.5).
- **`xash.cmd` no se ha leído nunca**: si el fichero no se abre, el arranque sigue igual
  y sólo faltará la traza `cmdline:`.
- **El presupuesto de memoria empeora otra vez**: 4,6 MB de XBE, más el pushbuffer de
  2 MB de pbgl, más el heap del juego, en una consola de 128 MB. Sigue sin medirse.
- **Half-Life, no Opposing Force.** Cuando esto arranque, cambiar de rama es repetir
  §24.1-24.3 con `opfor` — y ahí sí aplican los avisos de §18 sobre su censo.

### 24.10 Estado del entorno

```
~/opposing-force-x/
├── hlsdk-hl/                       NUEVO — worktree de master con soporte nxdk
├── hlsdk-hl-nxdk.patch             NUEVO — 4 ficheros, +183/-5 (316 lineas)
├── gamelibs-hl/                    NUEVO — server.lib, client.lib y sus manifiestos
├── xash-xbox-boot-trace.patch      10 ficheros del motor, +226/-6 (567 lineas)
└── research/
    ├── ofx-hlsdk-setup.sh          NUEVO — worktree + parche
    ├── ofx-hlsdk-build.sh          NUEVO — compila HL base con nxdk
    ├── ofx-hlsdk-package.sh        NUEVO — .lib, manifiestos y renombrados
    ├── ofx-xbox-assets.sh          prepara, sube y verifica los assets
    ├── ofx-xbox-deploy.sh          sube el xbe, baja el log
    ├── ofx-xbox-nosse2.sh          verifica 0 instrucciones SSE2
    ├── ofx-xbox-symbols.sh         recalcula STATIC_MODULE_RENAMES
    ├── ofx-xbox-verify.sh          build desde limpio
    └── ofx-pkgconf-shim.sh         el shim de 19.6a

En la consola, E:\Apps\xash\
├── default.xbe        4.608.000 B
├── xash.cmd           "-noip"
├── xash-boot.log      el de la seccion 23
└── valve\             262 ficheros, 30.984.611 B
```

20 ficheros nuestros en el motor, **2.879 líneas**. Diez ficheros del fork modificados
más uno de hlsdk, los dos con su diff aparte. `hlsdk-portable/` (rama `opfor`, §18),
`game/`, `game-win32/` y `game-offsets/`: intactos.

## 25. Paso 2i — `SV_Init`: el gamedll se toca por primera vez

**Resumen: el arranque llega a `Host_Main: SV_Init` y ahí se acaba el log. No hay nada
después, ni siquiera la línea del vigilante, así que no fue un cuelgue: el fallo se
llevó el título entero. `xash.cmd` sí se lee —está en el log— pero `-noip` no servía
para nada porque `nxNetInit` se llamaba **antes** de mirarlo. Las dos cosas arregladas e
instrumentadas.**

| | |
|---|---|
| Log | **43 líneas**, última: `Host_Main: SV_Init` |
| ¿Volvió sola a los 30 s? | **No.** Ni línea del vigilante (§25.3) |
| `xash.cmd` | ✅ leído: `argc=2`, `argv[1] = -noip` |
| `-noip` | ❌ sin efecto: `NET_Init` seguía tardando **9,01 s** (§25.2) |
| Trazas nuevas | `SV_Init`, `SV_InitGame`, `SV_LoadProgs` paso a paso (§25.4) |

### 25.1 El log, íntegro

```
[0000     2612] trace up: log=D:\xash-boot.log fb=0x83eb4000 serial=no uart at 0x3F8
[0001     2613] watchdog armed: back to the dashboard after 30 s of silence
[0002     2614] crt initializers: alive before main()
[0003     2615] main: entered, argc=0 argv=0xd0061c84 argv0=(null)
[0004     2691] main: XVideoSetMode 640x480x32 done
[0005     2692] main: Xbox_GetArgv -> argc=0 argv=0xd0061c84
[0006     2693] cmdline: D:\xash.cmd gave 1 argument
[0007     2694] cmdline:   argv[1] = -noip
[0008     2694] main: calling Host_Main, gamedir=valve
[0009     2695] Host_Main: enter
[0010     2696] Host_InitCommon: enter, argc=2 basedir=valve
[0011     2697] Host_InitCommon: Sys_ParseCommandLine ok
[0012     2697] Host_InitCommon: exename=default
[0013     2698] Host_InitCommon: Memory_Init
[0014     2699] Host_InitCommon: Zone Engine pool ok
[0015     2699] Host_InitCommon: Cmd/Cvar ok, Sys_InitLog
[0016     2701] Host_InitCommon: Con_Init ok
[0017     2702] Host_InitCommon: Platform_Init (SDL2 on this target)
[0018     2703] Host_InitCommon: Platform_Init ok
[0019     2704] Host_InitCommon: FS_Init
[0020     2704] FS_Init: FS_DetermineRootDirectory (mounts D: on this target)
[0021     2719] FS_Init: rootdir=D:\
[0022     2719] FS_Init: rodir= gamedir=valve
[0023     2720] FS_LoadProgs: COM_LoadLibrary(filesystem_stdio)
[0024     2721] FS_LoadProgs: handle=0x466b34
[0025     2722] FS_LoadProgs: filesystem_stdio ready
[0026     2722] FS_Init: InitStdio
[0027     2749] FS_Init: filesystem mounted
[0028     2750] Host_InitCommon: FS_Init ok
[0029     2750] Host_InitCommon: Image_Init / Sound_Init
[0030     2751] Host_InitCommon: Image/Sound ok
[0031     2752] Host_InitCommon: FS_LoadGameInfo
[0032     2799] Host_InitCommon: gamedir=valve dll_path=cl_dlls
[0033     2826] Host_InitCommon: gfx/conchars found
[0034     2841] Host_InitCommon: decals/HPAK ok, IN_Init
[0035     4812] Host_InitCommon: done
[0036     4813] Host_Main: Host_InitCommon returned
[0037     4814] Host_Main: Mod_Init
[0038     4814] Host_Main: NET_Init
[0039    13828] Host_Main: NET_Init ok
[0040    13828] Host_Main: NET_InitMasters ok
[0041    13829] Host_Main: Netchan_Init ok
[0042    13843] Host_Main: SV_Init
```

Y **nada más**. `SV_Init` entró y no salió.

Un dato que el log regala de paso: los 1,97 s entre `decals/HPAK ok, IN_Init` y
`Host_InitCommon: done` son **`IN_Init`**, es decir SDL2 enumerando el USB. Confirmado el
otro sospechoso de §23.4.

### 25.2 `xash.cmd` sí se lee. `-noip` no servía

Las dos preguntas del encargo, con respuesta separada porque son cosas distintas.

**El fichero se lee.** Líneas 0006 y 0007: `cmdline: D:\xash.cmd gave 1 argument` y
`argv[1] = -noip`. Y la prueba de que llega al motor está en la 0010:
`Host_InitCommon: enter, argc=2` — antes era `argc=0`. El mecanismo de §24.4 funciona.

**`-noip` no hacía nada, y la culpa es del orden.** `engine/common/net_ws.c`, en
`NET_Init`:

```c
#elif XASH_XBOX
	nx_net_parameters_t params = { 0 };
	params.ipv4_mode = NX_NET_DHCP;      // <- espera una concesion DHCP
	nxNetInit( &params );
#endif

	NET_InitializeCriticalSections();

	net.allow_ip = !Sys_CheckParm( "-noip" );   // <- 20 lineas mas abajo
```

`nxNetInit` con `NX_NET_DHCP` se llama **incondicionalmente y antes** de mirar `-noip`.
Lo que ese switch controla es si se abren sockets IP, no si se levanta la pila. Los 9 s
estaban en la pila.

Confirmado además el número exacto: `NET_Init` a 4814, `NET_Init ok` a 13828,
**9,01 s**, idéntico a la medición por descarte de §23.4. La inferencia era correcta; lo
que faltaba era la palanca.

Ahora, en este target, `-noip` significa «no levantes la pila», que es la única lectura
que ahorra la espera. Con traza propia, para que el próximo log diga cuál de los dos
caminos tomó.

> Nota de honestidad: esto cambia ligeramente el significado de `-noip` respecto a las
> demás plataformas, donde la pila del sistema ya está levantada y el switch sólo evita
> abrir sockets. Aquí no hay pila hasta que alguien la levanta.

### 25.3 El vigilante no saltó, y eso también es información

`SV_Init` es la última línea y **no hay** `watchdog: 30 s with no output`. El vigilante
estaba armado —línea 0001— y su hilo debería haberse despertado cada segundo. Que no lo
hiciera dice que **no fue un bloqueo del hilo principal**: si el hilo principal se
atasca, el vigilante sigue vivo, escribe y devuelve la consola al dashboard. Eso es
precisamente lo que §21.5 dejó escrito que distinguiría un caso del otro:

> Y de paso distingue cuelgue de fallo duro: si el vigilante salta y la consola vuelve,
> el hilo seguía vivo y era un bloqueo; si no vuelve, ni el vigilante sobrevivió y fue
> una excepción que se llevó el título entero.

No volvió. **Fue una excepción**, y el kernel del Xbox se llevó por delante todos los
hilos, el vigilante incluido. Un acceso a memoria inválido en el código de juego encaja;
un `#UD` ya no, porque el binario está limpio de SSE2 (verificado otra vez, 0 en 420
objetos).

No se ha intentado instalar un manejador de última oportunidad. En i386 eso significa
SEH, que clang no genera de forma fiable para este target, y montarlo sería un proyecto
en sí mismo. La vía barata es la de siempre: **estrechar el cerco con trazas**.

### 25.4 Instrumentado el sitio exacto

`SV_Init` son ~150 líneas de `Cvar_RegisterVariable` y luego, al final, tres llamadas.
La tercera es la que importa:

```c
	SV_InitFilter();
	SV_ClearGameState();                                  // borra los *.hl temporales
	SV_InitGame( GI->gamemode != GAME_SINGLEPLAYER_ONLY ); // <- carga el gamedll
```

`SV_InitGame` → `SV_LoadProgs("server")` → y ahí dentro está **la primera instrucción de
código de juego que ejecuta la consola en toda la historia de este port**. Trazas nuevas,
una por paso:

| Fase | Traza |
|---|---|
| `SV_Init` | antes de `SV_InitFilter`, `SV_ClearGameState`, `SV_InitGame`, y al salir |
| `SV_InitGame` | el nombre con el que pide el módulo (`"server"`, por `XASH_INTERNAL_GAMELIBS`) |
| `SV_LoadProgs` | `COM_LoadLibrary` y el handle; las direcciones de `GetEntityAPI`, `GetEntityAPI2`, `GetNewDLLFunctions` y `GiveFnptrsToDll` resueltas desde la tabla |
| | **antes y después de `GiveFnptrsToDll`** — la primera llamada al gamedll |
| | antes de `GetEntityAPI2`, y al terminar la API de entidades |
| | la reserva de `max_edicts` |
| | antes de `pfnGetGameDescription`, de `pfnGameInit` y de `pfnRegisterEncoders` |

Las direcciones importan tanto como el orden: si `GiveFnptrsToDll` sale `0x0`, el fallo
es de las tablas generadas en §24.2; si sale un puntero y la llamada no vuelve, el fallo
está dentro del gamedll y las tablas son correctas.

Los tres `pfn*` del final son las llamadas que más pueden doler, porque van por punteros
que el propio gamedll acaba de rellenar en `svgame.dllFuncs`.

### 25.5 Qué esperar en el próximo log

```
Host_Main: SV_Init
SV_Init: cvars done, SV_InitFilter
SV_Init: SV_ClearGameState
SV_Init: SV_InitGame (this loads the server game dll)
SV_InitGame: SV_LoadProgs(server)
SV_LoadProgs: COM_LoadLibrary(server)
SV_LoadProgs: handle=0x...                <- distinto de 0 = las tablas de 24.2 valen
SV_LoadProgs: entry points resolved (API=0x... API2=0x... New=0x0 Give=0x...)
SV_LoadProgs: calling GiveFnptrsToDll     <- primer codigo de juego en la consola
SV_LoadProgs: GiveFnptrsToDll returned
SV_LoadProgs: calling GetEntityAPI2
SV_LoadProgs: entity API ok
SV_LoadProgs: allocating 1200 edicts
SV_LoadProgs: calling pfnGetGameDescription
SV_LoadProgs: calling pfnGameInit
SV_LoadProgs: pfnGameInit returned, SV_InitClientMove
SV_LoadProgs: calling pfnRegisterEncoders
SV_LoadProgs: server game dll is up
SV_Init: done
Host_Main: CL_Init (loads ref_gl and menu)
```

`New=0x0` es lo esperado: Half-Life base no exporta `GetNewDLLFunctions`, y el motor lo
trata como opcional (§24.2).

Y con `-noip` ya efectivo, el arranque debería bajar de ~14 s a **unos 5**.

### 25.6 Qué NO está verificado

- **Que el arreglo de `-noip` funcione.** Está subido, no ejecutado. Si `nxNetInit` no
  era el culpable, el próximo log lo dirá con su propia traza.
- **La causa del fallo en `SV_Init` sigue sin conocerse.** Lo único establecido es que no
  fue un bloqueo, porque el vigilante no llegó a correr.
- **Las tablas del gamedll de §24 no se han ejercitado todavía**: el log se corta antes
  de `COM_LoadLibrary("server")`.
- **Los 838 renombrados del cliente** siguen sin ejecutarse, y ahora tampoco los del
  servidor.
- **El presupuesto de memoria** sigue sin medirse, y `SV_LoadProgs` reserva
  `sizeof(edict_t) * max_edicts` de golpe — 1200 edicts según el `gameinfo.txt` de
  `valve/`. Es la primera reserva grande del arranque y un sospechoso razonable si el
  fallo resulta ser de memoria.

### 25.7 Estado

`default.xbe` 4.612.096 B, subido. `xash.cmd` sigue con `-noip`, que ahora sí hace algo.
0 instrucciones SSE2 en 420 objetos, y el build de Linux sigue saliendo entero.

Ficheros del fork modificados: 14, con el diff aparte en
`~/opposing-force-x/xash-xbox-boot-trace.patch`. Los cuatro nuevos de esta sesión
—`net_ws.c`, `sv_main.c`, `sv_init.c`, `sv_game.c`— son todos trazas salvo el arreglo de
`-noip`.

## 26. Paso 2j — El gamedll de servidor corre en la consola

**Resumen: `server` arrancó entero. Las cuatro direcciones se resolvieron desde la tabla
generada, `GiveFnptrsToDll` y `GetEntityAPI2` volvieron, se reservaron 1200 edicts,
`pfnGameInit` y `pfnRegisterEncoders` corrieron. Ese es todo el mecanismo de §24
funcionando en hardware. Y el arranque bajó de 13,8 s a 2,8 s con `-noip` ya efectivo.**

| | |
|---|---|
| Log | **61 líneas**, última: `SV_LoadProgs: server game dll is up` |
| ¿Volvió sola? | **No.** Sin línea del vigilante: excepción dura otra vez (§26.2) |
| `-noip` | ✅ `NET_Init` de **9.010 ms a 1 ms** |
| Arranque completo | 13,8 s → **2,8 s** |
| Trazas nuevas | `CL_Init`, `CL_LoadProgs` export por export (§26.3) |

### 26.1 El log, de la línea 0036 en adelante

```
[0036     2734] Host_Main: Host_InitCommon returned
[0037     2735] Host_Main: Mod_Init
[0038     2735] Host_Main: NET_Init
[0039     2736] NET_Init: -noip, skipping nxNetInit (no DHCP wait)
[0040     2737] Host_Main: NET_Init ok
[0041     2751] Host_Main: NET_InitMasters ok
[0042     2752] Host_Main: Netchan_Init ok
[0043     2753] Host_Main: SV_Init
[0044     2755] SV_Init: cvars done, SV_InitFilter
[0045     2756] SV_Init: SV_ClearGameState
[0046     2757] SV_Init: SV_InitGame (this loads the server game dll)
[0047     2758] SV_InitGame: SV_LoadProgs(server)
[0048     2758] SV_LoadProgs: COM_LoadLibrary(server)
[0049     2759] SV_LoadProgs: handle=0x45dd78
[0050     2760] SV_LoadProgs: entry points resolved (API=0x1799f0 API2=0x179a20 New=0 Give=0x17aec0)
[0051     2761] SV_LoadProgs: calling GiveFnptrsToDll
[0052     2762] SV_LoadProgs: GiveFnptrsToDll returned
[0053     2762] SV_LoadProgs: calling GetEntityAPI2
[0054     2763] SV_LoadProgs: entity API ok
[0055     2764] SV_LoadProgs: allocating 1200 edicts
[0056     2784] SV_LoadProgs: calling pfnGetGameDescription
[0057     2785] SV_LoadProgs: calling pfnGameInit
[0058     2791] SV_LoadProgs: pfnGameInit returned, SV_InitClientMove
[0059     2799] SV_LoadProgs: calling pfnRegisterEncoders
[0060     2800] SV_LoadProgs: server game dll is up
```

Y ahí acaba.

**Lo que esas 13 líneas demuestran**, que es mucho más de lo que parece:

- **`handle=0x45dd78`** — `lib_static.c` encontró `server` en
  `generated_library_tables.h`. La dirección es la de la tabla, en `.data`.
- **`API=0x1799f0 API2=0x179a20 New=0 Give=0x17aec0`** — las etiquetas asm de §24.2
  funcionan. Los tres punteros no nulos apuntan a código del gamedll dentro del XBE, y
  `New=0` es lo previsto: Half-Life base no exporta `GetNewDLLFunctions` y el motor lo
  trata como opcional. **El puente entre nombre de export y símbolo decorado es
  correcto**, incluido `_GiveFnptrsToDll@8`, que es el caso `__stdcall`.
- **`GiveFnptrsToDll returned`** — la primera instrucción de código de juego que ha
  ejecutado esta consola, y volvió.
- **1200 edicts en 20 ms** — la reserva grande que §25.6 señalaba como sospechosa de
  memoria pasó sin problema.
- **`pfnGameInit returned`** — el `GameDLLInit()` de hlsdk entero: registra decenas de
  cvars del servidor a través de `g_engfuncs`, es decir **llamadas del gamedll de vuelta
  al motor**, cruzando la frontera en las dos direcciones.

Es el mecanismo de §24 completo, ejercitado, en hardware.

**Y `-noip` funcionó.** Línea 0039: `NET_Init: -noip, skipping nxNetInit`. `NET_Init`
pasó de 9.010 ms a **1 ms**, y el arranque entero de 13,8 s a **2,8 s**. El diagnóstico
de §25.2 era correcto: los nueve segundos eran la concesión DHCP, y el switch no la
evitaba porque se miraba veinte líneas después de levantar la pila.

### 26.2 Otra vez excepción dura, y otra vez en un sitio raro

No hay `watchdog: 30 s with no output`, así que se repite el patrón de §25.3: **no fue
un bloqueo**. Si el hilo principal se atasca, el vigilante sigue vivo y devuelve la
consola. No lo hizo, luego el fallo se llevó todos los hilos.

Lo desconcertante es **dónde**. Entre `server game dll is up` y la siguiente traza no
hay ni una llamada:

```c
	XTRACE( "SV_LoadProgs: server game dll is up" );
	return true;                    // <- SV_LoadProgs
}
...
	if( !SV_LoadProgs( dllpath )) { ... }
	return true;                    // <- SV_InitGame
}
...
	SV_InitGame( ... );
	XTRACE( "SV_Init: done" );      // <- nunca sale
```

**Dos returns.** Un fallo ahí apunta a la pila: un desbordamiento de buffer en algún
sitio del camino que pisó una dirección de retorno, y que sólo se cobra la factura al
volver. `SV_LoadProgs` es una función larga con `static enginefuncs_t`, `static
globalvars_t` y `static playermove_t` —esos no están en pila— pero el gamedll ha estado
escribiendo en `svgame.globals` y en `gpEngfuncs` durante todo el tramo anterior.

Para partir ese hueco en dos hay ahora tres trazas donde había una:
`SV_InitGame: SV_LoadProgs returned`, `SV_Init: done` y `Host_Main: SV_Init ok`. El
próximo log dirá cuál de los dos returns es.

> **Y una fragilidad del trazador que podría haberme engañado, corregida.**
> `Xbox_TraceWriteFile` ponía `xbox_trace_path = NULL` al primer fallo de apertura, «para
> dejar de intentarlo». Eso significa que **un solo hipo del disco terminaba el log en
> una línea arbitraria**, y eso se lee exactamente igual que «el arranque murió aquí».
> El motor desmonta y vuelve a montar `D:` en `FS_MountXboxRootdir`, así que el hipo no
> es hipotético. Ahora se reintenta siempre y se cuenta lo perdido: si alguna línea no
> llegó al disco, la siguiente lo dice —`(N lines lost to disc errors)`— y no hay forma
> de confundir un log corto con un arranque corto.
>
> No hay indicio de que esto haya pasado en esta corrida. Pero era una conclusión falsa
> esperando su turno, y no se puede descartar retroactivamente para §25.

### 26.3 El cliente, instrumentado igual

Lo siguiente en la secuencia es `CL_Init`, y ahí está el módulo interesante: `client`
lleva **838 símbolos renombrados** frente a **1** del servidor (§24.3). Si algo resolvió
a la copia equivocada, se ve ahí.

| Dónde | Traza |
|---|---|
| `CL_Init` | `CL_InitLocal`, `VID_Init` con entrada y salida, `CL_LoadProgs`, `S_Init`, `Voice_Init`, `ID_Init`/`SteamBroker_Init`, fin |
| `CL_LoadProgs` | el nombre pedido y el handle; el paso de vgui; cuántos exports se van a resolver |
| | **una línea por cada export que no resuelva**, con su nombre |
| | antes de `pfnInitialize`, con su dirección, y al volver |
| | `CL_InitCDAudio`/`titles`/`particles`/`beams`/`tempents`, `R_InitRenderAPI`, `CL_InitEdicts`/`CL_InitClientMove` |

La línea por export que falta es la que importa para el punto 2 del encargo. El cliente
de Half-Life **no exporta `GetClientAPI` ni `F`** (§24.2), así que el motor resuelve los
~69 nombres de `cdll_exports[]` uno a uno contra la tabla generada. Un agujero ahí es un
agujero en el manifiesto de §24.2 —el que dejó 63 de 64— y saldrá con nombre y apellidos
en vez de como un `missing essential exports` genérico.

`pfnInitialize` se traza con su dirección por la misma razón que `GiveFnptrsToDll` en el
servidor: si sale `0x0`, el fallo es de la tabla; si sale un puntero y no vuelve, el
fallo está dentro del cliente.

### 26.4 Qué esperar en el próximo log

```
Host_Main: SV_Init
...
SV_LoadProgs: server game dll is up
SV_InitGame: SV_LoadProgs returned        <- si falta, el fallo es el primer return
SV_Init: done                             <- si falta, es el segundo
Host_Main: SV_Init ok
Host_Main: CL_Init (loads ref_gl and menu)
CL_Init: CL_InitLocal
CL_Init: VID_Init (renderer and menu)
R_LoadProgs / R_Init_Video / UI_LoadProgs <- lo de la seccion 23, que ya funcionaba
CL_Init: VID_Init ok
CL_Init: CL_LoadProgs(client)
CL_LoadProgs: handle=0x...                <- distinto de 0 = la tabla del cliente vale
CL_LoadProgs: base exports resolved       <- sin "WITH HOLES"
CL_LoadProgs: calling pfnInitialize (0x...)
CL_LoadProgs: pfnInitialize returned ok
CL_LoadProgs: client game dll is up
CL_Init: done
Host_Main: entering frame loop -- the engine is up
```

Tres cosas que mirar, por orden de lo que enseñan:

- **Si el log ya no se corta en `server game dll is up`**, el problema de §26.2 era el
  del trazador y no una excepción — y entonces habrá que releer §25 con esa duda.
- **`MISSING export <nombre>`**: agujero en el manifiesto del cliente.
- **Si muere en `pfnInitialize`**: el cliente entra pero se rompe dentro, y ahí sí
  entrarían en juego los 838 renombrados.

### 26.5 Qué NO está verificado

- **El cliente sigue sin ejecutarse.** Todo lo de §26.3 es instrumentación, no resultado.
- **Los 838 renombrados del cliente** siguen sin ejercitarse; los del servidor eran uno.
- **La causa del corte tras `server game dll is up` no se conoce**: hay dos hipótesis
  —excepción en el camino de retorno, o el fallo del trazador de §26.2— y el próximo log
  las separa.
- **`pfnGameInit` corrió, pero nadie ha comprobado que hiciera lo correcto.** Que vuelva
  no dice que los cvars que registró tengan los valores buenos.
- **El menú principal** sigue siendo el objetivo, sin alcanzar.
- **El presupuesto de memoria** sigue sin medirse.

### 26.6 Estado

`default.xbe` 4.616.192 B, subido. `xash.cmd` con `-noip`. 0 instrucciones SSE2 en 420
objetos y el build de Linux entero.

Ficheros del fork modificados: 16. Los tres nuevos de esta sesión —`cl_main.c`,
`cl_game.c`, `sv_init.c`— son sólo trazas. Diff aparte, como siempre.

## 27. Paso 2k — La pila: 250 KB donde el gamedll espera 1 MB

> **VEREDICTO (medido en §28.1): la pila NO era la causa.** La marca de agua de §27.5
> dice **45.372 bytes de máximo uso**. Con los 256.000 originales sobraban 210 KB: nunca
> estuvo cerca. La hipótesis de §27.3 queda descartada, y el fallo siguió exactamente en
> el mismo sitio con 1 MB de pila. Lo que sí queda de esta sección es todo lo demás: el
> descubrimiento de que nxdk no corre `main()` en el hilo principal (§27.2), el cuarto
> caso de una flag anulada por orden (§27.4), y la herramienta de medida, que es la que
> permitió cerrar el asunto en una iteración. La causa real está en §28.2.

**Resumen: el log no perdió ni una línea —la guarda de §26.2 lo dice explícitamente— así
que el fallo es real y está en el camino de retorno de `SV_LoadProgs`. Siguiendo el hilo:
nxdk no corre `main()` en el hilo principal del proceso, sino en uno que crea con
`thrd_create`, y ese hilo recibe la pila que dice la cabecera del XBE — **250 KB**, no el
megabyte contra el que está escrito un gamedll de Windows. Subida a 1 MB, y con una marca
de agua que lo mide en vez de suponerlo.**

| | |
|---|---|
| `(N lines lost to disc errors)` | **no aparece**: 0 líneas perdidas |
| Secuencia | `[0000]`…`[0060]` sin huecos |
| Las tres trazas del hueco | **ninguna**, ni en pantalla ni en disco |
| Pila del hilo del motor | **256.000 B** (0x3E800) → **1.048.576 B** |
| Prueba directa | marca de agua sobre patrón `0xBEEFCAFE` (§27.5) |

### 27.1 Lo primero: descartar al mensajero

Dos comprobaciones antes de teorizar, porque §26.2 dejó abierta la duda de si el
trazador podía estar cortando el log por su cuenta:

1. **`grep 'lines lost to disc errors'` → 0.** La guarda añadida en §26.2 habría marcado
   cualquier línea que no llegase al disco. No hubo ninguna.
2. **La numeración va de `[0000]` a `[0060]` sin saltos**, y el fichero tiene 61 líneas.
   El contador se incrementa en `Xbox_Trace` antes de escribir, así que un hueco en la
   numeración delataría una línea perdida aunque la guarda fallase. No hay hueco.

Y una tercera, por si el compilador me estaba engañando a mí: **las cadenas de las tres
trazas están en el binario**. `strings` sobre `default.xbe` encuentra
`SV_LoadProgs returned`, `SV_Init: done` y `Host_Main: SV_Init ok`, y también en los
objetos `sv_init.c.2.o`, `sv_main.c.2.o` y `host.c.2.o`. El código está compilado y
enlazado; simplemente no se ejecuta.

**Conclusión: el fallo es real y ocurre entre `XTRACE("server game dll is up")` y la
siguiente instrucción trazada, que está a dos `return` de distancia sin ninguna llamada
por medio.** Un fallo en el camino de retorno.

### 27.2 De dónde sale la pila en este target, que no es obvio

Perseguir la hipótesis lleva a un sitio que no esperaba. `engine/wscript:280` pide

```python
conf.env.append_unique('LINKFLAGS', ['-Wl,/STACK:256000', '-Wl,/MAP'])
```

y lo natural es suponer que eso es lo que el motor tiene. Pero **el `main()` del motor no
corre en el hilo principal del proceso**. `nxdk/lib/pdclib/platform/xbox/crt0.c`:

```c
thrd_t main_thread;
thrd_create(&main_thread, main_wrapper, NULL);   // <- main() va aqui dentro
thrd_detach(main_thread);
```

`thrd_create` llama a `CreateThread(NULL, 0, ...)`, y `nxdk/lib/winapi/thread.c:69`:

```c
if (dwStackSize == 0) {
    dwStackSize = CURRENT_XBE_HEADER->SizeOfStack;
}
```

es decir, el campo `dwPeStackCommit` de la cabecera XBE, que `cxbe` rellena
(`tools/cxbe/Xbe.cpp:76`) copiando el `SizeOfStackReserve` del PE — el que pone
`/STACK:`. La cadena cierra: **el arranque entero corre sobre 256.000 bytes.**
Comprobado en los dos sitios:

```
PE:   SizeOfStackReserve: 256000
XBE:  (PE) Stack Commit : 0x0003E800    = 256000
```

**250 KB.** En Windows, el sistema le da a un proceso **1 MB** por defecto, y el gamedll
de Valve está escrito contra eso: hlsdk tiene funciones con buffers locales generosos y
cadenas de llamada profundas, y nadie las ha auditado nunca contra un cuarto de esa
cifra. `pfnGameInit` acababa de recorrer medio hlsdk registrando cvars.

> Ojo con el nombre del campo: se llama `dwPeStackCommit` pero el comentario del propio
> `cxbe` avisa —«StackCommit actually means StackReserve»— y lo que copia es el
> *reserve*. En un Xbox no hay diferencia: no hay paginación bajo demanda, la pila se
> reserva entera.

### 27.3 Por qué un desbordamiento se cobra la factura al volver

Merece la pena dejarlo escrito porque la intuición engaña. Un desbordamiento clásico
—escribir por debajo del fondo de la pila— falla en la *escritura*, no al volver. Lo que
encaja con el síntoma es lo contrario: algo escribió **hacia arriba**, más allá de un
buffer local, y pisó una dirección de retorno. Entonces el código sigue corriendo tan
tranquilo hasta que ejecuta el `ret` correspondiente y salta a la nada.

Que las últimas cinco trazas antes del corte salieran bien —`pfnGameInit returned`,
`SV_InitClientMove`, `Delta_Init`, `pfnRegisterEncoders`, `server game dll is up`— y que
el corte llegue exactamente en el primer `ret` posterior es la firma de eso.

Una pila más pequeña de lo esperado produce ese cuadro de dos maneras: bien porque una
función escribe un buffer dimensionado con la pila de Windows en mente, bien porque el
propio marco no cabe y el `__chkstk` que clang emite para marcos grandes acaba tocando
memoria que no es suya.

**Nada de esto está demostrado.** Es la hipótesis que mejor encaja, y por eso lleva
adjunta una medición.

### 27.4 Cuarta vez que el orden de las flags decide el resultado

Subir la pila debería ser una línea. No lo fue.

`conf.env.append_unique('LINKFLAGS', ['-Wl,/STACK:1048576'])` desde `xbox.py` **no
surtió efecto**: el PE seguía diciendo 256000. Porque `xbox.py` se carga en
`wscript:237` y el `/STACK:256000` del fork se añade en `wscript:280`, veinte líneas
después. El último gana.

La solución es la misma familia de trucos que ya lleva este port: `LDFLAGS`, que la orden
de enlace de waf emite **al final de todo**, después de `LINKFLAGS`. Verificado:

```
PE:   SizeOfStackReserve: 1048576
XBE:  dwPeStackCommit (offset 0x130): 1048576 bytes (1024 KB)
```

Es la **cuarta** vez en este port que una flag correcta queda anulada por otra posterior:
§19.8 (`-Wno-error=implicit-function-declaration` de `xcompile.py`), §20.2
(`-Werror=strict-prototypes`), §21.3 (`-msse2` filtrado por opus) y ahora ésta. Con una
diferencia incómoda: las tres primeras las descubrí porque el build fallaba o el binario
se caía. **Ésta no daba ningún síntoma**: el `configure` imprimía tan contento
`... stack : 1048576 bytes` mientras el binario llevaba 250 KB. La única razón de haberlo
pillado es que comprobé el valor en la cabecera en vez de fiarme del mensaje.

Queda como opción, `--xbox-stack=N`, con 1 MB por defecto.

### 27.5 La prueba directa: marca de agua sobre la pila

La sugerencia del encargo, implementada. `Xbox_StackPaint()` y `Xbox_StackReport()` en
`engine/platform/xbox/xbox_trace.c`:

- **Pintar.** `KeGetCurrentThread()` da `StackBase` (dirección alta, donde empieza) y
  `StackLimit` (baja, donde se acaba). Se rellena con `0xBEEFCAFE` desde el fondo hasta
  512 bytes por debajo del marco actual, para no tocar nada vivo. Se hace en
  `launcher.c`, justo después de leer `xash.cmd`: es el punto más superficial de todo el
  arranque, así que la marca cubre todo lo que venga después.
- **Medir.** Se busca desde el fondo la primera palabra que ya no es el patrón. La
  distancia hasta `StackBase` es el máximo que se ha usado.

Tres puntos de medida, todos en sitios que el log demuestra que se ejecutan:

| Dónde | Por qué |
|---|---|
| `Host_Main`, tras `Host_InitCommon` | línea base: el arranque sin gamedll |
| `SV_LoadProgs`, tras `pfnGameInit` | después de que el gamedll haya corrido de verdad |
| `SV_LoadProgs`, en la última línea | justo antes del `return` que no vuelve |

La primera línea del informe da además el dato en crudo:
`stack: base=... limit=... size=N KB, sp=...`, que confirma en ejecución lo que §27.2
deduce leyendo código.

**Y el resultado es interpretable pase lo que pase.** Si la marca de agua en
`SV_LoadProgs` supera los 256.000 bytes, queda demostrado que con la pila anterior no
cabía. Si se queda muy por debajo —digamos en 60 KB— entonces la pila no era el problema
y la hipótesis de §27.3 se cae, con el mismo log.

### 27.6 Qué esperar en el próximo log

```
cmdline:   argv[1] = -noip
stack: base=0x... limit=0x... size=1024 KB, sp=0x...
...
stack after Host_InitCommon: high water N of 1048576 bytes (x%), M free
...
SV_LoadProgs: pfnGameInit returned, SV_InitClientMove
stack after pfnGameInit: high water N of 1048576 bytes
SV_LoadProgs: calling pfnRegisterEncoders
SV_LoadProgs: server game dll is up
stack at end of SV_LoadProgs: high water N of 1048576 bytes   <- el numero que decide
SV_InitGame: SV_LoadProgs returned                            <- si sale, era la pila
SV_Init: done
Host_Main: SV_Init ok
Host_Main: CL_Init (loads ref_gl and menu)
```

Cuatro lecturas posibles, y las cuatro dicen algo:

- **Pasa de `SV_LoadProgs` y la marca supera 250 KB** → era la pila, demostrado por
  partida doble.
- **Pasa, pero la marca se queda muy por debajo** → la pila más grande arregló algo por
  casualidad (una reserva que ahora cae en otro sitio, por ejemplo). Sospechoso, y
  habría que seguir mirando.
- **Sigue muriendo en el mismo sitio** → no era la pila. La marca de agua acota entonces
  cuánto margen sobraba, que es dato para descartarla del todo.
- **Muere antes, en `Xbox_StackPaint`** → `KeGetCurrentThread()->StackBase/StackLimit`
  no son lo que creo en un hilo de `PsCreateSystemThreadEx`, y el propio pintado se sale.

### 27.7 Qué NO está verificado

- **La hipótesis de la pila sigue siendo una hipótesis.** Encaja con el síntoma y con un
  número real y equivocado (250 KB frente a 1 MB), pero nadie ha visto todavía la marca
  de agua.
- **1 MB es una cifra prestada**, la de Windows. No está medido que haga falta tanto ni
  que baste.
- **`Xbox_StackPaint` nunca se ha ejecutado.** Escribe sobre casi toda la pila del hilo
  guiándose por dos campos del `KTHREAD`; si esos campos no son lo que creo, el pintado
  es exactamente el tipo de código que corrompe la pila que pretende medir. Es el riesgo
  que se asume a cambio de una prueba directa, y por eso hay un margen de 512 bytes bajo
  el marco actual y ocho palabras de respeto en el fondo.
- **El cliente sigue sin ejecutarse**, con su instrumentación de §26.3 intacta y sin
  estrenar.
- **El presupuesto de memoria empeora**: la pila pasa de 250 KB a 1 MB. En 128 MB no es
  nada, pero es una cifra más que nadie ha sumado.

### 27.8 Estado

`default.xbe` 4.616.192 B, subido, con 1 MB de pila confirmado en la cabecera. `xash.cmd`
con `-noip`. 0 instrucciones SSE2 en 420 objetos, build de Linux entero.

Ficheros del fork modificados: 16, los mismos de §26 más las tres llamadas a
`Xbox_StackReport`. `xbox.py` gana `--xbox-stack`.

## 28. Paso 2l — La causa real: `GiveFnptrsToDll` se llamaba con la convención equivocada

**Resumen: la pila no era. 45.372 bytes de máximo uso, 4% de 1 MB; con los 256.000
originales sobraban 210 KB. Y el síntoma tampoco cambió: la línea `[0064]` que parecía
progreso es una traza mía que va *antes* del `return`. Persiguiendo el sitio exacto
aparece la causa de verdad, y es del fork: `engine/eiface.h:37` excluye al Xbox del
`__stdcall`, mientras el gamedll de hlsdk compila `GiveFnptrsToDll` como `WINAPI`. Los
dos limpiaban los mismos 8 bytes.**

| | |
|---|---|
| Marca de agua | **45.372 de 1.048.576 B (4%)**, idéntica en los tres puntos |
| ¿Pila con 256 KB? | sobraban **210 KB**. Descartada |
| ¿Cambió el síntoma? | **No** (§28.1) |
| Causa | desajuste de convención de llamada en `GiveFnptrsToDll` (§28.2) |
| Arreglo | una línea en `engine/eiface.h`, verificada en el desensamblado |

### 28.1 El síntoma no cambió: la línea 0064 es mía

Conviene deshacer el espejismo antes de nada, porque cambia la lectura de todo lo demás.

```
[0063     2837] SV_LoadProgs: server game dll is up
[0064     2845] stack at end of SV_LoadProgs: high water 45372 of 1048576 bytes (4%)
```

La `[0064]` es el `Xbox_StackReport( "at end of SV_LoadProgs" )` que §27.5 añadió, y está
**dentro de `SV_LoadProgs`, antes del `return`**. Las tres trazas del hueco siguen sin
aparecer —`SV_LoadProgs returned`, `SV_Init: done`, `Host_Main: SV_Init ok`— y `CL_Init`
tampoco. Comprobado por `grep` sobre el log, no por lo que se vio en pantalla.

**El punto de fallo es exactamente el mismo**: el primer `ret` después de la última
sentencia de `SV_LoadProgs`. Lo único que ha cambiado es que ahora hay una línea más
antes de él.

Y la marca de agua, además de matar la hipótesis, apoya esto: **45.372 bytes en los tres
puntos de medida**, idénticos. El máximo se alcanzó antes del primero (dentro de
`Host_InitCommon`, que escanea el filesystem y arranca SDL) y nada posterior bajó más.
El gamedll no gastó pila apreciable.

Sobre la sospecha de que pintar la pila enmascarase un fallo dependiente de basura no
inicializada: **es una preocupación legítima y aquí no aplica**, porque el fallo siguió
ocurriendo en el mismo sitio con la pila pintada. Si el pintado hubiera tapado algo, el
arranque habría avanzado. Aun así el pintado es una medición que altera lo que mide, así
que ahora se puede desactivar con `-nostackpaint` en `xash.cmd`, sin recompilar.

### 28.2 La causa: dos convenciones de llamada para la misma función

Con la pila descartada, quedaba explicar por qué falla un `ret` cuando todo lo anterior
funciona. El desensamblado de `SV_LoadProgs` da la primera pieza:

```
00003568 <_SV_LoadProgs>:
    3568:  pushl %edi
    3569:  pushl %esi
    356a:  pushl %eax          <- 4 bytes de locales, y ya
...
    3ac0:  addl  $4, %esp
    3ac3:  popl  %esi
    3ac4:  popl  %edi
    3ac5:  retl
```

**No hay marco de pila.** Con `-Os` clang omite el frame pointer, así que el epílogo
deshace el prólogo contando bytes: si `%esp` no vale exactamente lo que valía, los `pop`
leen basura y el `retl` salta a cualquier sitio.

La segunda pieza es de dónde sale el desajuste. `engine/eiface.h:37`, del fork:

```c
#if defined(_WIN32) && !XASH_XBOX
#define DLLEXPORT __stdcall
#else
#define DLLEXPORT /* */
#endif
```

Ese `&& !XASH_XBOX` lo puso el autor. Y `DLLEXPORT` se usa **en un solo sitio de todo el
motor**, `sv_game.c:40`:

```c
typedef void (DLLEXPORT *GIVEFNPTRSTODLL)( enginefuncs_t*, globalvars_t* );
```

Al otro lado, el gamedll viene de hlsdk compilado con **sus propias** cabeceras.
`dlls/h_export.cpp:47`:

```c
#define EXPORT2 WINAPI
...
extern "C" void DLLEXPORT EXPORT2 GiveFnptrsToDll( enginefuncs_t*, globalvars_t* )
```

y el símbolo lo dice sin lugar a interpretación —está en el manifiesto de §24.2—:

```
GiveFnptrsToDll _GiveFnptrsToDll@8      <- @8: stdcall, el callee limpia 8 bytes
GetEntityAPI    _GetEntityAPI           <- sin @: cdecl, limpia el caller
```

Así que el motor empujaba dos argumentos y limpiaba 8 bytes él mismo, y la función
—stdcall— limpiaba otros 8 con su `ret 8`. **`%esp` volvía 8 bytes demasiado alto.**

Y de ahí el cuadro entero:

- La llamada vuelve bien.
- Todo lo que sigue funciona: `SV_LoadProgs` casi no usa la pila —cuatro bytes de
  locales, el resto son `static` en `.data`— así que un `%esp` desplazado no rompe nada.
  Por eso `GetEntityAPI2`, los 1200 edicts, `pfnGameInit` y `pfnRegisterEncoders`
  salieron todos correctos, con valores sensatos.
- El desajuste espera hasta el epílogo, que restaura `%esp` sumando en vez de desde
  `%ebp`, y el `retl` salta a basura. Se lleva el título entero, por eso el vigilante
  tampoco corría (§25.3).

Lo que se veía en pantalla era el reflejo exacto de esto: **todo bien hasta el momento
justo de volver**.

Confirmado en el desensamblado, antes y después del arreglo:

```
antes:   calll *0   ->   addl $8, %esp     <- el caller tambien limpiaba
despues: calll *0   ->   pushl $0          <- ya no; limpia solo el callee
```

Y las otras tres llamadas indirectas de la función (`GetEntityAPI`, `GetEntityAPI2`,
`GiveNewDllFuncs`) **siguen** con su `addl $8, %esp`, que es lo correcto: sus símbolos no
llevan `@`, son cdecl en los dos lados. El cambio tocó exactamente una llamada.

### 28.3 El arreglo

Quitar el `&& !XASH_XBOX`. Una línea, y es segura precisamente porque `DLLEXPORT` sólo se
usa en ese typedef: no hay superficie que pueda romper. Deja al motor exactamente igual
que en un build de Windows, que es contra lo que está compilado el gamedll.

**Es un bug del fork**, no del port. En cualquier combinación donde el gamedll venga de
hlsdk con sus cabeceras —es decir, siempre— esa exclusión rompe la llamada. Se une a la
lista de §19.8: `MSGBOX_XBOX` sin valor, `XASH_CUSTOM_SWAP` roto, el
`-Wno-error=implicit-function-declaration` inerte, y ahora ésta.

No se puede saber por qué el autor la puso. Una explicación plausible es que su gamedll
—si llegó a tener uno— estuviera compilado sin `WINAPI` en esa función, en cuyo caso su
exclusión era correcta *para su combinación* y venenosa para cualquier otra. Es
especulación.

### 28.4 Lo que esta sesión deja además

- **`-nostackpaint`**, para que la instrumentación de §27.5 pueda apagarse desde
  `xash.cmd`. Una medición que altera lo que mide tiene que ser opcional.
- **`Xbox_CmdlineHas()`**, que responde a lo mismo que `Sys_CheckParm` pero **antes** de
  que `Host_InitCommon` parsee nada, para switches que cambian lo que hace `main()`.
- **La marca de agua se queda.** Ha cumplido su función —matar una hipótesis en una
  iteración, con un número— y cuesta unos milisegundos al arrancar.

### 28.5 Qué esperar en el próximo log

```
SV_LoadProgs: server game dll is up
stack at end of SV_LoadProgs: high water ~45000 of 1048576 bytes
SV_InitGame: SV_LoadProgs returned        <- la linea que lleva tres sesiones sin salir
SV_Init: done
Host_Main: SV_Init ok
Host_Main: CL_Init (loads ref_gl and menu)
CL_Init: CL_InitLocal
CL_Init: VID_Init (renderer and menu)
R_Init_Video: pbgl up, VID_SetMode        <- lo de la seccion 23
CL_Init: CL_LoadProgs(client)
CL_LoadProgs: handle=0x...
CL_LoadProgs: base exports resolved       <- sin "WITH HOLES"
CL_LoadProgs: calling pfnInitialize (0x...)
CL_LoadProgs: client game dll is up
CL_Init: done
Host_Main: entering frame loop -- the engine is up
```

Si `SV_InitGame: SV_LoadProgs returned` sale, el diagnóstico era correcto y el siguiente
territorio es el cliente, con toda la instrumentación de §26.3 esperando sin estrenar.

### 28.6 Qué NO está verificado

- **El arreglo no se ha ejecutado.** Lo demostrado es que el desensamblado ya no limpia
  dos veces esa llamada, que es la condición necesaria; que el `ret` vuelva es la
  predicción.
- **No se ha auditado el resto de la frontera motor↔gamedll** buscando desajustes
  parecidos. `DLLEXPORT` sólo se usa una vez en el motor, pero `enginefuncs_t` y
  `DLL_FUNCTIONS` tienen cientos de punteros a función, y nadie ha comprobado uno a uno
  que las dos partes los declaren igual. Que `pfnGameInit` y compañía funcionaran no
  prueba nada de las que llevan argumentos.
- **El cliente sigue sin ejecutarse.**
- **La pila queda en 1 MB** aunque la medida diga que sobraban con 250 KB. No hay razón
  para volver atrás —cuesta 750 KB de una consola de 128 MB— pero es un cambio que ya no
  se justifica por lo que lo motivó. `--xbox-stack` permite revertirlo.
- **El presupuesto de memoria** sigue sin medirse.

### 28.7 Estado

`default.xbe` 4.616.192 B, subido. `xash.cmd` con `-noip`. 0 instrucciones SSE2 en 420
objetos y build de Linux entero — este arreglo toca `engine/eiface.h`, que se compila en
todas las plataformas, así que la comprobación importaba más que de costumbre.

Ficheros del fork modificados: 17.

---

## 29. Paso 2m — El motor arranca entero en la consola

El arreglo de §28 —quitar `&& !XASH_XBOX` de `engine/eiface.h`— era una predicción. Esta
sección la comprueba. El log tiene 100 líneas, **0 perdidas**, y termina en:

```
[0099     3427] Host_Main: entering frame loop -- the engine is up
```

Es la primera vez desde §17 que el arranque no muere en ninguna parte. Motor, filesystem,
renderer, menú, gamedll de servidor y gamedll de cliente: los seis módulos están dentro
del XBE, cargados y corriendo, en 3,4 segundos desde el encendido del trazador.

### 29.1 Las cuatro preguntas, respondidas

**1. ¿Pasó el `ret`?** Sí. Las tres líneas que llevaban tres sesiones sin salir:

```
[0065     2853] SV_InitGame: SV_LoadProgs returned
[0066     2854] SV_Init: done
[0067     2855] Host_Main: SV_Init ok
```

El diagnóstico de §28 era correcto. `GiveFnptrsToDll` está declarada `__stdcall` en
`hlsdk` y se llamaba como `cdecl` desde el motor; el caller limpiaba una pila que ya
había limpiado el callee, y el desajuste no se manifestaba en la llamada sino tres
`ret` más arriba. Eso explica también por qué el hueco de §26 nunca escribió nada: no se
moría *ejecutando* código, se moría al *volver*.

**2. ¿Llega a `CL_Init` y `CL_LoadProgs`? ¿Hay algún `MISSING export`?**

Llega, y **no hay ninguno**:

```
[0084     3146] CL_LoadProgs: handle=0x45fd18
[0085     3146] CL_LoadProgs: vgui step done, resolving 37 exports
[0086     3147] CL_LoadProgs: base exports resolved
[0087     3148] CL_LoadProgs: calling pfnInitialize (0x1a1a30)
[0088     3149] CL_LoadProgs: pfnInitialize returned ok
```

`MISSING export` = 0, `WITH HOLES` = 0. Los 37 exports de la API base del cliente se
resolvieron todos, y `pfnInitialize` —la primera llamada real al gamedll de cliente—
entró y volvió.

Esto es relevante más allá del arranque. El `client.lib` traía **838 choques de símbolos**
frente a 1 del servidor (§24.5); la sospecha razonable era que el renombrado masivo con
`llvm-objcopy --redefine-syms` hubiera dejado alguna referencia apuntando al símbolo
equivocado. Los 37 exports resueltos y `pfnInitialize` volviendo bien dicen que al menos
la superficie que el arranque toca está intacta. **No dicen** que los 838 estén bien: dicen
que los que se usan hasta aquí lo están.

**3. ¿Dónde muere ahora?** No muere. Llega al final de `Host_Main` y entra en el bucle de
frames:

```
[0092     3292] CL_LoadProgs: client game dll is up
[0093     3296] CL_Init: client game dll is up, S_Init
[0094     3421] CL_Init: S_Init ok, Voice_Init
[0095     3421] CL_Init: ID_Init / SteamBroker_Init
[0096     3423] CL_Init: done
[0097     3424] Host_Main: CL_Init ok
[0098     3426] Host_Main: post-init configs done
[0099     3427] Host_Main: entering frame loop -- the engine is up
```

Ni `returning to dashboard` (0 apariciones) ni `START+BACK` (0). El vigilante de 30 s se
desarma justo al entrar en el bucle, por diseño, así que su silencio no prueba nada. La
consola se apagó a mano.

**Lo que esto significa exactamente:** el motor llegó al bucle de frames y **escribió esa
línea en disco**. No dice cuántos frames ha corrido después, ni si el bucle sigue vivo, ni
qué está dibujando. Ver §29.4.

### 29.2 El log íntegro

`research/logs/xash-boot-20260817-045525.log`, 5.386 B, 100 líneas, 0 perdidas.
Columnas: número de línea y milisegundos desde `Xbox_TraceInit`.

```
[0000      543] trace up: log=D:\xash-boot.log fb=0x83eb4000 serial=no uart at 0x3F8
[0001      544] watchdog armed: back to the dashboard after 30 s of silence
[0002      545] crt initializers: alive before main()
[0003      546] main: entered, argc=0 argv=0xd0122c84 argv0=(null)
[0004      622] main: XVideoSetMode 640x480x32 done
[0005      623] main: Xbox_GetArgv -> argc=0 argv=0xd0122c84
[0006      624] cmdline: D:\xash.cmd gave 1 argument
[0007      625] cmdline:   argv[1] = -noip
[0008      625] stack: base=0xd0123000 limit=0xd0023000 size=1024 KB, sp=0xd0122c58
[0009      630] main: calling Host_Main, gamedir=valve
[0010      631] Host_Main: enter
[0011      632] Host_InitCommon: enter, argc=2 basedir=valve
[0012      632] Host_InitCommon: Sys_ParseCommandLine ok
[0013      633] Host_InitCommon: exename=default
[0014      634] Host_InitCommon: Memory_Init
[0015      634] Host_InitCommon: Zone Engine pool ok
[0016      635] Host_InitCommon: Cmd/Cvar ok, Sys_InitLog
[0017      637] Host_InitCommon: Con_Init ok
[0018      638] Host_InitCommon: Platform_Init (SDL2 on this target)
[0019      639] Host_InitCommon: Platform_Init ok
[0020      653] Host_InitCommon: FS_Init
[0021      654] FS_Init: FS_DetermineRootDirectory (mounts D: on this target)
[0022      655] FS_Init: rootdir=D:\
[0023      655] FS_Init: rodir= gamedir=valve
[0024      656] FS_LoadProgs: COM_LoadLibrary(filesystem_stdio)
[0025      657] FS_LoadProgs: handle=0x468b34
[0026      657] FS_LoadProgs: filesystem_stdio ready
[0027      658] FS_Init: InitStdio
[0028      684] FS_Init: filesystem mounted
[0029      685] Host_InitCommon: FS_Init ok
[0030      686] Host_InitCommon: Image_Init / Sound_Init
[0031      687] Host_InitCommon: Image/Sound ok
[0032      687] Host_InitCommon: FS_LoadGameInfo
[0033      735] Host_InitCommon: gamedir=valve dll_path=cl_dlls
[0034      761] Host_InitCommon: gfx/conchars found
[0035      776] Host_InitCommon: decals/HPAK ok, IN_Init
[0036     2747] Host_InitCommon: done
[0037     2748] Host_Main: Host_InitCommon returned
[0038     2756] stack after Host_InitCommon: high water 45364 of 1048576 bytes (4%), 1003212 free
[0039     2757] Host_Main: Mod_Init
[0040     2771] Host_Main: NET_Init
[0041     2772] NET_Init: -noip, skipping nxNetInit (no DHCP wait)
[0042     2773] Host_Main: NET_Init ok
[0043     2773] Host_Main: NET_InitMasters ok
[0044     2774] Host_Main: Netchan_Init ok
[0045     2775] Host_Main: SV_Init
[0046     2778] SV_Init: cvars done, SV_InitFilter
[0047     2778] SV_Init: SV_ClearGameState
[0048     2780] SV_Init: SV_InitGame (this loads the server game dll)
[0049     2780] SV_InitGame: SV_LoadProgs(server)
[0050     2781] SV_LoadProgs: COM_LoadLibrary(server)
[0051     2782] SV_LoadProgs: handle=0x45ed78
[0052     2783] SV_LoadProgs: entry points resolved (API=0x179c80 API2=0x179cb0 New=0 Give=0x17b150)
[0053     2784] SV_LoadProgs: calling GiveFnptrsToDll
[0054     2784] SV_LoadProgs: GiveFnptrsToDll returned
[0055     2785] SV_LoadProgs: calling GetEntityAPI2
[0056     2786] SV_LoadProgs: entity API ok
[0057     2787] SV_LoadProgs: allocating 1200 edicts
[0058     2807] SV_LoadProgs: calling pfnGetGameDescription
[0059     2807] SV_LoadProgs: calling pfnGameInit
[0060     2814] SV_LoadProgs: pfnGameInit returned, SV_InitClientMove
[0061     2835] stack after pfnGameInit: high water 45364 of 1048576 bytes (4%), 1003212 free
[0062     2844] SV_LoadProgs: calling pfnRegisterEncoders
[0063     2845] SV_LoadProgs: server game dll is up
[0064     2852] stack at end of SV_LoadProgs: high water 45364 of 1048576 bytes (4%), 1003212 free
[0065     2853] SV_InitGame: SV_LoadProgs returned
[0066     2854] SV_Init: done
[0067     2855] Host_Main: SV_Init ok
[0068     2855] Host_Main: CL_Init (loads ref_gl and menu)
[0069     2856] CL_Init: CL_InitLocal
[0070     2858] CL_Init: VID_Init (renderer and menu)
[0071     2862] R_LoadProgs: COM_LoadLibrary(ref_gl)
[0072     2863] R_LoadProgs: handle=0x468b24
[0073     2864] R_LoadProgs: GetRefAPI ok, calling the renderer's R_Init
[0074     2866] R_Init_Video: pb_size 2 MB, pbgl_init
[0075     2920] R_Init_Video: pbgl up, VID_SetMode
[0076     2921] R_Init_Video: mode set, GL_InitExtensions
[0077     2924] R_Init_Video: done
[0078     2934] R_LoadProgs: renderer up
[0079     2935] UI_LoadProgs: COM_LoadLibrary(menu)
[0080     2936] UI_LoadProgs: handle=0x468b58
[0081     3136] CL_Init: VID_Init ok
[0082     3144] CL_Init: CL_LoadProgs(client)
[0083     3145] CL_LoadProgs: COM_LoadLibrary(client)
[0084     3146] CL_LoadProgs: handle=0x45fd18
[0085     3146] CL_LoadProgs: vgui step done, resolving 37 exports
[0086     3147] CL_LoadProgs: base exports resolved
[0087     3148] CL_LoadProgs: calling pfnInitialize (0x1a1a30)
[0088     3149] CL_LoadProgs: pfnInitialize returned ok
[0089     3150] CL_LoadProgs: CD audio, titles, particles, beams, tempents
[0090     3287] CL_LoadProgs: R_InitRenderAPI
[0091     3288] CL_LoadProgs: CL_InitEdicts / CL_InitClientMove
[0092     3292] CL_LoadProgs: client game dll is up
[0093     3296] CL_Init: client game dll is up, S_Init
[0094     3421] CL_Init: S_Init ok, Voice_Init
[0095     3421] CL_Init: ID_Init / SteamBroker_Init
[0096     3423] CL_Init: done
[0097     3424] Host_Main: CL_Init ok
[0098     3426] Host_Main: post-init configs done
[0099     3427] Host_Main: entering frame loop -- the engine is up
```

### 29.3 Dónde se va el tiempo

Con el `-noip` ya corregido (§25.4), el arranque entero cuesta **3,4 s**. Los tramos que
pesan, medidos entre líneas consecutivas:

| Tramo | Líneas | Coste |
|---|---|---|
| `IN_Init` → `Host_InitCommon: done` | 0035→0036 | **1.971 ms** |
| `UI_LoadProgs` → `VID_Init ok` (init del menú) | 0080→0081 | 200 ms |
| `S_Init` | 0093→0094 | 125 ms |
| tempents/beams → `R_InitRenderAPI` | 0089→0090 | 137 ms |
| `pbgl_init` | 0074→0075 | 54 ms |
| todo lo demás | | < 30 ms cada uno |

Los 1.971 ms de `IN_Init` son **el 58% del arranque** y no están instrumentados por dentro:
la traza sale antes de `IN_Init` y la siguiente ya es el `done` de `Host_InitCommon`, así
que dentro caben `IN_Init`, el mando y lo que `Host_InitCommon` haga después. Es el
siguiente sitio obvio donde partir una traza si el tiempo de arranque llega a importar.
Hoy no importa.

### 29.4 Los dos testigos que hemos perdido

Esto es lo incómodo de la sesión y conviene decirlo antes que el titular. **La pantalla ya
no es testigo, y la red tampoco.**

- **La pantalla**, desde §23: `debugPrint` de nxdk escribe en el framebuffer de nxdk, y
  en cuanto `pbgl_init` toma la GPU (línea 0074) ese framebuffer deja de mostrarse. Todo
  lo que el trazador pinte a partir de ahí se escribe en memoria que ya nadie enseña. El
  "amago de imagen y pantalla negra" que se ve en la consola es **exactamente el mismo
  síntoma** que en §23, cuando el motor moría mucho antes: el síntoma visible no ha
  cambiado aunque el motor haya avanzado 60 líneas de log. Una pantalla negra aquí no
  distingue "colgado" de "corriendo y sin dibujar nada".
- **La red**, desde §25: con `-noip` no hay pila de red, así que "la consola no responde
  al ping" ya no significa nada. Antes era una señal burda de vida.

Queda **el log en disco**, y sólo eso. Que es de escritura síncrona (open/write/close por
línea, §20.3) y por tanto sobrevive a un cuelgue duro, que es justo por lo que se hizo
así. Pero es un testigo que sólo habla cuando alguien le pregunta, por FTP, después.

**Consecuencia directa:** no se sabe si el bucle de frames ha dado un frame o diez mil. La
línea 0099 se escribe *al entrar*, no dentro. Lo primero de la próxima ronda tiene que ser
un latido —una traza cada N frames, o cada segundo— porque sin él "entering frame loop"
es lo mismo que "murió en la primera instrucción del bucle".

### 29.5 Por qué el motor no imprime nada suyo en el log

Las 100 líneas del log están numeradas: **todas** son `Xbox_Trace`. Ni una sola línea de la
consola del motor. Y eso pese a que `sys_con.c` está enganchado al trazador y a que en
§25 se metió a mano un `Con_Printf( "Xbox networking disabled by -noip\n" )` que tampoco
sale.

La cadena está bien —se verificó símbolo a símbolo: `Con_Printf` → `Con_Printfv` →
`Sys_Print` → `Sys_PrintLog` (incondicional, `system.c:547`) → `Sys_PrintStdout` → rama
Xbox → `Xbox_TraceRaw`, y `llvm-nm --undefined-only` confirma que `sys_con.c.o` referencia
`_Xbox_TraceRaw`. El filtro está una llamada antes de todo eso, en la primera línea de
`Con_Printf`:

```c
void GAME_EXPORT Con_Printf( const char *szFmt, ... )
{
	if( !host.allow_console )
		return;
```

Y `host.allow_console` sale de `host.c:1038`:

```c
host.allow_console = DEFAULT_ALLOWCONSOLE || DEFAULT_DEV > 0;
```

**`DEFAULT_ALLOWCONSOLE` vale 0 en este target.** No porque nadie lo definiera para Xbox
—el fork lo define— sino porque **la rama de Xbox de `common/defaults.h` es código muerto**:

```c
// common/defaults.h:140
#if XASH_WIN32                       // <- se cumple: nxdk define _WIN32
	#define DEFAULT_FULLSCREEN   "0"
#elif XASH_NSWITCH
	...
#elif XASH_PSVITA
	...
#elif XASH_XBOX                      // <- inalcanzable, XASH_WIN32 ya ganó
	#define DEFAULT_MODE_WIDTH   640
	#define DEFAULT_MODE_HEIGHT  480
	#define DEFAULT_FULLSCREEN   "2"
	#define DEFAULT_M_IGNORE     "1"
	#define DEFAULT_ALLOWCONSOLE  1
	#define XASH_NO_IPV6_RESOLVE  1
#endif
```

Comprobado con el preprocesador real de nxdk y las banderas del build, no leyendo el
fuente:

```
$ nxdk-cc -E -P -DXASH_XBOX=1 -I... defprobe.c
const char *probe_win32        = "XASH_WIN32=" "1";
const char *probe_xbox         = "XASH_XBOX=" "1";
const char *probe_allowconsole = "DEFAULT_ALLOWCONSOLE=" "0";
const char *probe_fullscreen   = "DEFAULT_FULLSCREEN=" "0";
const char *probe_width        = "DEFAULT_MODE_WIDTH=" "DEFAULT_MODE_WIDTH";   <- sin definir
const char *probe_height       = "DEFAULT_MODE_HEIGHT=" "DEFAULT_MODE_HEIGHT"; <- sin definir
const char *probe_mignore      = "DEFAULT_M_IGNORE=" "0";
```

`XASH_WIN32` y `XASH_XBOX` valen 1 **a la vez**, que es correcto —el Xbox *es* Win32, esa
es toda la premisa de nxdk— y precisamente por eso la cadena `#if/#elif` no sirve para
distinguirlos: la rama de Xbox tenía que ir **antes** que la de Win32, o dentro de ella.

Es **el quinto bug del fork** de la lista de §19.8, y de la misma familia exacta que el
primero (`MSGBOX_XBOX` definido sin valor): código escrito para Xbox que el preprocesador
nunca ve. Lo que se pierde:

| Se pierde | Queda en | Consecuencia |
|---|---|---|
| `DEFAULT_ALLOWCONSOLE 1` | `0` | **`Con_Printf` no imprime nada.** Es lo que estamos viendo |
| `DEFAULT_FULLSCREEN "2"` | `"0"` (ventana) | inerte hoy: `vid_xbox.c` llama a `R_ChangeDisplaySettings(0,0,WINDOW_MODE_FULLSCREEN)` a pelo y no mira el cvar |
| `DEFAULT_MODE_WIDTH/HEIGHT 640×480` | sin definir | inerte hoy: sólo los usa `vid_sdl2.c`, y el backend de vídeo aquí es `vid_xbox.c` |
| `DEFAULT_M_IGNORE "1"` | `"0"` | el motor espera ratón donde no hay |
| `XASH_NO_IPV6_RESOLVE 1` | sin definir | `net_ws.c:244` toma el camino de resolución IPv6. Con `-noip` no se llega; **sin** `-noip` es candidato a parte de los 9 s históricos |

El único con efecto demostrado hoy es el primero. Los otros cuatro son deuda: hoy están
tapados por `vid_xbox.c` y por `-noip`, y saldrán en cuanto se quite cualquiera de los dos.

**Arreglo** (pendiente, no aplicado en esta ronda): mover la rama `XASH_XBOX` delante de
`XASH_WIN32`. Es una línea movida, pero toca un fichero que compila en todas las
plataformas y no se ha querido meter en la misma ronda que la comprobación de §28.

**Atajo inmediato mientras tanto:** `host.c:1040` hace `if( Sys_CheckParm( "-dev" ))
host.allow_console = true;`, así que poner `-dev 2` en `D:\xash.cmd` enciende la consola
del motor sin tocar código. Vale la pena en la próxima ronda por sí solo: es la diferencia
entre tener las trazas que hemos escrito nosotros y tener además las que el motor lleva
puestas de fábrica.

### 29.6 Qué NO está verificado

Esto es más largo que el titular, y con razón.

- **No se sabe si el bucle de frames corre.** La línea 0099 se escribe al entrar. Sin
  latido, "entró en el bucle" y "murió en la primera instrucción del bucle" producen
  exactamente el mismo log. Ver §29.4.
- **No hay nada dibujado.** El menú principal —el objetivo declarado de §24— **no se ha
  visto**. `UI_LoadProgs` cargó el módulo (línea 0080) y `VID_Init` volvió bien, pero
  nadie ha comprobado que el menú *renderice*. La pantalla negra no lo desmiente ni lo
  confirma.
- **`pfnVidInit` del cliente no aparece en el log.** El cliente cargó y `pfnInitialize`
  volvió, pero la llamada que le dice "hay vídeo, prepárate" no está instrumentada.
- **De los 838 choques de símbolos del cliente, sólo se ha ejercitado lo que toca el
  arranque.** 37 exports y `pfnInitialize`. El resto sigue sin evidencia.
- **La frontera motor↔gamedll sigue sin auditar** (§28.6 sin cambios): `enginefuncs_t` y
  `DLL_FUNCTIONS` tienen cientos de punteros y nadie ha comparado declaración con
  declaración. §28 encontró uno; no hay razón para creer que fuera el único.
- **Sonido:** `S_Init` volvió en 125 ms. No se ha oído nada.
- **Mando:** `IN_Init` volvió. No se ha pulsado nada que llegue al motor.
- **El presupuesto de memoria** sigue sin medirse (§28.6).
- **La rama muerta de `defaults.h` está diagnosticada, no arreglada.**

### 29.7 Lo que toca ahora

En este orden, porque cada uno hace visible al siguiente:

1. **Latido en el bucle de frames.** Sin él no se puede afirmar nada de lo que viene
   después. Barato: una traza cada segundo con el contador de frames.
2. **`-dev 2` en `xash.cmd`**, para encender `Con_Printf` sin tocar código, y de paso ver
   si el motor lleva rato quejándose de algo por un canal que estaba mudo.
3. **Arreglar la rama de `defaults.h`**, que es lo correcto y hace innecesario el punto 2.
4. **Devolver el trazador a la pantalla después de pbgl**: pintar por encima de pbgl, o
   por serie si aparece cable, o aceptar que el log es el único testigo y añadir latido
   suficiente. Sin esto, cada iteración cuesta un viaje de FTP.
5. Entonces, y sólo entonces, preguntarse por qué el menú no se ve.

### 29.8 Estado

`default.xbe` 4.616.192 B, subido, sin cambios respecto a §28.7 — esta sección **no ha
compilado nada**, sólo ha leído lo que aquel binario produjo. `xash.cmd` con `-noip`.
Ficheros del fork modificados: 17. Ficheros nuestros: 20.

El motor de Xash3D-FWGS con Half-Life dentro arranca entero en una Xbox de 2001 en 3,4
segundos. No dibuja nada todavía.

## 30. Paso 2n — Testigos: latido de frames, la consola del motor, y el trazador de vuelta en pantalla

**Resumen: los cuatro encargos de §29.7, hechos y subidos. Un latido de una línea por
segundo dentro del bucle de frames; `-dev 2` en `xash.cmd` para encender `Con_Printf`;
la rama muerta de `defaults.h` movida a su sitio y la familia entera auditada con un
script — tras el arreglo quedan **0** ramas Xbox muertas; y un overlay que repinta las
últimas líneas del trazador sobre el framebuffer de pbgl en cada swap, leyendo del CRTC
qué búfer se está mostrando en vez de suponerlo. Nada de esto ha corrido todavía.**

| | |
|---|---|
| Latido | `host.c`, 1 línea/s con contador de frames y fps |
| `xash.cmd` | `-noip -dev 2`, verificado releyéndolo por FTP |
| `defaults.h` | rama `XASH_XBOX` delante de `XASH_WIN32`; probe: los 5 valores correctos |
| Auditoría | 21 cadenas con XBOX en `#elif`; **0 muertas** tras el arreglo |
| Overlay | anillo de 10×76 sobre pbgl, fuente unscii de nxdk, `PCRTC_START` leído por frame |
| `default.xbe` | 4.616.192 → **4.620.288 B**, SSE2 = 0 en 420 objetos, **subido** |

### 30.1 El latido

En el `while` de `Host_Main`, tras `COM_Frame`, un bloque `#if XASH_XBOX`: cuenta frames
y cada vez que pasa un segundo de `Platform_DoubleTime` emite

```
heartbeat: frame 1234, 59.8 fps
```

Es la respuesta a §29.4: sin esto, «entró en el bucle» y «murió en la primera
instrucción del bucle» producen el mismo log. Cuesta un open/append/close por segundo
(la escritura síncrona de §20.3), que en el bucle de frames es ruido.

### 30.2 `-dev 2`

`D:\xash.cmd` dice ahora `-noip -dev 2` — reescrito por FTP y releído para confirmar.
`host.c:1040` hace `if( Sys_CheckParm( "-dev" )) host.allow_console = true`, así que la
consola del motor habla aunque el binario viejo siguiera puesto; con el `defaults.h`
arreglado (30.3) `allow_console` ya vale 1 de fábrica y `-dev 2` queda para lo que es:
subir el nivel de developer y ver los mensajes que el motor lleva callando desde §17.

**Aviso hecho a sabiendas:** cada línea de `Con_Printf` pasa por `Xbox_TraceRaw`, y eso
es un open/append/close en disco *por línea*. Con `-dev 2` el arranque imprime cientos;
el próximo arranque será más lento y el log mucho más gordo. Es el precio de oír al
motor la primera vez. Si estorba, se baja a `-dev 1` o se quita.

### 30.3 `defaults.h`, y la auditoría de toda la familia

El arreglo es el anunciado en §29.5: la rama `XASH_XBOX` del bloque «Platform
overrides» movida **delante** de la de `XASH_WIN32`, con un comentario que dice por qué
(nxdk define `_WIN32`; ésa es la premisa del toolchain). Verificado con el preprocesador
real de nxdk (`ofx-q.sh`), no leyendo el fuente:

```
DEFAULT_ALLOWCONSOLE=1   (era 0 — el bug de §29.5)
DEFAULT_FULLSCREEN="2"   (era "0")
DEFAULT_MODE_WIDTH=640   (era: sin definir)
DEFAULT_MODE_HEIGHT=480  (era: sin definir)
DEFAULT_M_IGNORE="1"     (era "0")
```

Y `XASH_NO_IPV6_RESOLVE=1` queda definido, así que `net_ws.c` dejará de intentar
resolución IPv6 el día que se quite `-noip`. Las cinco deudas de la tabla de §29.5
quedan saldadas de un movimiento.

**La norma, para que no haya un sexto:**

> **En este fork, cualquier `#elif XASH_XBOX` detrás de una rama que ya es cierta en
> este target es código muerto por construcción.** nxdk define `_WIN32`, luego
> `XASH_WIN32` vale 1 siempre que `XASH_XBOX` vale 1: un `#if XASH_WIN32` delante se
> queda con la cadena entera. Las dos formas correctas, ambas ya en uso en el fork:
> poner la rama Xbox **primero**, o escribir la de escritorio como
> `XASH_WIN32 && !XASH_XBOX`. Cada vez que se toque un condicional de plataforma,
> pasar `research/ofx-audit-elif-xbox.py`, que busca el patrón en todo el árbol.

La auditoría no fue un grep a ojo: el script reconstruye cada cadena `#if/#elif` de
todos los `.c/.h` del fork y clasifica cada `#elif` con XBOX según si alguna rama
anterior puede ganarle. Encontró 21 cadenas; tras el arreglo, **ninguna muerta**. Los
casos que parecían sospechosos y no lo son, dicho en voz alta:

- `common/port.h:64` y las de `identification.c`: la rama anterior es `#if !XASH_WIN32`,
  que en Xbox es **falsa** — no eclipsa, la rama Xbox sí se alcanza. (El primer
  clasificador las marcó mal por no mirar la negación; se revisaron a mano las 21.)
- Las cadenas que empiezan por `#if XASH_SDL...`: `XASH_SDL` **no está definido** en el
  build Xbox (comprobado en los `DEFINES` del `c4che` de waf), valen 0 y caen bien.
- `platform.h`: `Platform_NanoSleep` en Xbox cae al `#else return false`. No es de esta
  familia (no hay rama Xbox escrita que se pierda), pero queda anotado como deuda menor.

### 30.4 El trazador de vuelta en pantalla

Desde §23 la pantalla dejó de ser testigo: `pbgl_init` toma el CRTC y `debugPrint`
escribe en un framebuffer que ya nadie muestra. El arreglo tiene tres piezas, todas en
`xbox_trace.c` + dos ganchos en `vid_xbox.c`:

1. **Un anillo de 10 líneas × 76 columnas.** Todo lo que pasa por `Xbox_Trace` y
   `Xbox_TraceRaw` (o sea, también la consola del motor) se copia al anillo.
2. **`Xbox_TraceOverlayDraw()`, llamado desde `GL_SwapBuffers` después de cada
   `pbgl_swap_buffers()`.** Repinta el anillo entero, cada frame, arriba a la
   izquierda, blanco sobre negro. El swap siguiente lo borra con el frame viejo y la
   llamada siguiente lo repinta: nunca tapa nada más de un frame.
3. **El búfer de destino no se supone: se lee.** pbkit reprograma `PCRTC_START`
   (0xFD600800) en cada flip (`pbkit.c:812`), así que el overlay lee ese registro en
   cada draw y convierte el físico a puntero con el mapeo fijo del kernel
   (`0x80000000 | phys` — la misma región en la que viven los punteros de
   `XVideoGetFB`). Al doble búfer no se le puede esconder el texto.

Los glifos son la fuente de debug de nxdk (`hal/font_unscii_16.h`, 8×16, dominio
público), incluida directamente del header que usa `debug.c` — el overlay no le debe
nada a `debugPrint` ni a su idea de cuál es el framebuffer actual. Dos caminos que se
descartaron y por qué: `XVideoSetFB` escribe el CRTC y se pelearía con el flip de
pbkit; y retargetear `debugPrint` no se puede porque `synchronizeFramebuffer()` machaca
`SCREEN_FB` con `XVideoGetFB()` en cada llamada.

Coste: ~10 líneas × 76 columnas × 8×16 px × 4 B ≈ 0,4 MB de escrituras por frame en el
peor caso (todas las líneas llenas); en la práctica las líneas cortas pintan solo lo que
ocupan. Un `sfence` al final, el mismo que `XVideoFlushFB`.

**Qué debería verse en la próxima ejecución:** las últimas 10 líneas del trazador
flotando sobre lo que pbgl muestre (aunque sea negro), con el latido cambiando cada
segundo. La lectura es binaria: si la pantalla late, el port vuelve a tener testigo
visual y cada iteración se ahorra el viaje por FTP; si el log late y la pantalla no,
el overlay tiene mal alguna de sus tres suposiciones (30.6) y el log lo dirá.

### 30.5 Qué haría falta para `map c1a0` desde `xash.cmd`

Es la siguiente ronda, y es corta:

1. **Subir `pak0.pak`** (313.133.320 B, `game/valve/pak0.pak`) a
   `E:\Apps\xash\valve\`. `c1a0.bsp` — y casi todo lo que referencia: modelos,
   sonidos, sprites — vive **dentro** del pak; los `maps/*.bsp` sueltos de `game/valve`
   son solo los multijugador. El filesystem ya sabe leerlo: `FS_AddGameDirectory`
   registra los `.pak` igual que registró los `.wad` de §23. A ~5-8 MB/s de FTP son
   uno o dos minutos, una vez.
2. **`+map c1a0` en `xash.cmd`**, que quedaría `-noip -dev 2 +map c1a0`. Los argumentos
   `+` los recoge `Cbuf_ExecStuffCmds` (`host.c:1338`), que corre justo antes de entrar
   al bucle de frames — exactamente el mecanismo que usaría un `map` tecleado en
   consola, sin teclado.

Lo que ese intento va a ejercitar por primera vez, y dónde puede morir:

- **La frontera motor↔gamedll entera** (§28.6, sin auditar): `SV_SpawnServer` llama a
  cientos de `enginefuncs` y resuelve entidades contra los 499 exports de §24.2. Aquí
  es donde el siguiente `GiveFnptrsToDll` está esperando, si lo hay. Con latido,
  consola y overlay, esta vez se verá dónde.
- **La memoria.** `XASH_LOW_MEMORY=2` está puesto, pero nadie ha medido el presupuesto
  (§29.6): mapa + texturas del BSP + modelos sobre 64 MB, con el pushbuffer de 2 MB y
  los framebuffers de pbgl ya dentro.
- **El loopback con `-noip`.** El singleplayer de Xash va por `NS_LOOPBACK`, que en
  teoría no necesita la pila de red que `-noip` apaga. En teoría. Sin verificar.
- **El input sigue sin existir**: si el mapa carga, será una cámara quieta en
  c1a0. Ver algo dibujado ya sería la foto de la sesión.

El orden correcto sigue siendo: primero validar esta ronda (que el latido y el overlay
funcionan), después el pak y el `+map` — si algo muere cargando el mapa, hay que poder
distinguirlo de un bucle que nunca corrió.

### 30.6 Qué NO está verificado

- **Nada de esta ronda ha corrido en hardware.** Latido, `-dev 2`, defaults nuevos y
  overlay: compilado, SSE2 = 0, subido — no ejecutado. Exactamente el mismo estado que
  pbgl tuvo entre §17.2 y §23.
- **El overlay descansa en tres suposiciones razonadas y cero comprobadas:** que el
  scanout de pbgl es 32 bpp con el pitch que dice `pb_back_buffer_pitch()`; que
  `0x80000000 | (PCRTC_START & 0x03FFFFFF)` es un puntero escribible al búfer mostrado;
  y que `GL_SwapBuffers` se llama cada frame aunque no haya nada que dibujar.
- **El coste real de `-dev 2`** sobre el tiempo de arranque (disco síncrono por línea)
  no está medido.
- Todo lo de §29.6 que no toca esta ronda sigue igual: menú sin verse, 838 choques del
  cliente sin ejercitar, frontera motor↔gamedll sin auditar, memoria sin medir.

### 30.7 Estado

En la consola: `default.xbe` **4.620.288 B** (nuevo) y `xash.cmd` con `-noip -dev 2`
(verificado). El log rescatado antes de pisar el XBE era el mismo de §29, 100 líneas.
Ficheros del fork modificados: **18** (entra `common/defaults.h`). Nuestros:
`xbox_trace.c` +~150 líneas de overlay, ganchos en `vid_xbox.c` y `host.c`, y dos
scripts nuevos en `research/`: `ofx-audit-elif-xbox.py` (el auditor de la norma de
30.3) y `ofx-xbox-build.sh` (build incremental, sin el distclean del verify).

La próxima sesión empieza encendiendo la consola y mirando la pantalla: si el latido
se ve, el port ha recuperado el testigo que perdió en §23.

## 31. Paso 2o — El primer frame moría en una aserción de SDL, y la caza en frío de sus hermanos

**Resumen: el log de §30 (272 líneas, la consola del motor por fin hablando) enseñó que
el bucle de frames SÍ arrancó y murió dentro del primer segundo: el gamedll de cliente
pidió el centro de la ventana, no hay ventana, la aserción de SDL no encontró messagebox
que mostrar y respondió con `abort()`. Arreglado el sitio, instalado un manejador de
aserciones que convierte esa clase entera de muertes en líneas de log, corregido el
auditor con el valor real de `XASH_SDL` — que encontró un hermano de verdad durmiendo
en la cadena MESSAGEBOX — y documentado con neón que el printf de este target tira los
floats y desalinea lo que venga detrás.**

| | |
|---|---|
| ¿Latido? | No llegó a emitir: el bucle murió en el frame ~1, antes del primer segundo |
| ¿`-dev 2`? | **Sí**: 272 líneas, toda la consola del motor en el log |
| Muerte | `pfnGetWindowCenterX` → `SDL_GetWindowPosition(NULL)` → aserción → `abort()` |
| Auditor v3 | `XASH_SDL=2` real, dos contextos; **1 hermano nuevo cazado** (MESSAGEBOX) |
| printf | **cualquier `%f` del proyecto sale vacío y desalinea los args siguientes** |
| `default.xbe` | 4.620.288 B, SSE2 = 0 en 420 objetos, **subido**; `xash.cmd` sin cambios |

### 31.1 Lo que dijo el log de la ronda §30

`research/logs/xash-boot-20260817-150636.log`. Las cuatro preguntas, en orden:

1. **No hay latido, pero no porque el bucle no corriera.** Entró (línea 0102, 3,7 s),
   completó al menos un frame — detectó el mando, imprimió `Time to first frame` — y
   murió antes de que el latido emitiera su primera línea, que es al segundo. La
   distinción que el latido existe para hacer («¿entró o corrió?») quedó respondida por
   los efectos colaterales del propio frame 1.
2. **`-dev 2` funcionó entero.** `Developer level: 2`, el filesystem hablando (`Adding
   WAD/ZIP/directory`), los 28 `GL_CheckExtension`, y warnings útiles que estaban mudos:
   falta `gfx/palette`, faltan los sprites de muzzleflash (cosas que viven en
   `pak0.pak`), `vgui_support` ausente (esperado, no existe en este port).
3. **Del overlay no se pudo afirmar nada**: `overlay up: 640x480 pitch 2560` salió, pero
   con un solo swap antes del abort no hay distinción posible entre «no pinta» y «no
   llegó a pintar». Esta ronda instrumenta al pintor (31.5).
4. **Dónde murió, con la cadena entera:** el gamedll de cliente centra el ratón en su
   primer frame y llama a `gEngfuncs.GetWindowCenterX` → `cl_game.c:2069`
   `SDL_GetWindowPosition( host.hWnd )` con `host.hWnd == NULL` porque aquí no hay
   ventana → aserción en `SDL_video.c:1908` → el manejador por defecto de SDL intenta
   un messagebox (`No message system available`, visible en el log) → «oh well», dice
   el comentario de `SDL_assert.c`, y devuelve `SDL_ASSERTION_ABORT` → `abort()` con el
   vigilante ya desarmado → título muerto, consola fuera de la red, pantalla negra.
   **La no-red ya lo decía desde fuera**: ni `Sys_Error` ni vigilante — cualquiera de
   los dos habría devuelto la consola al dashboard con su FTP.

### 31.2 `XASH_SDL=2`: la corrección al auditor, y el hermano que encontró

**§30.3 afirmó que `XASH_SDL` no está definido en este build, y es falso.** Se miró la
lista global de `DEFINES` del c4che de waf; el define es **por target**:
`engine/wscript:305` añade `XASH_SDL=2` solo a las TUs del engine (y en Xbox excluye
`vid_sdl*` de las fuentes, que es por lo que `vid_xbox.c` corre aunque el macro diga
`VIDEO_SDL`). El `Platform_Init (SDL2 on this target)` del log era literal: input,
sonido y timer de este port son SDL2 sobre nxdk. ref/ y filesystem/ compilan sin
`XASH_SDL`.

El auditor v3 modela los dos contextos (engine con SDL=2; el resto sin; `common/*.h`
evaluado en ambos) y corrige además el fallo de la v1 con las negaciones (`#if
!XASH_WIN32` es falso en Xbox: no eclipsa). Sobre las 21 cadenas, marcó 5; revisadas a
mano las 5:

| Cadena | Veredicto |
|---|---|
| `defaults.h` VIDEO (`#if XASH_SDL` → `#elif XASH_XBOX`) | **Muerta en el engine, benigna hoy**: `XASH_VIDEO` queda en `VIDEO_SDL`, pero nadie compara con `VIDEO_XBOX` — la selección real la hace el wscript excluyendo `vid_sdl*`. **No se toca a propósito**: reordenarla dejaría `XASH_INPUT`/`XASH_SOUND` sin definir y rompería lo que funciona. Documentada aquí para que el macro mentiroso no engañe a nadie más. |
| `defaults.h` MESSAGEBOX | **Hermano de verdad, arreglado.** Ver abajo. |
| `defaults.h` TIMER | En Xbox gana `TIMER_SDL`, correcto (no existe TIMER_XBOX). |
| `platform.h:197` NanoSleep | `XASH_SDL == 3` es falso (SDL=2); en Xbox cae al `#else return false`. Deuda menor, no de la familia. |
| `identification.c:371` | La rama marcada es la de escritorio, correctamente excluida. |

**El hermano:** con `XASH_SDL >= 2` cierto, la cadena MESSAGEBOX elegía `MSGBOX_SDL`, y
la implementación Xbox de `Platform_MessageBox` que el fork lleva en `sys_con.c:443`
**era código muerto** — mientras cada messagebox real iba a un
`SDL_ShowSimpleMessageBox` que en esta máquina solo sabe decir «No message system
available». Estaba a la vista en el log de la muerte. Arreglo doble: la rama
`XASH_XBOX` puesta **primero** en la cadena (con comentario), y la implementación
reescrita sobre `Xbox_Trace`, que llega a los cuatro testigos (pantalla, anillo del
overlay, serie, disco) en vez de a un `debugPrint` que pbgl dejó sin pantalla.

La cuenta queda así: la familia del preprocesador («código Xbox que el compilador nunca
ve») suma **seis** con éste. Y `pfnGetWindowCenterX/Y` estrena una familia nueva —
**«código SDL que asume ventana»** — cuyo tratamiento no es cazarlos uno a uno sino el
manejador de 31.3, que convierte al que quede en una línea de log.

### 31.3 Los tres arreglos

1. **`SDL_SetAssertionHandler`, primera línea de `main()`** (`launcher.c`), antes que
   el trazador siquiera. Registra `SDL assert IGNORED: '<cond>' at <file>:<line>` vía
   `Xbox_Trace` y devuelve `SDL_ASSERTION_ALWAYS_IGNORE` — ALWAYS, para que un aserto
   dentro del bucle de frames cueste una escritura síncrona en total, no una por frame.
   Válido antes de `SDL_Init`: solo almacena el puntero.
2. **`pfnGetWindowCenterX/Y`**: el bloque SDL guardado con `&& !XASH_XBOX`. Sin
   ventana no hay offset que sumar: el centro es `host.window_center_x` a secas, que
   `vid_common.c:51` pone a `ancho/2` en `VID_SetMode`.
3. **MESSAGEBOX**, arriba.

### 31.4 ⚠⚠ EL PRINTF DE ESTE TARGET TIRA LOS FLOATS — Y CORROMPE LO QUE VENGA DETRÁS ⚠⚠

La pdclib de nxdk (`_PDCLIB_print.c:452`) tiene los `case 'f' 'F' 'e' 'E' 'g' 'G' 'a'
'A'` como **`break` vacío**: el especificador se consume, `base` queda en 0 y la etapa
de salida no emite nada. Dos consecuencias, una visible y una traicionera:

1. **Todo `%f`/`%g`/`%e` del proyecto entero imprime cadena vacía.** El log de §30 está
   lleno: `Rendering Bitmap Font(16, 500) took  seconds`, `player_mins:   `. El motor
   lleva imprimiendo floats vacíos en esta máquina desde el primer arranque.
2. **El `va_arg` del double NUNCA se consume.** Cualquier conversión *posterior* a un
   float en la misma cadena lee el argumento equivocado: un `%s` detrás de un `%f`
   toma los primeros 4 bytes del double como puntero. **Una línea de log con basura —
   o un cuelgue dentro de un printf — puede ser esto y no el bug que se investiga.**

**Norma:** en las trazas nuestras, floats prohibidos — entero escalado, como el latido
(`fps ×10` → `%u.%u`). Auditado el árbol entero: **0 conversiones float** en
`Xbox_Trace`/`XTRACE` (la del latido de §30 era la única y salió con esta ronda). Las
del motor se quedan: toda línea del motor con un float se lee con desconfianza, y jamás
se diagnostica a partir del hueco. Si algún día hace falta de verdad, la salida es
enlazar un `vsnprintf` de reemplazo con soporte de coma flotante por delante de pdclib.

### 31.5 El latido y el pintor, ahora con testigos propios

- El latido imprime `heartbeat: frame N, 59.8 fps` con la fracción hecha de enteros.
- `Xbox_TraceOverlayDraw` traza `overlay draw #N: fb=%p` en su primera llamada y luego
  una vez cada 3600 (~1/min a 60 fps), **antes** de pintar, para que salga aunque el
  pintado casque. El puntero `fb` alternando entre dos valores es el doble búfer hecho
  visible; un `fb` fijo o fuera de rango sería el diagnóstico de §30.6 servido.

Lectura del próximo log, por orden de información: `overlay draw #1` dice que el swap
corre; el latido dice que el bucle vive y a cuánto; `SDL assert IGNORED` diría que la
familia nueva tiene más miembros — cada uno con su fichero y su línea, gratis.

### 31.6 Qué NO está verificado

- **Todo lo de esta ronda**: manejador, center-fix, MSGBOX, latido entero, trazas del
  pintor. Compilado, 0 SSE2, subido — no ejecutado. Las tres suposiciones del overlay
  (§30.6) siguen sin comprobar; esta vez el log dirá cuál falla, si falla.
- El manejador cubre **aserciones**. Una llamada SDL sin ventana que casque de otra
  manera (deref directo de NULL) sigue siendo una muerte súbita.

### 31.7 Estado

`default.xbe` 4.620.288 B subido (mismo tamaño que §30: el alineado de secciones se
tragó la diferencia); `xash.cmd` sigue en `-noip -dev 2`, sin `+map` a propósito.
Ficheros del fork modificados: **18** — `cl_game.c`, `launcher.c` y `sys_con.c` ya
estaban en la lista por las trazas. El auditor v3 en `research/ofx-audit-elif-xbox.py`.

**Sobre `pak0.pak` y `+map c1a0`:** subir el pak **ya** tiene sentido — es independiente
de todo lo anterior, el filesystem lo registra solo, y sus 313.133.320 B tardan uno o
dos minutos de FTP que no bloquean nada. El `+map` conviene que espere **un** log con
latido: si se añade ahora y algo muere, no se sabrá si murió el mapa o seguía muerto el
frame 1, y distinguir eso es exactamente lo que esta ronda ha comprado. Con un log que
lata, añadir `+map c1a0` a `xash.cmd` cuesta un segundo de FTP.

## 32. Paso 2p — HITO: el menú de Half-Life en pantalla, a 59,9 fps

**El primer render real del port. El menú principal de Half-Life, dibujado por pbgl en
la GPU de una Xbox de 2001, con el overlay del trazador encima marcando `heartbeat:
frame 231, 59.9 fps`. Hay foto. Todo lo que las tres últimas secciones construyeron a
ciegas funcionó a la primera: el latido late, el overlay pinta, el manejador de
aserciones no tuvo que ignorar ni una, y START+BACK — pendiente desde §21 — devolvió
la consola al dashboard limpiamente. Y la sesión regaló el siguiente bug, ya
diagnosticado: los paks y los backslashes no se hablan.**

| | |
|---|---|
| Menú | **en pantalla**, renderizado por pbgl |
| Latido | 107 líneas, 59,9 fps estables (50,2 el primer segundo, arranque) |
| Overlay | **probado**: `draw #1 fb=0x839d0000`, `#3600 fb=0x838a4000` — el doble búfer, visible |
| `SDL assert IGNORED` | **0** — el arreglo de `pfnGetWindowCenter*` era el único que faltaba |
| `pak0.pak` | montado: `Adding PAK: D:\valve\pak0.pak (3195 files)` |
| START+BACK | **verificado por primera vez**: `returning to dashboard in 5 s: START+BACK` |
| Siguiente bug | `maps\t0a0.bsp` no carga **del pak** — diagnóstico en 32.2 |

### 32.1 Las tres suposiciones del overlay: las tres eran buenas

El log (`research/logs/xash-boot-20260817-162110.log`, 404 líneas, 114 s de sesión)
cierra lo que §30.6 dejó abierto: `GL_SwapBuffers` corre cada frame (60 draws por
segundo de latido), el mapeo `0x80000000 | PCRTC_START` apunta a memoria que se muestra
(la foto es la prueba), y el pitch de 2560 con 32 bpp era el correcto. El puntero `fb`
alternando entre `0x839d0000` y `0x838a4000` en las trazas del pintor es exactamente la
firma que 31.5 pedía: al doble búfer ya no se le puede esconder nada.

El latido, además, resultó ser un profiler gratis: 59,9 clavados en el menú, 42-46
después de un `Host_Error` con su recuadro encima. Primer dato de rendimiento real del
port, sin instrumentar nada más.

### 32.2 El regalo de la sesión: `t0a0` no carga, y ya se sabe por qué

Se probó «Hazard Course» desde el menú. El motor intentó el mapa y:

```
Host_Error: Could not load model maps\t0a0.bsp from disk
Server was killed due to an error
```

Dos cosas, la buena primero: **el motor no murió**. `Host_Error` mató al servidor, el
menú siguió, el latido siguió (42 fps), y START+BACK salió limpio. La resiliencia que
§31 compró — errores que antes eran pantallas negras — ya se nota.

La mala, con el diagnóstico completo, **aún sin arreglar y sin verificar**:

- El backslash del mensaje delata la ruta: `Mod_LoadModel` pasa el nombre por
  `COM_FixSlashes` justo antes de `FS_LoadFile` (`engine/common/model.c:314`).
- `COM_FixSlashes` tiene una rama `#if XASH_XBOX` **añadida por el fork**
  (`public/crtlib.h:372`) que convierte `/` en `\` — «separador nativo». Upstream hace
  lo contrario **incluso en Windows**: normaliza a `/` y deja que la capa de
  filesystem traduzca al abrir ficheros reales.
- `FS_FindFile_PAK` busca por **bisección con `Q_stricmp`** contra la tabla del pak
  (`filesystem/pak.c:231`), y el formato PAK guarda las rutas con barra normal:
  `maps/t0a0.bsp`. Backslash ≠ barra → «no existe», con el fichero a tres entradas de
  distancia entre los 3195.
- Por eso **todo lo suelto funcionaba** — CreateFile acepta backslashes — y lo primero
  que de verdad vive dentro del pak (un `.bsp`) fue lo primero en fallar. No es que
  los paks no funcionen en Xbox: es que las rutas llegan torcidas a su tabla.

**Predicción explícita, para contrastar con el próximo log:** `+map c1a0` desde
`xash.cmd` morirá con `Could not load model maps\c1a0.bsp from disk` mientras esa rama
de `COM_FixSlashes` exista. El arreglo candidato es quitar la rama Xbox (dejar el
comportamiento upstream, barras normales) y dejar que `filesystem_stdio` traduzca en
la frontera con el sistema — exactamente el reparto de papeles que upstream ya usa en
Windows. Un cambio de tres líneas, pero toca **todas** las rutas del motor: no se mete
sin su propia ronda.

### 32.3 Deuda menor, anotada y a propósito sin tocar

Cosmética, del propio log:

- **Localización**: `Localize_AddToDict( resource/gameui_english.txt ): couldn't open
  file` — ídem `mainui_english.txt` y `valve_english.txt` (la pista venía sonando
  desde §23). Los rótulos del menú salen con su clave `GameUI_*` en vez del texto. Son
  ficheros de texto que no están en los assets subidos.
- **`gamestartup`**: `FS_OpenStream: couldn't open "media/gamestartup"` — la música
  del menú. Silencio en su lugar.

Ninguna de las dos toca el camino del mapa. Cuando haya un mapa cargando, se suben los
`resource/*.txt` y el `media/` que falten y listo.

### 32.4 Estado

`xash.cmd` = `-noip -dev 2 +map c1a0` (subido y releído). `default.xbe` sin cambios
desde §31 — esta sección no ha compilado nada; ha leído un log y ha visto una foto.

De la lista de «no verificado» que se arrastraba: latido ✔, overlay ✔ (las tres
suposiciones), `-dev 2` ✔, manejador de aserciones ✔ (cero disparos), START+BACK ✔,
`pak0.pak` montado ✔. Quedan: el menú *interactivo* (la foto es del menú quieto; nadie
ha navegado con el mando todavía), la frontera motor↔gamedll bajo un mapa de verdad, y
el presupuesto de memoria.

El port dibuja. Lo siguiente es que cargue un mapa, y el bug que lo impide ya tiene
nombre, fichero y línea.

## 33. Paso 2q — `+map c1a0`: la misma bala, sin chaleco

**El reinicio al dashboard no fue un cuelgue ni el vigilante: fue una salida ordenada de
libro, con el error de las barras de §32.2 apareciendo letra por letra donde se
predijo. La diferencia con el `t0a0` del menú no está en el bug — es el mismo — sino en
el momento: `host.c:710` convierte cualquier `Host_Error` de los tres primeros frames
en `Sys_Error` fatal, y un `+map` de línea de órdenes se ejecuta exactamente ahí. La
hipótesis del encargo era correcta, con un matiz medible: no murió *antes* del bucle,
murió *dentro*, en el frame 1-2, que para el motor sigue siendo «el arranque».**

| Pregunta | Respuesta del log |
|---|---|
| ¿El error de barras? | **Sí, exacto**: `Could not load model maps\c1a0.bsp from disk` |
| ¿`returning to dashboard`? | Sí: **`caught an error`** — ni vigilante ni cuelgue |
| ¿Ordenado o a medias? | **Ordenado**: shutdown completo, configs escritas, audio cerrado |
| ¿Antes o después del bucle? | **Dentro**: `entering frame loop` (4453 ms) → `overlay draw #1` (4510) → `Spawn Server: c1a0` → error (4600) |

### 33.1 La línea temporal, al milisegundo

`research/logs/xash-boot-20260817-163232.log`, 294 líneas:

```
[0104     4453] Host_Main: entering frame loop -- the engine is up
[0105     4510] overlay draw #1: fb=0x839d0000
Spawn Server: c1a0
[0106     4600] *** Sys_Error: Host_ErrorInit: Could not load model maps\c1a0.bsp from disk
[0107     4601] MSGBOX Xash Error: Host_ErrorInit: Could not load model maps\c1a0.bsp from disk
Note: Issuing host shutdown due to reason "caught an error"
Server shutdown / CL_Shutdown() / configs / audio      <- secuencia entera, limpia
[0108     4813] returning to dashboard in 5 s: caught an error
```

El `map` de `xash.cmd` no corre en `Cbuf_ExecStuffCmds` a pelo: la orden queda en el
búfer y se ejecuta en el primer `Cbuf_Execute` del bucle — por eso `Spawn Server`
aparece *después* del primer swap. Murió a 147 ms de vida del bucle, antes del primer
latido (que emite al segundo).

### 33.2 Por qué `t0a0` dejó el menú vivo y `c1a0` mató el motor

`engine/common/host.c:710`, sin tocar por nosotros, upstream puro:

```c
if( host.framecount < 3 )
    Sys_Error( "%sInit: %s", __func__, hosterror1 );   // -> Host_ErrorInit
```

`Host_Error` tiene dos regímenes. Con el motor rodado (el `t0a0` del menú: frame
~5700), mata el servidor, deja el menú y sigue. En los tres primeros frames considera
que un error es un error *de arranque* y lo escala a `Sys_Error` — no hay menú al que
volver porque, a efectos del motor, todavía no ha terminado de nacer. Un `+map` de
línea de órdenes cae siempre en el segundo régimen: **misma bala, sin chaleco**. La
hipótesis del encargo, confirmada y con el mecanismo señalado.

Consecuencia operativa, que conviene dejar escrita: **aunque se arreglen las barras,
cualquier error de un `+map` de `xash.cmd` será siempre salida al dashboard**, no menú
con recuadro. Para una consola es un comportamiento razonable — error → dashboard con
el log en el disco — pero hay que saberlo al leer futuros logs: `Host_ErrorInit` en un
arranque con `+map` no señala una regresión del motor, señala el mapa.

### 33.3 Todo lo demás funcionó — incluida una pieza de §31 que estrenaba

- **La predicción de §32.2 queda confirmada por la segunda vía.** Menú y línea de
  órdenes mueren en la misma línea de `Mod_LoadModel` con la misma ruta torcida. Dos
  caminos, un bug: la rama `#if XASH_XBOX` de `COM_FixSlashes` (`public/crtlib.h:372`)
  contra la tabla de barras normales de `FS_FindFile_PAK` (`filesystem/pak.c:231`).
- **`MSGBOX Xash Error: …` es la primera línea real del `Platform_MessageBox` de Xbox**
  que §31 resucitó de código muerto. Antes de esa ronda, este mismo error habría
  intentado un messagebox de SDL y el log habría dicho `No message system available`.
- **La salida ordenada es la de §21.5** haciendo exactamente su trabajo: shutdown
  entero, log completo en disco, consola de vuelta en la red — este análisis existe
  porque el FTP respondió a la primera.
- `overlay draw #1: fb=0x839d0000` — mismo búfer de primer frame que en §32, tercera
  ejecución consecutiva del overlay sin sorpresas.

### 33.4 Qué NO está verificado

- **El arreglo de las barras sigue sin existir.** Diagnóstico completo desde §32.2,
  dos confirmaciones independientes, cero líneas cambiadas. Es la próxima ronda, y
  toca todas las rutas del motor: `COM_FixSlashes` vuelve al comportamiento upstream
  (normalizar a `/`) y la traducción a backslash queda donde upstream la tiene, en la
  frontera del filesystem con el sistema operativo.
- Si al quitar la rama aparece algún camino Xbox que de verdad necesitara backslashes
  (montajes de `D:`, `CreateFile` con rutas relativas), saldrá en el log de esa ronda.

### 33.5 Estado

Consola en el dashboard, `xash.cmd` = `-noip -dev 2 +map c1a0` (se deja: la próxima
ronda quiere exactamente este arranque para validar el arreglo), `default.xbe` sin
cambios desde §31. Esta sección no ha compilado nada.

Tres ejecuciones, tres logs, un solo bug en pie. Nombre, fichero, línea y dos
confirmaciones. La ronda de las barras es lo único entre el port y su primer mapa.

## 34. Paso 2s — La ronda de las barras: VFS en `/`, frontera en `\`

**La rama Xbox de `COM_FixSlashes` está fuera y el VFS habla barra normal en todas
partes, como upstream. Pero la auditoría previa cambió la forma del arreglo: quitarla
a secas habría roto todo lo suelto, porque en esta máquina nadie traduce las barras
por ti. El arreglo completo es el reparto de papeles que upstream ya usa en Windows —
VFS con `/`, traducción en la frontera con el OS — solo que aquí la frontera hay que
escribirla a mano: `FS_SysPath`, aplicada en los nueve syscalls de ruta del
filesystem. Y de paso, quince constructores de rutas de los guards del fork volteados
a `/` para que el VFS no quede con separadores mixtos. Compilado, 0 SSE2, subido;
`xash.cmd` intacto para validar contra el mismo fallo.**

### 34.1 La auditoría que pedía el encargo, y lo que cambió

La pregunta era: ¿hay algo más que dependa del backslash? La respuesta resultó ser
**todo lo suelto**, y se comprobó en fuente, no por suposición:

- `CreateFileA` de nxdk (`lib/winapi/fileio.c`) hace `RtlInitAnsiString` y entrega la
  ruta **tal cual** a `NtCreateFile`. No hay capa Win32 que convierta `/` en `\` — esa
  capa es exactamente lo que un PC tiene y esta consola no.
- `fopen`/`_open` de pdclib (`_PDCLIB_open.c`) → `CreateFileA`, también verbatim.
- El object manager de NT parte las rutas **solo** por `\`.

O sea: «CreateFile traga ambas» — la premisa cómoda — **es falsa aquí**. Y con toda
probabilidad es la razón por la que el autor del fork escribió su rama: los ficheros
sueltos con `/` no le abrían. Su rama era correcta para su combinación (todo suelto,
sin paks) y veneno para la nuestra (pak0.pak con su tabla de barras normales, por
definición del formato). **Séptimo bug de la familia** — §19.8 lista los cuatro
primeros, §29.5 el quinto, §31.2 el sexto — y queda apuntado con el nombre que merece:
*rama Xbox del autor, correcta para su combinación, veneno para la nuestra.*

Con `FS_SysPath` en la frontera, el interrogante del kernel deja de importar: al OS
siempre le llega `\`, hable el VFS como hable.

### 34.2 El cambio, pieza a pieza

1. **`public/crtlib.h`**: la rama `#if XASH_XBOX` de `COM_FixSlashes` eliminada;
   comportamiento upstream en todas las plataformas (normalizar a `/`, comprimir
   duplicadas), con comentario que apunta aquí.
2. **`filesystem/filesystem.c`**: `FS_SysPath( path, out )` — `/`→`\` en un búfer
   local — aplicada en los **nueve** puntos donde una ruta cruza al OS:
   `_open` (`FS_SysOpen`), `_stat` ×4 (`FS_SysFileTime`, `FS_SysFileExists`,
   `FS_SysFolderExists`, `FS_SysFileOrFolderExists`), `_findfirst` (patrón de
   `listdirectory`, convertido in situ), `_mkdir` (`FS_CreatePath`), `rename`
   (`FS_Rename`) y `remove` (`FS_Delete`).
3. **Quince constructores de rutas** en guards `XASH_XBOX` del fork, volteados de
   `\` a `/`: los searchpaths de `FS_AddGameHierarchy` (`_hd`, `_lv`, `_addon`,
   idioma, `custom`, `downloaded`), `gamedir_path`/`gameinfo.txt`/`liblist.gam`,
   el barrido de rodir/rootdir, el camino directo de `FS_FindFile` y el
   `dllInfo->fullPath` de `FS_FindLibrary`. Cuatro de ellos quedaron idénticos a su
   `#else` upstream y el guard se eliminó. La semántica propia del fork — rutas
   absolutas `D:/...` donde upstream usa relativas — se conserva; solo cambia el
   separador.

   Sin esto, el VFS habría quedado mixto (`D:/valve\...`), que funciona en los
   syscalls (la frontera lo arregla) pero reabre una clase entera de fallos
   silenciosos: cualquier `stricmp` entre la forma con `/` y la forma con `\` del
   mismo directorio — la detección de searchpath duplicado, sin ir más lejos — deja
   de casar.
4. **Dejado a propósito**: los retoques Xbox de `dir.c` (aceptan ambos separadores al
   trocear — inofensivos con entrada uniforme) y el «try backslashes» de la ruta
   relativa de gamedll (tolerancia de *comparación*, no construcción).

### 34.3 Qué NO está verificado

- **Nada ha corrido.** La predicción para el próximo log, en orden: `Adding PAK`
  igual que en §32; `Spawn Server: c1a0`; y donde §33 decía `Could not load model
  maps\c1a0.bsp` — con backslash — ahora o bien el mapa carga, o bien falla *más
  adentro* con otra cara. Cualquiera de las dos es progreso; la segunda diría que la
  frontera motor↔gamedll de §28.6 ha cobrado su siguiente pieza.
- Si algo suelto se rompe por el cambio de régimen, la firma será `couldn't exec
  config.cfg` (que §32 escribió y §33 leyó bien) o errores de escritura de configs al
  salir. El log de arranque en sí no corre peligro: `xbox_trace.c` usa rutas
  literales `D:\...` que no pasan por nada de esto.
- El presupuesto de memoria con un BSP cargado sigue sin medirse — si c1a0 entra,
  esa es la siguiente incógnita en la cola.

### 34.4 Estado

`default.xbe` 4.620.288 B (idéntico en tamaño otra vez: el alineado se lo traga),
SSE2 = 0 en 420 objetos, subido. `xash.cmd` **sin tocar** y releído para confirmar:
`-noip -dev 2 +map c1a0` — el mismo arranque que §33, contra el mismo fallo, con el
arreglo debajo. Ficheros del fork modificados: **20** (entran `public/crtlib.h` y
`filesystem/filesystem.c`).

Si el próximo log dice `heartbeat` con un mapa debajo, el port habrá pasado de
dibujar un menú a correr Half-Life.

## 35. Paso 2t — Las barras funcionan; el muro nuevo es la memoria

**El error cambió, y cambió como debía: ni una sola backslash en las 200 líneas del
log. `Adding PAK: D:/valve/pak0.pak (3195 files)` con la ruta limpia, el filesystem
entero hablando `/`, y el arreglo de §34 verificado donde manda la norma — en el
artefacto corriendo en la consola, no en el configure. El muro nuevo es otro y está
más adentro: `_Mem_Alloc: out of memory` decodificando un BMP del menú, con el tamaño
del alloc **comido por el bug del printf de §31.4**, que aquí dio su demostración en
vivo. Y una ironía de las buenas: este OOM es *consecuencia* del arreglo — el menú por
fin encuentra sus imágenes dentro del pak, y al decodificarlas se acaba la memoria.**

| Pregunta | Respuesta |
|---|---|
| ¿Cambió el error? | **Sí, por completo**: ya no hay `Could not load model`, ni backslashes |
| ¿Pak montado? | `Adding PAK: D:/valve/pak0.pak (3195 files)`, ruta limpia |
| ¿El .bsp? | **Ni se llegó a pedir**: murió antes de entrar al bucle, el `+map` nunca se ejecutó |
| ¿Hasta dónde? | Renderer up → menú cargado → **770 ms cargando imágenes** → OOM en `img_bmp.c:203` |
| ¿Salida? | Ordenada: `Sys_Error` → shutdown → `MSGBOX` (§31) → dashboard |

### 35.1 La verificación de §34, en el artefacto

`research/logs/xash-boot-20260817-165541.log`. Todas las rutas del filesystem salen
con `/`: el pak, el pk3, los cuatro wads, `D:/valve/`. En §33 la línea equivalente era
`D:\valve\pak0.pak`. El VFS uniforme de §34.2 está funcionando en hardware, y la
prueba más elocuente no es una ruta impresa sino el sitio del fallo: **el motor murió
decodificando una imagen que las cinco ejecuciones anteriores ni siquiera
encontraban.** `FS_FindFile_PAK` está sirviendo contenido del pak por primera vez.

### 35.2 El OOM, y el printf comiéndose la evidencia

```
*** Sys_Error: _Mem_Alloc: out of memory (alloc size ^1s,pri,ntf throw exception^7
 at ../engine/common/imagelib/img_bmp.c:203)
```

`img_bmp.c:203` es la reserva del bitmap decodificado: `image.size = width * height *
bpp; image.rgba = Mem_Malloc(...)`. La cabecera del BMP pasó la validación de tamaño
contra el fichero real (la comprobación de `:197`), así que no es una cabecera
corrupta pidiendo gigas: es una imagen legítima del pak y una reserva plausiblemente
modesta que **aun así falló**.

¿Cuánto pedía? No se sabe, y el porqué es exactamente §31.4: el mensaje formatea el
tamaño con `Q_pretifymem`, que hace `Q_snprintf( "%.*f %s", ... )` — pdclib consume la
precisión `*`, tira el double sin consumirlo, y el `%s` siguiente lee los bytes del
double como puntero. La basura `^1s,pri,ntf throw exception^7` es una cadena ajena de
la memoria del proceso. **La predicción de §31.4 («una línea corrupta puede ser el
printf, no el bug») se cumplió en el primer error importante que pasó por un float.**
Lo único que la ruta del código garantiza es que el valor no era integral en su bin —
o sea, la reserva superaba 1 KB. Nada más.

### 35.3 Lo que sí se sabe, y las hipótesis en orden

Murió a los 4,2 s, tras 770 ms de carga de imágenes del menú con el pak sirviendo de
verdad. En el aire, por orden de sospecha para la próxima ronda:

1. **Agotamiento real del montón de nxdk.** Nadie ha medido nunca el presupuesto
   (§29.6, deuda desde entonces): 64 MB totales menos XBE (4,6 MB), pilas, pushbuffer
   (2 MB), framebuffers de pbgl, tabla de 3195 entradas del pak, fuentes ya
   rasterizadas… y ahora los BMP de menú decodificados a RGBA. Puede simplemente no
   caber como está.
2. **Un límite artificial del malloc.** Cómo respalda nxdk su `malloc` (¿arena fija?
   ¿páginas bajo demanda?) quedó sin respuesta en una inspección rápida del
   toolchain. Si hay arena fija, el número mágico está ahí. El `xbox_sbrk.c` de
   §19.3 — escrito y nunca incorporado al build — es contexto, no diagnóstico: alimenta
   el *swap allocator* del motor, no el malloc del CRT.
3. **Una fuga o una reserva absurda previa** que dejó el montón exhausto antes del
   BMP. Menos probable: el arranque hasta ahí es idéntico al de §32, que vivió 114 s.

### 35.4 Efecto secundario de §34, detectado y acotado

`COM_StripDirectorySlash` (`filesystem_engine.c:470`, upstream) solo pela `/`. Antes
el rootdir `D:\` le pasaba por delante intacto; ahora `D:/` sí se pela y queda
**`D:`**. Consecuencias visibles en el log: `Adding directory: D:` (el searchpath
raíz) y los dos únicos constructores sin comprobación de separador — `downloaded` y
`custom` — producen `D:valve/downloaded/` y `D:valve/custom/`. **Hoy inofensivo**:
esos directorios no existen y todo lo real entra por `D:/valve/`, que se construye
con la forma consciente del separador. Pero es formato malo latente; el arreglo es
darles a esos dos builders el mismo `has_sep` que ya tienen sus hermanos. Una línea
cada uno, próxima ronda.

### 35.5 Lo que toca ahora

1. **Que el OOM diga su número**: el mensaje de `_Mem_Alloc` con el tamaño en bytes y
   enteros (`%u`), inmune a §31.4. Sin eso, todo lo demás es adivinar.
2. **Medir el montón**: total, usado y mayor bloque libre, trazado al entrar al menú
   y en el momento del fallo. El presupuesto de §29.6 deja de ser deuda o deja de ser
   misterio, una de las dos.
3. Con el número en la mano, decidir: ¿cabe Half-Life en lo que queda, o hay un
   límite artificial que levantar?
4. De paso: los dos builders de 35.4.

### 35.6 Estado

Consola en el dashboard. `xash.cmd` = `-noip -dev 2 +map c1a0`, sin tocar.
`default.xbe` de §34 en la consola. Esta sección no ha compilado nada: ha leído un
log, ha confirmado un arreglo y ha identificado el muro siguiente con su
instrumentación pendiente.

La cadena de muros, para perspectiva: pantalla negra (§21) → filesystem (§22) → GPU
(§23) → gamedll (§24-28) → arranque completo (§29) → testigos (§30-31) → menú visible
(§32) → barras (§33-34) → **memoria**. Cada uno más adentro que el anterior.

## 36. Paso 2u — El XBE pedía la mitad de la máquina: Limit64MB, y la memoria instrumentada

**El titular se confirmó sin encender la consola: el `default.xbe` llevaba en sus init
flags el bit `Limit64MB` — `0x00000005: [Mount Utility Drive] [Limit Devkit Run Time
Memory to 64MB]`, leído en el dump del artefacto — porque cxbe lo graba a fuego
(`tools/cxbe/Xbe.cpp:68`) y no ofrece opción para quitarlo. En una consola con 128 MB
soldados, cada arranque de este port ha corrido con la mitad de la máquina tirada a la
basura. El bit está fuera: parche post-cxbe en `xbox.py`, verificado en los bytes del
binario. Y la ronda de instrumentación completa va dentro del mismo XBE: el OOM ya
dice su número en enteros, y `Xbox_MemReport` traza RAM del kernel y mayor malloc
concedible en tres momentos del arranque. Nada ejecutado aún; el veredicto en runtime
lo dará la línea `mem at boot` del próximo log.**

| | |
|---|---|
| Init flags antes | `0x00000005` — `Limit64MB` **puesto** (offset 0x124: `05 00 00 00`) |
| Init flags ahora | `0x00000001` — verificado con `xxd` en el artefacto: `01 00 00 00` |
| Quién lo ponía | cxbe, hardcodeado (`Xbe.cpp:68`), sin opción de línea de órdenes |
| OOM con número | `_Mem_Alloc: out of memory (alloc size %u bytes ...)` — enteros, inmune a §31.4 |
| `Xbox_MemReport` | RAM total/libre del kernel + mayor malloc, en `at boot`, `at frame loop` y en cada fallo de alloc |
| `default.xbe` | 4.620.288 B, SSE2 = 0, **subido**; `xash.cmd` sin tocar |

### 36.1 El punto 4, comprobado donde manda la norma

En el artefacto, dos veces:

1. **Antes**: `build/engine/default.xbe.txt` (el dump que cxbe escribe al convertir)
   decía `Init Flags: 0x00000005 [...] [Limit Devkit Run Time Memory to 64MB]`, y
   `xxd -s 0x124` sobre el binario confirmaba `05 00 00 00`. No era el configure: era
   el XBE subido a la consola desde §20.
2. **Después**: el parche corre dentro de la tarea `cxbe` de `xbox.py`, justo después
   del relink — comprueba la firma `XBEH`, lee los flags en 0x124, exige que el bit
   esté puesto (si cxbe cambia sus defaults algún día, lo dice en voz alta en vez de
   fallar en silencio) y lo limpia. `xxd` sobre el `default.xbe` recién salido:
   `01 00 00 00`. `Mount Utility Drive` intacto.

   **Trampa documentada**: el `.txt` del `-DUMPINFO` se escribe *antes* del parche y
   seguirá mostrando el flag puesto. La verificación es sobre los bytes, no sobre el
   txt — está avisado en el propio código.

La semántica del bit, para el registro: con él puesto, el kernel limita el título a
64 MB *aunque la máquina tenga 128*. Es el mecanismo pensado para que los devkits de
128 MB emulen una retail. Nuestra consola lleva los 128 soldados; el título los verá
solo si además el kernel de la placa los expone — eso es exactamente lo que la línea
`mem at boot` del próximo log va a responder con un número: `ram total 65536 KB` o
`ram total 131072 KB`.

### 36.2 Los tres puntos de instrumentación

1. **El OOM dice su número.** Los dos `Sys_Error` de `zone.c` (`Mem_Alloc` y
   `Mem_Realloc`) imprimen en Xbox `alloc size %u bytes` — enteros — en vez del
   `Q_memprint` cuyo `%.*f` se comió el dato en §35.2. El camino de escritorio queda
   como estaba.
2. **`Xbox_MemReport( tag )`** (`xbox_trace.c`, declarada en `nxdk_compat.h`): vuelca
   `MmQueryStatistics` del kernel — RAM total, libre, y páginas comprometidas de VM,
   pool, pilas e imagen, todo en KB y enteros — y a continuación mide **el mayor
   malloc que el CRT concede ahora mismo**, por bisección entre 0 y 256 MB, reservando
   y liberando en el acto. Ese segundo número separa las hipótesis 1 y 2 de §35.3 sin
   discusión: si el kernel dice 40 MB libres pero el malloc mayor es de 4, hay un
   límite artificial entre medias; si ambos números se parecen, el montón es lo que
   es y el presupuesto es real.
3. **Tres momentos**: `at boot` (en `main`, antes de que el motor reserve nada — la
   línea del veredicto 64/128), `at frame loop` (motor entero arriba, contra la
   baseline), y `Mem_Alloc/Mem_Realloc failure` (el estado exacto en el instante del
   fallo, pegado al tamaño que se pedía).

### 36.3 El punto de §35.4, también dentro

Los tres sitios que el pelado del rootdir dejó malformados llevan ahora el mismo
`has_sep` que sus hermanos: el searchpath raíz (era `Adding directory: D:`, será
`D:/`), y los builders de `downloaded/` y `custom/` (eran `D:valve/...`). Con esto,
todas las rutas del log del próximo arranque deberían salir bien formadas, sin
excepción.

### 36.4 Qué NO está verificado

- **Nada de esta ronda ha corrido.** En particular, que el kernel de esta placa
  exponga los 128 MB con el bit fuera es **plausible, no medido** — la consola lleva
  BIOS de aftermarket y chips soldados, que es la combinación correcta, pero el
  veredicto es la línea `mem at boot`.
- Si el kernel expone 64 pese a todo, la ronda no se pierde: quedan los números del
  presupuesto real (§35.3, hipótesis 1 contra 2) y el OOM con su tamaño, que era el
  plan original de tres puntos.
- La sonda de mayor-malloc toca el montón durante el fallo de alloc; en teoría
  reservar-y-liberar no perturba nada, en la práctica está sin estrenar.

### 36.5 Estado

`default.xbe` 4.620.288 B con flags `0x00000001`, SSE2 = 0 en 420 objetos, subido.
`xash.cmd` = `-noip -dev 2 +map c1a0`, intacto — el mismo arranque de §33 y §35, ahora
con el doble de máquina si el kernel coopera, y con la memoria hablando en números si
no. Ficheros del fork modificados: **21** (entra `scripts/waifulib/xbox.py`... que ya
estaba; entra `engine/common/zone.c` — 21 en total).

El próximo log responde tres preguntas de una tacada: cuánta RAM ve el proceso, cuánto
malloc concede el CRT, y — si con 128 MB el menú carga y el `+map` por fin se ejecuta —
si Half-Life corre en una Xbox. La línea que hay que buscar primero: `mem at boot`.

## 37. Paso 2v — 128 MB confirmados, la «basura» era ausencia de dibujo, y c1a0 estaba cargando

**Las tres respuestas del log, en orden de titular: el proceso ve `ram total 131072 KB`
— los 128 MB, medidos en runtime, con un mayor malloc de 112 MB; el OOM de §35 ha
desaparecido y el motor llegó más lejos que nunca — menú cargado con el arte del pak,
`+map c1a0` ejecutado, `c1a0.bsp` cargado y el precache de modelos en marcha cuando el
log se corta sin error; y la pantalla de basura no era corrupción de memoria sino los
framebuffers de pbgl sin limpiar, expuestos 27 segundos porque el primer arranque que
carga todo tarda mucho más en llegar al primer swap. Debajo de la basura, Half-Life
estaba cargando su primer mapa.**

| Pregunta | Respuesta |
|---|---|
| ¿`mem at boot`? | **`ram total 131072 KB, free 115192 KB`** — 128 MB; mayor malloc **115072 KB** |
| ¿Hasta dónde? | **Más lejos que nunca**: menú OK (sin OOM), `Spawn Server: c1a0`, BSP cargado, precache de armas — el log acaba en `loading models/p_crossbow.mdl`, **sin error** |
| ¿Escribe algo fuera de sitio? | **No.** fb de arranque en `0x83eb4000` — el mismo de siempre, bajo 64 MB |
| ¿La basura? | Framebuffers de pbgl **sin limpiar**, en pantalla de 7,1 s a 33,7 s |
| ¿Aislamiento (-nooverlay)? | **Innecesario**: el log exonera al overlay; el kit queda en el arsenal |

### 37.1 El punto 4 de §36, respondido con números

```
[0012      608] mem at boot: ram total 131072 KB, free 115192 KB, committed vm 12056 KB ...
[0013     4334] mem at boot: largest malloc 115072 KB
[0107    28907] mem at frame loop: ram total 131072 KB, free 64728 KB, ...
[0108    33670] mem at frame loop: largest malloc 64640 KB
```

El bit `Limit64MB` fuera **duplica la máquina de verdad**: kernel y CRT concuerdan
(115 MB libres, 112 concedibles de una pieza). Y la segunda pareja de líneas es la
primera medición del presupuesto real del port: **el motor entero, con el menú y sus
imágenes del pak cargadas, consume ~50 MB** — con 64 MB quedaban ~14 para el mapa, y
por eso §35 murió donde murió. El OOM no volvió a aparecer: pasó justo donde §35
caía, y el arranque entero hasta el bucle costó 28,9 s (los BMP del menú decodificando
por primera vez, más 8,5 s de las dos sondas de malloc — reducidas a granularidad de
1 MB en esta ronda).

### 37.2 La basura, explicada y exonerados los sospechosos

La caza estática de §36→37 dejó a cada sospechoso en su sitio, comprobado en fuente:

- **pbkit se auto-limita a 64 MB**: framebuffers y depth-stencil con
  `HighestAcceptable 0x03FFB000`, pushbuffer y DMA con `MAXRAM 0x03FFAFFF`. Sus
  máscaras `& 0x03FFFFFF` son coherentes con sus propias reservas. Inocente.
- **pbgl igual** (`PBGL_MAXRAM 0x03FFAFFF` para texturas). Inocente.
- **El framebuffer de arranque de nxdk** se reserva con límite `0x7FFFFFFF` — el
  único candidato real a acabar arriba — pero el log lo pone en `0x83eb4000`, el
  mismo sitio de siempre: en esta máquina la memoria contigua sigue saliendo de abajo.
  Inocente (con la reserva anotada: si algún día se mueve, `pbkit.c:2345` guarda esa
  dirección truncada a 26 bits).
- **El overlay**: su máscara `& 0x03FFFFFF` sí arrastraba la suposición de 64 MB —
  cuarta suposición que §30.6 no listó — y queda ensanchada a `0x07FFFFFF`. Pero no
  pintó nada fuera: el fb que lee estaba bajo 64 MB, y su `draw #1` salió en el
  mismo búfer de siempre.

Lo que quedó en pantalla fue otra cosa: **pbgl no limpia sus framebuffers al nacer**.
Lo que hubiera en esa RAM es lo que el CRTC enseña hasta el primer swap real. Siempre
fue así — es el «amago de imagen» de §23, que entonces duraba un segundo y medio.
Este arranque, el primero en que el menú decodifica de verdad sus imágenes del pak,
tardó 26 segundos en dar su primer swap, y el destello se convirtió en media pantalla
de minuto de basura con un motor perfectamente sano debajo. Y cuando el mapa empezó a
cargar (bloqueante: sin swaps), la pantalla volvió a congelarse sin testigo — el
reinicio manual llegó a mitad del precache, y no hay forma de saber por el log si el
motor seguía moliendo o se había atascado: las líneas `loading` van sin marca de
tiempo.

### 37.3 El arreglo: que la pantalla nunca más se quede sin testigo

Tres piezas, todas en la familia del trazador:

1. **Limpieza de la cadena de swap al iniciar pbgl** (`vid_xbox.c`): tres ciclos de
   `glClear` + swap dejan ambos búferes en negro antes de que nadie los vea.
2. **Repintado del overlay en cada línea de traza** (`Xbox_Trace` y `Xbox_TraceRaw`):
   durante una carga bloqueante no hay swaps, pero sí hay líneas — cada `loading
   models/...` repinta el anillo, así que la carga del mapa se ve como un scroll en
   pantalla. Guarda antirrecursión (`Draw` traza su propia línea de testigo).
3. **Sondas a granularidad de 1 MB**: mismos números útiles, sin los 8,5 s.

Con esto, el próximo arranque enseña texto desde `pbgl_init` hasta el frame loop, y
el precache entero línea a línea. Si el motor se atasca, la última línea queda a la
vista además de en el disco.

### 37.4 Qué NO está verificado

- **Si el precache terminaba.** El log se corta sin error a ~50 modelos; reinicio
  manual en medio. El próximo arranque, con el scroll visible, lo dirá — y si muere,
  dirá exactamente dónde.
- **La limpieza de framebuffers y el repintado por traza**: compilados (objetos
  recompilados verificados, 0 SSE2, flag `0x01`), no ejecutados.
- **El coste del repintado por línea** durante el precache (una pasada de overlay por
  cada `loading`): estimado en decenas de µs por línea, no medido.
- El presupuesto de memoria **con el mapa cargado** — la medición que §29.6 pedía —
  sigue pendiente de un arranque que termine de cargar.

### 37.5 Estado

`default.xbe` 4.620.288 B (flags `0x00000001`, SSE2 = 0) **subido**; `xash.cmd` =
`-noip -dev 2 +map c1a0`, sin tocar — quinta vez el mismo arranque, cada vez contra
un muro distinto y más lejano. El kit de aislamiento (`-nooverlay`,
`ofx-xbe-limit64.sh`) queda en `research/` sin estrenar: el log exoneró al overlay
antes de necesitarlo.

La foto del estado real: con 128 MB, Half-Life pasó de morir en el menú a estar
cargando las armas de c1a0. Lo único que faltó fue paciencia — y un testigo que
dijera que había que tenerla.

## 38. Paso 2w — Cuelgue mudo en `gman.mdl`: sin excepción, sin vigilante, y sin reloj para medirlo

**El log acaba exactamente en `loading models/gman.mdl` y no hay una línea más: ni
`Sys_Error`, ni `Host_Error`, ni `SDL assert IGNORED`, ni vigilante, ni
`returning to dashboard`. Es un cuelgue mudo, y no saltó ningún mecanismo de salida
por una razón concreta y corregible: el vigilante se desarmaba al entrar al bucle de
frames desde §29, cuando el silencio todavía era normal. Con el latido de §30 ya no lo
es, así que vuelve a estar armado — 60 s — y el próximo atasco volverá solo al
dashboard con el estado del montón. Además, las líneas del motor pasan a ir selladas
con la hora: sin eso era imposible responder si el precache iba lento o parado, que es
justo lo que esta sección no puede afirmar.**

| Pregunta | Respuesta |
|---|---|
| ¿Dónde acaba? | `loading models/gman.mdl`, línea 367. **Nada después** |
| ¿Cuelgue o excepción? | **Cuelgue mudo**: 0 `Sys_Error`, 0 `Host_Error`, 0 asertos SDL, 0 vigilante |
| ¿Saltó algún mecanismo? | **Ninguno, y no podía**: el vigilante estaba desarmado desde el frame loop |
| ¿(a) lento o parado? | **Indeterminable**: las líneas del motor no llevaban hora (arreglado) |
| ¿(b) montón? | `mem at frame loop: free 64728 KB` — 63 MB libres antes del mapa. Sin traza en el punto de muerte (arreglado) |
| ¿(c) frontera? | Sospechoso principal, con evidencia nueva: la textura 188×345 de gman |

### 38.1 El orden real, que la pantalla no contaba bien

El log (`research/logs/xash-boot-20260817-220458.log`, 367 líneas) termina así:

```
*Graph Loaded!
Chapter title: C0A1TITLE
loading models/scientist.mdl
loading models/barney.mdl
loading models/filecabinet.mdl
monster_furniture has no view_ofs!
loading models/gman.mdl
```

Esto ya no es el precache de armas de §37: es el **spawn de entidades** del BSP, cada
una precargando lo suyo. c1a0 es «Anomalous Materials», donde el G-Man aparece tras
el cristal — el motor llegó hasta la entidad que lo instancia y ahí se quedó.

Nótese que el orden que se leyó en pantalla (gman, luego el título de capítulo, luego
el grafo) **no es el real**: es el bug del pintor de 38.4. La pantalla mezclaba
residuo con líneas nuevas; el log manda.

### 38.2 Por qué no saltó nada, y el arreglo

`host.c`, desde §29, desarmaba el vigilante justo al entrar al bucle con este
razonamiento, correcto entonces: *«a partir de aquí el silencio es normal, un juego
que corre no imprime»*. Pero §30 metió el latido, que imprime **una línea por
segundo**. Desde ese momento el silencio dejó de ser normal y nadie volvió a mirar la
premisa: cinco minutos de silencio absoluto y el vigilante desarmado mirando.

El arreglo es de una línea y cambia el ciclo de trabajo entero:

- `Xbox_TraceWatchdogArm( 60 )` en lugar de `Disarm()` al entrar al bucle. Cualquier
  traza —latido, línea de consola, `loading ...`— refresca el contador, así que un
  precache **lento** no lo dispara; solo el silencio real.
- Al dispararse, el vigilante vuelca `Xbox_MemStat( "at watchdog" )` y sale al
  dashboard. El log queda en disco y la consola vuelve a la red **sola**.
- **Detalle que importa**: el vigilante usa `Xbox_MemStat` (solo `MmQueryStatistics`,
  una llamada al kernel), nunca `Xbox_MemReport`. Si el hilo principal está colgado
  *dentro* del asignador, sondear `malloc` desde el vigilante se bloquearía en el
  lock del montón y el vigilante no llegaría nunca al dashboard — el diagnóstico
  matando al diagnosticador.

Este cuelgue costó un reset manual y una espera a que la consola volviera a la red.
El siguiente costará 60 segundos.

### 38.3 Sin reloj no hay hipótesis (a)

La pregunta «¿sigue vivo pero lentísimo?» tiene una respuesta honesta: **no se puede
saber con este log**. Las líneas del motor (`Con_Printf` → `Xbox_TraceRaw`) se
escribían *verbatim*, sin marca de tiempo, por diseño de §20. Con una línea por
modelo y ningún reloj, no hay nada que medir ni extrapolar.

Arreglado: `Xbox_TraceRaw` sella cada línea al principio de línea con
`[con %8u] `, distinguible de un vistazo de las trazas numeradas. Solo al principio
de línea, porque `Con_Printf` no siempre entrega líneas enteras y una continuación con
un reloj pegado en medio sería peor que nada. Con esto, el próximo log da el coste de
cada modelo y la respuesta a (a) es aritmética.

### 38.4 El overlay se solapaba: bug del pintor, confirmado en el código

No era síntoma del cuelgue. `Xbox_OverlayDrawLine` recorría `line[col]` hasta el
`NUL`, así que **una línea corta nunca borraba la cola de la larga que ocupaba esa
fila antes**. Fue invisible durante diez secciones porque cada swap limpiaba el frame
entero; saltó a la vista justo ahora porque §37 introdujo el repintado *sin* swaps
para sobrevivir a las cargas bloqueantes. Es decir: lo arregló un cambio y lo destapó
el siguiente.

Arreglo: cada fila recuerda cuántas celdas pintó (`xbox_overlay_rowlen`) y repinta al
menos esas, rellenando con espacios. En el caso normal no cuesta nada extra.

**Corolario operativo:** hasta que esto se vea funcionando, lo que aparezca en pantalla
no vale como orden de eventos. El log en disco es el único testigo con orden fiable.

### 38.5 Lo que sí se sabe de `gman.mdl`, medido en el pak

Con el pak en el PC se puede comparar el que cuelga contra los que cargaron bien
(`research/ofx-pak-ls.sh` y `ofx-mdl-textures.sh`, nuevos):

| Modelo | .mdl | texturas | mayor textura | mayor POT tras reescalar |
|---|---|---|---|---|
| `scientist` | 174.732 B | `scientistt.mdl` 126.940 B, 31 | 72×115 | 128×128 |
| `barney` | 129.104 B | `barneyt.mdl` 102.604 B, 26 | 92×109 | 128×128 |
| `filecabinet` | 14.880 B | 2 internas | 48×120 | 64×128 |
| **`gman`** | **76.028 B** | `gmant.mdl` **144.276 B**, 23 | **188×345** | **256×512** |

Dos conclusiones limpias: **el tamaño del modelo no es la causa** — gman es la mitad
que scientist, que cargó — y **todas las texturas son NO-POT**, así que tampoco es eso
por sí solo. Lo que destaca es una sola cifra: `GMan_SuitTop2.bmp`, 188×345, que al
redondear a potencia de dos son **256×512, cuatro veces el mayor de los que
funcionaron**. Es la primera textura de ese tamaño que este port intenta subir.

Y encaja con la forma del fallo. `GL_ARB_texture_non_power_of_two` sale **failed** en
el log, así que el motor reescala; el camino de subida acaba en pbgl, y pbkit espera
a la GPU con `while( pb_busy( ));` — **un bucle sin una sola línea de salida**. Un
cuelgue ahí es exactamente lo que se ha visto: silencio absoluto, sin excepción, sin
consumir memoria, sin volver. Los `OpenGL Error: GL_INVALID_ENUM while uploading` que
aparecen desde §32 dicen que ese camino ya tenía problemas menores.

**Es una hipótesis con evidencia, no un diagnóstico.** Las alternativas siguen vivas:
un bucle infinito en el propio cargador studio, o algo del gamedll en la frontera de
§28.6, que sigue sin auditar.

### 38.6 Qué NO está verificado

- **Nada de esta ronda ha corrido**: vigilante rearmado, sellos de hora, pintor
  arreglado y `mem before map load`. Compilado, verificado en el artefacto (las cinco
  cadenas nuevas presentes, flags `0x00000001`), 0 SSE2, subido.
- **La hipótesis de la textura no está probada.** Lo probado es que gman no es más
  grande y que su textura mayor sí lo es, con mucho.
- **El montón en el punto de muerte** sigue sin conocerse para *este* cuelgue; el del
  próximo lo dará el vigilante.

### 38.7 Estado y qué esperar del próximo arranque

`default.xbe` 4.620.288 B subido; `xash.cmd` = `-noip -dev 2 +map c1a0`, sin tocar
—sexta vez el mismo arranque—. El log siguiente debería contestar tres cosas sin
intervención humana: cuánto tarda cada modelo (`[con ...]`), si el precache avanza o
está parado, y —si vuelve a atascarse— el vigilante dará el estado del montón y
devolverá la consola al dashboard en un minuto, en vez de costar un reset y una espera.

Y si el atasco vuelve a ser `gman.mdl`, el siguiente paso ya tiene forma: instrumentar
el camino de subida de texturas del renderer, empezando por dónde se mete la de
256×512.

## 39. Paso 2x — El vigilante estaba armado y no disparó: la ronda de discriminar

**El log de la repetición confirma lo peor y lo mejor a la vez. Lo peor: `watchdog
armed ... 60 s` está en el log (27,2 s), el motor calló en `gman.mdl` a los 108 s, el
usuario esperó más de cinco minutos — y el vigilante no escribió ni una línea. O la
máquina entera muere (a), o el vigilante vive pero su primera acción lo mata (b). Lo
mejor: los sellos `[con]` de §38 funcionaron a la primera y responden la pregunta del
precache: iba lento — 0,4 a 3 s por modelo — pero avanzando, y gman es una parada en
seco, no lentitud. Esta ronda mete el discriminador de (a)/(b) en el vigilante, las
trazas por textura en el camino del renderer, esperas acotadas en pbgl, y deja
identificado el mapa de control sin monstruos.**

| Dato del log | Valor |
|---|---|
| Vigilante | armado a 60 s en `27215 ms`; **cero líneas al colgarse** |
| Precache | 0,4–3 s por modelo (scientist 2,7 s, barney 2,7 s), **avanzando** |
| Parada | `[con 107974] loading models/gman.mdl` — última línea; 5+ min de silencio |
| Montón | `mem before map load: free 64472 KB` — la memoria queda descartada |
| `default.xbe` | 4.620.288 → **4.624.384 B** (trazas + bandas), flags `0x01`, 0 SSE2, subido |

### 39.1 Lo que el log ya deja cerrado

- **(a) de §38 —«¿lento o parado?»— respondida con aritmética**: entre `Spawn Server`
  (31,4 s) y gman (108 s) hay 76 s de precache a 0,4–3 s por entrada, sin ningún
  hueco mayor de 3,2 s. Luego, nada durante cientos de segundos. Es un muro, no una
  cola.
- **(b) memoria: descartada.** 63 MB libres justo antes del mapa, y el cuelgue no
  consume nada (la pantalla se quedó helada, no degradándose).
- **El vigilante armado y mudo** convierte la pregunta de §38.2 en la de esta ronda:
  ¿por qué un hilo independiente, que solo duerme y mira un contador, no llegó ni a
  su primera línea?

### 39.2 Las dos historias del silencio, y el discriminador

**(a) La máquina entera muere.** Un `while( pb_busy( ))` es un spin vivo — el
planificador seguiría y el vigilante correría. Pero hay una versión peor: si la GPU se
encalla a mitad de una lectura de registro, la transacción de bus **no termina nunca**,
las interrupciones no se sirven, y ningún hilo vuelve a ejecutarse. Un solo acceso
colgado = consola muerta. Con eso, ni vigilante ni START+BACK ni nada.

**(b) El vigilante vive pero su primera acción lo mata.** Su primera acción era
`Xbox_Trace` → `CreateFileA` sobre el FATX. La previsión de §38.2 cubrió el lock del
*montón* (por eso usa `Xbox_MemStat` y no la sonda de malloc), pero **no el del
filesystem**: si el hilo principal cayó dentro de una operación de E/S sosteniendo un
lock del FS, la escritura del log se bloquea para siempre. El matiz en contra: la
última línea (`loading models/gman.mdl`) se imprime *después* de que `FS_LoadFile`
devuelva — el fichero ya estaba leído — así que el principal no debería estar dentro
del FS. En contra de (a): un spin normal no explica el silencio. Ninguna historia
gana sin medir.

**El discriminador**, en el orden exacto del disparo del vigilante:

1. **Banda ROJA** en lo alto de la pantalla — `Xbox_WatchdogMark`: escrituras de
   memoria al framebuffer que el CRTC muestra, una lectura de registro y nada más.
   Sin disco, sin montón, sin locks. Lo más primitivo que este port sabe hacer.
2. La línea de log y `Xbox_MemStat` — la parte que puede bloquearse en (b),
   deliberadamente *después* de la banda.
3. **Banda VERDE**: la escritura sobrevivió.
4. `XLaunchXBE( NULL )` directo — sin la cortesía de 5 s de `Xbox_ReturnToDashboard`,
   que son lujos para una máquina en estado desconocido.

Lectura forense de la pantalla en el próximo cuelgue: **nada = (a)**, máquina muerta,
ningún vigilante software puede ayudar; **rojo solo = (b)** con la escritura como
asesina; **rojo + verde + dashboard = vigilante entero**, y el log trae el montón del
momento.

### 39.3 El ataque a gman, por los tres flancos

1. **`GL_UploadTexture` instrumentado** (`ref/gl/gl_image.c`, con `Xbox_Trace`
   directo — la cabecera compat se fuerza en todas las TU, §20): una línea
   `upload: <nombre> WxH -> WxH tipo N` por textura, y `uploaded: <nombre> ok` al
   salir para las de ≥256. Un `upload:` sin su `uploaded:` es el culpable con nombre
   y dimensiones. Coste real en disco síncrono durante la carga; asumido hasta cazar
   el cuelgue.
2. **Esperas acotadas en pbgl** — y un hallazgo del barrido: **el camino de subida de
   texturas no tiene ninguna espera de GPU** (es memcpy + swizzle de CPU). Los
   `while( pb_busy( ))` viven en init, swap y `glFinish`. Todos acotados a 5 s con
   `PBGL_BOUNDED_WAIT` (traza `pbgl: wait timeout at <sitio>` y sigue), `libpbgl.lib`
   recompilada (§17.2) y el engine relinkado a mano — waf no vigila el `.lib`
   externo. Implicación honesta: si el cuelgue está dentro de la subida, es CPU puro
   (¿resample 188×345→256×512? ¿swizzle?) y las trazas de (1) lo dirán; si está en el
   swap posterior, el timeout hablará.
3. **Mapa de control identificado**: el pak trae los multijugador — `stalkyard`
   (700 KB, cero monstruos) es el control limpio. **No se cambia todavía**: la
   próxima ejecución repite `c1a0` a propósito, porque es la que discrimina (a)/(b)
   y nombra la textura. El control va después: `-noip -dev 2 +map stalkyard`.

### 39.4 Qué NO está verificado

- **Todo lo de la ronda**: bandas, trazas de subida, esperas acotadas. Compilado y
  comprobado en el artefacto (`pbgl: wait timeout` ×2, `upload:`, `uploaded:`,
  bandas en el camino del vigilante; flags `0x01`; 0 SSE2), no ejecutado.
- **La hipótesis del bus colgado** no es medible desde dentro si es cierta — por
  construcción. Su evidencia sería negativa: ninguna banda en pantalla.
- El parche de pbgl vive en el **toolchain** (`/opt/toolchains/xbox/pbgl`), fuera del
  árbol del fork. Pendiente de exportar como `.patch` en `~/opposing-force-x/` como
  los demás parches de toolchain.

### 39.5 Estado y guion de las dos próximas ejecuciones

`default.xbe` 4.624.384 B subido; `xash.cmd` = `-noip -dev 2 +map c1a0`, sin tocar.

**Ejecución 1 (repetición discriminante):** mismo `c1a0`. Si cuelga: mirar la
pantalla (bandas → (a)/(b)) y, si vuelve sola, el log trae la última `upload:` — la
textura culpable con dimensiones — y el montón del vigilante. **Ejecución 2
(control):** `+map stalkyard`, cero monstruos — si carga y late, el problema es de
los modelos de personaje, no del mapa ni del mecanismo.

La cadena de §37 sigue viva: el port pasó de menú a cargar armas, de armas a spawn de
entidades, y ahora la frontera es un modelo concreto con la textura más grande que el
port ha intentado subir. Cada ronda, un muro más adentro.

## 40. Paso 2y — No era una textura: era la 765ª. Y un splash de 42 MB

**El cuelgue no tiene nada que ver con `GMan_Shin1`, una textura de 32×64 que ocupa
3 KB. La hipótesis de acumulación era la correcta, y ahora tiene número: al llegar a
ella pbgl llevaba **50,1 MB reservados de los 64** que puede direccionar, y **42,67 MB
de esos —el 85%— son una sola imagen del menú**, `gfx/shell/splash`, que entra a
3840×2000 y se redondea a 4096×2048 RGBA. Los tiempos crecientes que se veían en el
log (177 → 196 → 806 ms) son la búsqueda de páginas físicamente contiguas cada vez
más difícil en una región casi llena. Esta ronda mete el techo de 512 px que quita
esos 42 MB de en medio, instrumenta el asignador que estaba completamente ciego, y
corrige un defecto de mi propia instrumentación que estuvo a punto de mandar la
investigación detrás de una textura inocente.**

| | |
|---|---|
| Bandas del vigilante | **Ninguna**, y ninguna línea suya en el log — hipótesis (a) |
| Culpable real | **La 765ª reserva**, no la textura: 50,1 MB de 64 en uso |
| `gfx/shell/splash` | 3840×2000 → **4096×2048 RGBA = 42,67 MB con mipmaps** |
| Tiempos crecientes | 177 / 196 / 806 ms — búsqueda de páginas contiguas |
| Arreglo | `GL_MAX_TEXTURE_SIZE` limitado a **512** en Xbox: el splash pasa a <1 MB |
| `default.xbe` | 4.624.384 B, flags `0x01`, 0 SSE2, subido |

### 40.1 Antes de nada: mi instrumentación tenía un defecto y hay que decirlo

La lectura «`GMan_Shin1` tiene `upload:` y nunca su `uploaded: ok`» era razonable y
**la señal era falsa**: el cierre de §39 se emitía solo para texturas de 256 px o más

```c
if( tex->width >= 256 || tex->height >= 256 )
    Xbox_Trace( "uploaded: %s ok", tex->name );
```

y `GMan_Shin1` mide 32×64. **Nunca habría impreso su cierre, ni cargando
perfectamente.** El recuento del log lo deja claro: 769 `upload:` contra 30
`uploaded:`. Lo único que Shin1 demuestra es lo que ya decía su posición: que es la
**última línea**, es decir donde el motor se detuvo. Ni siquiera está probado que el
cuelgue sea *dentro* de su subida y no en el hueco hasta la siguiente.

Corregido: el cierre se emite ahora para todas, con su tamaño. Una instrumentación que
solo habla de los casos grandes convierte cada silencio pequeño en falso positivo, y
eso es peor que no instrumentar.

### 40.2 Las bandas: ninguna, y qué significa

En el log no hay `watchdog: ... with no output` ni `at watchdog`, y en pantalla no
apareció ni la roja ni la verde. El vigilante **no llegó a ejecutar su primera
instrucción**, que era pintar memoria — sin disco, sin montón, sin locks.

Eso descarta (b) —«vive pero muere al escribir el log»— y deja **(a): la máquina
entera parada**. Con una salvedad honesta que conviene dejar escrita: si el CRTC
estuviera congelado, la banda podría haberse escrito y no verse. Pero la banda va
*antes* de la línea de log, y tampoco hay línea de log; la escritura a disco no
depende de la GPU. Que falten las dos cosas señala al hilo, no a la pantalla.

Y hay un mecanismo que lo explica sin invocar hardware roto: `MmAllocateContiguousMemoryEx`
es una llamada al **gestor de memoria del kernel**, que busca páginas físicas
contiguas con alineación. Esa búsqueda se hace con el lock del gestor tomado y a IRQL
elevado — **con el planificador efectivamente apagado**. Un hilo principal metido ahí
durante minutos no es un hilo colgado: es la máquina entera sin planificar, y por eso
ni vigilante, ni START+BACK, ni nada. Encaja con todo lo observado, y con que los
tiempos vinieran creciendo justo antes.

**No está probado** —medirlo desde dentro es, por construcción, imposible— pero es la
única historia que explica las cuatro cosas a la vez: silencio total, tiempos
crecientes, ausencia de excepción y consumo de memoria plano.

### 40.3 Los tiempos, y qué distingue a las lentas

La alternancia (~5 ms / ~180 ms) no la explica el tamaño de la textura: `Mouth1`
(32×16, la más pequeña de todo el tramo) tardó 176 ms y `Collar1` (64×32, el doble)
tardó 6. Lo que distingue a una reserva lenta de una rápida en un asignador de páginas
contiguas no es lo que pides, sino **si hay un hueco de ese tamaño ya disponible**: las
rápidas caen en un hueco libre del tamaño justo, las lentas obligan a recorrer la
región. Con 50 de 64 MB ocupados y 765 reservas de tamaños dispares, el mapa de huecos
está hecho trizas.

Por eso la cifra que importa no es la de ninguna textura concreta sino la acumulada:

```
#  1  acumulado   0.0 MB   *default
#192  acumulado  45.8 MB   #maps/c1a0.bsp:+7~lab1_cmp1.mip
#383  acumulado  47.2 MB   #models/v_rpg/M17CL.mdl
#574  acumulado  48.7 MB   #sprites/zerogxplode(frame:00).spr
#765  acumulado  50.1 MB   #models/gman/GMan_Shin1.mdl
```

El salto está **antes de la textura 192**: en las primeras cien entradas ya hay 45,8 MB
puestos. Es el menú, no el mapa. Todo c1a0 —geometría, armas, personajes, sprites—
cabe en los 4,3 MB restantes; lo que no cabe es lo que el menú dejó ahí.

### 40.4 El culpable, con nombre y factura

| Textura | Destino | VRAM con mips |
|---|---|---|
| **`gfx/shell/splash`** | **4096×2048 RGBA** | **42,67 MB** |
| `#Tahoma_11_500_o1_stb_font.bmp` | 256×512 | 0,67 MB |
| `gfx/shell/head_load` / `head_save` | 512×64 | 0,17 MB cada una |
| `cached/conback` | 256×256 | 0,09 MB |
| `#models/gman/GMan_SuitTop2.mdl` | 128×256 | 0,04 MB |

Una imagen de fondo del menú principal ocupa **mil veces** lo que la textura mayor del
G-Man. Y explica la cronología entera del port: el splash **no existía** hasta §37,
porque vive dentro de `pak0.pak` — §35 lo tiene en el log como
`FS_LoadImage: couldn't load "gfx/shell/splash"`. Subir el pak dio al motor sus
imágenes de menú y, con ellas, 42 MB menos para el juego. **El muro de la memoria y
el mapa cargando llegaron en el mismo envío.**

De paso queda explicado por qué `GMan_SuitTop2` (188×345) acabó en 128×256 y no en
256×512: `NearestPOW` con `gl_round_down` redondea *hacia abajo* cuando la distancia a
la potencia superior es grande (188 → 128, 345 → 256), y hacia arriba cuando es
pequeña (3840 → 4096). Consistente, y la razón de que la hipótesis de §38.3(c) —«la
textura de 256×512»— fuera doblemente falsa: ni era tan grande ni era la que falla.

### 40.5 Lo que lleva esta ronda

1. **El techo de 512 px** (`gl_opengl.c`, tras leer `GL_MAX_TEXTURE_SIZE`). pbGL
   responde 4096 porque es lo que el NV2A sabe direccionar, no un presupuesto; el
   presupuesto real es el pool contiguo de 64 MB compartido con framebuffers y
   pushbuffer. 512 se elige **contra la salida, no contra el hardware**: la pantalla
   es de 640×480 y no puede enseñar más. El splash cae de 42,67 MB a menos de 1.
2. **El asignador deja de estar ciego** (`pbgl/src/memory.c`): total vivo, pico,
   número de reservas, y una línea cuando una tarda ≥100 ms o falla. Silencioso
   cuando todo va bien. Es la instrumentación «dentro de la subida» que faltaba: si
   el próximo muro vuelve a ser memoria, dirá el KB exacto y cuánto había puesto.
3. **El cierre de subida para todas las texturas** (40.1).
4. `pbgl-bounded-waits.patch` actualizado a 185 líneas con los tres ficheros.

### 40.6 Qué NO está verificado

- **Nada de esto ha corrido.** Compilado, comprobado en el artefacto (las cinco
  cadenas nuevas, flags `0x01`, 0 SSE2), subido.
- **Que el techo de 512 sea suficiente** es aritmética, no medición: quita 42 MB de
  50, y deja el uso previsto en torno a 8 MB de 64. Si el mapa sigue muriendo por
  memoria, ahora habrá una línea `pbgl mem:` diciéndolo en vez de un silencio.
- **El mecanismo del IRQL** (40.2) es la mejor explicación disponible, no un hecho
  medido.
- **La fragmentación** no se ataca en esta ronda; el techo la hace irrelevante por
  ahora. Si reaparece, el arreglo de verdad es liberar las texturas del menú antes de
  cargar mapa.

### 40.7 Estado

`default.xbe` 4.624.384 B subido; `xash.cmd` = `-noip -dev 2 +map c1a0`, sin tocar —
séptima vez el mismo arranque. Lo que hay que mirar en el próximo log, por orden:
`Xbox: capping GL_MAX_TEXTURE_SIZE 4096 -> 512` al iniciar el renderer, el
`uploaded:` del splash con su tamaño nuevo, la ausencia de líneas `pbgl mem: slow`, y
después de `loading models/gman.mdl` — por primera vez en cuatro secciones — otra
línea cualquiera.

## 41. Paso 2z — El techo funciona, y la culpa es de una textura de 3 KB

**El techo de 512 hizo exactamente lo que prometía: el splash pasa de 42,67 MB a
384 KB, la VRAM total de 50,1 MB a 6,7, las subidas de 177/196/806 ms a 2-5 ms
constantes, y el mismo punto del mapa se alcanza en 20,8 s en vez de 128. Y el motor
muere otra vez exactamente en `#models/gman/GMan_Shin1.mdl`. Con el cierre ya
instrumentado para todas las texturas, la señal es limpia por primera vez: **769
`upload:` y 768 `uploaded:`** — falta uno y solo uno. La acumulación queda descartada
por su propio experimento de control: misma textura, mismo punto, con una cuarta
parte de memoria en uso. Esta ronda descarta también el tamaño, el formato, la
posición en el modelo y la integridad del fichero, y mete trazas dentro de la subida
para que el próximo log señale el paso exacto.**

| | |
|---|---|
| Techo de textura | `capping GL_MAX_TEXTURE_SIZE 4096 -> 512`, splash a **384 KB** |
| VRAM total | **6,7 MB** de 64 (antes 50,1) — `pbgl mem: slow` **cero veces** |
| Tiempo al mismo punto | **20,8 s** (antes 128 s) |
| Señal | 769 `upload:` / **768** `uploaded:` — falta el de `GMan_Shin1` |
| Vigilante | ninguna banda, ninguna línea: máquina parada otra vez (§40.2) |

### 41.1 Lo que esta ronda descarta, con evidencia

- **Acumulación**: descartada por el mejor control posible — la misma textura, en el
  mismo orden, con 6,7 MB en uso en vez de 50,1. Si fuera presión de memoria, con una
  cuarta parte del consumo habría caído más tarde o no habría caído.
- **Tamaño**: `GMan_Shin1` es 40×54 → 32×64, 2.160 píxeles indexados, **3 KB**.
- **Formato y destino**: idénticos a tres vecinas que suben sin problema —
  `GMan_Pelvis1` (44×51), `GMan_Shoe1` (24×57) y `GMan_Arm-R1` (44×65) **acaban las
  tres en 32×64 type 2**, y las tres cierran en 2-5 ms.
- **Ser la última del modelo** (tu hipótesis 4): **no lo es**. Es la #12 de 23; le
  sigue `GMan_Arm-L1` (40×58 → 32×64), que ni siquiera llega a imprimir su `upload:`.
  El cuelgue es dentro de la subida de Shin1, no en el cierre del precache del modelo.
- **Integridad del MDL**: `research/ofx-mdl-texdata.sh` (nuevo) disecciona la tabla de
  las 23 texturas de `gmant.mdl`: offsets consecutivos sin un solo hueco ni solape,
  la última acaba **exactamente** en el byte 144.276 que mide el fichero, dimensiones
  positivas, paletas de 768 B completas. Shin1 está en el offset 108.512 y usa 35
  índices distintos. No hay nada roto en el fichero.

Queda una conclusión incómoda y sólida: **es esa textura concreta, por su contenido**,
y el estado del sistema no tiene nada que ver.

### 41.2 Lo que la lectura estática agotó

Se auditaron los tres caminos, y los tres están acotados sobre el papel:

- **El resample del motor** (`Image_Resample8Nolerp`, `img_utils.c:1054`): doble bucle
  con límites fijos y aritmética de punto fijo. No puede no terminar.
- **La cadena de mipmaps de pbgl** (`tex_alloc`, `texture.c:504`): el
  `while( width >= 1 || height >= 1 )` con **OR** llama la atención, pero traza bien:
  32×64 termina en 7 niveles, y `TEX_MAX_MIPS` es 12. Sin desbordamiento.
- **La generación de mips indexados** (`tex_mip8`, `texture.c:305`): bucles acotados;
  es una búsqueda de color más cercano sobre 256 entradas por píxel de salida —de ahí
  los 2-5 ms por textura— pero finita.

Y un detalle del propio autor de pbgl que conviene tener delante, tres líneas encima
del código que nos ocupa:

```c
// NOTE: reading directly from contiguous memory where the palette is stored
// seems to be very slow and sometimes seemingly causes hangs for some reason
```

Es decir: **pbgl ya tenía cuelgues conocidos y no explicados alrededor de la paleta**,
y su solución fue copiarla a la pila antes de usarla. Que nuestro cuelgue sea
data-dependiente en un camino de textura paletizada, en la única biblioteca del port
cuyo autor documenta cuelgues raros justo ahí, es la coincidencia más sugerente que
tenemos. No es prueba.

### 41.3 Las trazas de dentro (tu punto 3)

Igual que se hizo con `SV_LoadProgs` en §26-28, el hueco se parte hasta que el log
señale un paso y no una función:

**Lado motor** (`ref/gl/gl_image.c`, rama indexada de Xbox):
`up1: resample WxH -> WxH` → `up2: resampled` → `up3: raw WxH` → `up4: raw done` →
`up5: checked`.

**Lado pbgl** (`texture.c`): `pbgl tex: store WxH L0, mips yes` → dentro,
`pbgl mips: N..M, bpp B, palnum P`, y **por cada nivel**
`pbgl mips: L<n> WxH from WxH` → `pbgl mips: L<n> scaled, swizzling` →
`pbgl mips: L<n> done` → al salir, `pbgl tex: stored`.

Con eso, el próximo log dice si se queda en el resample del motor, en la reserva, en
la generación de un mip concreto, en el swizzle de ese mip, o en la llamada GL. Cuesta
unos seis apuntes por textura —unas 4.600 líneas de disco síncrono, quizá 7 s más de
arranque— y a cambio nombra el paso.

### 41.4 Qué NO está verificado

- **Nada de esta ronda ha corrido.** Compilado, verificado en el artefacto (las seis
  cadenas nuevas, flags `0x01`, 0 SSE2), `libpbgl.lib` reconstruida entera y el motor
  relinkado a mano, subido.
- **La sospecha de la paleta** es una coincidencia documentada, no un diagnóstico.
- **Por qué la máquina entera muere** en vez de solo el hilo sigue sin explicación
  medida; la mejor sigue siendo la de §40.2 (algo que no permite planificar), y ahora
  con un matiz importante en contra: aquí no hay reserva de memoria contigua de por
  medio —la VRAM está holgada y ningún `pbgl mem:` se imprimió—, así que la historia
  del kernel buscando páginas ya no sirve. Si el próximo log muere dentro de un
  `swizzle_rect` o un `tex_mip8`, habrá que explicar cómo un bucle de usuario congela
  el planificador. Esa será la pregunta de §42.

### 41.5 Estado

`default.xbe` 4.624.384 B subido; `xash.cmd` = `-noip -dev 2 +map c1a0`, sin tocar —
octava vez. `pbgl-bounded-waits.patch` en 252 líneas, con memoria, esperas y trazas.

Dos scripts nuevos en `research/`: `ofx-mdl-texdata.sh` (integridad de la tabla de
texturas de un MDL) y `ofx-vram-sum.sh` (VRAM acumulada según el log).

Lo que hay que buscar en el próximo log es una sola línea: la última antes del
silencio, que ahora será una de once posibles en vez de un `upload:` mudo.

## 42. Paso 3a — Muere antes de `up1`, donde no hay ningún bucle: el sospechoso es el trazador

**El log responde la pregunta con una precisión que da la vuelta a la investigación:
tras `upload: #models/gman/GMan_Shin1.mdl 40x54 -> 32x64 type 2` **no hay `up1`**. No
muere en el resample, ni en la llamada GL, ni dentro de pbgl: muere antes de todos
ellos, en un tramo de código que —leído línea a línea— no contiene un solo bucle,
llamada bloqueante ni reserva de memoria. Ese callejón sin salida obliga a mirar al
único código que sí se ejecuta ahí y que hasta ahora estaba fuera de sospecha por ser
«el que mira»: el propio `Xbox_Trace`, que desde §37 termina cada línea leyendo un
registro de la GPU y escribiendo 390 KB en memoria write-combined. La línea se escribe
en disco *antes* de eso, que es exactamente por lo que la vemos y por lo que la
siguiente no existe.**

| Paso | ¿Llegó? |
|---|---|
| `upload:` (entrada) | **sí** — es la última línea del log |
| `up1: resample` | **no** |
| `up2`…`up5`, `pbgl tex: store`, `pbgl mips: L<n>` | no se alcanzan |
| Vecina `GMan_Arm-R1` | el mismo camino entero en **2 ms** |

### 42.1 El tramo, auditado línea a línea

Entre las dos trazas hay exactamente esto, y todo se leyó en el fuente:

| Código | Veredicto |
|---|---|
| Copia de `fogParams`, cálculo de `buf`/`bufend`/`offset`/`texsize` | aritmética |
| `Con_Reportf` de `s&3` | **no se ejecuta**: 40×54 = 2160, y 2160 & 3 = 0 |
| `pglBindTexture` | **pbgl `texture.c:729`: asignaciones puras**, sin bucles |
| `pglColorTableEXT` | **pbgl `texture.c:958`: valida y asigna punteros**, sin bucles |
| `GL_CalcMipmapCount` | acotado a 16 iteraciones (`gl_image.c:441`) |
| Comprobación de desbordamiento de búfer | dispararía `Host_Error`, que imprimiría |

Ninguno puede colgarse. Y no es teoría: las tres vecinas con **destino idéntico**
(32×64 type 2) recorren ese mismo tramo en 2 ms cada una, la última de ellas
—`GMan_Arm-R1`— doce milisegundos antes.

### 42.2 El testigo como sospechoso

Cuando el código que puede fallar no puede fallar, el que sobra es el que no se estaba
mirando. `Xbox_Trace` hace, en este orden:

1. `debugPrint` a la pantalla,
2. serie (desactivada, no hay UART),
3. **escritura síncrona al disco** — abrir, añadir, cerrar,
4. `Xbox_OverlayPush` al anillo,
5. **`Xbox_TraceOverlayDraw()`** — añadido en §37.

El paso 3 es la razón de que la línea `upload:` exista en el fichero. Todo lo que
falle en 4 o 5 deja el log exactamente como lo encontramos: con la línea puesta y sin
continuación. **El trazador está dentro del hueco que estamos midiendo**, y lo ha
estado desde §37 sin que nadie lo pusiera en la lista.

Y el paso 5 hace dos cosas que ningún otro código del port hace tan a menudo:

```c
fb = (unsigned char *)( 0x80000000u | ( XBOX_OVERLAY_PCRTC_START & 0x07FFFFFFu ));
```

- **Lee un registro MMIO de la GPU** (`0xFD600800`, `PCRTC_START`) — una transacción
  de bus contra el NV2A, una por cada línea trazada.
- Escribe hasta 390 KB en memoria **write-combined** que el CRTC está escaneando, otra
  vez por cada línea.

Esto encaja con lo que ninguna hipótesis anterior lograba explicar: **por qué muere la
máquina entera y no solo el hilo**. Un bucle de usuario no apaga el planificador, pero
una lectura MMIO que no completa **sí** cuelga la CPU en el bus, y con ella las
interrupciones, el planificador, el vigilante y START+BACK. Es la misma familia que la
nota que el autor de pbgl dejó escrita tres líneas encima de su código de paletas:
*«leer directamente de la memoria contigua donde está la paleta parece muy lento y a
veces provoca cuelgues»*. Accesos a memoria especial que a veces no vuelven.

**Qué no explica todavía:** por qué en esa textura y no en las 6.371 trazas anteriores.
La respuesta honesta es que no se sabe, y que un fallo dependiente de estado del bus
puede muy bien parecer determinista si el estado se reproduce igual en cada arranque —
que es justo lo que hace un precache idéntico ocho veces seguidas.

### 42.3 El experimento, que no necesita compilar nada

`-nooverlay` existe desde §39 y nunca se usó. Ahora vale su peso en oro:

```
xash.cmd = -noip -dev 2 +map c1a0 -nooverlay
```

Con el pintor apagado no hay lectura de `PCRTC_START` ni escrituras a VRAM, pero el log
sigue completo — el disco es un camino independiente. La lectura del resultado es
binaria:

- **Pasa de `GMan_Shin1`** → el pintor es el culpable y el port tiene un bug propio,
  no de pbgl ni del gamedll. El arreglo sería acotar el pintor: repintar como mucho
  cada N ms en vez de en cada línea, y cachear el puntero del framebuffer en lugar de
  releer el registro 6.000 veces.
- **Muere igual, pero ahora con `up0a`/`up0b`** (añadidas en esta ronda alrededor de
  `pglBindTexture` y `pglColorTableEXT`) → el pintor queda exonerado y el hueco se
  parte en tres, con `glColorTableEXT` como siguiente sospechoso pese a no tener
  bucles: sería un fallo de hardware/estado, no de lógica.
- **Muere sin llegar a `up0a`** → sólo queda el trazador, aunque no pinte, y habría que
  mirar `debugPrint` y la escritura FATX.

Cualquiera de los tres resultados elimina al menos una mitad del espacio de búsqueda,
y ninguno cuesta una recompilación.

### 42.4 Qué NO está verificado

- **La hipótesis del bus es una inferencia por eliminación**, apoyada en que explica el
  único hecho que las demás no explicaban (la muerte del planificador). No hay medida.
- **El XBE nuevo** (con `up0a`/`up0b`) está compilado, verificado y subido, pero **no
  ejecutado**.
- Sigue sin explicarse la **selectividad**: por qué esta textura, siempre.

### 42.5 Estado

`default.xbe` 4.624.384 B subido, con `up0a`/`up0b`. `xash.cmd` **sí cambia esta vez**,
por primera vez desde §33: `-noip -dev 2 +map c1a0 -nooverlay`, ya escrito y releído en
la consola.

Ocho ejecuciones han hecho falta para llegar aquí, y el recorrido merece decirse: el
muro pasó de ser «el mapa no carga» a «una textura concreta», de ahí a «un paso
concreto de su subida», y ahora a «el instrumento que usábamos para mirar». Es el
riesgo clásico de instrumentar una máquina sin depurador — y también la única razón
por la que hay algo que leer.

## 43. Paso 3b — El pintor era inocente: el cuelgue vive en cinco líneas sin un solo bucle

**Veredicto del experimento de §42.3: el tercer caso. Con `-nooverlay` confirmado en
el log (`overlay disabled by -nooverlay`, sin una sola lectura de `PCRTC_START` ni
escritura a VRAM en toda la ejecución), el motor muere **en el mismo sitio exacto** y
sin llegar siquiera a `up0a`. El trazador queda exonerado por completo y con él la
hipótesis del bus MMIO que §42 había construido. Lo que queda es un tramo de cinco
sentencias —copias de campos, dos funciones aritméticas y un `glBindTexture` que es
puro asignar— en el que no hay un bucle, ni una reserva, ni una llamada bloqueante, y
del que el motor no sale. Esta ronda le pone una traza a cada sentencia.**

| | |
|---|---|
| `-nooverlay` | activo y verificado: `overlay disabled by -nooverlay` (línea 137) |
| Muerte | `upload: #models/gman/GMan_Shin1.mdl` — línea 8166, la última |
| `up0a: bound` | **no llega** |
| Vecina `GMan_Arm-R1` | el camino completo, `up0a`→`uploaded`, en **25 ms** |
| Veredicto | el pintor **no** es el culpable; el trazador tampoco |

### 43.1 Por qué esto exonera al trazador entero, no solo al pintor

El orden interno de `Xbox_Trace` es: `debugPrint` → serie → **escritura al disco** →
`OverlayPush` → `OverlayDraw`. Que la línea `upload:` esté en el fichero demuestra que
los tres primeros pasos terminaron. `OverlayPush` es una copia a un anillo en memoria
normal, y `OverlayDraw` con `-nooverlay` **retorna en la primera instrucción**. No
queda nada del instrumento donde esconderse: cuando el motor sale de esa llamada a
`Xbox_Trace`, está vivo y la máquina responde.

Con eso caen las dos hipótesis que §42 había levantado: ni la lectura MMIO de
`PCRTC_START`, ni las escrituras write-combined al framebuffer. Y cae también, de
rebote, la explicación que teníamos para el hecho más raro de todos —que muera la
máquina entera y no solo el hilo—, que se apoyaba precisamente en esos accesos.

### 43.2 El hueco, ahora de cinco sentencias

Entre la traza que sí sale y la que no, esto es **todo** lo que se ejecuta:

```c
tex->fogParams[0..3] = pic->fogParams[0..3];   // 4 copias de float
// (el Con_Reportf de s&3 no entra: 40*54 = 2160, 2160 & 3 == 0)
buf    = pic->buffer;
bufend = pic->buffer + pic->size;
offset = Image_CalcImageSize( pic->type, 40, 54, pic->depth );
texsize = GL_CalcTextureSize( tex->format, 32, 64, tex->depth );
normalMap = ...; numSides = ...; texture3d = ...;
glState.currentTextures[...]      = tex->texnum;
glState.currentTexturesIndex[...] = tex - gl_textures;
pglBindTexture( tex->target, tex->texnum );
```

`GL_CalcTextureSize` (`gl_image.c:316`) es un `switch` con multiplicaciones.
`glBindTexture` de pbgl (`texture.c:729`) son comparaciones y asignaciones. Ninguna
de las dos tiene un bucle. Y `GMan_Arm-R1`, con **el mismo destino 32×64 type 2**,
recorrió este tramo 36 ms antes sin despeinarse.

De modo que solo quedan dos formas de que el flujo no llegue a la siguiente línea:

1. **Una de esas sentencias falla al tocar memoria** — un puntero corrupto en `pic`,
   o un `tex->texnum` fuera de rango — y la excepción se lleva el título por delante.
2. **Algo ajeno a la CPU detiene la máquina** justo ahí, y la coincidencia con esta
   textura es de calendario, no de causa.

La primera es ahora la favorita, y por un motivo nuevo: **la consola apareció en el
dashboard**. En las cuatro roturas anteriores hubo que resetear a mano. Una excepción
no capturada en un título de Xbox termina en reinicio del kernel, que es exactamente
lo que se ve; un bus colgado, no.

**Y eso hay que confirmarlo**, porque cambia el diagnóstico entero: no es lo mismo
«volvió sola» que «la reseteaste». Es la primera pregunta del próximo mensaje.

### 43.3 Una traza por sentencia

Cinco marcas nuevas, en `ref/gl/gl_image.c`, con datos y no solo posición:

| Traza | Qué demuestra si sale |
|---|---|
| `upA: pic=… buf=… size=… pal=… flags=… depth=… mips=…` | `pic` es legible entero, con sus punteros a la vista |
| `upB: fog+bounds ok` | las copias de `fogParams` y la aritmética de punteros pasaron |
| `upC: imagesize N` | `Image_CalcImageSize` volvió |
| `upD: texsize N, fmt X` | `GL_CalcTextureSize` volvió |
| `upE: binding texnum N target X` | se llega a la llamada, con el número de textura delante |

`upA` es la más valiosa de las cinco: si sale y enseña un `buf` o un `pal` absurdo, el
diagnóstico está hecho y el culpable es quien cargó el modelo, no quien lo sube. Si no
sale, el fallo está en la propia lectura de `pic` — y entonces `pic` es basura desde
antes, lo que apunta al `rgbdata_t` del cargador studio.

### 43.4 Qué NO está verificado

- **Nada de esta ronda ha corrido**: compilado, 0 SSE2, flags `0x01`, subido.
- **Si la consola volvió sola al dashboard** — pendiente de confirmación, y es la
  bisagra entre «excepción» y «cuelgue de bus».
- **La selectividad** sigue sin explicación: la misma textura, siempre, ahora también
  con el pintor apagado. Ocho ejecuciones, ocho veces la 765ª.

### 43.5 Estado

`default.xbe` 4.624.384 B subido con `upA`…`upE`. `xash.cmd` se queda en
`-noip -dev 2 +map c1a0 -nooverlay`: el pintor apagado es una variable menos y ya no
hace falta para nada — el log en disco basta como testigo, que es como empezó todo en
§20.

El cerco está en cinco sentencias. O una de ellas toca memoria que no debe, y el
próximo log lo dirá con nombres y punteros, o el problema no está en el código que
ejecuta la CPU — y entonces habrá que preguntarse qué hace el NV2A mientras tanto.

## 44. Paso 3c — Textura 768: un `realloc` que mueve el array y deja punteros colgando

**Ocho ejecuciones apuntando a una textura de 3 KB, y no tenía nada que ver con la
textura. El log de esta ronda pone el número encima de la mesa:
`upE: binding texnum 768` es la última línea, y `up0a: bound` —tres instrucciones más
allá— no llega nunca. Muere **dentro de `glBindTexture`**, con el número de textura
**768**, que es exactamente el múltiplo de `TEX_ALLOC_STEP` (256) donde pbGL amplía su
array de texturas con `realloc`. Y `realloc` mueve el bloque: `pbgl.tex[unidad].tex`
es un puntero crudo dentro de ese array que nadie rebasa, así que la primera cosa que
hace `glBindTexture` a continuación es escribir a través de un puntero a memoria
recién liberada. Arreglado, verificado en el artefacto y subido.**

| | |
|---|---|
| Última línea | `[11752 33846] upE: binding texnum 768 target de1` |
| Paso siguiente | `up0a: bound` — **nunca sale** |
| Número de textura | **768** = 3 × `TEX_ALLOC_STEP`, el punto de crecimiento del array |
| Vecina `GMan_Arm-R1` | mismo camino, texnum 767, **9 ms** antes, perfecta |
| Arreglo | rebasar `pbgl.tex[u].tex` tras el `realloc` (`pbgl/src/texture.c:699`) |

### 44.1 El mecanismo, completo

```c
// pbgl/src/texture.c, glGenTextures()
texture_t *newtex = realloc(textures, sizeof(texture_t) * (tex_cap + step));
...
textures = newtex;          // <- el array puede haberse MOVIDO
```

```c
// pbgl/src/texture.c, glBindTexture()
if (pbgl.tex[pbgl.active_tex_sv].tex)
    pbgl.tex[pbgl.active_tex_sv].tex->bound = GL_FALSE;   // <- puntero al bloque viejo
```

`pbgl.tex[unidad].tex` se rellena en cada `glBindTexture` con `&textures[n]`. Cuando el
array crece y `realloc` lo reubica, esos punteros —uno por unidad de textura— siguen
apuntando al bloque que `realloc` acaba de liberar. Nadie los actualiza. La secuencia
que nos mata es exactamente:

1. El motor pide una textura nueva para `GMan_Shin1`; toca el número **768**, que ya no
   cabe en `tex_cap`.
2. `glGenTextures` hace `realloc` a 1024 entradas. El bloque se mueve.
   `pbgl.tex[0].tex` queda colgando.
3. El motor llama a `pglBindTexture( GL_TEXTURE_2D, 768 )`.
4. La comprobación `tex >= tex_cap` pasa (768 < 1024), y la línea siguiente **escribe**
   a través del puntero muerto.

Y explica limpiamente todo lo que la investigación no lograba encajar:

- **Por qué esa textura y no otra**: porque es la 768ª, no porque sea ella. El mapa
  entero es determinista, así que el cruce cae siempre en el mismo sitio.
- **Por qué daba igual el tamaño, el formato y el contenido**: nunca se llegó a mirar
  su contenido; muere antes de tocarlo.
- **Por qué daba igual la presión de memoria** (50 MB en §40 contra 6,7 MB en §41): el
  disparador es el **contador de texturas**, no los bytes.
- **Por qué el pintor era inocente** (§43): el problema estaba tres llamadas más
  adelante, en pbGL.
- **Por qué muere la máquina entera y no solo el hilo**: escribir en un bloque recién
  liberado corrompe la estructura de la que salen, poco después, las direcciones
  físicas que se programan en el NV2A. Una dirección inválida en la GPU sí puede dejar
  el bus sin completar una transacción, y con él la CPU sin interrupciones, sin
  vigilante y sin START+BACK. **Esta última parte sigue siendo inferencia**: lo
  demostrado es dónde muere, no qué hace la GPU después.

### 44.2 El arreglo

En `glGenTextures`, justo después del `realloc` y antes de publicar el puntero nuevo:

```c
if (newtex != textures) {
  Xbox_Trace("pbgl tex: array moved %p -> %p (cap %u -> %u), rebasing units", ...);
  for (GLuint u = 0; u < TEXUNIT_COUNT; ++u)
    if (pbgl.tex[u].tex)
      pbgl.tex[u].tex = newtex + (pbgl.tex[u].tex - textures);
}
```

Solo hay que rebasar esos punteros: los demás punteros de un `texture_t`
(`palette.data`, `mips[].data`) apuntan **dentro de su propio bloque de VRAM
contigua**, que el `realloc` del array no toca, y `shared_palette` apunta al pool
global. La traza deja constancia del momento del movimiento, que es un suceso raro y
que conviene ver en el log cuando ocurre.

**Es un bug de pbGL, no del port**: cualquier proyecto que pase de 256, 512 o 768
texturas lo encuentra. No es una peculiaridad de la Xbox ni de este motor.

### 44.3 Lo que el log deja de propina

- `upA: pic=0xf991f0 buf=0xc71960 size=2160 pal=0xc71540 flags=0 depth=1 mips=1` — el
  `rgbdata_t` estaba perfecto. La sospecha de §43 sobre punteros corruptos en `pic`
  queda descartada con datos.
- `upC: imagesize 0` — `Image_CalcImageSize` devuelve **0** para `PF_INDEXED_32`.
  Inofensivo aquí (`offset` no se usa en el camino indexado de Xbox), pero es una cifra
  falsa que puede engañar a quien lea este código en el futuro. Anotado como deuda.
- `upD: texsize 2048, fmt 80e5` — `GL_COLOR_INDEX8_EXT`, correcto.
- `target de1` = `GL_TEXTURE_2D`, correcto.

### 44.4 Qué NO está verificado

- **El arreglo no se ha ejecutado.** Compilado, `libpbgl.lib` reconstruida entera,
  motor relinkado, cadenas `array moved` y `rebasing units` verificadas dentro del
  `default.xbe`, 0 SSE2, flags `0x01`, subido.
- **La cadena que va del puntero colgante al cuelgue del bus** es la mejor explicación
  disponible, no una medida.
- **Qué hay después de la textura 768** es territorio virgen: el precache de c1a0
  sigue, y quedan las 11 texturas restantes del G-Man, el resto de entidades, y todo
  lo que venga después de `SV_SpawnServer`. El siguiente muro puede estar a diez
  líneas o a diez mil.

### 44.5 Estado

`default.xbe` 4.624.384 B subido con el arreglo y con `upA`…`upE` puestas — se dejan a
propósito una ronda más: si el arreglo funciona, la textura 768 pasará de largo y esas
trazas lo dirán con nombre y número; después se quitan, porque cuestan seis escrituras
a disco por textura.

`xash.cmd` sigue en `-noip -dev 2 +map c1a0 -nooverlay`.
`pbgl-bounded-waits.patch`, que ya no le hace justicia al contenido, pasará a llamarse
por lo que es: esperas acotadas, instrumentación del asignador, trazas de mipmaps y
**el arreglo del `realloc`** — el primero de los tres que es un bug de verdad y no un
testigo.

Lo que hay que buscar en el próximo log es una línea que nunca se ha visto:
`pbgl tex: array moved`, seguida —por primera vez en nueve ejecuciones— de
`up0a: bound`.

## 45. Paso 3d — HITO: Black Mesa en la consola. c1a0 cargado y renderizando

**El arreglo de §44 era el último muro. `+map c1a0` carga entero y la Xbox está
dibujando Black Mesa: el sector Anomalous Materials, con su geometría, sus texturas y
su iluminación, en una máquina de 2001. Es el primer mapa jugable del port y el final
del camino que empezó en §21 con una pantalla negra sin una sola línea de log. Nueve
ejecuciones hicieron falta para cruzar la textura 768; la décima entró en el mapa.**

| | |
|---|---|
| Mapa | **c1a0 cargado y renderizando** |
| Causa raíz vencida | el `realloc` de pbGL que dejaba punteros colgando (§44) |
| Cadena completa | menú (§32) → barras (§34) → memoria (§36-37) → textura 768 (§44) → **mapa** |
| Pendiente a la vista | **los NPC no aparecen**: ni científicos ni Barney |
| Trazas retiradas | `upA`…`upE`, `up1`…`up5`, mipmaps de pbgl — seis a veintiuna escrituras por textura |

### 45.1 Lo que este hito confirma de una vez

Todo lo que se construyó a ciegas durante veinticinco secciones funciona a la vez y en
la misma ejecución:

- **pbgl y el NV2A** dibujando geometría de verdad, no un menú: BSP, lightmaps,
  texturas del pak.
- **Los seis módulos estáticos** dentro del XBE (§24-29), con el gamedll de servidor
  ejecutando la lógica de spawn del mapa.
- **El filesystem** sirviendo un `pak0.pak` de 299 MB con las barras correctas (§34).
- **El presupuesto de memoria** aguantando un mapa entero con el techo de 512 px (§36).
- **El bucle de frames** vivo y con latido (§30), sobre 128 MB reales (§36).

Y el arreglo de §44 queda validado por el hecho más simple posible: la textura 768 ya
no mata la máquina.

### 45.2 Los NPC no están, y hay que distinguir dos cosas muy distintas

`scientist.mdl` y `barney.mdl` **se cargaron en el precache** —está en los logs desde
§38— y sin embargo no hay nadie en el laboratorio. Eso admite dos explicaciones que no
se parecen en nada:

1. **No existen**: el gamedll de servidor no llegó a instanciar las entidades, o el
   cliente nunca recibió su estado.
2. **Existen y no se dibujan**: llegan al cliente y el renderer de modelos estudiados
   no los pinta.

Descartado ya por lectura de código, antes de gastar una ejecución: **el loopback no
está roto por `-noip`**. Era la sospecha natural —§30.5 lo dejó anotado como sin
verificar— pero `net.initialized` se pone a `true` al final de `NET_Init` pase lo que
pase (el log lo confirma con `Base networking initialized`), y el camino local no usa
sockets: `NET_SendPacket` desvía a `NET_SendLoopPacket` y `NET_GetPacket` lee de
`NET_GetLoopPacket`, dos búferes en memoria. `-noip` apaga la pila IP, no el loopback.

### 45.3 La instrumentación que separa las dos historias

Dos pulsos, uno a cada lado de la frontera, ambos una vez por segundo:

**En el motor**, ampliando el latido de §30:

```
heartbeat: frame N, 30.0 fps, sv ents A, cl ents B
```

`sv ents` es `svgame.numEntities` —lo que el gamedll de servidor creó de verdad— y
`cl ents` es `cl.frames[cl.parsecountmod].num_entities`, los estados de entidad que
cruzaron el netchan y llegaron al cliente **este frame**.

**En el renderer** (`R_EndFrame`, dentro de ref_gl, que es donde vive `r_stats` y el
motor no lo ve):

```
renderpulse: client ents C, studio drawn D (P polys), sprites S, world W polys / L leafs
```

La tabla de verdad es directa:

| `sv ents` | `cl ents` | `studio drawn` | Diagnóstico |
|---|---|---|---|
| 0 | 0 | 0 | el gamedll no creó nada: problema de spawn en el servidor |
| alto | 0 | 0 | creadas pero no llegan: el netchan de loopback o la visibilidad (PVS) |
| alto | alto | 0 | llegan y no se dibujan: **el renderer de modelos estudiados** |
| alto | alto | >0 | se dibujan pero no se ven: transformación, escala o cámara |

Los contadores de mundo (`world polys / leafs`) sirven de control: si salen altos, el
BSP se está dibujando y el renderer está vivo, con lo que un cero en `studio drawn`
señala específicamente al camino de modelos y no al renderer entero.

### 45.4 Trazas retiradas

Con el bug de §44 cerrado, se van las que costaban disco por textura: `upA`…`upE`
(cinco), `up1`…`up5` (cinco) y las de pbgl por nivel de mipmap (**veintiuna** por
textura, las más caras que ha tenido este port). Se quedan `upload:`/`uploaded:`, dos
por textura, que son las que hicieron inequívoca la señal en §41 y valen su precio si
aparece otro muro en el camino de texturas.

### 45.5 Qué NO está verificado

- **El log de esta ejecución todavía no se ha recogido**: la consola desaparece de la
  red mientras el título corre (§21.5), y el juego seguía en pantalla. Los números de
  arranque, memoria y fps del primer mapa llegan con él.
- **El XBE con la instrumentación de NPC está compilado y verificado** (los pulsos
  presentes, las trazas caras ausentes, flags `0x01`, 0 SSE2) **pero no subido**, por
  la misma razón.
- **Nadie ha jugado todavía**: no se ha comprobado el mando, ni las armas en primera
  persona, ni el HUD, ni el sonido en el mapa. La foto es de una cámara quieta.
- **El presupuesto de memoria con el mapa cargado** sigue sin medirse.

### 45.6 Estado

El port arranca, monta su filesystem, enciende la GPU, carga los seis módulos, ejecuta
el gamedll, entra en el bucle de frames y **dibuja un mapa de Half-Life en una Xbox
original**. Lo que queda de aquí en adelante ya no es «conseguir que arranque»: es
que el juego se comporte como un juego. Los científicos ausentes son el primer punto
de esa lista nueva.

## 46. Paso 3e — Veintisiete minutos a 60 fps, y la zona muerta que FWGS no reescala

**El log del primer mapa jugable trae tres cosas: la confirmación en vivo del arreglo
de §44 —tres reubicaciones del array de texturas registradas, una de ellas saltando de
`0xfb8658` a `0x7fce0008`, dos gigas de distancia—, una sesión de **92.878 frames en
27 minutos a 59,9-60,0 fps sostenidos**, y el primer `START+BACK` de la historia del
port ejecutado desde dentro de un mapa. Sobre el mando: FWGS sí trae zonas muertas y
sensibilidades, y llegan bien; el problema es cómo aplica la zona muerta —un corte
seco sin reescalar, que hace saltar el eje al 12,5% de la velocidad máxima en cuanto
lo cruzas—. Arreglado, con curva de respuesta añadida, y ajustable por FTP sin
recompilar.**

| | |
|---|---|
| Sesión | **92.878 frames, 27 min, 59,9-60,0 fps** |
| `START+BACK` | **funciona desde el mapa** — `returning to dashboard: START+BACK` |
| Arreglo de §44 | 3 reubicaciones registradas: 256→512, 512→768, **768→1024** |
| Memoria en el bucle | 96.980 KB libres, mayor malloc 96.256 KB |
| Texturas | 823 subidas, la 819 enlazada sin incidente (antes moría en la 768) |
| NPC | **existen en el servidor** — el gamedll ejecuta su código de IA |

### 46.1 El arreglo de §44, visto desde dentro

```
[3917   19372] pbgl tex: array moved 0xc5da58   -> 0xfb8658   (cap 256 -> 512)
[7802   26597] pbgl tex: array moved 0xfb8658   -> 0x7fce0008 (cap 512 -> 768)
[11749  33883] pbgl tex: array moved 0x7fce0008 -> 0x7fbd0008 (cap 768 -> 1024)
```

Tres veces se movió el array durante la carga del mapa, y la segunda **cambió de
región de memoria entera**: de `0x00fb8658` a `0x7fce0008`. Un puntero sin rebasar no
apuntaba «unos bytes más allá», apuntaba a dos gigabytes de distancia. Se entiende que
la máquina no sobreviviera a escribir ahí.

Y la tercera reubicación —la que cruza 768— es la que mataba el port desde §38. Ahora
sale en el log, se rebasan las unidades, y nueve líneas después la textura 769 se sube
como cualquier otra.

### 46.2 El mando: qué trae FWGS y qué está mal

**Sí trae todo lo que hacía falta**, y llega bien a este target:

| cvar | Por defecto | Qué hace |
|---|---|---|
| `joy_pitch_deadzone`, `joy_yaw_deadzone` | `4096` (12,5%) | zona muerta de la cámara |
| `joy_forward_deadzone`, `joy_side_deadzone` | `4096` | zona muerta del movimiento |
| `joy_pitch`, `joy_yaw` | `100.0` | grados/segundo a fondo de stick |
| `joy_side`, `joy_forward` | `1.0` | escala de movimiento, negativo invierte |
| `joy_side_key_threshold`, `joy_forward_key_threshold` | `24576` | a partir de dónde el eje simula una tecla |

`DEFAULT_JOY_DEADZONE` vale `4096` en este build (`common/defaults.h:188`), los cvars
se registran en `Joy_Init` y el camino de eventos es el normal:
`SDLash_HandleGameControllerEvent` → `Joy_AxisMotionEvent` → `Joy_ProcessStick`. Nada
roto ahí. **Curva de respuesta no trae ninguna**: la conversión es lineal.

**Lo que sí está mal es cómo aplica la zona muerta** (`in_joy.c:245`, upstream):

```c
if( value < deadzone && value > -deadzone )
    value = 0;   // y el resto del recorrido se queda como estaba
```

Corta el centro y **no reescala lo que queda**. Consecuencia directa: mientras estás
dentro de la zona muerta no pasa nada, y en el instante en que la cruzas el eje vale
4096/32767 — **la cámara arranca de golpe al 12,5% de la velocidad máxima**. No hay
transición: se pasa de quieto a girar a doce grados por segundo. Con ratón nadie lo
nota porque el ratón no reposa en un borde; con un stick es exactamente la sensación
de «se mueve con la mínima presión».

### 46.3 El arreglo, y cómo ajustarlo sin recompilar

En `Joy_ProcessStick`, bajo `#if XASH_XBOX`, la zona muerta pasa a **reescalarse**:

```
t = (|v| - deadzone) / (32767 - deadzone)
```

con lo que la respuesta empieza en cero y llega a fondo igual que antes, sin saltos. Y
encima, una curva opcional, `joy_curve`, que mezcla lineal con cuadrática:

```
t' = t·(1-c) + t²·c
```

`0.0` es lineal (lo de FWGS), `1.0` cuadrática pura, y el valor por defecto es `0.5`:
control fino cerca del centro sin perder velocidad al fondo. Se implementa como mezcla
y no con un exponente a propósito — `powf` no es de fiar en pdclib y esto no necesita
más.

**El ajuste va por FTP, no por recompilación.** `Cmd_Userconfigd_f` (`cmd.c:1396`)
ejecuta todos los `.cfg` de `userconfig.d/` al final del arranque, y **el motor nunca
escribe ahí** —al contrario que `config.cfg`, que sobrescribe al salir—. Así que el
sitio correcto es:

```
E:\Apps\xash\valve\userconfig.d\joystick.cfg
```

Ya está subido, con las cuatro zonas muertas, la curva, las sensibilidades y un
comentario por línea explicando qué mueve cada cosa. Se edita por FTP, se reinicia el
título y listo. Los valores de partida son `6000` de zona muerta en la cámara (18%),
`5000` en el movimiento y `joy_curve 0.5`.

Y para no volver a preguntarse si los valores llegan, `Joy_Init` los dice en el log:

```
joystick: deadzones pitch N yaw N fwd N side N, sens pitch N yaw N, curve x100 N
```

### 46.4 Los NPC: existen, y ahora se sabe

El log lo resuelve a medias, y con una prueba sólida:

```
[con 33548] monster_furniture has no view_ofs!
[con 34220] monster_gman has no view_ofs!
```

Ese mensaje sale de `dlls/monsters.cpp:2598` del **gamedll**, dentro del código de
monstruos. Es decir: el servidor no solo creó `monster_gman`, sino que **ejecutó su
código de IA**. Los NPC existen. Lo que falla está aguas abajo: o sus estados no
llegan al cliente, o llegan y el renderer no los dibuja. La instrumentación de §45
—`sv ents` / `cl ents` en el latido, `renderpulse` en el renderer— está subida y lo
decide en la próxima ejecución.

### 46.5 Hallazgos colaterales de 27 minutos de juego

- **El cielo no carga**: `OpenGL Error: GL_INVALID_OPERATION while uploading
  gfx/env/desertrt.bmp [2D]`. Las seis caras del skybox pasan por un camino distinto
  al de las texturas normales; el techo de 512 de §44 no debería afectarlas, pero hay
  que mirarlo.
- **Faltan sonidos**: `Could not load sound sound/plats/vehicle_ignition.wav`.
- **Cvars de sonido desconocidos** (`s_refgain`, `s_occfactor`…): son de una versión
  distinta de la config; inofensivos.
- **El arranque es más rápido**: 96 MB libres al entrar al bucle contra los 64 de §37,
  gracias al techo de textura.

### 46.6 Qué NO está verificado

- **El arreglo del mando no se ha probado**: XBE subido (4.624.384 B, flags `0x01`,
  0 SSE2) y `joystick.cfg` en la consola, sin ejecutar.
- **Los valores de partida son una apuesta razonada**, no medida: el bueno se
  encuentra jugando, que es justo para lo que el fichero es editable.
- **Los NPC**: sigue sin saberse si el fallo es de red o de renderizado.
- El skybox y los sonidos que faltan, sin investigar.

### 46.7 Estado

El port corre Half-Life en una Xbox original **a 60 fps estables durante media hora**,
sale limpio con START+BACK y se configura editando un fichero de texto por FTP. Lo que
queda ya no es hacer que funcione: es que se juegue bien. La lista, por orden de lo
que rompe más la ilusión: los NPC invisibles, el cielo que no carga, y el tacto del
mando —que a partir de esta ronda se ajusta sin tocar el compilador.

## 47. Paso 3f — Los NPC son invisibles porque el mundo y ellos usan caminos GL distintos

**El log que se recogió no traía los contadores nuevos, y por una razón que merece
quedar escrita: la consola estaba corriendo un XBE viejo. El despliegue de §46 no
llegó y nadie lo comprobó, así que se leyó un log de la ronda anterior creyendo que
era de esta. Desde ahora el despliegue se verifica bajando el binario de vuelta y
comparando md5 (`ofx-xbox-verify-deploy.sh`). Con eso corregido, el análisis del
camino de dibujado da una respuesta que encaja con todo lo que se ve: **el mundo y los
modelos estudiados no usan el mismo camino de OpenGL**. El mundo cae a modo inmediato
porque pbGL no tiene VBOs; el studio usa vertex arrays sin condición. Uno se ve y el
otro no. Y por el camino, un bug real de pbGL con el stride 0.**

| | |
|---|---|
| Log recogido | de un XBE **anterior**: sin `joystick:`, sin `renderpulse`, con trazas ya retiradas |
| Causa | el despliegue de §46 no subió; **no se verificó** |
| Remedio | `ofx-xbox-verify-deploy.sh`: sube, **baja de vuelta y compara md5** |
| Mundo | camino **VBO → falla la extensión → modo inmediato** (pushbuffer) → **visible** |
| Studio | `R_StudioDrawArrays` → vertex arrays → DMA de la GPU → **invisible** |
| Bug encontrado | pbGL no traduce `stride 0` (OpenGL: «empaquetado»); arreglado |
| Segundo problema | cuelgue duro durante una escena guionizada; sin rastro |

### 47.1 El fallo de método, dicho antes que nada

El log de esta ronda tenía 13.135 líneas, latidos a 59,9 fps y ni uno de los tres
contadores que §45 añadió. La explicación no era un bug: **el XBE de la consola no era
el que se había compilado**. El comando de despliegue de §46 se truncó a cuatro líneas
de salida por conveniencia y nadie miró si la subida había ocurrido; el listado FTP
mostraba el mismo tamaño de fichero —4.624.384 B en las dos versiones— así que
tampoco delataba nada.

La norma de la casa era «verifica en el artefacto, no en el configure». Se queda corta:
hay que verificar **en el artefacto que está en la máquina**. `ofx-xbox-verify-deploy.sh`
sube el XBE, lo vuelve a bajar y compara md5. Cuesta dos segundos y cierra una clase
entera de sesiones perdidas.

### 47.2 Por qué el mundo se ve y los NPC no

Los dos dibujan geometría, pero por caminos que no se parecen:

**El mundo** (`gl_rsurf.c`) tiene un camino de VBO —`pglDrawElements` sobre búferes de
la GPU— y otro de modo inmediato. El de VBO exige `GL_ARB_vertex_buffer_object`, y el
log lo lleva diciendo desde §32:

```
GL_CheckExtension: GL_ARB_vertex_buffer_object - failed
```

Así que el mundo cae al modo inmediato: `glBegin`/`glEnd`, con cada vértice empujado
por el **pushbuffer**. La GPU nunca lee memoria de la CPU; recibe los datos ya
copiados. Funciona, y por eso Black Mesa se ve.

**Los modelos estudiados** (`gl_studio.c:2196`) no tienen alternativa: `R_StudioDrawArrays`
llama a `pglVertexPointer`/`pglTexCoordPointer`/`pglColorPointer` y luego a
`pglDrawElements`, **sin comprobar ninguna extensión**. Ahí la GPU no recibe datos:
recibe **direcciones**, y va a buscarlos ella por DMA a la memoria del proceso.

Ese cambio de mecanismo es la frontera exacta entre lo que se ve y lo que no. Y el
dato que aportaste lo respalda desde otro ángulo: en PC, con llvmpipe, el cuelgue de
§23 moría en `R_StudioDrawArrays` → `R_StudioSubmitMesh`. El mismo camino, roto en dos
implementaciones de GL distintas, es una pista fuerte de que el problema está en lo
que Xash pide, no en quién se lo da.

Descartado por comprobación directa, para que no quede como sospecha: las ocho
funciones implicadas (`glEnableClientState`, `glVertexPointer`, `glTexCoordPointer`,
`glColorPointer`, `glDrawElements`…) están **las ocho** en la tabla de `vid_xbox.c` y
**las ocho** implementadas en pbGL. No hay punteros nulos ni funciones que falten.

### 47.3 El bug del stride 0

Buscando en ese camino apareció uno seguro (`pbgl/src/array.c:34`):

```c
arr->stride = stride;   // y de aqui, tal cual, al registro del NV2A
```

En OpenGL, `stride 0` **no** significa «sin salto»: significa «empaquetado», y la
implementación tiene que calcular `size * sizeof(type)`. pbGL metía el cero en
`NV097_SET_VERTEX_DATA_ARRAY_FORMAT_STRIDE`, y un stride de cero en el NV2A hace que
**todos los vértices lean el mismo elemento**.

Y es exactamente lo que el studio le pasa (`gl_studio.c:2202` y `:2207`):

```c
pglVertexPointer  ( 3, GL_FLOAT,         12, g_studio.arrayverts );  // stride explicito
pglTexCoordPointer( 2, GL_FLOAT,          0, g_studio.arraycoord );  // 0 = empaquetado
pglColorPointer   ( 4, GL_UNSIGNED_BYTE,  0, g_studio.arraycolor );  // 0 = empaquetado
```

Arreglado en `array_set`: si el stride llega a cero se calcula. **No es la explicación
completa** —con las posiciones correctas los modelos se verían mal texturizados, no
invisibles— pero es un bug de pbGL de manual y estaba en el camino.

### 47.4 Lo que decide la próxima ejecución

Dos trazas, una a cada lado de la llamada, ambas limitadas a una de cada 1800 (≈ una
por segundo) más las dos primeras:

```
studio draw#N: V verts, E elems (from a/b), chrome C     <- ref_gl, antes de pedir el dibujo
gl draw#N: E idx, pos[en/sz/st/@] tc[...] col[...]        <- pbgl, con lo que la GPU recibe
```

La segunda incluye **las direcciones de los arrays**, y eso importa: pbGL enmascara los
punteros a 26 bits (`array.c:65`, `& 0x03FFFFFF`) para convertir virtual en físico, lo
cual solo vale por debajo de 64 MB. Desde que se soltó el límite de memoria en §36, el
proceso puede tener datos por encima. Si `@` sale con una dirección alta, el
enmascarado está mandando a la GPU a leer donde no debe — y eso encajaría además con
los cuelgues duros.

Y los tres contadores de §45 (`sv ents` / `cl ents` / `renderpulse`) por fin correrán,
ahora que el binario correcto está en la consola.

### 47.5 El segundo problema: el bloqueo en una zona

El log acaba así, sin más:

```
[con 149009] Playing sentence !GM_1MUMBLE ()
[12541 150563] heartbeat: frame 6201, 58.9 fps
[con 150993] Firing: (argue_1)
[con 150994] Found: scripted_sentence, firing (argue_1)
```

Es la escena guionizada de los dos científicos discutiendo, con el G-Man mascullando
de fondo. Después, nada: **ni vigilante, ni excepción, ni línea de salida**. Otro
cuelgue duro de los que exigen el botón, igual que los de §38-§43.

Lo único que se sabe es cuándo: 150 segundos de partida, a 58,9 fps hasta el último
latido —sin degradación previa—, en pleno `scripted_sentence`. Puede ser la misma
causa que los modelos invisibles (una GPU a la que se manda leer memoria equivocada
acaba colgando el bus, §42) o algo del gamedll en las secuencias guionizadas. **No hay
pista suficiente**, y esta ronda no añade instrumentación específica para ello a
propósito: si el arreglo de los modelos hace desaparecer el cuelgue, la causa era
compartida, y eso es información. Si no, se instrumenta entonces con la certeza de que
son dos cosas.

### 47.6 Qué NO está verificado

- **Nada de esta ronda ha corrido.** XBE compilado, 0 SSE2, y esta vez **verificado en
  la consola por md5** (`bf088756...`).
- **La hipótesis del camino de arrays no está probada**, solo razonada. Lo probado es
  que el mundo usa modo inmediato y el studio no.
- **El enmascarado a 26 bits** es sospechoso pero no medido; las trazas traen las
  direcciones justamente para eso.
- El cuelgue de la escena guionizada, sin diagnóstico.

### 47.7 Estado

`default.xbe` en la consola, verificado byte a byte, con el arreglo del stride, las dos
trazas del camino de studio y —por fin ejecutándose— los contadores de §45 y el ajuste
de mando de §46. `xash.cmd` sigue en `-noip -dev 2 +map c1a0 -nooverlay`.

En el próximo log, tres cosas y en este orden: si `studio draw#` aparece (el renderer
lo intenta), qué direcciones enseña `gl draw#` (si son altas, ahí está la causa), y qué
dicen `sv ents` / `cl ents` / `renderpulse` (que ya solo sirven para confirmar lo que
se sabe por otra vía: que existen y no se dibujan).

## 48. Paso 3g — No es que se dibujen mal: es que no se dibujan. Cero llamadas

**El log responde las tres preguntas y deja la sospecha de §47 sin efecto, porque no
llega a aplicarse: `R_StudioDrawArrays` **no se llama ni una sola vez** —cero trazas
`studio draw#`, cero `gl draw#`, `studio drawn 0 (0 polys)`— mientras el mundo dibuja
**534.948 polígonos por frame**. No hay direcciones que examinar porque nunca se pide
un dibujo. La máscara de 26 bits de `array.c:65` queda descartada por el mejor motivo
posible: el código que la usa no se ejecuta. La ruptura está más arriba, y esta ronda
instrumenta la cadena entera —siete puntos— para localizar el eslabón exacto.**

| Pregunta | Respuesta |
|---|---|
| ¿Direcciones sobre 0x04000000? | **No hay direcciones**: cero llamadas a `glDrawElements` desde studio |
| `sv ents` / `cl ents` | **208-330** en el servidor, **24-32** llegando al cliente |
| `studio drawn (polys)` | **0 (0 polys)** — y el mundo, 534.948 polígonos |
| ¿Se colgó? | **No**: el log acaba limpio, con el motor vivo |
| Zonas muertas | `pitch 4096 yaw 4096 fwd 4096 side 4096, curve x100 50` |

### 48.1 Lo que los números cierran

- **Los NPC existen y llegan al cliente.** El servidor tiene entre 208 y 330 entidades
  y el cliente recibe de 24 a 32 estados por frame. La red no es el problema, como ya
  decían la física y las voces.
- **El renderer está sano.** Medio millón de polígonos de mundo por frame a 50-60 fps.
- **Y nadie le pide un solo modelo estudiado.** `c_studio_models_drawn` = 0,
  `R_StudioDrawArrays` = 0 llamadas.

Tus dos observaciones lo confirman desde fuera y valen más que cualquier contador: las
sillas que se mueven solas —física del servidor sin dibujo— y sobre todo **la puerta
del principio del mapa**, que se abre mientras sus barrotes cilíndricos se quedan
quietos. La puerta es geometría BSP; los barrotes, un modelo estudiado. Es la misma
frontera, en la misma entidad, sin IA ni red de por medio, y a treinta segundos del
arranque. Es el caso de prueba con el que hay que trabajar.

### 48.2 Por qué la máscara de §47 queda descartada

`array.c:65` enmascara los punteros de vertex array a 26 bits, lo que rompería
cualquier dato por encima de 64 MB. Sigue siendo una bomba de relojería —y hay que
arreglarla algún día— pero **no es esto**: `glDrawElements` no se llama desde el
camino de studio ni una vez en todo el log. Lo mismo vale para el bug del `stride 0`
arreglado en §47: real, pero no la causa.

Merece la pena decirlo claro porque es la tercera vez en esta investigación que una
hipótesis bien razonada resulta irrelevante por estar aguas abajo del fallo (§38 la
textura, §42 el pintor, ahora la máscara). La lección se repite: **medir dónde muere
antes de explicar por qué**.

### 48.3 La cadena, y dónde puede romperse

Del dispatch al triángulo hay siete escalones, y uno de ellos **cruza a otro módulo**:

```
R_DrawEntitiesOnList        (ref_gl)      case mod_studio:
  R_DrawStudioModel         (ref_gl)      gl_studio.c:3607
    R_StudioDrawModelInternal (ref_gl)    gl_studio.c:3572
      pStudioDraw->StudioDrawModel        <-- CLIENT DLL, CStudioModelRenderer
        ...                               (hlsdk C++, enlazado estatico)
          R_StudioDrawPoints  (ref_gl)    gl_studio.c:2246
            R_StudioDrawArrays (ref_gl)   gl_studio.c:2196   <-- CERO
```

Con el mundo dibujado (`RI.drawWorld`), los modelos **no** los pinta el motor: los
pinta el `CStudioModelRenderer` del **client.dll**, a través de `pStudioDraw`. Y ese
client.dll es el C++ de hlsdk que §24 metió dentro del XBE **renombrando 838 símbolos**
para resolver choques con el servidor. §29 dejó anotado que de esos 838 solo se había
ejercitado lo que toca el arranque; este es el primer código que los usa en serio.

Sospechosos, por orden:

1. **Las entidades no llegan a la lista de dibujo**: `R_AddEntity` las rechaza. Sus
   dos salidas tempranas son `!clent->model` y `r_drawentities` a cero. Si fuera la
   primera, los modelos de brush tampoco se dibujarían — y la puerta se ve, así que
   apunta a algo específico de los estudiados.
2. **`pStudioDraw` mal enganchado**: el client.dll devolvió su interfaz pero alguna de
   sus entradas apunta donde no debe, cortesía del renombrado masivo.
3. **El `CStudioModelRenderer` corre y se rinde por dentro**: sin modelo, sin secuencia,
   sin huesos, o con una comprobación de visibilidad que siempre falla.

### 48.4 La instrumentación: siete contadores, una línea

En vez de trazar por modelo —que a 60 fps y decenas de entidades ahogaría el disco—,
siete contadores acumulativos y **una línea por segundo** junto al `renderpulse`:

```
studiochain: ents N -> draw N -> internal N (iface N, back N) -> engine N -> points N
```

Cada número es un escalón de 48.3. La lectura es inmediata:

| Dónde se queda en cero | Qué significa |
|---|---|
| `ents` | las entidades studio nunca entran en la lista de dibujo → `R_AddEntity` |
| `draw` | están en la lista y no se despachan → imposible salvo corrupción |
| `internal` | `R_DrawStudioModel` sale antes de tiempo (`RP_ENVVIEW`) |
| `iface` | no se toma el camino del client.dll (`RI.drawWorld` falso) |
| `back` **menor que** `iface` | **el client.dll entra y no vuelve** — se pierde dentro |
| `points` | el client.dll vuelve pero nunca pide geometría → su renderer se rinde |

Y el `renderpulse` pasa a dar además `solid` y `trans`, el tamaño real de las listas de
dibujo, en lugar del `c_client_ents` que usé en §45 y que resultó ser mal proxy: solo
cuenta entidades `ET_FRAGMENTED`, así que su cero no significaba nada.

### 48.5 Qué NO está verificado

- **Nada de esta ronda ha corrido.** XBE compilado, 0 SSE2 y **verificado por md5 en la
  consola** (`78c0eb67…`).
- **El cuelgue de §47 no se repitió** en esta sesión, pero tampoco se ha arreglado nada
  que lo explicara: sigue abierto.
- Las zonas muertas del mando salen a `4096`, o sea que `joystick.cfg` **no se aplicó**
  —los valores son los de fábrica, no los 6000/5000 del fichero—. Hay que mirar si
  `userconfig.d` se ejecuta en este target; queda para la próxima, con la instrucción
  de buscar la línea `userconfig.d/joystick.cfg aplicado` en el log.

### 48.6 Estado

`default.xbe` verificado en la consola con los siete contadores. `xash.cmd` sin
cambios. El caso de prueba es ahora la puerta del principio del mapa: treinta segundos
de arranque y la respuesta en la primera línea `studiochain` que salga.

## 49. Paso 3h — El client.dll entra a dibujar 11.729 veces y no dibuja nada

**La cadena de §48 da el diagnóstico con una precisión que no admite discusión:**

```
studiochain: ents 11729 -> draw 11729 -> internal 11729 (iface 11729, back 11729)
             -> engine 0 -> points 0
```

**Las entidades se despachan, `R_DrawStudioModel` se llama, se toma el camino del
client.dll, y se vuelve de él — once mil setecientas veintinueve veces, sin perder
una sola. Y `R_StudioDrawPoints` no se alcanza nunca. El `CStudioModelRenderer` del
client.dll entra, hace lo que hace, y sale sin pedirle al motor un solo triángulo.
El fallo está dentro de ese renderer.** Y el cuelgue de la escena guionizada resulta
ser reproducible al milímetro: dos de dos, en las mismas dos líneas de log.

| | |
|---|---|
| Estudiados en la lista | **11.729** despachados, ninguno dibujado |
| Dónde se pierde | dentro de `pStudioDraw->StudioDrawModel` (client.dll) |
| ¿Se cuelga ahí? | **No**: `back` = `iface`, vuelve siempre |
| Mundo | 970.849 polígonos/frame, 58-60 fps |
| Cuelgue | **mismo punto exacto**, dos ejecuciones: `scripted_sentence (argue_1)` |
| `joystick.cfg` | **sí se aplica** — el problema era mi traza, no el fichero |

### 49.1 Lo que la cadena descarta y lo que deja

Descartado, con números: que las entidades no lleguen a la lista de dibujo (llegan),
que el dispatch falle (no falla), que el motor salga antes de tiempo (no sale), que el
client.dll se cuelgue dentro (vuelve siempre, `back` == `iface`), y que el problema
sea la máscara de 26 bits o el `stride 0` de §47 (nunca se llega a esas llamadas).

Lo que queda es una sola cosa: **el `CStudioModelRenderer` del client.dll se ejecuta y
decide no dibujar**. Es código C++ de hlsdk que §24 metió dentro del XBE renombrando
**838 símbolos** para resolver choques con el servidor, y §29 dejó anotado que de esos
838 solo se había ejercitado la superficie que toca el arranque. Este es el primer
código que los usa a fondo.

### 49.2 La instrumentación: preguntarle al que no habla

No se puede trazar dentro del client.dll sin recompilar hlsdk, pero **no hace falta**:
ese renderer no puede hacer nada sin llamar de vuelta al motor. El motor le entrega
`gStudioAPI`, una tabla de 46 funciones (`gl_studio.c:3970`), y el orden en que las
llama es conocido. Contando cuáles se invocan se ve exactamente dónde se rinde:

```
studiocb: getent N, bbox N (ok N), setupmodel N, lighting N, renderer N
```

| Si se para en | Significa |
|---|---|
| `getent` = 0 | ni siquiera arranca: su interfaz está mal enganchada |
| `bbox` alto, `ok` = **0** | **descarta todos los modelos por recorte de volumen** |
| `setupmodel` = 0 | se rinde antes de elegir el cuerpo del modelo |
| `lighting` = 0 | falla al calcular la iluminación |
| `renderer` = 0 | llega hasta el final y no entra a pintar |

`bbox ok = 0` es el sospechoso de cabecera: `R_StudioCheckBBox` es la primera
comprobación del renderer y devuelve el resultado de `R_StudioComputeBBox`. Si esa
prueba dice siempre «fuera de pantalla», todos los modelos se descartan en silencio
mientras el resto del motor funciona — que es exactamente el cuadro clínico. Por eso
se cuentan por separado las llamadas y los aciertos.

### 49.3 El cuelgue: reproducible al milímetro

Los dos logs terminan en **las mismas dos líneas**:

```
[con 100813] Firing: (argue_1)
[con 100814] Found: scripted_sentence, firing (argue_1)
```

Uno a los 150 s de partida, el otro a los 100 s, sin degradación previa —58,7 y 59,7
fps en los últimos latidos— y sin vigilante, sin excepción y sin línea de salida. Dos
de dos en el mismo evento: la discusión guionizada de los dos científicos, que un
`multi_manager` (`argument_loop` → `argument_mm`) dispara.

Y hay un detalle que lo hace más interesante: **otras frases sí suenan** —
`!GM_1MUMBLE`, `!BA_button`— así que el camino de sentences funciona en general y algo
de *ésta* no.

Instrumentado en `VOX_LoadSound` (`s_vox.c:444`), que es donde el motor convierte el
nombre de una frase en una lista de palabras y sus wav:

```
vox: '<frase>' -> dir '<carpeta>', text '<texto crudo>'
vox: parsed N words
vox:   word i '<ruta del wav>'      <- una por palabra
vox: N words kept, loading first
vox: done
```

Si el log se corta en mitad de las palabras, el culpable es un `.wav` concreto y
tendrá su nombre escrito. Si se corta antes, es el análisis de la frase. Si llega a
`done`, el cuelgue no está aquí y hay que mirar el `scripted_sequence` del gamedll.

### 49.4 El mando: era mi traza, no el fichero

`joystick.cfg` **sí se ejecuta**:

```
[con 8611] execing userconfig.d/joystick.cfg
[con 8625] userconfig.d/joystick.cfg aplicado
```

La línea que decía `deadzones pitch 4096 …` sale en el instante **2950 ms**, y el
fichero se ejecuta en el **8611**: mi traza está en `Joy_Init`, que corre dentro de
`Host_InitCommon`, mucho antes de que se ejecuten las configuraciones al final de
`Host_Main`. Estaba imprimiendo los valores de fábrica y llamándolos efectivos. El
mecanismo de §46 funciona; la instrumentación era la equivocada.

De paso queda visto que el motor busca `joystick.cfg` también en la raíz del gamedir
(`execing joystick.cfg` / `couldn't exec`), lo cual es inofensivo.

### 49.5 Qué NO está verificado

- **Nada de esta ronda ha corrido.** XBE compilado, 0 SSE2, verificado por md5
  (`9b3901a2…`).
- **La hipótesis del recorte de volumen** (`bbox ok = 0`) es la favorita, no un hecho.
- **Que el cuelgue esté en el camino de sentences** es una apuesta: lo único probado es
  que ocurre mientras se dispara `argue_1`.
- Los valores efectivos del mando siguen sin verse en el log; la traza sigue donde
  estaba, ahora sabiendo lo que mide.

### 49.6 Estado

`default.xbe` verificado en la consola con doce contadores de studio y cinco trazas de
sentences. `xash.cmd` sin cambios.

Dos preguntas, un solo arranque: la línea `studiocb` dice por qué no se ven los NPC, y
lo que salga después de `Firing: (argue_1)` dice qué cuelga la máquina. Con la puerta
de los barrotes a treinta segundos y la discusión al minuto y medio, ambas caben en la
misma partida.

## 50. Paso 3i — `bbox 2484 (ok 0)`: todos los modelos se descartan por recorte

**El contador señala el eslabón exacto y no deja lugar a interpretación:**

```
studiocb: getent 2484, bbox 2484 (ok 0), setupmodel 0, lighting 0, renderer 0
```

**`R_StudioCheckBBox` se llama 2.484 veces y devuelve «no visible» las 2.484. Todo lo
que viene después —elegir el cuerpo del modelo, iluminarlo, pintarlo— se queda a cero
por consecuencia. El `CStudioModelRenderer` del client.dll funciona: pregunta si el
modelo está en pantalla, el motor le dice que no, y hace lo correcto, que es saltárselo.
El fallo está en la respuesta, no en quien pregunta.** Y el cuelgue de la escena
guionizada resulta **no** ser `argue_1`: esta vez se disparó dos veces sin matar nada.

| | |
|---|---|
| `bbox` | **2.484 llamadas, 0 aciertos** |
| `setupmodel` / `lighting` / `renderer` | **0** — nunca se llega |
| Mundo | 198.772 polígonos/frame, 55-56 fps |
| Salida | limpia, `START+BACK` a los 41 s |
| `argue_1` | **disparó dos veces sin colgar** — la hipótesis de §49 cae |
| Sentences | funcionan: `gman/gman_mumble1` y `scientist/cascade`, cargadas enteras |

### 50.1 Dónde puede fallar la respuesta

`R_StudioCheckBBox` (`gl_studio.c:1319`) delega en `R_StudioComputeBBox`, que tiene
exactamente dos formas de decir «no»:

```c
if( !m_pStudioHeader )
    return false;                                   // (1) sin cabecera del modelo
...
if( !bbox && R_CullModel( e, studio_mins, studio_maxs ))
    return false;                                   // (2) fuera del frustum
```

**(1) Sin cabecera.** `m_pStudioHeader` lo pone el **client.dll** llamando a
`StudioSetHeader` con lo que le devolvió `Mod_Extradata`. Si esa llamada devuelve
vacío, el puntero se queda a NULL y todas las comprobaciones fallan desde la primera
línea. Sería un fallo de plomería entre módulos: justo el tipo de cosa que los **838
símbolos renombrados** de §24 pueden haber roto, y que §29 dejó anotado como sin
ejercitar.

**(2) Recortado.** La caja se calcula transformando las esquinas del modelo por
`g_studio.rotationmatrix`, y en este target **esa matriz la escribe el client.dll** a
través de un puntero que le da el motor. Si llega a ceros, las ocho esquinas colapsan
en el origen del mundo, que está fuera del frustum desde cualquier sitio: modelos
invisibles y todo lo demás funcionando. Encaja igual de bien.

Las dos historias producen el mismo síntoma y se distinguen con un contador.

### 50.2 La instrumentación que las separa

```
studiobbox: no header N, culled N | setheader N (null N)
```

- **`no header` alto** → el client.dll nunca entregó una cabecera válida; el problema
  es `Mod_Extradata` o el enganche de la interfaz. `setheader (null N)` lo confirma
  desde el otro lado.
- **`culled` alto** → la cabecera está bien y lo que falla es la geometría de la
  prueba.

Y para el segundo caso, las dos primeras veces que ocurra se vuelca todo lo que
interviene:

```
cull: ent N model <nombre>
cull:   box mins X Y Z maxs X Y Z (x1000)
cull:   matrix row0 ... row1 ... row2 ... (x1000)
cull:   viewer X Y Z (x1000)
```

Con eso se ve de un vistazo si la matriz es una identidad razonable o un bloque de
ceros, y si la caja resultante cae encima del jugador o en el origen del mundo. **Todo
en enteros escalados por mil**, porque el `printf` de este target se traga los floats
(§31.4) — la misma lección que costó una sección entera.

### 50.3 El cuelgue: `argue_1` queda absuelto

§49 dio por reproducible que el cuelgue ocurría al disparar `scripted_sentence
(argue_1)`, porque dos logs seguidos terminaban en esas dos líneas. **Esta ejecución lo
desmiente**: `argue_1` se disparó a los 22,8 s y otra vez a los 30,8 s, y el motor
siguió corriendo hasta que se salió a mano por START+BACK a los 41 s.

Lo que sí se ve, y funciona, es el camino de sentences completo:

```
vox: '#778' -> dir 'scientist/', text 'cascade'
vox: parsed 1 words
vox:   word 0 'scientist/cascade'
vox: 1 words kept, loading first
vox: done
```

La lectura correcta de los dos logs anteriores es otra: la escena de la discusión corre
**en bucle** (un `multi_manager` la relanza), así que sus líneas son ruido de fondo
constante. Que dos cuelgues terminaran ahí dice cuándo, no por qué. Y la pista buena es
la que diste tú: *«llegué a una zona»* — **el cuelgue va con el sitio, no con el
guion**. Probablemente con lo que hay que cargar o dibujar al llegar allí.

Queda pendiente, y esta vez con la hipótesis correcta: reproducirlo caminando al mismo
punto y mirar qué se está cargando en ese instante. Las trazas de `vox` se quedan
puestas —son baratas y ya han demostrado que ese camino está sano—.

### 50.4 Qué NO está verificado

- **Nada de esta ronda ha corrido.** XBE compilado, 0 SSE2, verificado por md5
  (`85a31d82…`).
- **Cuál de las dos causas de 50.1 es la buena.** Ese es todo el objetivo del próximo
  arranque.
- **El cuelgue por zona** no se ha reproducido con instrumentación adecuada.

### 50.5 Estado

El port está a **un contador** de saber por qué no se ve un solo modelo estudiado. Es
de los momentos en que conviene mirar atrás: la investigación ha ido de «los NPC no
aparecen» a «el renderer no los dibuja», de ahí a «no se le pide que los dibuje», luego
a «el client.dll no llega a pedirlo», y ahora a «el client.dll pregunta si son visibles
y el motor contesta que no, 2.484 veces seguidas». Cada paso ha descartado media
investigación.

Con arrancar y mirar la puerta de los barrotes treinta segundos, la línea
`studiobbox` dice cuál de las dos ramas es — y con ella, dónde está el arreglo.

## 51. Paso 3j — CAUSA RAÍZ: el fork encogió un struct de ABI compartida en un solo lado

**El log dio la pista (`setheader 9500 (null 9500)`: el client.dll entrega una
cabecera nula todas las veces, y el recorte ni se ejecuta — `no header 0, culled 0`
porque `CheckBBox` sale antes, por un puntero de modelo que no es un modelo). Pero el
diagnóstico no salió de la consola: salió de **comparar los layouts de los structs
compartidos con el volcado de registros de clang**, sin gastar un solo arranque. El
fork añadió a `common/cl_entity.h` — la cabecera de interfaz que el gamedll también
compila — un `#if XASH_XBOX` que encoge `HISTORY_MAX` de 64 a 16 y borra `syncbase`.
El motor cree que `cl_entity_t` mide 1652 bytes; el client.dll, compilado con su copia
estándar, cree que mide **3000**. Todo campo después de `ph[]` — `mouth`, `latched`,
`origin`, `angles`, **`model`** — está a 1348 bytes de donde el otro lado lo busca.
El cliente leía `->model` de en medio de la nada, `Mod_Extradata` recibía basura,
devolvía NULL, y todos los modelos estudiados del juego eran invisibles mientras la
física, la IA y el sonido — que viven en el lado del servidor, con otros structs —
funcionaban perfectamente. Noveno bug de la familia, y el más caro de encontrar.**

| struct | engine (antes) | hlsdk | ahora |
|---|---|---|---|
| `cl_entity_s` | **1652** | 3000 | **3000 == 3000** |
| `entity_state_s` | 340 | 340 | == |
| `clientdata_s` / `usercmd_s` / `weapon_data_s` / `local_state_s` | iguales | iguales | == |
| `ref_params_s` | 232 | **240** | 232+cola acolchada |

### 51.1 La cadena causal completa, de la cabecera al síntoma

1. `common/cl_entity.h:62` (fork): `#if XASH_XBOX → HISTORY_MAX 16`. Ahorra 1344
   bytes por entidad… **solo en el motor**. Y `:105`: borra `syncbase` — 4 más.
2. El client.dll de hlsdk compila contra su propia copia con el 64 estándar.
3. `CStudioModelRenderer::StudioDrawModel` hace `m_pCurrentEntity->model` — lee el
   offset **2964** de una estructura que el motor termina en el 1620.
4. Ese "modelo" es basura; `Mod_Extradata(basura)` → NULL; `StudioSetHeader(NULL)`;
   y `SetRenderModel(basura)` deja `RI.currentmodel` inválido, por lo que
   `CheckBBox` falla en su primera línea — de ahí que los contadores de §50
   (`no header`, `culled`) se quedaran a cero: **la respuesta era «ninguna de las
   dos ramas: la pregunta llega rota»**.
5. Física, IA, voces: intactas — viven en el servidor, cuyos structs (`entvars_t`,
   `edict_t`) no fueron tocados.

Explica hasta el detalle más fino: por qué los barrotes de la puerta (studio, lado
cliente) no se movían mientras la puerta (BSP, dibujada por el motor) sí.

### 51.2 El método que lo encontró: layouts medidos, no leídos

`research/ofx-abi-structs.sh` (nuevo, permanente): compila un TU mínimo en el
contexto del engine y otro en el de hlsdk con `-Xclang -fdump-record-layouts` — el
volcado de layouts del propio clang, el mismo compilador que genera el código — y
compara struct a struct. Es §15 (offsets de save/restore) aplicado a la frontera
motor↔cliente, y responde en dos segundos lo que cinco rondas de instrumentación en
hardware fueron acorralando.

**Regla nueva de la casa**: correr el prober tras cualquier cambio en `common/*.h` o
al actualizar hlsdk. Un byte de diferencia = campos desplazados = síntomas
imposibles de correlacionar.

De la auditoría completa de guards `XASH_XBOX` en `common/` salieron además:
`defaults.h` (config, no ABI — ok), `port.h` (mismo layout — ok), `sound_api.h`
(un `volatile`, no cambia layout — ok), y `xash3d_types.h` (`MAX_QPATH 48` — **solo
afecta a structs internos del motor**; `player_info_t` y `model_t` usan 64 literal a
ambos lados, verificado).

### 51.3 Los arreglos

1. **`cl_entity.h`**: fuera los dos guards. `HISTORY_MAX 64` y `syncbase` de vuelta,
   con el porqué escrito encima. Coste: ~1,6 MB más de entidades de cliente — con
   96 MB libres (§46), irrelevante. La lección, en una frase que queda en la
   cabecera: *un struct de una cabecera compartida no puede cambiar de tamaño en un
   solo lado de la frontera*.
2. **`ref_params_t`, el hallazgo colateral**: hlsdk-portable le añade al final
   `fov_x, fov_y` («Xash3D extension», 8 bytes) que el struct del motor — upstream
   incluido — no tiene. El `V_CalcRefdef` del cliente **escribe 8 bytes más allá del
   buffer del motor en cada frame**; upstream sobrevive porque caen en el vecino de
   `.bss` que toque. Arreglado en `cl_view.c`: el buffer es ahora un struct con
   16 bytes de cola en propiedad. No era la causa de los NPC, pero era una escritura
   fuera de límites real, en cada frame, esperando su momento.

### 51.4 Qué NO está verificado

- **El arreglo no ha corrido en la consola.** Verificado estáticamente (el prober da
  `3000 == 3000` y «ABI limpia»), compilado, 0 SSE2, desplegado por md5
  (`ae06b31a…`). La prueba de fuego es la puerta de los barrotes.
- **El cuelgue por zona sigue abierto** (§50.3): esta ronda no lo toca. Con la ABI
  arreglada cabe la posibilidad de que fuera otro efecto de los offsets — el cliente
  también *escribe* en `cl_entity_t` (latched, mouth) a 1348 bytes de su sitio, es
  decir, **corrompía memoria del motor en cada frame con NPC visibles cerca**. Si el
  cuelgue desaparece con este arreglo, era eso; si no, se instrumenta con dos
  problemas claramente separados.
- Los contadores de §48-50 se quedan una ronda más para ver la cadena entera en
  verde; luego se retiran.

### 51.5 Estado

`default.xbe` desplegado y verificado. `xash.cmd` sin cambios. El prober de ABI en
`research/`, con veredicto automático.

Nueve ejecuciones de instrumentación acorralaron el fallo desde «no se ven los NPC»
hasta «la pregunta llega rota», y el volcado de layouts hizo el resto sin encender la
consola. Si la próxima partida enseña un científico en Anomalous Materials, el port
pasa de dibujar un mapa a dibujar el juego.

## 52. Paso 3k — HITO: los NPC se ven. Y los dos bugs que quedan comparten sistema nervioso

**El arreglo de ABI de §51 era el correcto: los científicos y el guardia están en
Anomalous Materials, con física, voz y modelo. El port dibuja el juego. Quedan dos
defectos visibles — todos los NPC miran «al revés», y los cuatro cilindros de la
esclusa no giran — y la investigación en frío de esta ronda los deja conectados:
**ambos son de ángulos de entidad**, y el segundo ni siquiera es de modelos
estudiados: los cilindros son brushes. La ronda descarta estáticamente el formato
studio como causa (244/176/112/80 bytes, idénticos en ambos lados), localiza las
cuatro entidades exactas de los cilindros en el lump del BSP, y despliega el volcado
que separa las tres hipótesis del giro con una sola línea de log.**

| | |
|---|---|
| NPC | **visibles** — la cadena de §48-51, cerrada |
| Bug 1 | todos los NPC girados (¿180? ¿signo? ¿colapso a 0?) |
| Bug 2 | `doorwheels`: 4 × `func_door_rotating` — **brush, no studio** |
| Formato studio | idéntico entre engine y hlsdk: descartado en frío |
| Hacks del fork en el camino | ninguno (solo `r_studio_drawelements 0`, que es lo que hace funcionar el modo inmediato) |

### 52.1 Lo que se descartó sin encender la consola

- **El formato studio**: `studiohdr_t` 244, `mstudioseqdesc_t` 176, `mstudiobone_t`
  112, `mstudiotexture_t` 80 — byte a byte iguales en los dos árboles. Las cabeceras
  difieren en constantes (`MAXSTUDIOVERTS`, flags) que no tocan disco.
- **Hacks del fork en `gl_studio.c`**: los 17 guards `XASH_XBOX` son las trazas de
  §48-50 más uno legítimo: `r_studio_drawelements 0` — que explica, de paso, por qué
  studio dibuja en modo inmediato y no por el camino de arrays que §47 investigó.
- **La premisa de §48 sobre los cilindros**: falsa, y lo dice el propio mapa. El lump
  de entidades de `c1a0.bsp` (extraído del pak con `ofx-bsp-ents.sh`, nuevo) tiene
  **un solo** `.mdl` colocado — el archivador del headcrab. Los cilindros son
  `doorwheels`: cuatro `func_door_rotating` sobre los brushes `*36`-`*39`. **Brush,
  no studio.** La puerta que sí se abre es un `func_door` que traslada; lo que no se
  ve es la **rotación** de entidades brush.

Eso deja un cuadro sugerente: modelos studio con la orientación mal, brushes
rotatorios sin rotación visible. Dos síntomas, un sospechoso común: **el camino de
ángulos de entidad**, del servidor al renderer.

### 52.2 El transform del cliente, leído — y las tres hipótesis

`StudioSetUpTransform` (hlsdk, `StudioModelRenderer.cpp:451`) calcula para monstruos
(`MOVETYPE_STEP`):

```
yaw = curstate.angles[YAW] + (e->angles[YAW] - latched.prevangles[YAW])~ * f,  f ∈ [-1, 0]
```

Tres fuentes, todas rellenadas por el motor. Según cuál llegue mal:

| Si el log dice | Diagnóstico |
|---|---|
| `MATRIX yaw == curstate + 180` | el cliente lo gira: convención — buscar el flip en su código |
| `MATRIX yaw == -curstate` | signo invertido en la red o en la interpolación del motor |
| `MATRIX yaw == 0` siempre | colapso: `e->angles`/`prevangles` mal y el término `d·f` anula el yaw |
| `MATRIX yaw == curstate` y aun así de espaldas | la matriz es correcta: el fallo está en los huesos |

El volcado (`ang:`) va en el punto perfecto — el éxito de `CheckBBox`, cuando el
cliente ya escribió su matriz en `g_studio.rotationmatrix` —: dos entidades por
segundo, con `curstate yaw / ent yaw / prev yaw / MATRIX yaw` (el de la matriz,
recuperado con `atan2(m[1][0], m[0][0])`), más `animtime`/`prevanimtime`/`cltime` y
el movetype, todo en enteros (§31.4).

### 52.3 Y el gemelo para los cilindros

En `R_DrawBrushModel`: un contador de dibujos de brush **con ángulos no nulos**
(`brushrot: N rotated brush draws since boot`, en el pulso) y el volcado de los dos
primeros por segundo (`brushang: ent N *36 angles P Y R`). La lectura es binaria:
si al accionar la esclusa el contador no se mueve, los ángulos de los `doorwheels`
no llegan al renderer — mismo sistema nervioso que los NPC girados. Si el contador
sube y los ángulos cambian, la rotación llega y el fallo está en cómo se aplica.

### 52.4 El cuelgue por zona (tu punto 3)

Sin noticias en esta sesión — que es exactamente lo que la hipótesis de §51.4
predice: el cliente llevaba desde el primer frame **escribiendo** `latched` y `mouth`
1348 bytes más allá de su sitio, es decir, corrompiendo memoria del motor cada frame
con NPC en pantalla. Si tras este arreglo el cuelgue no reaparece en sesiones largas,
era eso y queda cerrado sin más trabajo. Sin confirmar hasta que se acumulen paseos.

### 52.5 Qué NO está verificado

- **Nada de la ronda ha corrido**: volcados compilados, 0 SSE2, desplegado y
  verificado por md5 (`94db3893…`).
- **Las tres hipótesis del giro** siguen las tres vivas hasta la primera línea `ang:`.
- **La conexión NPC↔cilindros** es una sospecha con buena pinta, no un hecho.

### 52.6 Estado

`default.xbe` verificado en la consola. `xash.cmd` sin cambios. Dos scripts nuevos:
`ofx-bsp-ents.sh` (entidades de un BSP dentro del pak) y el prober de studio dentro
de la familia ABI.

La partida que toca: mirar al guardia (línea `ang:`), accionar la esclusa (contador
`brushrot:`), y pasear un rato largo (el cuelgue). Tres respuestas, una sesión.

## 53. Paso 3l — No era 180: era el neón de §31.4 borrando todos los ángulos del mapa

**La cronología corregida era la pista buena, pero la conclusión da otra vuelta: los
síntomas no los creó §51 — §51 los hizo *visibles*. El volcado `ang:` no muestra ni
180 ni signo invertido: muestra **ceros en las tres fuentes y en la matriz** — los
ángulos de los monstruos nunca salen del servidor. Y la causa es la advertencia que
§31.4 dejó escrita con neón, cobrándose su tercera víctima y la primera funcional:
`sv_game.c:5032` convierte la clave legacy `"angle"` de los mapas con `"%g %g %g"`, y
el printf de este target tira los floats — cada `"angle" "180"` del mapa se convertía
en una cadena vacía y cada NPC y cada scripted_sequence del juego nacía mirando a yaw
0. De la misma pasada cae `Cvar_SetValue`, cuya rama fraccionaria formatea con `%f`:
cualquier cvar puesto por código a un valor no entero quedaba en 0 en silencio.**

| Pregunta | Respuesta |
|---|---|
| `ang:` (tu punto 4) | `curstate 0, ent 0, prev 0, MATRIX 0` — **todo ceros**, en todos los NPC |
| `brushrot:` | funciona: la silla de la intro rota 0→-90, y **los doorwheels llegan a 147°** |
| ¿Más ABI rota? (punto 1) | formato studio idéntico; prober «ABI limpia»; el commit del fork solo tocó límites internos además de los dos venenos ya arreglados |
| ¿Por qué encogió el fork el struct? (punto 2) | commit `28bdf8d` «define xbox protocol limits»: dieta de memoria |
| ¿Buffer de tamaño 3-4? (punto 3) | no apareció ninguno; hipótesis mejor abajo |

### 53.1 La cadena causal, esta vez entera

1. Los mapas de Half-Life usan la clave **legacy `"angle"`** (un solo valor) en
   monstruos y scripted_sequences — c1a0 entero es así, verificado en su lump de
   entidades: `"angle" "180"`, `"177"`, `"273"`…
2. El motor la convierte a `"angles"` para el gamedll: `Q_snprintf( "%g %g %g", … )`.
3. En este target, `%g` **no imprime nada** (§31.4). La cadena resultante: vacía.
4. El gamedll parsea esa cadena → `pev->angles = 0 0 0`.
5. Todos los NPC miran al este del mundo; todos los scripts apuntan a yaw 0. El
   guardia «mira a la puerta» y los científicos «al revés» — no era un flip de 180:
   era que sus orientaciones reales quedaron borradas y las poses que se veían eran
   las que el azar de la geometría dictara.

**Y estuvo así desde el primer arranque.** Antes de §51 nadie podía verlo porque los
modelos eran invisibles. La cronología del descubrimiento no era la cronología del
bug — la corrección que enviaste era el dato que faltaba, solo que su lectura
correcta era «§51 destapó», no «§51 rompió».

### 53.2 La segunda víctima: `Cvar_SetValue`

La auditoría de formatos float que alimentan lógica (no logs) dio cuatro hits, y uno
es estructural: las **dos** funciones de poner cvars por valor —`Cvar_SetValue` y
`Cvar_DirectSetValue`— usan `%d` para valores enteros (por eso casi todo funciona) y
**`%f` para fraccionarios**: todo cvar puesto por código a `0.5`, `2.5`, etc. quedaba
en `""` → 0. Arreglado con un formateador de microunidades en enteros puros
(`Cvar_FloatToString`: exacto a 6 decimales, ningún float llega jamás a printf). Los
otros dos hits (`addip %g` de un ban con `-noip`, y un tiempo en el userinfo) quedan
anotados como inertes.

El gamedll está limpio: cero formatos float alimentando comandos, y sus ocho
`CVAR_SET_FLOAT` pasan por el motor, ya arreglado.

### 53.3 Tus puntos 1 y 2, con datos

- **Punto 2**: el fork encogió `HISTORY_MAX` y borró `syncbase` en el commit
  `28bdf8d` — «define xbox protocol limits» — junto a una dieta de límites internos
  (`MAX_MODELS 512`, `CMD_BACKUP 16`, `MAX_VISIBLE_PACKET 128`…). El motivo era
  memoria en la era de los 64 MB. Los límites internos son legales (viven solo en el
  motor); los dos cambios de `cl_entity.h` eran los únicos venenosos, y ya están
  revertidos. Restaurarlos no rompe nada más: no había otra dependencia.
- **Punto 1**: el prober extendido al formato studio (244/176/112/80, idénticos) y
  los siete structs de frontera dan «ABI limpia». No apareció ningún campo más
  desalineado.
- **Punto 3**: ningún buffer de 3-4 en los structs de eventos. La hipótesis mejor
  para puertas y microondas es ahora otra: **las escenas guionizadas de c1a0
  dependen de los ángulos que este bug borraba**. La esclusa la abre el guardia
  ejecutando su script; con cada scripted_sequence apuntando a yaw 0, los NPC giran
  y caminan hacia donde no es, los scripts se atascan y las cadenas de
  multi_manager que abren puertas no llegan a disparar. El microondas «3 usos» puede
  ser parte de la misma maraña de scripts de la cocina. **Es hipótesis**: el arreglo
  de los ángulos está desplegado y la próxima partida la confirma o la separa.

### 53.4 Lo que el log de tu sesión ya desmiente

- **Los doorwheels giran**: `brushang: ent 115 *36 angles 0 0 147`. La rotación de
  brushes llega y se aplica — la silla de la intro hace su 0→-90 de libro. Lo que
  fuera que viste como «puertas que no abren» no es el camino de rotación del
  renderer.
- **La cadena studio, en verde por primera vez**: `studio drawn 7684 (5.3M polys)`,
  `points 20846`. Los contadores de §48-50 han cumplido; se retirarán cuando esta
  ronda quede validada.

### 53.5 Qué NO está verificado

- **El arreglo de ángulos no ha corrido** (compilado, 0 SSE2, desplegado por md5
  `e160f158…`). La prueba: los científicos de la entrada deben mirar cada uno a su
  sitio — 180, 177, 273 — y el guardia a ti.
- **La hipótesis de los scripts** para puertas/microondas, pendiente de esa partida.
- **El cuelgue por zona**: sin reaparecer. Suma sesiones a favor de que era la
  escritura desplazada de §51.4, sin confirmar aún.

### 53.6 Estado

`default.xbe` desplegado y verificado. La norma de §31.4 sube de rango: no era solo
«no uses floats en trazas» — era **«ningún float puede llegar a un printf de este
target, y menos si el resultado se parsea»**. La auditoría de formatos-que-alimentan-
lógica (`ofx-audit-floatfmt.sh`) queda en `research/` para correr tras cada
actualización del fork.

Si la próxima partida enseña al guardia mirándote a los ojos y la esclusa abriéndose
sola, tres bugs habrán caído con dos líneas de formato.

## 54. Paso 3m — HITO: el juego funciona. Y la ronda de armas, preparada

**La partida de validación de §53 lo confirma todo de una vez: científicos y guardia
orientados cada uno a su sitio, linterna, traje HEV, cargadores de pared de salud y
batería — la cadena entera de interacción del juego, funcionando. Sin cuelgues. Dos
líneas de formato (`%g` en la conversión del `"angle"` legacy, `%f` en los cvars
fraccionarios) tenían secuestrados los ángulos de todos los NPC y scripts del juego, y
con ellas cayó la cadena entera: scripts que se atascaban, puertas que no se abrían,
orientaciones absurdas. El port ha pasado de dibujar un mapa a **jugarse**. Queda un
fleco (el microondas) y un bloque grande sin ejercitar: las armas. Esta sección
prepara esa ronda.**

| | |
|---|---|
| NPC | orientados correctamente; scripts completándose |
| Interacción | linterna, HEV, cargadores de salud/batería: **todo funciona** |
| Cuelgues | ninguno — más peso a que era la escritura desplazada de §51.4 |
| Deuda | **microondas: 3 usos y se atasca** (no urgente, sin diagnosticar) |
| Deuda | **`env_light`**: convierte el vector solar con `%f` → luz exterior rota |
| Siguiente | armas: predicción, viewmodel, eventos, HUD, muzzleflash, decals |

### 54.1 El microondas, a la lista de deuda

Tres usos y deja de responder. Con puertas y scripts ya funcionando, queda como
defecto aislado. Sin diagnosticar a propósito: la ronda de armas ejercita eventos y
sonido — si el microondas comparte causa con algo de eso, saldrá solo; si no, tendrá
su ronda.

### 54.2 Trucos sin teclado (tu punto 1)

No hace falta consola — y de hecho la consola sin teclado no serviría de nada (se
puede abrir con un bind, pero no teclear en ella). Las dos piezas van por las vías
que ya existen:

- **`sv_cheats 1` por `xash.cmd`**: los argumentos `+` se ejecutan en orden en el
  primer `Cbuf_Execute`, así que `+sv_cheats 1 +map c1a1` activa los trucos **antes**
  de que el servidor arranque. Ya está escrito en la consola.
- **`impulse 101` por bind**: `userconfig.d/weapons-test.cfg` (subido) hace
  `bind DPAD_UP "impulse 101"` — el dpad-arriba era el spray, que no se pierde nada.
  Los nombres de botón salen de `keys.c:109`: `A_BUTTON`, `DPAD_UP`, `R1_BUTTON`…
  La botonera de fábrica ya es jugable: R1 dispara, R2 alt-fire, B usa, X recarga,
  dpad izquierda/derecha cambia de arma, Y linterna (verificada por ti).

El cfg se borra al acabar la ronda; lo dice su propia cabecera.

### 54.3 El mapa (tu punto 2)

**`c1a1`** («Unforeseen Consequences», recién pasada la cascada): zombis y headcrabs
desde el primer pasillo, espacio variado, y todo dentro del pak (2,7 MB). Mejor que
c1a0 (sin combate real), y mejor que el Hazard Course (t0a0 encadena scripts de
tutorial que meten ruido). Como control sin IA queda `stalkyard` (multijugador, cero
monstruos): si algo peta en c1a1, la misma arma en stalkyard separa «bug de armas» de
«bug de monstruos». `xash.cmd` queda en `-noip -dev 2 +sv_cheats 1 +map c1a1`.

### 54.4 La instrumentación (tu punto 3): siete contadores, una línea

`struct xbox_dbg_weapons` (en el compat header, visible desde motor y ref), un
`++` por llamada en los puntos que nombraste, y una línea por segundo en el pulso:

```
wpulse: vm N, events Q/F/M (q/fired/missing), spr N, muzzle N, decal N
```

| Contador | Dónde | Qué vigila |
|---|---|---|
| `vm` | `R_DrawViewModel` (`gl_studio.c:3852`) | el viewmodel — studio en primera persona, donde llvmpipe moría en PC |
| `events q` | `CL_QueueEvent` | eventos de disparo entrando a la cola |
| `events fired` | `CL_FireEvent` → hook | despachados al gamedll de cliente |
| `events missing` | ídem, rama de error | sin hook o sin precache — **si M sube al disparar, ahí está el bug** |
| `spr` | `SPR_DrawGeneric` | HUD dibujando sprites |
| `muzzle` | `R_MuzzleFlash` | fogonazos (tempents) |
| `decal` | `CL_DecalShoot` | impactos en paredes — el precedente de ref_soft en §15 |

La lectura en juego es inmediata: se aprieta el gatillo y se mira qué contadores se
mueven. `vm` quieto = viewmodel no se dibuja; `q` sube y `fired` no = el evento se
pierde en la cola; `muzzle`/`decal` quietos con `fired` subiendo = el hook corre pero
sus efectos no llegan al renderer.

### 54.5 El audit de floats sobre las armas (tu punto 4): limpias

`ofx-audit-floatfmt` sobre `dlls/wpn_shared`, `weapons.cpp` y `cl_dll/ev_*`:

- **Los eventos llevan los floats en binario**: `fparam1`/`fparam2` viajan dentro de
  `event_args_t` (structs verificados idénticos en §51) — spread y daño llegan al
  cliente sin pasar por ningún printf. La munición, cadencias y conos son constantes
  compiladas (`VECTOR_CONE_*`). Cero `atof` en el camino.
- **Un hit real, fuera de armas**: `dlls/lights.cpp:188` — `env_light` convierte el
  vector del sol con `sprintf("%f")` para meterlo en cvars. En este target eso son
  cadenas vacías: **la dirección de la luz solar de los mapas exteriores se pierde**.
  Deuda anotada; c1a1 es interior, no molesta todavía. El arreglo será el
  `Cvar_FloatToString` de §53 o tocar esas tres líneas del gamedll.

### 54.6 Qué NO está verificado

- **Nada de la ronda ha corrido**: contadores compilados, 0 SSE2, XBE en cola de
  despliegue verificado (la consola estaba fuera de la red al cierre; el script
  espera y compara md5 al subir).
- **`impulse 101` con `sv_cheats` por cmdline**: el orden de ejecución es correcto
  sobre el papel; la primera pulsación de dpad-arriba lo dice de verdad.
- **El microondas y `env_light`**, en deuda consciente.

### 54.7 Estado

El port juega Half-Life: mapa, NPC, scripts, interacción, HUD básico y 60 fps. La
ronda de armas tiene mapa (`c1a1`), trucos accesibles desde el mando, siete
contadores esperando el primer gatillo, y el arsenal del gamedll de cliente —
predicción, eventos, viewmodel — como último gran territorio sin pisar.

## 55. Paso 3n — Las armas funcionan enteras. Los decals: 3.167 peticiones, cero agujeros

**La ronda de armas valida el último gran bloque del gamedll de cliente: viewmodels,
eventos de disparo (98 encolados, 98 despachados, **cero perdidos**), muzzleflash,
sprites del HUD por decenas de miles, sonido, daño, y hasta los virotes de ballesta
clavándose con su ángulo — el volcado `ang:` los muestra con `curstate = ent = prev =
MATRIX = -34`, el arreglo de §53 firmando en cada campo. El único fallo es de precisión
quirúrgica: los agujeros de bala no aparecen, y el contador responde tu primera
pregunta al instante: `decal 3167` — **se piden y no se pintan**. Esta ronda parte ese
subsistema en cuatro etapas contadas y descarta en frío las texturas, el polygon
offset y los floats.**

| Contador (fin de sesión) | Valor | Lectura |
|---|---|---|
| `vm` | 11.932 | viewmodel dibujándose cada frame |
| `events q/fired/missing` | **98/98/0** | el sistema de eventos, perfecto |
| `spr` | 132.477 | HUD vivo |
| `muzzle` | 82 | fogonazos funcionando |
| `decal` | **3.167** | **peticiones que jamás llegan a la pared** |

### 55.1 Lo que la sesión deja validado

El bloque que §54 llamó «el último territorio sin pisar» está pisado: la predicción
corre, `CL_QueueEvent → CL_FireEvent → hook del gamedll` no pierde un solo evento, el
viewmodel (studio en primera persona, donde llvmpipe moría en PC en §23) dibuja sin
queja, y el flujo completo disparo→evento→efecto→daño funciona con todas las armas.
`impulse 101` por dpad y `sv_cheats` por cmdline funcionaron a la primera.

### 55.2 Los decals: descartes en frío antes de instrumentar

- **Las texturas suben.** Las trazas `upload:` de §41 siguen puestas y muestran 43
  texturas de decal (`{scorch2`, `{crack1`, `{dent6`, `{shot…`) subidas con cierre
  `ok`. Y las texturas de mapa con el mismo formato de transparencia por paleta
  (`{fence2`, `{ladder1`) también — las vallas del mapa se ven, así que el índice
  255 transparente funciona en pbgl.
- **`decals.wad` montado e indexado**: 222 decals en `Host_InitDecals`, y el motor
  carga las listas por mapa (`Loading decals from c1a1`).
- **El polygon offset existe en pbgl**: `glPolygonOffset` guarda estado, el flush lo
  programa (`NV097_SET_POLYGON_OFFSET_*`) y el enable (`GL_POLYGON_OFFSET_FILL`)
  está en su `glEnable`. No es el sospechoso fácil de los polígonos coplanares.
- **El camino de floats, limpio** (tu punto 4): cero conversiones float en formatos
  de `gl_decals.c` y `cl_tent.c` — la proyección es aritmética binaria que jamás toca
  printf. El `Decals:  K` sin número que se ve en el log es un listado cosmético del
  servidor (`sv_client.c:3485`, `%.2fK`), anotado y sin consecuencias.
- **Sin errores en el log**: ni `Decal has invalid texture`, ni `Decals must hit
  mod_brush` — las salidas ruidosas de `R_DecalShoot` no se disparan.

### 55.3 El eslabón §15, y la tubería en cuatro etapas

Tu punto 3: el crash de ref_soft en PC era en su propio dibujado de decals — código
distinto, pero **las etapas compartidas entre renderers son las mismas**: la petición
del motor, la proyección BSP (`R_DecalNode`), el enganche a la superficie
(`surf->pdecals`) y el batch de dibujo. La instrumentación corta exactamente por esas
juntas:

```
decpulse: req N -> shot N -> created N -> drawcalls N, batched now N
```

| Etapa | Dónde | Si aquí se corta |
|---|---|---|
| `req` | `CL_DecalShoot` (motor) | — (ya sabemos que sube) |
| `shot` | `R_DecalShoot` pasadas las salidas tempranas | el modelo/textura no resuelve |
| `created` | `R_DecalCreate` con superficie | **la proyección BSP no encuentra pared** — `R_DecalNode`/`R_DecalSurface`, pura geometría |
| `drawcalls` | `DrawSurfaceDecals` | se crean pero el batch nunca se consume |
| `batched now` | `tr.num_draw_decals` en el pulso | cuántas superficies con decals ve el frame |

El hueco entre la primera etapa que se queda quieta y la anterior es el diagnóstico,
igual que en §48-50 con los modelos.

### 55.4 Qué NO está verificado

- **Nada de esta ronda ha corrido**: contadores compilados, 0 SSE2, desplegado y
  verificado por md5 (`d9265690…`).
- **La hipótesis favorita** —que `created` se quede a cero y el fallo sea la
  proyección BSP— es solo la más probable a priori; los descartes de arriba dejan
  poco más en pie, pero el contador manda.
- El microondas y `env_light` siguen en deuda, sin cambios.

### 55.5 Estado

`default.xbe` con la tubería de decals contada, desplegado. `xash.cmd` sigue en
`-noip -dev 2 +sv_cheats 1 +map c1a1`. La partida que toca es corta: entrar, vaciar
un cargador contra una pared, y salir — la primera línea `decpulse:` con fuego hecho
dice en qué etapa mueren los agujeros de bala.

## 56. Paso 3o — La tubería de decals está viva entera: mueren en el último centímetro

**El `decpulse` de tu cargador contra la pared responde la pregunta de la forma más
incómoda posible: no mueren en ninguna etapa contable.**

```
decpulse: req 265 -> shot 332 -> created 377 -> drawcalls 4869, batched now 0
```

**Las peticiones llegan (265), sobreviven a las salidas tempranas (332 — más que las
peticiones porque ahí caen también los decals restaurados del servidor), la proyección
BSP encuentra superficie y los crea (377), y el dibujado se invoca a miles — 120
llamadas por segundo repintándolos cada frame. El `batched now 0` es normal: el pulso
lee después de consumir el batch. Se emiten los polígonos GL y no aparece un píxel: el
fallo está en el último centímetro, entre el `glBegin` y el framebuffer.** La ronda
descarta en frío las teorías fáciles de ese tramo y planta dos sondas que separan las
dos que quedan.

| Etapa | Valor | Veredicto |
|---|---|---|
| `req` (motor) | 265 | ✔ |
| `shot` (pasadas las salidas) | 332 | ✔ |
| `created` (proyección BSP → superficie) | **377** | ✔ — la geometría funciona |
| `drawcalls` | **4.869** | ✔ — se dibujan cada frame |
| Píxeles en pantalla | 0 | **aquí** |

### 56.1 El último centímetro, leído línea a línea

`DrawSurfaceDecals` para paredes normales (sin `SURF_TRANSPARENT`, así que el bloque
de stencil ni se ejecuta) acaba en `DrawSingleDecal`: bind de la textura del decal,
`glBlendFunc( SRC_ALPHA, ONE_MINUS_SRC_ALPHA )`, y un `glBegin( GL_POLYGON )` con sus
vértices — **exactamente el mismo modo inmediato con el que el mundo entero se dibuja
y se ve**. Descartes:

- **No son vertex arrays** (la sospecha heredada de §47): es `glBegin`, como el mundo.
- **No es el stencil** ni el `glColorMask`: ese bloque solo corre sobre superficies
  transparentes, y las paredes de c1a1 no lo son.
- **No es polygon offset**: pbgl lo implementa entero (§55), y además un fallo ahí
  daría z-fighting parpadeante, no invisibilidad limpia.
- **No es la conversión de paleta de pbgl**: `tex_store_palette` preserva el alfa
  (`dst[3] = src[3]`) cuando el formato interno es RGBA, que es como el fork la sube.

### 56.2 Las dos hipótesis que quedan, y la sonda de cada una

Un polígono con blend `SRC_ALPHA` solo puede ser invisible por dos motivos:

1. **Alfa cero.** Los decals de GoldSrc son texturas indexadas cuya paleta lleva el
   **gradiente de alfa** (los `{fence`/`{ladder` binarios funcionan con alpha-test,
   pero un decal necesita la rampa entera). Si la paleta llega al motor sin la rampa
   —o el imagelib no la construye en este camino—, cada texel sale con alfa 0 y el
   blend pinta exactamente nada, con toda la tubería sana. Sonda: `decalpal:` vuelca
   la rampa (`a[0], a[64], a[128], a[192], a[255]`) de cada paleta de decal en el
   momento de la subida. Una rampa creciente = paleta buena; todo ceros o todo 255 =
   culpable encontrado.
2. **Geometría degenerada.** `R_DecalSetupVerts` produce los vértices; si salen
   colapsados o con UVs fuera de rango, el polígono se emite y no cubre ni un texel.
   Sonda: `decdraw#:` vuelca los primeros cuatro dibujados enteros — textura, flags
   TF, número de vértices, primer vértice y su UV — y uno de cada 600 después.

Las dos sondas caben en la misma partida corta de siempre: entrar, un cargador a la
pared, salir.

### 56.3 Notas del log

- El sistema de armas se mantiene impecable en la segunda sesión: `events 64/64/0`,
  `muzzle 92`, viewmodel vivo.
- `studiobbox` ahora muestra el mundo sano: `culled 6992 | setheader 11258 (null 0)`
  — **null 0**: la cabecera del modelo llega siempre desde §51, y el recorte hace su
  trabajo de verdad (descarta lo que no está en pantalla, no todo).

### 56.4 Qué NO está verificado

- **Las dos sondas no han corrido**: compiladas, 0 SSE2, desplegadas por md5
  (`f3689ec1…`).
- **La suposición de que las paredes de c1a1 no son `SURF_TRANSPARENT`** es
  razonable pero no medida; si `decalpal` y `decdraw` salen limpios, el bloque de
  stencil pasa a sospechoso y habrá que contar por esa rama.

### 56.5 Estado

`xash.cmd` sin cambios. Los agujeros de bala están acorralados en una función de
veinte líneas: o su textura no tiene alfa, o sus vértices no tienen área. La próxima
línea `decalpal:` o `decdraw#:` elige.

## 57. Paso 3p — Las dos sondas absuelven a sus sospechosos; queda el z-buffer, y el experimento es gratis

**Las dos hipótesis de §56 firmaron «inocente» en la misma partida. La paleta de los
decals lleva su rampa de alfa perfecta — `a[0]=0, a[64]=64, a[128]=128, a[192]=192` —
el gradiente de manual de un decal GoldSrc, con las vallas mostrando su patrón binario
255/0 al lado como control. Y la geometría está sana: 4 vértices, coordenadas de mundo
razonables, UVs dentro de rango, la textura correcta enlazada. Además los sprites del
HUD son indexados+blend igual que un decal, y se ven: el muestreo de alfa por paleta y
el blending funcionan. La eliminación deja UNA diferencia entre un decal y todo lo que
sí se ve: es coplanar con una pared que ya escribió su profundidad — vive o muere por
el margen del depth test. Y ahí hay un número concreto que acusa.**

| Sonda | Resultado | Veredicto |
|---|---|---|
| `decalpal` | rampa 0→192, índice 255 transparente | **paleta: inocente** |
| `decdraw` | 4 verts, `v0 711 1488 -113`, st 0..1, tex correcta | **geometría: inocente** |
| Sprites HUD (control) | indexados+blend, visibles | **muestreo y blend: inocentes** |
| Diferencia restante | coplanaridad con la pared | **el depth test, acusado** |

### 57.1 El número que acusa

El motor pide el offset de polígono estándar: `glPolygonOffset( -1, -gl_polyoffset )`
con `gl_polyoffset` por defecto a **2**. En OpenGL de PC, el parámetro `units` se
multiplica por la mínima diferencia resoluble del z-buffer — el driver lo convierte a
«ticks». **pbgl pasa el bias crudo al registro del NV2A**
(`NV097_SET_POLYGON_OFFSET_BIAS = -2.0`), y en un z-buffer de 24 bits fijos con el
rango completo en ~16,7 millones de ticks, eso son **dos ticks de margen**. El quad
del decal se rasteriza con vértices distintos a los triángulos de la pared sobre el
mismo plano; el redondeo de interpolar Z puede comerse dos ticks sin despeinarse. El
decal pierde el test píxel a píxel y no queda nada — invisibilidad total con la
tubería entera sana, que es exactamente el cuadro.

En PC el mismo `2` funciona porque el driver escala las unidades. Aquí nadie escala:
es la misma familia que la máscara de 26 bits o el `%g` — código correcto en su casa,
descalibrado en ésta.

### 57.2 El experimento: cero recompilaciones

`gl_polyoffset` se lee **cada frame** (`DrawDecalsBatch`), así que se ajusta por
`userconfig.d` como el mando en §46. Subido `decals-test.cfg` con `gl_polyoffset 64`
y el guion de bisección escrito dentro:

1. Con `64`: si los agujeros aparecen, hipótesis confirmada — bajar 32 → 16 → 8 hasta
   el mínimo estable, y ese valor pasa a código como default de Xbox.
2. Si con `64` no: subir a `1024` como prueba burda.
3. Si ni con `1024`: la profundidad queda exonerada de verdad, y la siguiente ronda
   desactiva el depth test del dibujado de decals por código para partir lo que quede.

### 57.3 Qué NO está verificado

- **La hipótesis del margen**, hasta que dispares con el cfg puesto. Las sondas
  `decalpal`/`decdraw` se quedan una ronda más y se retiran con el arreglo.
- La semántica exacta del BIAS del NV2A (¿ticks?, ¿unidades de NDC?) está deducida de
  cómo pbgl lo programa, no medida — la bisección la medirá de facto.

### 57.4 Estado

XBE sin cambios (el de §56, verificado). `decals-test.cfg` en la consola. La partida
de siempre: un cargador a la pared — pero esta vez el resultado se ve sin abrir el
log.

## 58. Paso 3q — 1024 de offset aplicado y verificado: la profundidad exonerada, y el careo de estados desplegado

**La verificación que pediste antes de tocar código, hecha y con lección incluida: el
log demuestra que `userconfig.d/decals-test.cfg` se ejecutó — `aplicado
(gl_polyoffset 1024)`, sin ningún aviso de cvar protegido — mientras los decals
seguían emitiendo miles de draws invisibles. Con mil ticks de margen y nada en
pantalla, **la profundidad queda exonerada y la hipótesis de los ticks de §57 estaba
equivocada**. También cae la última teoría estática (`GL_POLYGON` en modo inmediato:
el mundo visible lo usa en seis sitios de `gl_rsurf.c`). Se acabaron las teorías: esta
ronda despliega el careo — el estado interno real de pbgl volcado en el instante de un
draw de decal y en el de un sprite con blend que sí se ve. El diff de esas líneas es
el diagnóstico.**

| Verificación | Resultado |
|---|---|
| ¿Se ejecutó el cfg? | **Sí**: `execing userconfig.d/decals-test.cfg` → `aplicado (gl_polyoffset 1024)` |
| ¿Algún rechazo del cvar? | Ninguno en el log |
| ¿Decals dibujándose mientras tanto? | 12.860 drawcalls, `decdraw` con geometría sana |
| ¿Aparecieron? | No → **profundidad exonerada** |
| `GL_POLYGON` inmediato | el mundo lo usa (`gl_rsurf.c:879, 966…`) → exonerado |

### 58.1 El estado de la eliminación, honesto

Absueltos con evidencia: la tubería entera (§56), la paleta y su rampa de alfa, la
geometría, el muestreo indexado+blend (los sprites del HUD lo usan y se ven), la
profundidad (este experimento), el polygon offset, el stencil (no corre en paredes
opacas), y la primitiva. **Mi hipótesis de §57 era errónea** y queda anotada como
tal: con bias 1024 los agujeros habrían aparecido sí o sí si el margen fuera el
problema.

Lo que queda no es una lista de sospechosos: es la certeza de que **algún elemento del
estado GL difiere** entre el draw del decal y los draws que funcionan — y que leerlo
en el fuente ya no alcanza, porque todo lo legible está leído.

### 58.2 El careo: `pbgl_dbg_state`

Nueva función de volcado **dentro de pbgl** (`state.c`, en el parche), que imprime el
estado que la librería tiene de verdad en el momento de la llamada — no el que el
motor cree haber pedido:

```
glstate TAG: flags[at A bl B cull C zt D st E tex0 F], blend SRC/DST, alpha FUNC ref R
glstate TAG: depth func F mask M, colormask CCCCCCCC, texenv MODE, color x1000 R G B A
glstate TAG: tex0 fmt XX WxH pal P mips a/b alloc A
```

Llamada dos veces desde cada lado de la comparación:

- **`DECAL`** — en `DrawSingleDecal`, justo antes del `glBegin` del polígono
  invisible.
- **`SPRITE`** — en `R_DrawSpriteQuad`, el dibujado con blend **visible** (los
  muzzleflash que funcionan pasan por aquí).

Seis líneas en el log, a difear a ojo. La corazonada que el volcado incluye a
propósito: el **color corriente** (`varray[COLOR1].value`) — `DrawSingleDecal` nunca
pone `glColor`, y con `texenv GL_MODULATE` un alfa corriente de 0 heredado de
cualquier draw anterior multiplica todo el decal a nada. En PC el orden de dibujado
deja otro color colgado; aquí puede dejar el letal. Si `color x1000 … 0` aparece en
la línea DECAL y `… 1000` en la SPRITE, ese es el bug — y el arreglo, un
`pglColor4ub( 255, 255, 255, 255 )` en el sitio justo.

### 58.3 Qué NO está verificado

- **El careo no ha corrido**: `libpbgl.lib` reconstruida (parche a 375 líneas), motor
  relinkado, 0 SSE2, desplegado por md5 (`4b0880ee…`).
- La corazonada del color corriente es la favorita **por ser la única variable de
  estado que ningún descarte ha tocado**, no por evidencia directa.
- `decals-test.cfg` sigue en la consola con 1024; inocuo, se limpia con el arreglo.

### 58.4 Estado

La misma partida corta de siempre: un disparo a la pared basta (el volcado salta en
el primer decal y el primer sprite). Seis líneas de log, un diff a ojo, y el
subsistema de decals — lo único que le falta al port para jugarse completo — queda
con su bug señalado por comparación directa de máquina, no por teoría.

## 59. Paso 3r — El careo iguala todo menos el depth test, y reabre la profundidad por el signo

**Las seis líneas del careo hacen su trabajo: matan la hipótesis del color — el decal
dibuja con `color 1000 1000 1000 1000`, alfa corriente perfecto — e igualan todo lo
demás: mismo blend src, mismo texenv MODULATE, mismo colormask, mismo formato de
textura indexada, mismo depth func LEQUAL. El diff completo entre el draw invisible y
el visible cabe en tres celdas: el sprite lleva alpha test (irrelevante para
invisibilidad), blend aditivo en vez de alpha-blend, y — la que importa — **el sprite
dibuja con el depth test APAGADO**. Y esa celda destapa el agujero del experimento de
§58: el volcado no imprime el polygon offset, y la «exoneración» de la profundidad
descansaba en asumir que el BIAS del NV2A comparte signo con OpenGL. Si la convención
está invertida, el `1024` de la ronda anterior empujó los decals mil ticks **detrás**
de la pared: el experimento no descartó la profundidad — la agravó.**

| Estado | DECAL (invisible) | SPRITE (visible) |
|---|---|---|
| color corriente | `1000 1000 1000 1000` | `1000 1000 1000 784` |
| blend | `0302/0303` | `0302/0001` (aditivo) |
| alpha test | off | on |
| **depth test** | **on** | **off** |
| depth func / mask, colormask, texenv, formato | idénticos | idénticos |

### 59.1 Lo que muere y lo que resucita

- **Muerta la hipótesis del color heredado** (la favorita de §58): el alfa corriente
  del decal es 1.0. El `pglColor4ub` no hace falta.
- **Resucita la profundidad**, y con mejor mecánica que la versión de §57: no es el
  *tamaño* del margen, es el **signo**. En GL, bias negativo acerca al espectador;
  si el registro `NV097_SET_POLYGON_OFFSET_BIAS` del NV2A interpreta el signo al
  revés — o pbgl debería negarlo y no lo hace —, todos los offsets que el motor pide
  («acércame el decal») lo alejan. Un decal empujado detrás de su propia pared falla
  LEQUAL en el cien por cien de los píxeles: invisibilidad limpia, tubería sana, y
  los dos experimentos anteriores (2 y 1024) fallando por el mismo motivo con el
  investigador mirando el valor y no el signo.
- Coherente además con el careo: lo visible-con-blend o bien no usa depth test
  (sprites del muzzleflash) o bien no es coplanar con nada (el mundo escribe primero).

### 59.2 El experimento, gratis otra vez

El motor llama `GL_PushPolygonOffset( -1, -gl_polyoffset )`, así que un valor
**negativo** del cvar produce un bias **positivo** en el hardware. `decals-test.cfg`
actualizado y subido:

- `gl_polyoffset -1024`: si los agujeros aparecen, el signo está invertido —
  confirmación instantánea. Bisecar después (-256, -64, -16, -4) al mínimo estable, y
  el arreglo definitivo va a pbgl: negar el bias (y el factor) en el flush, con la
  convención documentada, dejando el cvar en su 2 de siempre.
- Si con -1024 tampoco: la profundidad queda exonerada **con las dos direcciones
  probadas**, y la siguiente ronda replica por código el estado exacto del sprite
  (depth test off) en el dibujado del decal — que es feo pero es lo que GoldSrc
  hacía en software, y desempata entre «depth» y «otra cosa» de una vez.

### 59.3 Qué NO está verificado

- **El signo del BIAS es la hipótesis, no un hecho**: nace de que el careo dejó a la
  profundidad como única variable con mecánica plausible, no de una medida del
  registro.
- El careo se tomó en superficies opacas; el estado del decal sobre superficies
  transparentes (bloque de stencil) sigue sin comparar.

### 59.4 Estado

XBE sin cambios (el de §58, con el careo dentro — se retira con el arreglo).
`decals-test.cfg` con `-1024` en la consola, verificado subido. Un disparo a la
pared: si hay agujero, era el signo y el arreglo son tres líneas en pbgl; si no,
la profundidad cae con honores y el siguiente careo es contra el estado del sprite.

## 60. Paso 3s — Tres celdas, tres colores: el experimento que desempata de un solo cargador

**El `-1024` tampoco. Tercer fracaso de la profundidad por offset (2, 1024, -1024:
ambos signos probados) y fin de las rondas de un-cvar-una-hipótesis: esta ronda el
experimento vive en el código y responde a las tres celdas del careo **a la vez**.
Cada decal recibe una — y solo una — diferencia del diff DECAL vs SPRITE de §58,
elegida por su puntero (estable frame a frame) y **teñida de un color que la nombra
en pantalla**: ROJO = depth test apagado, VERDE = alpha test encendido, AZUL = blend
aditivo, sin tinte = control intacto. Un cargador contra la pared reparte las cuatro
variantes entre los agujeros y el veredicto se lee a ojo: el color que aparezca es
la celda culpable; varios colores, combinación; ninguno, las tres celdas absueltas
de golpe y el volcado no cubre al asesino.**

### 60.1 El experimento: variantes por identidad, no por orden

En `DrawSingleDecal` (`gl_decals.c`), bloque `XASH_XBOX` alrededor del
`glBegin(GL_POLYGON)`:

```c
int variant = (int)(((uintptr_t)pDecal ) >> 5 ) & 3;
```

- La variante sale del **puntero del decal**, no de un contador de draws: los decals
  se repintan cada frame, y un contador daría a cada agujero una variante distinta
  por frame (parpadeo inleíble). El puntero es la identidad del decal — misma
  variante siempre.
- **Variante 1 (ROJO, 255/60/60)**: `glDisable(GL_DEPTH_TEST)` — la celda del
  sprite que motivó el plan B. Lo que GoldSrc software hacía, feo y definitivo.
- **Variante 2 (VERDE, 60/255/60)**: `glEnable(GL_ALPHA_TEST)` — el func/ref ya son
  idénticos en ambas líneas del careo (`GREATER ref 0`), solo falta el enable.
- **Variante 3 (AZUL, 60/60/255)**: `glBlendFunc(GL_SRC_ALPHA, GL_ONE)` — el dst
  aditivo del sprite en lugar del `ONE_MINUS_SRC_ALPHA` del decal.
- **Variante 0 (control)**: intocada. Si el control aparece, algo más cambió y la
  ronda queda invalidada — el control está para eso.
- Restauración simétrica tras el `glEnd` (depth on, alpha off, color blanco); el
  blend no se restaura porque cada decal y cada caller ponen el suyo.

El tinte va por `glColor4ub` con alfa 255 — con `texenv MODULATE` colorea sin tocar
la rampa de alfa de la textura, así que no puede *causar* la aparición: solo la
etiqueta.

### 60.2 La auditoría del polygon offset: la cadena está entera

Tu pregunta de si `glPolygonOffset` llega de verdad al hardware, auditada eslabón a
eslabón — **ninguno está roto**:

| Eslabón | Veredicto |
|---|---|
| `GL_PushPolygonOffset` (motor, `gl_backend.c:481`) | primer push hace `glEnable(GL_POLYGON_OFFSET_FILL)` + `glPolygonOffset(factor, units)` — orden correcto |
| Tabla de punteros (`vid_xbox.c:126`) | `glPolygonOffset` cableado al de pbgl, no hay stub |
| `glPolygonOffset` (pbgl `state.c:774`) | **no está vacía**: guarda factor/units y marca `polyofs.dirty` si cambian |
| `glEnable` case `GL_POLYGON_OFFSET_FILL` (`state.c:489`) | `FLAG_DIRTY_IF_CHANGED(polyofs, …)` — el enable también marca `polyofs.dirty` |
| Flush (`state.c:290`) | empuja `NV097_SET_POLY_OFFSET_FILL_ENABLE` + `SCALE_FACTOR` + `BIAS` y limpia dirty |
| ¿Y el modo inmediato flushea? | sí: `glBegin` llama `pbgl_state_flush()` (`immediate.c:55`) |

Conclusión: los tres registros del NV2A **reciben** los valores. Lo que nadie
garantiza es la *semántica* — pbgl empuja `units` crudo como BIAS, sin el escalado
por «unidad resoluble del z-buffer» que OpenGL manda, y a estas alturas (±1024 sin
efecto visible) la sospecha razonable es que el registro espera otra escala u otra
etapa del pipeline. Por eso el desempate de esta ronda importa: si el ROJO aparece,
el problema ES depth y la siguiente excavación va a la semántica del BIAS; si no
aparece ni el rojo, la semántica del BIAS deja de importar.

Para verlo en vivo y no por fe, `pbgl_dbg_state` gana una cuarta línea:

```
glstate TAG: polyofs en E factor x1000 F units x1000 U dirty D
```

En la próxima línea DECAL debe salir `en 1 factor x1000 -1000 units x1000 -2000
dirty 0` (el push de `DrawDecalsBatch` con el cvar de vuelta a 2, ya flusheado). Si
`dirty` sale 1 en pleno draw, el flush no corrió y toda la tabla de arriba queda
desmentida por la máquina — por eso se imprime.

### 60.3 Qué NO está verificado

- **El experimento no ha corrido**: esta ronda es despliegue. La asignación por
  puntero reparte las variantes pseudo-aleatoriamente — un cargador entero (10-12
  agujeros) da ~3 decals por variante; con 2 disparos podría tocarte dos veces la
  misma.
- El careo de §58 se tomó en paredes opacas; el experimento hereda ese alcance.
- La semántica del BIAS del NV2A sigue sin medir — a propósito: primero el
  desempate, después (solo si hace falta) la metrología.
- `gl_polyoffset` vuelve a `2` en el cfg para no mezclar variables: el offset queda
  como estaba de fábrica mientras el experimento prueba otra cosa.

### 60.4 Estado

`libpbgl.lib` reconstruida (parche regenerado pendiente de `git diff`), motor
relinkado con `gl_decals.c` recompilado (verificado por timestamp del objeto y por
la cadena `polyofs en` dentro del XBE), 0 SSE2 en 420 objetos, desplegado con
md5 round-trip (`7eeb932b…`). `decals-test.cfg` con `gl_polyoffset 2` subido y
verificado. La partida: un cargador entero a la misma pared, y me dices **qué
colores ves** — esa frase tuya es el diagnóstico completo.

## 61. Paso 3t — La ronda era válida, la profundidad queda enterrada, y la caza sale del volcado

**Ningún color: las tres celdas del careo absueltas simultáneamente, y validado que
absueltas DE VERDAD — la ronda fue limpia. `sizeof(decal_t) = 60` medido con el
mismo target del build (`clang -fdump-record-layouts`, i386-pc-windows-msvc:
`sizeof=60, align=4`), el pool asigna secuencial (`gDecalPool[gDecalCount]`), y
`(base + 60·i)>>5 & 3` recorre las cuatro clases en cualquier ventana de 16 decals
consecutivos — con 491 decals creados en la partida, cada variante corrió decenas
de veces. Con el depth test APAGADO por código y nada en pantalla, la profundidad
queda enterrada tras cuatro intentos (2, 1024, -1024, disable). Y el log de esta
partida trae dos regalos: la línea `polyofs` **verifica por máquina** que el offset
llega al flush, y la línea SPRITE sale esta vez con `zt 1` — el sprite visible
dibujó con depth test ENCENDIDO, así que la celda estrella de §59 ni siquiera era
un discriminador estable. El asesino está FUERA del volcado, y esta ronda le quita
los dos escondites que le quedan: el estado no volcado, y la emisión misma.**

| Verificación pedida | Resultado |
|---|---|
| ¿Corrieron las 4 variantes? | **Sí** — distribución `[0,1,3,1,3,1,3,1,3,0,2,0,2,0,2,0]` por ventana de 16; 491 decals creados |
| `polyofs` en la línea DECAL | `en 1 factor x1000 -1000 units x1000 -2000 dirty 1` (1er decal) → `dirty 0` (2º) |
| ¿Qué significa ese dirty? | el volcado corre ANTES del `glBegin` (que es quien flushea): dirty 1 pendiente → dirty 0 flusheado. **El flush corre y los tres registros llegan al NV2A** |
| SPRITE esta partida | `zt 1` — visible CON depth test. La celda de §59 era circunstancial |
| cfg | `aplicado (gl_polyoffset 2)`, sin rechazos |

### 61.1 El replanteo: qué NO cubría `pbgl_dbg_state`

Lo volcado estaba igualado y el decal sigue muerto, luego el bug vive en lo no
volcado. Los puntos ciegos, ahora impresos (dos líneas nuevas en el careo):

```
glstate TAG: units en ABCD env E0 E1 E2 E3 act sv S cl C scis X
glstate TAG: view X Y WxH z N F, mv id I sum x1000 M, pr id J sum x1000 P
```

- **Unidades de textura 1-3**: el volcado solo miraba la 0. El sospechoso con mejor
  mecánica de todos los que quedan: los decals dibujan por el camino *single*
  (`batched now 0` en el log), es decir **inmediatamente después del pintado de la
  pared, con la pasada de lightmaps recién terminada**. Una unidad que esa pasada
  dejara encendida multiplica su etapa en el combiner; si su alfa es 0, el decal
  muere bajo `SRC_ALPHA` — y mata **las cuatro variantes de §60 a la vez**, porque
  todas conservaron `src=SRC_ALPHA` y la textura puesta. Los sprites, que dibujan
  mucho más tarde y tras otros reseteos, ni se enteran. Encaja con cada síntoma.
- **Matrices y viewport**: modelview/projection con flag identity y checksum
  (suma de los 16 elementos ×1000), viewport completo, scissor. Una transformación
  rota en ese punto del frame manda el polígono fuera de pantalla con todo el
  estado por-draw perfectamente sano.

### 61.2 La prueba bruta de existencia: dos sondas, dos colores

En `DrawSingleDecal`, tras el draw normal (que queda como control):

- **MAGENTA** — el mismo polígono del decal redibujado con TODO apagado: sin
  textura, sin blend, sin alpha test, sin depth test, color sólido. Si el NV2A
  emite algo en este punto del frame con estas matrices, hay parches magenta en la
  pared.
- **AMARILLO** — una vez por frame, un cuadrado en la esquina inferior izquierda
  bajo matrices IDENTIDAD (push/LoadIdentity/pop en projection y modelview), mismo
  todo-apagado. Prueba la emisión cruda en esa posición del frame sin depender de
  las matrices de la escena.

| Lectura | Diagnóstico |
|---|---|
| magenta SÍ | emisión y matrices sanas → el asesino es estado no rastreado (unidades 1-3 favoritas; el careo extendido lo nombra) |
| magenta NO, amarillo SÍ | matrices/viewport rotas en el momento del decal (los checksums lo confirmarán) |
| ninguno | ese punto del frame no llega al framebuffer: la investigación se muda a la emisión/pushbuffer |
| magenta Y agujeros normales | (no esperado) el control curado por accidente — ronda a repetir |

### 61.3 Qué NO está verificado

- **La prueba no ha corrido**: esta ronda es despliegue + validación retroactiva.
- La hipótesis de la unidad fantasma es la favorita por mecánica, no por medida —
  las dos líneas nuevas del careo la confirman o la matan en la próxima partida.
- Observado de paso en el log, anotado como deuda: `GL_INVALID_OPERATION while
  uploading gfx/env/desertrt.bmp` (cielo; cuadra con los avisos `alpha_sky`/
  `solid_sky`). No toca a los decals.
- El careo sigue disparándose 2 veces por tag y por arranque; suficiente.

### 61.4 Estado

`libpbgl.lib` reconstruida (parche regenerado, 415 líneas), motor relinkado con
`gl_decals.c` recompilado (verificado: cadena `units en` en el XBE, objetos
posteriores a la lib), 0 SSE2 en 420 objetos, desplegado con md5 round-trip
(`7bb26332…`). `decals-test.cfg` sin cambios (`gl_polyoffset 2`). La partida: un
solo disparo a la pared basta, y me dices dos cosas — **¿magenta en la pared?
¿amarillo en la esquina?** Con eso y las dos líneas nuevas del log, el asesino se
queda sin escondidos.

## 62. Paso 3u — La corrección que cambia el sospechoso: alterna por DISPARO, y la caza se muda a los eventos

**Tu observación mata la pista del doble búfer antes de nacer: la alternancia es
uno-sí-uno-no POR DISPARO a cualquier cadencia, y eso no es un artefacto de frames —
es un toggle en el camino del evento. En GoldSrc cada disparo vive dos veces
(predicción del cliente + confirmación del servidor) con un protocolo para que solo
UNA vida produzca efecto: el arma dispara `FEV_NOTHOST`, el servidor se salta al
host (`sv_ev_skip`) porque la predicción ya lo reprodujo, y la predicción solo
ejecuta cuando `g_runfuncs` es verdadero (la primera pasada del comando, no los
replays). Un toggle roto en CUALQUIERA de esas tres piezas produce exactamente tu
síntoma. Esta ronda instrumenta las tres capas de la vida doble — y de camino cayó
un bug del empaquetador que impedía enlazar: el detector de choques se
realimentaba de su propio enlace anterior.**

### 62.1 Tus tres preguntas, y qué las responde

1. **¿Se descartan alternativamente las dos vidas?** El careo de §54 (`ev_queued`/
   `ev_fired`) contaba el total sin distinguir origen. Ahora se parte: `ev_q_loc`
   (encolado por la predicción vía `CL_PlaybackEvent`) contra `ev_q_srv` (llegado
   por red vía `CL_ParseEvent`/`CL_ParseReliableEvent`), más `sv_ev`/`sv_ev_skip`/
   `sv_ev_sent` en el servidor. La contabilidad sana con `cl_lw 1` es: **1 disparo
   = 1 `sv_ev` = 1 `sv_ev_skip` = 1 `ev_q_loc`, con `ev_q_srv` = 0**. La columna
   que pierda los disparos silenciosos nombra al culpable.
2. **¿98/98/0 escondía la alternancia?** Posible y ahora medible: aquellos números
   decían que todo lo encolado se despachó, pero no CUÁNTAS veces se encoló por
   apretón de gatillo. Si eran 49 disparos × 2 vidas, o 98 × 1 vida con la mitad
   tragada ANTES de encolar, el careo antiguo no lo distinguía. El nuevo sí:
   `evpulse` en el log una vez por segundo, más una línea por evento (`evq LOC/SRV/
   REL`, `evfire`, `svev`, tapadas a 64) para emparejar disparo a disparo.
3. **¿El gamedll?** Instrumentado en las tres puertas: `hudpbev` en
   `HUD_PlaybackEvent` imprime **el valor de `g_runfuncs` en cada intento** — si
   los disparos silenciosos aparecen como `run 0` en su primer intento, la
   alternancia vive en la contabilidad de la predicción (`processedfuncs` en
   `cl_pmove.c:1060`), no en los eventos; `evbullets` en `EV_HLDM_FireBullets` (el
   manejador que hace chispas y decals); `evdecal` en `EV_HLDM_DecalGunshot` con el
   veredicto de si la traza dio en algo decalable.

### 62.2 La tensión honesta con los contadores viejos

Tu hipótesis fuerte — «el decal que no se ve NUNCA SE CREÓ» — explica la mitad
silenciosa, pero los contadores de §55-61 muestran la otra mitad VIVA: 491 decals
creados y 33.000 drawcalls en la última partida, y el magenta de §61 pintándose un
frame. Es decir: hay dos bugs apilados. La alternancia (mitad de los eventos
tragada: ni chispas ni decal) es NUEVA y esta ronda la caza; la invisibilidad de
los decals que SÍ se crean y SÍ se dibujan sigue abierta con el protocolo
magenta/amarillo de §61 intacto en el XBE. Si tu corazonada acierta del todo,
las dos investigaciones colapsan en una; si no, tenemos las dos instrumentadas
a la vez y una partida responde a ambas.

### 62.3 El bug del empaquetador: el detector que se comía su propia cola

El primer enlace de esta ronda murió con 7 símbolos sin resolver
(`_HUD_PostRunCmd`, `_IN_Client*`, `_V_CalcRefdef`…) referenciados por `<root>` —
la firma de una petición del propio enlazador, no de un objeto. La cadena, pieza a
pieza:

1. Los objetos del cl_dll llevan directivas COFF `/EXPORT:` **incrustadas**
   (sección `.drectve`, invisible para `llvm-nm`) en las funciones `dllexport`.
2. El enlace del XBE las obedece y lld genera `build/engine/xash.lib` con esos
   exports — un **subproducto** del enlace.
3. `ofx-hlsdk-package.sh` construye su lista de «símbolos ya en el XBE» con TODOS
   los `.lib` del build del motor — incluido ese subproducto. Al re-empaquetar
   (primera vez desde §24 con un enlace previo en el build), vio los exports del
   propio gamedll como símbolos del motor, los declaró choques y los renombró:
   la directiva incrustada quedó colgando y el enlace murió.

Arreglo: `! -name 'xash.lib'` en el detector, con el porqué documentado en el
script. La prueba del veneno: de **500 choques del server a 1**, y de 878 del
client a 838, al excluirlo. Pariente conceptual de la familia «la herramienta se
cree su propia salida anterior» — como el configure de la norma de §35, el
dumpinfo de §36, y ahora el empaquetador.

### 62.4 Qué NO está verificado

- **La partida no ha corrido**: esta ronda es instrumentación pura; cero cambios
  de comportamiento fuera de las trazas.
- El hlsdk no define `XASH_XBOX` (defines: `CLIENT_WEAPONS=1`…), así que sus tres
  sondas van bajo `#if 1` con comentario — retirarlas con el arreglo.
- `sizeof` y reparto de §60 siguen válidos; el protocolo magenta/amarillo de §61
  sigue dentro del XBE, sin tocar.

### 62.5 Estado

`client.dll`/`server.lib` recompilados y re-empaquetados con el detector
arreglado, motor relinkado, 0 SSE2 en 420 objetos, las 6 cadenas de sonda
verificadas dentro del XBE, desplegado con md5 round-trip (`760b1761…`). El
arreglo del bucle de START+BACK de la ronda anterior sigue dentro (la traza vieja
de `XLaunchXBE` ausente del binario). La partida: **cuenta los apretones de
gatillo** — un cargador lento, apuntando a la pared — y trae el log. `evpulse`
dirá en qué columna se pierden los silenciosos, `hudpbev` dirá si `g_runfuncs`
alterna, y de propina el magenta/amarillo de §61 responde lo suyo.

## 63. Paso 3v — El dato que absuelve al renderer entero, la sonda que nunca fue control, y los eventos exonerados por columna

**Tu observación 1 hace en una frase lo que ocho rondas no pudieron: absuelve al
renderer, al estado GL, a la profundidad y a pbgl DE GOLPE — el magenta aparece y
queda permanente en cualquier mapa salvo el inicial, luego el hardware y todo el
camino de dibujo funcionan. El fallo es del mapa inicial o de la primera carga, y
la prueba directa ya está en la consola: `xash.cmd` cambiado a `+map c1a2` y
verificado por round-trip — el próximo arranque ES el experimento. Tu observación
2 destapa además un error de diseño mío que hay que escribir con la misma tinta
que el de §40: el cuadrado amarillo vivía DENTRO de `DrawSingleDecal`, solo
corría cuando un decal dibujaba — nunca fue el control independiente por-frame
que §61 decía que era. Y el log de tu partida responde a C con contabilidad
perfecta: la alternancia NO está en los eventos.**

### 63.1 (C) Los eventos, columna a columna: cadena completa en TODOS los disparos registrados

```
evpulse: sv 36 skip 34 sent 2, q loc 34 srv 2
```

Los 2 `sent`/`srv` son los dos eventos reliable del propio mapa al cargar (idx 15
y 9, `flags 26`). Los otros 34 son apretones que llegaron al arma, y **los 34
tienen la cadena entera**: `svev idx 4` → `svev skip NOTHOST` (el servidor cede
al cliente, correcto con `cl_lw 1`) → `hudpbev run 1 final 1` (ni un solo `run
0`: la predicción nunca tragó un intento por replay) → `evq LOC` → `evfire` →
`evbullets #N` secuencial → `evdecal` con `solid 4` (BSP, decalable). La
contabilidad sana de §62 se cumple al 100%: 1 disparo = 1 sv_ev = 1 skip = 1 q
loc, `q srv` = 0.

Conclusión: **si la mitad de tus apretones no producen nada, esos apretones
nunca llegan a ser disparos** — mueren ANTES del arma, en la puerta de ataque.
Nueva sonda `atkgate` en `ItemPostFrame` (hlsdk, `hl_weapons.cpp`): una línea por
flanco de subida de IN_ATTACK con el cooldown que la puerta vio en ese instante
(`next x1000`, entero). Si los apretones silenciosos salen con `next > 0`, la
alternancia es la contabilidad del cooldown (`m_flNextPrimaryAttack -= msec`) y
tenemos al culpable; si ni siquiera aparecen, el botón no llega y la caza sube al
camino del usercmd.

### 63.2 (B) El amarillo: la sonda que se auditaba a sí misma

`grep` lo confirma sin ambigüedad: el bloque del amarillo estaba dentro del
`#if XASH_XBOX` de `DrawSingleDecal`, tras el draw del decal. Solo se ejecutaba
cuando un decal dibujaba — tu observación «solo al disparar a un cristal, solo
donde los decals funcionan» es exactamente su condición de ejecución vista desde
fuera. **Nunca fue un control independiente**, y las filas de la tabla de §61 que
dependían de él («amarillo sí/no») quedan anuladas. Lo que SÍ sobrevive de §61 es
el magenta, que es quien ha dado el dato decisivo de esta ronda. El amarillo se
retira; su pregunta («¿emite el NV2A en ese punto del frame?») la responde el
magenta permanente de los mapas sanos. Hermano del error de §40: una
instrumentación que solo corre bajo condición convierte cada silencio en un falso
positivo — va a la lista de patrones.

### 63.3 (A) Las dos hipótesis, y la tercera que traen los contadores

La prueba directa (a)-contra-(b) ya está armada: `+map c1a2` en la consola. Si
falla c1a2 y funciona c1a1, es «el primer mapa cargado»; si c1a2 funciona, es
c1a1 en concreto. Pero el log de tu partida deja una miga que sugiere una
tercera: `decpulse: req 127 -> shot 299 -> created 575` — **más shots que
requests y más created que shots**, cuando en §56 la cadena era monótona. Algo
re-dispara decals sin pasar por `CL_DecalShoot`: el camino de RESTAURACIÓN (los
ficheros de transición de nivel re-aplican los decals guardados al re-entrar).
Si tu «vuelvo al mapa inicial y dejan de funcionar» fue por changelevel, ese
re-entrar pasó por la restauración — y el mapa del arranque también recibe una
pasada equivalente al activarse. Hipótesis (c): **el camino de re-aplicación
corrompe o desvincula lo que toca**.

Para partir el empate, el ciclo de vida instrumentado (`declife`, por segundo):

```
declife: runs R clip0 C unlink U (created N)
```

- `runs` planos + `unlink` disparado → algo ARRANCA los decals de sus
  superficies cada frame (y `R_DecalUnlink` es LA función de borrado: contador
  dentro).
- `clip0` disparado → el clipper empieza a devolver 0 vértices (el early-out de
  `DrawSingleDecal` ahora cuenta en vez de callar).
- `runs` sanos y nada crece → dibujan y algo los tapa (y el magenta lo diría).

### 63.4 Qué NO está verificado

- La prueba c1a2 no ha corrido; la hipótesis (c) es la favorita por la miga de
  los contadores, no por medida directa.
- `atkgate` detecta el flanco con un `static` que en replays de predicción
  contaría de más; en single-player con esta latencia es inocuo.
- Confirmado por ti y anotado: sonido vuelto tras arranque en frío (APU
  reinicializado por el reinicio completo) y START+BACK sin bucle — los dos
  arreglos de §61-62 validados en máquina.

### 63.5 Estado

XBE con `declife` + `atkgate` dentro y el amarillo fuera (verificado: cadenas
nuevas presentes, `YELLOW` ausente), gamedll recompilado y re-empaquetado
(detector de choques estable: server 1 / client 838), 0 SSE2, desplegado md5
round-trip (`eb1784c3…`). `xash.cmd` en consola: `+map c1a2`, verificado. La
partida pide tres cosas: (1) arranca y dispara en c1a2 — ¿fallan AHÍ ahora?;
(2) viaja a c1a1 y a un tercer mapa y dispara en cada uno; (3) **cuenta los
apretones** en una ráfaga lenta. El log trae `declife`, `atkgate` y `evpulse` —
tres respuestas de una partida.

## 64. Paso 3w — El ascensor no es un bug: es el cepo de diseño de c1a2, y ya tiene botón de escape

**Tu ascensor está diagnosticado por código y por datos del mapa, sin necesitar
el log: NO es pariente del microondas ni ningún fallo del port — es el juego
original. c1a2 se diseñó para entrarse por changelevel desde c1a1c, y arrancarlo
en frío deja al jugador en un cepo que en el PC original se comporta EXACTAMENTE
igual. La salida ya está montada: un bind de mando que dispara las puertas por
`ent_fire` (herramienta del propio fork), staged y con subida automática en
cuanto la consola reaparezca en la red. El log de tu partida sigue dentro de la
consola — el FTP es del dashboard y ahora mismo no hay consola en la red — así
que `atkgate`, `declife` y la confirmación del magenta en c1a2 quedan para el
apéndice de esta sección en cuanto el vigilante lo baje.**

### 64.1 La anatomía del cepo, entidad a entidad

Del lump de entidades de c1a2 (extraído del pak con `tmp-entdump.py`, bounds de
brushes con `tmp-bspmodels.py`):

| Entidad | Qué es | Dato clave |
|---|---|---|
| `info_player_start` (2320 −1056 −540) | donde nace el jugador con `+map c1a2` | DENTRO de la cabina (la zona sur) |
| `*39` + `*67` | las dos hojas de la puerta, `targetname startele1` | plano y≈−842; toggle, `wait -1` |
| `*133` (2235 −853) | botón INTERIOR | **`master "button_lock"`** |
| `*134` (2408 −831) | botón del pasillo, sin master | inalcanzable con puertas cerradas |
| `button_lock` | multisource | su ÚNICA fuente es la puerta `*39` |

La cadena que te dejó encerrado: la puerta `*39` tiene `target button_lock` y lo
dispara **al terminar de abrirse** (`DoorHitTop`, doors.cpp:677). En arranque
limpio la puerta nunca se ha abierto → la fuente del multisource está OFF →
`IsTriggered` = 0 → botón interior bloqueado. Y el síntoma exacto que
describiste está en `buttons.cpp`: `ButtonActivate()` **emite el sonido del
botón (:732) ANTES de consultar al master (:734)**; con `locked_sound 0` el
bloqueo es mudo. «Hace ruido y no pasa nada» = botón bloqueado por master, al
pie de la letra.

En el flujo pensado por Valve nunca se llega ahí: vienes de c1a1c, el
`trigger_changelevel` trae `changetarget elestartmm` (un `CFireAndDie` cruza la
transición contigo, triggers.cpp:1468), y ese multi_manager simula el viaje
(sonidos + shake) y dispara `startele1` a los 8,5 s: las puertas se abren solas,
`*39` arma el multisource al llegar arriba, y el botón interior queda operativo
para el viaje de vuelta (cerrar puertas → `DoorHitBottom` dispara su `netname`
`eledoordelaymm` → changelevel a c1a1c). El gamedll del port es el hlsdk vanilla
línea por línea en toda esta cadena: **en el PC original, `map c1a2` por consola
deja el mismo softlock**. No hay nada que arreglar en el port.

### 64.2 El botón de escape: `ent_fire` por mando

El fork trae los enttools de FWGS (`sv_client.c:3093`): `ent_fire <patrón>
<orden>` corre server-side si `sv_enttools_enable 1` y el jugador ya nació.
`ent_fire startele1 use` con el jugador de activador entra por
`CBaseDoor::Use` → `DoorActivate` → `DoorGoUp` en las dos hojas — y al llegar
arriba `*39` arma el multisource: el ascensor queda EXACTAMENTE como si
hubieras llegado de c1a1c, botón interior incluido. Eso convierte el cepo en la
prueba de transición que pedía §63: sales, disparas, vuelves a entrar, botón
interior → changelevel a c1a1c.

Nuevo `userconfig.d/ascensor-escape.cfg` (staged en `research/stage/`):

```
sv_enttools_enable 1
bind STICK2 "ent_fire startele1 use"
```

STICK2 (click del stick derecho) duplicaba `+duck` con L1, así que no se pierde
nada; `+duck` sigue en L1. Un click del stick derecho dentro del ascensor y las
puertas se abren. BORRAR el cfg cuando `xash.cmd` deje de arrancar en c1a2.

### 64.3 Qué NO está verificado (y quién lo verificará)

- **El log de tu partida c1a2 no se ha podido bajar**: la consola no está en la
  red (el FTP es del dashboard; en juego desaparece, y la última IP del 17-ago
  ya no responde). `tmp-xbox-watch.sh` corre en segundo plano: barre la subred
  cada 90 s, identifica la consola por su FTP (busca `default.xbe` en
  `E:/Apps/xash/`), baja `xash-boot.log` y sube `ascensor-escape.cfg` con md5
  round-trip. Tus tres preguntas del log (atkgate, declife, magenta en c1a2 pese
  a ser mapa de arranque — y qué tiene c1a1 de particular) se responden en
  cuanto caiga.
- El careo estructural c1a1 vs c1a2 ya está hecho y NO señala al infodecal como
  distintivo: c1a1 trae 68 y c1a2 60 — los dos aplican decals de mapa al
  arrancar. Lo que distinga a c1a1 tendrá que decirlo `declife`.
- El cfg de escape no ha corrido en máquina (cadena verificada solo en código:
  enttools → `pfnUse` → `CBaseDoor::Use` → `DoorActivate`).

### 64.4 Estado

Diagnóstico del ascensor cerrado por código + datos del mapa; escape staged con
subida automática pendiente de consola; XBE sin tocar (el de §63, md5
`eb1784c3…`, sigue siendo el bueno — esta ronda no necesitaba recompilar nada).
La partida siguiente: dentro del ascensor, click del stick derecho, y el resto
del guion de §63 sigue vivo — disparar en c1a2, viajar (puerta norte a c1a2a, o
ascensor de vuelta a c1a1c) y contar apretones.

## 65. Paso 3x — El log de c1a2 cierra tres cazas de golpe: la alternancia era el ARMA, el arranque queda absuelto, y al asesino solo le queda el alfa

**La partida del ascensor (log `xash-boot-20260819-115352`, 8050 líneas) responde
todo lo que §63 dejó armado y algo más. Uno: la alternancia nada-magenta NUNCA fue
un bug — es la cadencia de tracers del MP5, comportamiento vanilla del arma con la
que probabas. Dos: `declife` demuestra que en c1a2-de-arranque tu decal se dibuja
CADA frame de principio a fin de la partida — la hipótesis (b) «es el primer mapa
cargado» cae, como apostaste, y lo que queda es (a): c1a1 en concreto, pendiente de
una partida con `declife` arrancando allí. Tres: el careo extendido de §61 llegó en
este log y mata al último sospechoso de estado — unidades 1-3 APAGADAS, matrices,
viewport y scissor de libro — mientras `decalpal` enseña la rampa de alfa llegando
perfecta a `glColorTableEXT`. Reinterpretando §60 con esto: las cuatro variantes
conservaban textura y blend por alfa — si el alfa MUESTREADO es 0, las cuatro mueren
juntas y el magenta (sin textura) vive. Todo converge en un único hueco sin mirar:
qué lee el NV2A de esa textura. La ronda nueva lo mira con los ojos.**

### 65.1 (4) `atkgate`: la puerta estaba abierta — el silencio era del arma

24 flancos de IN_ATTACK, TODOS con `next x1000 -1` (≤ 0: puerta abierta, el arma
disparó cada vez; el `-1000` del #1 es el descuento del despliegue). `evpulse`
cuadra por columnas (26 disparos = 26 q loc, sv 28 = 26 skip + 2 reliables del
mapa) — pero `evbullets 26` contra **`evdecal 13`: exactamente la mitad**. La
mitad que falta no muere en ninguna puerta: muere en `ev_hldm.cpp:719` — el MP5
dispara con `iTracerFreq 2`, y en la rama `BULLET_PLAYER_MP5` (`:472-477`) **el
disparo con tracer no hace ni sonido de impacto ni decal**. Vanilla puro; a
bocajarro el tracer ni se ve (el segmento nace por detrás del plano de la pared).
La partida de §63 no lo enseñó porque fue con la glock: freq 0, decal SIEMPRE — y
sus 34 disparos eran 2 cargadores de 17. La caza de la alternancia **se cierra**:
era el arma. (El clip del log lo firma: 25→1 y recarga a 50 — el cargador del MP5.)

### 65.2 (2)(3) `declife`: c1a2-de-arranque dibuja SANO — (b) cae

La serie por segundos, leída entera:

| fase | runs | unlink/created | lectura |
|---|---|---|---|
| carga (t≈25s) | 12 y plano | 175/280 | decals del mapa aplicados; los 105 vivos están FUERA del ascensor (PVS) — no dibujan, correcto |
| disparos (t=42-110s) | crece ≈ fps, sin parar | +14/+15 | **un decal vivo dibujándose CADA frame**; cada disparo nuevo solapa y desvincula al anterior (unlink +1 por created +1) |
| t=115s→fin | plano en 982 | — | dejaste de jugar (el log acaba en `returning to dashboard: START+BACK` a t=118,9s) |

`decdraw#600` lo firma con nombre: el MISMO `{shot1` en (2429 −852 −508) —tu
balazo en la pared de la cabina— dibujándose frame tras frame, 4 verts sanos,
texcoords 0..1. **En c1a2 de arranque el ciclo de vida es sano de punta a punta:
la hipótesis (b) cae y el fallo del magenta-un-frame es de c1a1 en concreto.**
Qué tiene c1a1 de particular NO está medido aún — `declife` no existía en las
partidas de c1a1 — y es una partida barata: `xash.cmd` a `+map c1a1` cuando
cerremos la ronda del alfa. `clip0 = 0` toda la partida: el clipper, absuelto.

### 65.3 (§61 saldado) El careo extendido: el último estado sospechoso, muerto

```
glstate DECAL: units en 1000 env 2100 2100 2100 2100 act sv 0 cl 0 scis 0
glstate DECAL: view 0 0 640x480 z 0 1, mv id 0 sum ..., pr id 0 sum -7672
```

Unidades 1-3 **apagadas** (la unidad fantasma de §61, favorita por mecánica,
muerta por medida), scissor off, viewport completo, proyección idéntica a la del
SPRITE visible. Y el estado por-draw del decal es de libro: blend 0302/0303,
MODULATE, color blanco, zt LEQUAL con offset −1/−2. A la vez, `decalpal` (§56)
enseña la rampa: `{shot1 a[0]=0 a[64]=64 a[128]=128 a[192]=192 a[255]=0` — el
alfa correcto ENTRA en `glColorTableEXT`. Con §60 releído a esta luz (las cuatro
variantes conservaban `src=SRC_ALPHA` y la textura puesta → un alfa muestreado a 0
las mata a las cuatro y deja vivo al magenta), el espacio se estrecha a UNA
pregunta: **¿qué muestrea el NV2A de la textura del decal?** El código de pbgl
leído entero sale limpio (paleta BGRA con alfa en `tex_store_palette`, formato
`SZ_I8_A8R8G8B8`), que es exactamente el patrón de §61: cuando cada pieza mirada
por separado es sana, la respuesta la da la máquina, no el grep. Cabo suelto
honesto: el sprite, con el mismo formato paletado y MODULATE, ES visible — lo que
distinga su paleta de la del decal es parte de la respuesta.

### 65.4 La ronda nueva: TEXOPAQUE y `palstore`

- **TEXOPAQUE** (`DrawSingleDecal`, tras el magenta): el mismo polígono una
  TERCERA vez, ENCIMA del magenta, con la textura PUESTA y todo lo demás quitado
  — REPLACE (fragmento = téxel crudo, sin math de color ni alfa), sin blend, sin
  alpha test, sin depth. Un vistazo decide:
  | ves | veredicto |
  |---|---|
  | el dibujo del balazo en un cuadrado opaco | muestreo y RGB de paleta sanos → **el asesino es el ALFA que ve el combiner** |
  | cuadrado negro/basura | el lookup de paleta no llega a la GPU |
  | solo magenta, sin cuadrado | encender ESTA textura mata la emisión |
- **`palstore`** (pbgl, `tex_store_palette`): la otra punta del tubo de
  `decalpal` — la paleta BGRA tal cual queda ALMACENADA para el NV2A, mismos
  cinco índices, filtrada a paletas con rampa real (los cientos de paletas
  opacas del mundo callan). `decalpal` bien + `palstore` bien = la pérdida es
  posterior (DMA/combiner); `palstore` a ceros = el almacenamiento es el asesino.

### 65.5 Qué NO está verificado

- TEXOPAQUE y `palstore` no han corrido — esta ronda es su estreno.
- La particularidad de c1a1 sigue sin medir (siguiente: `+map c1a1` con declife).
- El cfg del ascensor (§64) no ha corrido en máquina todavía.

### 65.6 Estado

`libpbgl.lib` reconstruida (parche regenerado, 446 líneas), motor relinkado
(`gl_decals.o` nuevo, cadena `palstore#` verificada en el XBE), 0 SSE2 en 420
objetos, desplegado con md5 round-trip (`51b82fd4…`). `ascensor-escape.cfg` ya
está en la consola (md5 verificado). La partida pide poco y responde mucho:
arranca (sigue en `+map c1a2`), click del stick derecho para abrir el ascensor
(valida §64 de paso), **cambia a la glock** (dpad hasta la 9mm — decal en CADA
disparo, sin tracers que ensucien la cuenta), UN disparo a la pared y me dices
cuál de las tres celdas de 65.4 ves. El log trae `palstore` de propina. Si
además sales, cruzas al norte (c1a2a) y vuelves, las transiciones de §63 quedan
ejercitadas con el ascensor ya operativo.

## 66. Paso 3y — La paleta se guarda perfecta: el asesino vive entre la memoria y un registro, y hay un sospechoso que explica TODOS los síntomas

**Cuadrado negro opaco: veredicto limpio, y `palstore` responde tu pregunta 1 sin
ambigüedad — la paleta queda ALMACENADA perfecta. `a[0]=0 a[64]=64 a[128]=128
a[192]=192` en las 32 paletas de decal del log, exactamente la rampa que entró por
`glColorTableEXT`. Luego el almacenamiento NO es el fallo, y la caza se muda a los
últimos tres metros: memoria → registro → muestreo. La lectura completa de ese
camino (tu pregunta 2) deja dos sospechosos con mecánica, y uno de ellos explica de
un tirón TODO lo que llevamos ocho rondas viendo — incluido por qué el alfa muere:
si los ÍNDICES de la imagen son ceros, cada téxel resuelve a `pal[0]`, cuyo alfa es
0 en las 32 paletas. Invisible bajo blend, negro bajo REPLACE, magenta (sin
textura) vivo, y las cuatro variantes de §60 muertas a la vez. Y sobre tu pregunta
3 tengo que corregirte, porque la evidencia lo exige: la paleta NO puede explicar
c1a1-contra-c1a2. Esta ronda pone el volcado de memoria que decide, y arranca en
c1a1 para atacar la otra pregunta con datos en la misma partida.**

### 66.1 (1) `palstore`: el almacenamiento, absuelto

40 líneas, y las 32 de decal idénticas a su `decalpal`:

```
decalpal:   {shot1  a[0]=0 a[64]=64 a[128]=128 a[192]=192 a[255]=0     <- entra
palstore#22: w 256  a[0]=0 a[64]=64 a[128]=128 a[192]=192 a[255]=0     <- queda guardada
```

Las dos puntas del tubo, iguales. `w 256` en todas (el tamaño que el registro
espera). Las 8 paletas con `a[255]=255` son de mundo/modelo, no de decal. **La
rampa está en `pal->data` con los bytes correctos**: si el NV2A muestrea negro, lo
que falla está DESPUÉS de este punto.

### 66.2 (2) El camino completo, de la llamada al registro

| paso | dónde | qué pasa |
|---|---|---|
| 1 | motor, `gl_image.c:1191` | `glColorTableEXT(GL_TEXTURE_2D, ...)` — **no copia**: guarda `pal->source` (puntero) y `pal->width` |
| 2 | `glTexImage2D` → `tex_alloc` (`:918`) | reserva `tex->total = ALIGN(size,64) + 1024`; **`pal->data = tex->data + ALIGN(size,64)`** — la paleta vive PEGADA a la imagen, en el mismo bloque |
| 3 | `tex_gl_to_nv` (`:923`) | calcula `nv.palette = DMA_B \| LENGTH_256 \| (pal->data & 0x03FFFFC0)` |
| 4 | `tex_store` (`:936`) → `tex_store_palette` | copia BGRA con alfa — **verificado por `palstore`** |
| 5 | `state.c:203`, cada flush con la unidad sucia | `NV097_SET_TEXTURE_PALETTE` ← `nv.palette` |

El **orden** es correcto (reserva → registro → copia: cuando se calcula la
dirección, `pal->data` ya existe), y el registro se re-emite en cada flush. Los dos
sospechosos que sobreviven a la lectura:

- **(A) Alineamiento a 64.** El registro lleva la dirección enmascarada con
  `0x03FFFFC0`: **los 6 bits bajos se pierden**. `PAL_ALIGN` es 64, pero
  `pbgl_mem_alloc` pide `MEM_ALIGNMENT` = **16** a
  `MmAllocateContiguousMemoryEx`. Como `pal->data ≡ tex->data (mod 64)`, si el
  bloque cae en 16, 32 o 48 mod 64, **el NV2A lee la paleta hasta 48 bytes ANTES
  de donde está** — es decir, lee la COLA DE LA IMAGEN como paleta. En la práctica
  el kernel reparte por páginas y probablemente alinea de sobra; es sospechoso por
  mecánica, no por medida, y el volcado lo mata o lo corona.
- **(B) Los índices de la imagen.** Nadie ha mirado nunca los bytes de índice que
  se almacenan. Si son ceros, cada téxel resuelve a `pal[0]` — y `decalpal`/
  `palstore` dicen que **`a[0]=0` en las 32 paletas de decal**. Eso da: alfa 0 →
  invisible bajo blend (§55-62), color de índice 0 → cuadrado negro bajo REPLACE
  (§65), magenta intacto (no usa textura), y las CUATRO variantes de §60 muertas
  juntas (todas conservaban la textura). **Es el único candidato que explica todos
  los síntomas de ocho rondas con un solo mecanismo.**

De paso, un dato nuevo del log que merece vigilancia: el motor **reescala** las
imágenes de decal — `upload: {blood2 48x48 -> 64x64`, `{scorch1 96x96 -> 128x128`
— y reescalar una imagen de ÍNDICES promediando números de índice es aritmética sin
sentido. No es LA causa (`{shot1 16x16 -> 16x16` no se reescala y falla igual),
pero si (B) se confirma, el reescalado es el primer sitio donde mirar por qué.

### 66.3 (3) La corrección: la paleta NO puede explicar c1a1 contra c1a2

Tu tercera pregunta da por hecho que un solo culpable explica las dos cosas, y hay
que separarlas: **el magenta se dibuja con `glDisable(GL_TEXTURE_2D)` y color
sólido** (§61.2, código intacto en `gl_decals.c`). Sin textura no hay paleta, luego
lo que hace que el magenta aparezca en un mapa y no en otro **no puede ser la
paleta**. Son dos ejes distintos:

- **eje textura** (paleta/índices): por qué el decal es invisible — en TODOS los
  mapas, desde §55. Es el que esta ronda acorrala.
- **eje mapa** (c1a1): por qué en c1a1 el magenta se pinta un frame y para. Eso es
  `DrawSingleDecal` dejando de EJECUTARSE, no dejando de verse.

El log de esta partida trae datos del segundo eje —viajaste c1a2 → c1a1c → c1a2,
con el ascensor abierto por el bind, así que **§64 queda validado en máquina**—
pero no concluyentes: en c1a1c `runs` se queda plano en 12 porque no disparaste
allí y ningún decal entró en vista. Lo que sí se ve es la restauración trabajando
en cada carga (`created` 301 → 314 → 378: +64 decals re-aplicados al volver a
c1a2), sin `unlink` desbocado. Por eso esta ronda **arranca en c1a1**: un disparo
allí y `declife` responde el segundo eje con la misma partida que responde el
primero.

### 66.4 La sonda que decide: `texmem`

Tres líneas en `pbgl_dbg_state` (que ya corre en un decal y en un sprite visible,
así que el careo sale gratis):

```
texmem TAG: data ... mip0 ... size N bpp B | idx i0 i1 i2 i3 i64 i65 i128 i129
texmem TAG: pal PTR mod64 M w 256 reg REG gpuptr PTR2 (off O)
texmem TAG: pal[0] B G R A  pal[idx0=I] B G R A  pal[255] B G R A
texmem TAG: gpu sees pal[0] ...  pal[1] ...
```

| lectura | veredicto |
|---|---|
| `idx` todo ceros | **(B) confirmado**: la imagen no llegó; el asesino es el camino de subida/swizzle, no la paleta |
| `mod64` ≠ 0 y `gpu sees` ≠ `pal[0]` | **(A) confirmado**: truncamiento de dirección; la GPU lee la cola de la imagen como paleta |
| `idx` con valores y `mod64` 0 y todo casa | los dos caen: el fallo está en el muestreo/combiner y la caza se muda al DMA de paleta |
| `NO PALETTE` | el registro nunca se armó para esta textura |

El sprite visible imprime lo mismo a la vez: **el careo memoria-contra-memoria de
una textura paletada que SÍ se ve contra una que no**, que es la comparación que
lleva ocho rondas faltando.

### 66.5 Qué NO está verificado

- (A) y (B) son hipótesis con mecánica; el volcado las decide. No he «arreglado»
  nada a ciegas: alinear a 64 por si acaso habría enmascarado cuál era.
- El eje mapa sigue sin medir en c1a1 (por eso el arranque cambia).
- La sonda `texmem` lee `mips[0].data` en crudo (datos SWIZZLEADOS): sirve para
  distinguir ceros de no-ceros, no para reconstruir la imagen.

### 66.6 Estado — y una trampa nueva para la lista

`libpbgl.lib` reconstruida (parche regenerado, 492 líneas), 0 SSE2 en 420 objetos,
XBE desplegado con md5 round-trip (`e4621d56…`), `xash.cmd` cambiado a **`+map
c1a1`** y verificado por round-trip. El `ascensor-escape.cfg` se queda (inofensivo
en c1a1) y ya demostró que funciona.

**La trampa, documentada porque casi cuela un binario mentiroso**: waf **no ve
`libpbgl.lib` como dependencia**. El primer enlace de esta ronda dio `rc=0` y un
XBE con la fecha del build ANTERIOR — la lib nueva no entró. Se caza mirando la
fecha del XBE contra la de la lib, y se cura forzando el relink (borrar el `.xbe` y
tocar un fuente del ref). La verificación buena no es `rc=0`: es **buscar la cadena
nueva dentro del binario** (`texmem` ×2, `gpu sees pal` ×1). Hermana de la familia
«la herramienta se cree su propia salida anterior» (§35, §36, §62.3), pero al revés:
aquí la herramienta NO se cree una entrada que sí cambió.

La partida: arranca (ahora en c1a1, sin ascensor de por medio), **glock**, un
disparo a la pared, y con eso el log trae `texmem` (el veredicto de 66.4) y
`declife` (¿sigue c1a1 pintando un frame y parando?). Dos ejes, una partida.

## 67. Paso 3z — El cuadrado negro era MÍO, no del NV2A: la textura está sana entera, y el asesino del mapa es un pool de 64

**Tengo que empezar retirando mi propia conclusión: el cuadrado negro NO era un
veredicto. Las dos texturas que TEXOPAQUE volcó (`{dent6` y `{crack3`) son
**negras por diseño** — la entrada 255 de su paleta en `decals.wad`, que es
justo la que `LUMP_GRADIENT` convierte en el color entero del decal, vale
literalmente (0,0,0). Un cuadrado negro era la salida CORRECTA, y mi sonda no
podía distinguir «muestreo roto» de «funcionando». Misma familia que el cuadrado
amarillo de §63. Lo que sí trae el volcado `texmem` es el careo que faltaba
desde §55, y absuelve la cadena de textura ENTERA: índices presentes, paleta
alineada a 64 con `mod64 0`, registro correcto, y lo que la GPU lee byte a byte
igual a lo que escribimos. Las hipótesis (A) y (B) de §66, las dos muertas. Y
mientras eso se caía, `decpulse` y `declife` destapaban al culpable del eje de
mapa, que sí es un bug de verdad y estaba a la vista desde §53: **el fork
redefine `MAX_RENDER_DECALS` a 64 en Xbox, y los mapas colocan más decals que
eso ellos solos.** Tu pregunta 3 tiene respuesta, y no era la paleta.**

### 67.1 La cadena de textura, absuelta byte a byte

```
texmem DECAL:  data 833dc000 size 1024 bpp 1 | idx 4 3 3 2 1 1 0 0
texmem DECAL:  pal 833dc400 mod64 0 w 256 reg 033dc401 gpuptr 033dc400 (off 0)
texmem DECAL:  pal[0] 0 0 0 0  pal[idx0=4] 0 0 0 4  pal[255] 0 0 0 0
texmem DECAL:  gpu sees pal[0] 0 0 0 0  pal[1] 0 0 0 1
texmem SPRITE: data 833b5000 size 4096 bpp 1 | idx 255 252 252 255 ...
texmem SPRITE: pal 833b6000 mod64 0 w 256 reg 033b6001 gpuptr 033b6000 (off 0)
texmem SPRITE: pal[0] 246 254 253 255  pal[idx0=255] 0 0 0 255
```

| sospechoso de §66 | veredicto |
|---|---|
| (A) desalineamiento de 64 | **muerto**: `mod64 0`, `off 0`, `gpu sees` == lo escrito. El asignador de nxdk reparte por páginas; los 6 bits que el registro tira estaban a cero |
| (B) índices a cero | **muerto**: `idx 4 3 3 2 1 1 0 0` — hay imagen. (El segundo volcado sí sale a ceros, pero es la esquina transparente de un `{crack3` 64x64) |
| la paleta «degenerada» (0,0,0,i) | **es la CORRECTA**: `Image_SetPalette`, caso `LUMP_GRADIENT` (`img_utils.c:314`), pone RGB = entrada 255 del WAD y alfa = índice. El WAD dice `{dent6` → (0,0,0) y `{crack3` → (0,0,0). Negro es el color de un desconchón |

Y el careo con el sprite, que es lo que llevaba ocho rondas faltando, sale
idéntico en todo lo estructural: misma ruta, misma alineación, mismo registro
bien formado. La diferencia entre ambos es solo el CONTENIDO — y el contenido
del decal es el que su WAD manda. Comprobado además contra el fichero original
(`tmp-waddecal.py` / `tmp-wadhist.py` sobre `decals.wad`): `{shot1` usa índices
hasta 202, `{crack3` hasta 251, `{blood2` hasta 254 — es decir, **el alfa que
esas imágenes pueden producir es de sobra visible**. La rampa no está aplastada.

Lo único que queda sin cerrar del eje textura es si ese alfa llega al combiner,
y para eso va la sonda nueva de 67.4 — esta vez con una lectura que no puede
confundirse consigo misma.

### 67.2 (3) La respuesta a tu tercera pregunta: el pool de 64

`decpulse` de la partida en c1a1, con **un solo apretón de gatillo** en todo el
log (`atkgate #1`, `evbullets #1`, `evdecal #1`):

```
req  9 -> shot  76 -> created 121      t=27,7 s
req 19 -> shot  86 -> created 131      t=28,7 s
req 29 -> shot  96 -> created 141      t=29,7 s
   ...  +10/s en las TRES columnas, hasta el final del log
```

Diez decals por segundo, entrando por la primera columna (`CL_DecalShoot`), que
no son del jugador. Y `declife` en paralelo:

```
declife: runs  0 unlink  31 (created 112)   <- fin de carga
declife: runs 17 unlink  40 (created 121)   <- primer frame con decals a la vista
declife: runs 17 unlink  50 (created 131)   <- y AQUI SE CONGELA
declife: runs 17 unlink 152 (created 233)   <- 11 s despues: unlink +112, runs +0
```

**`runs` congelado en 17 con `unlink` subiendo +10/s es literalmente tu «se pinta
un frame y para».** Y el mecanismo estaba escrito en `protocol.h` desde la dieta
de límites de §53.3:

```c
#if XASH_XBOX
#define MAX_RENDER_DECALS   64      // max rendering decals per a level
```

`gDecalPool[64]`. `R_DecalAlloc` recorre el pool en círculo y **desvincula el
decal que va a reutilizar** — con 64 huecos y 10 creaciones por segundo, el pool
entero se recicla cada 6,4 segundos y arranca de la pared todo lo que hubiera.
Peor: **el pool es más pequeño que lo que los mapas colocan ellos solos**. c1a1
tiene 68 `infodecal` y crea 112 al cargar; c1a2 crea 229. El mapa desborda su
propio pool antes de que el jugador dispare, y la carga termina con la mayoría de
los decals del mapa ya desvinculados.

Eso explica la asimetría entera sin tocar la paleta: **c1a2 estaba quieto** (nada
pedía decals mientras no disparabas, §65: `runs` subiendo toda la partida y el
magenta permanente), **c1a1 tiene una fuente de 10/s** que barre el pool en
bucle. No era «el primer mapa», ni «la restauración», ni la textura: era la
capacidad del pool contra el ritmo de creación de cada mapa.

Arreglo: **64 → 512** (30 KB de pool; `sizeof(decal_t)` = 60 de §61, y el mapa de
enlace lo confirma: 0x7800 = 30720 = 512×60). Los arrays VBO de `gl_rsurf.c`
escalan con la misma constante y son código muerto en Xbox (`R_GenerateVBO` está
compilado fuera) — se llevan ~500 KB de BSS que nadie lee; queda anotado en el
propio `#define` por si el número vuelve a crecer.

### 67.3 Lo que NO explica el pool

Los 10 decals/s de c1a1 siguen sin nombre. No son del jugador y no son normales:
son un cadencia de 100 ms exacta, y la lista de texturas cargadas allí
(`{ding*`, `{dent*`, `{crack*`, `{scorch*`) es el juego de impactos de bala. El
pool grande deja de convertirlo en un borrado total, pero **la fuente es un
segundo bug, probablemente el de verdad**. Por eso va sonda: `decshoot`
(`cl_tent.c`, `CL_DecalShoot`) imprime las primeras 24 peticiones y luego una por
segundo, con textura, entidad, modelo, flags y posición. Si salen todas en el
mismo sitio, es algo re-disparando; si van con la entidad de un NPC, es un
monstruo; si el modelo cambia, es la restauración.

### 67.4 `ALPHASHAPE`: la sonda que sustituye a la que se engañó

TEXOPAQUE se retira. En su sitio, el mismo tercer dibujo **con el alpha test
encendido a 0.5**: el RGB es negro pase lo que pase, así que lo que informa es la
FORMA que el alpha kill recorta sobre el magenta.

| ves | veredicto |
|---|---|
| parche magenta con un agujero negro **con forma** de impacto | el alfa se muestrea bien: la cadena de textura está sana de punta a punta |
| parche magenta intacto | el alfa muestreado nunca pasa de 0.5: no llega al combiner |
| parche magenta **entero** cubierto de negro | el alfa se ignora: vale 1 en cada téxel |

Los tres casos son visualmente distintos entre sí, que es exactamente lo que le
faltaba a TEXOPAQUE y al amarillo de §63.

### 67.5 Qué NO está verificado

- El pool de 512 no ha corrido. La prueba es directa: en c1a1, un disparo debe
  **quedarse** en la pared (y `declife` debe enseñar `runs` subiendo en vez de
  congelado en 17).
- La fuente de los 10/s no está identificada; `decshoot` la nombra.
- `ALPHASHAPE` no ha corrido.
- El magenta y `{dent6`: esa textura concreta usa índices hasta 26 (alfa máximo
  10%), luego es casi invisible incluso funcionando bien. No es bug nuestro, pero
  conviene no elegirla como testigo.

### 67.6 Estado

`protocol.h` con el pool a 512 (verificado en el mapa de enlace, 30720 bytes),
`ALPHASHAPE` en `gl_decals.c`, `decshoot` en `cl_tent.c`, 2 objetos de decals y 1
de cl_tent recompilados por el cambio de cabecera, 0 SSE2 en 420 objetos, XBE
desplegado con md5 round-trip (`10df5430…`). `xash.cmd` sigue en **`+map c1a1`**,
que ahora es el mapa-prueba: es el que tenía la fuga.

La partida: arranca, **glock**, un disparo a la pared, y **quédate mirándolo diez
segundos**. Dos preguntas de un vistazo: ¿el impacto SIGUE ahí a los diez
segundos (pool arreglado), y qué forma tiene el recorte negro sobre el magenta
(67.4)? El log trae `decshoot` con el nombre del que pedía diez decals por
segundo.

## 68. HITO — Los decals funcionan. Y la dieta de límites, auditada: el pool no estaba solo

**Trece secciones después de §55, los agujeros de bala están en la pared y se
quedan. Tu confirmación cierra los dos ejes a la vez: la forma correcta recortada
sobre el magenta (el alfa se muestrea bien, la cadena de textura estaba sana desde
el principio) y permanencia de más de diez segundos (el pool ya no los arranca).
El cabo suelto de §67 también tiene nombre, y no es nuestro: los diez decals por
segundo son el `env_laser` de c1a1 quemando la pared, comportamiento vanilla de
Half-Life. Lo que convertía eso en un borrado total era `MAX_RENDER_DECALS 64`. Y
la auditoría que pediste encuentra que ese `#define` no estaba solo: el mismo
bloque de dieta llevaba un tope que **ya estaba mordiendo y avisando en el log sin
que nadie leyera el aviso** — 93 «Too many entities in visible packet list» en una
sola partida de c1a2. Sondas de decals retiradas; el port vuelve a verse como el
juego.**

| | |
|---|---|
| Decals | **funcionan**: forma correcta, permanentes |
| Causa 1 (invisibilidad) | nunca hubo tal: la cadena de textura estaba sana; lo que fallaba eran mis sondas |
| Causa 2 (desaparición) | `MAX_RENDER_DECALS 64` — pool más pequeño que lo que los mapas colocan |
| Fuente de los 10/s | `env_laser` de c1a1, `SF_BEAM_DECALS`, `nextthink +0.1` — **vanilla** |
| Hallazgo nuevo | `MAX_VISIBLE_PACKET 128` desbordando: entidades que no llegan al cliente |

### 68.1 (2) Quién pedía diez decals por segundo: el láser de c1a1

`decshoot` no dejó margen de duda. Las 88 peticiones salen **todas del mismo
sitio**, `at 475 1312 -86`, sobre `ent 0` / `mdl 1` (el mundo), rotando al azar
entre cinco texturas: 839, 840, 845, 846, 847 — y la 839 se subió al log un
instante antes de la primera petición con el nombre `{bigshot1`. Cinco `bigshot`
al azar, cadencia de 100 ms, punto fijo. Eso es una firma, y el código la tiene:

```c
void CLaser::StrikeThink( void ) {          // effects.cpp:1057
	...  FireAtPoint( tr );  pev->nextthink = gpGlobals->time + 0.1f;
}
void CBeam::BeamDamage( TraceResult *ptr ) { // effects.cpp:717
	... if( pev->spawnflags & SF_BEAM_DECALS )
		if( pHit->IsBSPModel() )
			UTIL_DecalTrace( ptr, DECAL_BIGSHOT1 + RANDOM_LONG( 0, 4 ) );
}
```

Y la entidad, en el lump de c1a1: `env_laser` en (539 1612 -56), `damage 5000`,
`spawnflags 97` = STARTON | SPARKEND | **SF_BEAM_DECALS (0x40)**, apuntando a un
`LaserTarget` fijo. Es el láser de seguridad rojo del principio de *Unforeseen
Consequences*: está encendido desde que carga el mapa y **quema la pared diez
veces por segundo**, para siempre.

**Veredicto: vanilla.** No es bug nuestro ni del gamedll; el Half-Life de PC hace
exactamente lo mismo y no se nota porque allí el pool son 4096 decals (siete
minutos hasta reciclar). Con 64 el láser barría el pool entero cada 6,4 segundos y
arrancaba de la pared todo lo demás — incluido tu disparo. Con 512, unos 51
segundos; la del jugador sobrevive de sobra, que es lo que confirmaste mirando.
Queda anotado como el ritmo natural de ese mapa, no como defecto.

### 68.2 (3) La auditoría de la dieta de §53

El fork encogió límites en cuatro cabeceras bajo `#if XASH_XBOX`. Criterio de
revisión: single-player, 128 MB con ~96 libres, y **evidencia** — el motor avisa
solo cuando un tope muerde, así que lo primero fue releer los logs buscando esos
avisos en vez de teorizar.

| límite | fork | sobremesa | veredicto |
|---|---|---|---|
| `MAX_VISIBLE_PACKET` | 128 | 2048 | **MORDÍA, MEDIDO** → 512 |
| `NUM_PACKET_ENTITIES` | 64 | 256 | su pareja: sube con él → 256 |
| `MAX_DECAL_SURFS` | 256 | 4096 | siguiente tope del camino recién arreglado → 512 |
| `MAX_RENDER_DECALS` | 64 | 4096 | arreglado en §67 → 512 |
| `MAX_DLIGHTS` / `MAX_ELIGHTS` | 16 / 32 | 32 / 128 | barato y cosmético → 32 / 64 |
| `MAX_MODELS` / `MAX_SOUNDS` | 512 / 512 | 1024 / 2048 | **vigilar**: sin desbordar en los logs; si aparece «limit exceeded», subir |
| `MAX_CHANNELS` / `MAX_DYNAMIC_CHANNELS` | 128 / 20 | 256 / 60 | **vigilar**: sin síntoma medido y el sonido acaba de estabilizarse (§61) — no se toca sin evidencia |
| `BLOCK_SIZE_MAX` / `MAX_LIGHTMAPS` | 128 / 64 | 1024 / 256 | **no tocar aún**: atado al tope de textura de 512 y al pool de 64 MB de pbgl; merece ronda propia |
| `MAX_TEXTURES` | 2048 | 4096 | holgado (los logs usan ~900) |
| `MAX_LIGHTSTYLES` | 64 | 256 | 64 es el límite del propio GoldSrc |
| `CMD_BACKUP` 16, `MULTIPLAYER_BACKUP`, `MAX_CUSTOM`, `NET_MAX_FRAGMENT` | — | — | multiplayer/latencia: inertes en single-player local |

**El hallazgo:** `MAX_VISIBLE_PACKET` a 128 estaba desbordando **y avisando**, y
el aviso llevaba ahí desde siempre:

```
Error: Too many entities in visible packet list. Ignored 11 entities
```

93 veces en la partida de c1a2, hasta 11 entidades descartadas de golpe. Eso es
`sv_frame.c:142`: entidades que el servidor **decide no enviar** porque el array
está lleno — cosas que deberían verse y no llegan al cliente. Con `sv ents 336` en
el heartbeat, un techo de 128 visibles se alcanza en cualquier escena poblada. Lo
mejor del caso: **el formato de red nunca hizo falta cambiarlo**.
`MAX_VISIBLE_PACKET_BITS` sigue siendo 11 (2048) porque el bloque Xbox no lo
redefinió — el campo del protocolo siempre tuvo sitio; lo único encogido era el
array. Coste medido en el mapa de enlace: `frame_ents` pasa de ~43 KB a 175 KB
(`sizeof(entity_state_t)` = 340 B).

Patrón para la lista, y es el mismo de §67 con otra cara: **un tope que se ignora
en silencio no se nota; uno que avisa tampoco, si el aviso vive en un log que solo
se lee buscando otra cosa.** El grep de «too many / exceeded / overflow» sobre los
logs viejos es ahora parte del ritual de cada ronda.

### 68.3 (1) Limpieza: el port vuelve a verse como el juego

Retiradas (verificado por ausencia de cadena en el XBE, que es la comprobación
buena de §66.6):

| sonda | dónde vivía | por qué se va |
|---|---|---|
| MAGENTA | `DrawSingleDecal` | tinte a pantalla; su pregunta está respondida |
| ALPHASHAPE | `DrawSingleDecal` | ídem; cumplió en una partida |
| `decdraw#` | `DrawSingleDecal` | volcado por draw |
| `decalpal` | `gl_image.c` | volcado por subida de textura |
| `palstore#` | pbgl `tex_store_palette` | ídem, del otro lado del tubo |
| `texmem` | pbgl `pbgl_dbg_state` | el volcado de memoria de §66 |
| `decshoot#` | `cl_tent.c` | **diez líneas por segundo**: la más cara de todas |

Se quedan, a coste de una línea por segundo, los contadores que son la alarma de
regresión de justo este bug: `declife` (runs / clip0 / unlink / created) y
`decpulse` (req → shot → created → drawcalls). Si el pool vuelve a estrangularse,
`runs` se congela y `unlink` se dispara: la firma es reconocible de un vistazo.
`pbgl_dbg_state` se queda en la lib pero **sin llamadas** — es un volcador de
estado de uso general, útil para la próxima caza.

En la consola: `ascensor-escape.cfg` borrado (su condición de retirada era dejar
de arrancar en c1a2) y `decals-test.cfg` borrado — su `gl_polyoffset 2` era
exactamente el valor por defecto del cvar, así que no cambiaba nada, pero era
deuda del experimento de §60. Quedan `joystick.cfg` y `weapons-test.cfg`.

### 68.4 Qué NO está verificado

- Los límites nuevos no han corrido: `MAX_VISIBLE_PACKET` 512,
  `NUM_PACKET_ENTITIES` 256, `MAX_DECAL_SURFS` 512, `MAX_DLIGHTS` 32,
  `MAX_ELIGHTS` 64. La prueba es directa y es la misma partida de siempre: el
  aviso «Too many entities» debe **desaparecer** del log de c1a2.
- La retirada de sondas no ha corrido: el riesgo es cosmético (que algún decal
  salga sin tinte donde antes había magenta, que es justo lo que se busca).
- `MAX_MODELS`/`MAX_SOUNDS`/canales de sonido y el bloque de lightmaps quedan en
  vigilancia consciente, no revisados en máquina.

### 68.5 Estado

`libpbgl.lib` reconstruida sin sondas (parche de vuelta a 416 líneas), motor
relinkado a la fuerza (la trampa de §66.6: waf no ve la lib), 0 SSE2 en 420
objetos, XBE desplegado con md5 round-trip (`68f05264…`). Pool de decals
verificado otra vez en el mapa de enlace (30720 B = 512×60) y `frame_ents` en
175 KB. `xash.cmd` se deja en **`+map c1a1`**: es el mapa con el láser, o sea el
peor caso para decals y el mejor sitio para ver de un vistazo que el arreglo
aguanta — cambiarlo a `+map c1a0` para una partida normal es un FTP de un segundo.

La partida ya no pide mirar tintes: **juega**. Dispara a las paredes y los
agujeros deben acumularse y quedarse, sin magenta. Y el log responde de propina si
el aviso de entidades ha desaparecido, que es el bug que esta auditoría acaba de
sacar de debajo de la alfombra.

## 69. Paso 4a — Opposing Force dentro del XBE: 731 puntos de entrada, ABI limpia, y un techo de diseño que hay que decidir

**El gamedll de Op4 está enlazado dentro del `default.xbe`: 660 puntos de entrada
de servidor y 71 de cliente, 0 duplicados, 0 símbolos sin resolver, 0 SSE2 en 233
objetos, y `{ "?Displace@CDisplacer@@QAEXXZ" ... }` en las tablas del cargador
estático. Los dos puntos que pediste primero por si traían sorpresas responden
así: el (1) sale casi gratis — de todos los parches sobre master, opfor solo
necesitaba UNO, porque el resto se hicieron originalmente sobre esta misma rama;
y el (2) da tres diferencias reales, todas benignas, más UNA sorpresa que no está
en el gamedll sino en el motor: **con enlazado estático, un XBE solo puede
contener UN mod.** El nombre que el motor busca no es el del gameinfo, es la
constante `"server"`. Hay salida barata y está documentada abajo. Nada
desplegado: la consola conserva el XBE de Half-Life que funciona, porque los
assets de gearbox aún no están subidas (tu punto 3).**

| | |
|---|---|
| Fuentes opfor compiladas | 233 objetos, `opfor.dll` + `client.dll` |
| SSE2 | **0** en 233 objetos (parche portado) |
| Puntos de entrada | server **660** (HL: 499), client **71** (HL: 63) |
| Choques renombrados | server **1** (igual que HL), client **1041** (HL: 838) |
| Exports sin resolver | **0** |
| ABI de frontera | **limpia**, idéntica a master |
| `default.xbe` | 4.628.480 → **5.017.600 B** |

### 69.1 (1) Los parches: uno, no todos

El censo de qué había que portar de master a opfor, fichero a fichero:

| parche de master | ¿lo necesita opfor? |
|---|---|
| `vcs_info` inline (`libs = []` + `game_shared/vcs_info.c`) | **ya lo tiene** — ese parche se escribió sobre ESTA rama en §18; el `excluded_files` con `gearbox/gearbox_client.cpp` lo delata |
| capa de compatibilidad `external/nxdk` + `-include nxdk_compat.h` | **ya lo tiene** (§18) |
| `scripts/waifulib/xcompile.py` | **byte a byte idéntico** entre las dos ramas |
| `-march=pentium3 -mno-sse2` en CPPFLAGS | **FALTABA** — único portado |

El de SSE2 faltaba porque se descubrió en §24, después de §18. Y es exactamente el
que no se puede olvidar: es una propiedad del **target**, no de la rama, y su
síntoma sería un opcode inválido en la primera operación en coma flotante del
código de juego, con un log apuntando a cualquier otro sitio (§21.2, §24.7). Se
portó con el porqué escrito al lado, y **verificado como manda §24.7: leyendo los
opcodes, no los flags.** Los `CFLAGS` del caché de waf siguen diciendo
`-march=pentium-m`; el arreglo vive en `CPPFLAGS`, que la orden de compilación
emite después. Medición: **0 instrucciones SSE2 en 233 objetos**.

Tres herramientas quedan parametrizadas por rama en vez de duplicadas —
`OFX_SDK=opfor` en `ofx-hlsdk-build.sh` y `ofx-abi-structs.sh`, y un directorio de
build como argumento en `ofx-xbox-nosse2.sh`.

Y una trampa del empaquetado, hermana de las tres de §24.6: **el nombre de la DLL
del servidor no es fijo**. master la llama `hl.dll` y opfor `opfor.dll`
(`mod_options.txt`: `SERVER_LIBRARY_NAME`). El script lo tenía escrito a mano;
suponerlo habría dado un `.lib` correcto con **0 exports** y ni un solo error —
la clase de fallo que sale a la luz tres sesiones después. Ahora lo busca.

### 69.2 (2) Qué necesita Op4 que HL no: tres diferencias, todas benignas

- **Cabeceras de frontera: ninguna.** `common/`, `engine/` y `public/` son
  **byte a byte idénticas** entre master y opfor. No hay structs propios de Op4 en
  la frontera con el motor. El prober de §51 sobre la rama opfor lo confirma por
  medida: `cl_entity_s` 3000, `entity_state_s` 340, `clientdata_s` 476,
  `usercmd_s` 52, `weapon_data_s` 88, `local_state_s` 6448 — todos iguales, y
  `ref_params_s` con la misma cola de 8 bytes ya conocida y acolchada.
  **Veredicto: ABI limpia.**
- **`pm_shared` sí difiere**, y es el único código compartido que cambia: Op4
  añade `CHAR_TEX_SNOW_OPFOR` con sus cuatro sonidos de pisada en nieve y un
  `FL_IMMUNE_SLIME`. Autocontenido en la copia del gamedll; el motor no tiene
  `pm_shared.c` propio que discrepe.
- **Límites propios del gamedll**: `MAX_WEAPON_SLOTS` 5→7 y `MAX_ITEM_TYPES` 6→8
  en `dlls/cdll_dll.h`. Los dos lados del gamedll los comparten y el motor no los
  ve, así que no es un problema de frontera — pero explica por qué el HUD de Op4
  tiene dos filas más de inventario.
- **Edicts: sin sorpresa.** El `gameinfo.txt` de gearbox pide `max_edicts 1200`,
  exactamente lo mismo que valve, y `MAX_EDICTS` del motor son 8192 (13 bits) que
  el bloque `#if XASH_XBOX` **no** encogió. Después de la auditoría de §68 esto
  había que mirarlo; está holgado.
- **Assets: completos y con la herencia bien declarada.** 470 MB en
  `game/gearbox/`, con `pak0.pak` de 165 MB, `liblist.gam` original
  (`startmap of0a0`, `trainmap ofboot0`) y un `gameinfo.txt` ya generado por Xash
  con **`basedir "valve"`** — la herencia que hace falta. Ojo al subirlos (tu
  punto 3): la copia WON trae la protección anticopia del CD —`clokspl.exe`,
  `secdrv.sys`, `drvmgt.dll`, `*.gbx`, `DQ2249.ICD`— que no pinta nada en la
  consola y son megas tontos.

### 69.3 La sorpresa: un XBE, un mod

No está en Op4 sino en el mecanismo de §24, y aparece al leer de dónde sale el
nombre que `COM_LoadLibrary` busca en las tablas:

```c
static void COM_GenerateServerLibraryPath( const char *alt_dllname, char *out, size_t size )
{
#ifdef XASH_INTERNAL_GAMELIBS // assuming library loader knows where to get libraries
	Q_strncpy( out, "server", size );
```

Con enlazado estático **el gamedir se ignora**: el motor pide literalmente
`"server"` y `"client"`, y la tabla generada tiene exactamente una entrada de cada.
Así que enlazar Op4 **sustituye** a Half-Life; no conviven. El `gamedll
"dlls/opfor.dll"` del gameinfo no se lee siquiera.

La salida buena está ya en el motor y no cuesta tocar código: `-dll` y
`-clientlib` rellenan `host.gamedll`/`host.clientlib`, y
`COM_GetCommonLibraryPath` los usa **verbatim** cuando no están vacíos
(`lib_common.c:253`). Es decir, con las tablas nombradas
`server_hl`/`server_op4`, un `xash.cmd` con `-dll server_op4 -clientlib
client_op4` elige mod sin recompilar. Lo que sí cuesta es el enlace: los dos
gamedlls comparten ~90% del código, así que el segundo entra entero renombrado
contra el primero (el script ya sabe hacerlo: es lo mismo que hace con el cliente
contra el servidor, 1041 choques) y el XBE crecería otros ~1,5 MB.

**Decisión de esta ronda: un solo mod, y que sea Op4** — es el objetivo del
proyecto, y Half-Life base ya cumplió su papel de validar el mecanismo. El XBE de
HL sigue en la consola y el de Op4 se guarda aparte
(`~/opposing-force-x/default-op4.xbe`); volver a HL es un `configure
--xbox-gamelibs=gamelibs-hl` y un FTP. Si más adelante interesa tener los dos en
un binario, la ruta está escrita arriba y no toca fuentes del motor.

### 69.4 Qué NO está verificado

- **Nada de esto ha corrido.** Es §24 otra vez: enlaza, los símbolos están donde
  el motor los busca, y no se ha ejecutado una sola instrucción. Deliberadamente
  **no se ha desplegado**: sin los assets de gearbox en la consola, el XBE de Op4
  solo serviría para romper el Half-Life que funciona.
- El renombrado de 1041 choques del cliente no está auditado uno a uno; lo que sí
  está medido es el resultado: 0 duplicados y 0 sin resolver en el enlace.
- `MAX_WEAPON_SLOTS 7` no se ha visto dibujar. Si el HUD sale raro, ése es el
  primer sitio donde mirar.
- Op4 usa los mismos límites de red que HL, pero **sus mapas son otros**: el
  aviso de entidades de §68 hay que volver a buscarlo en el primer log de
  gearbox, no darlo por resuelto.

### 69.5 Estado

`hlsdk-portable` (rama opfor) con el arreglo de SSE2 portado y comentado;
`gamelibs-op4/` empaquetado (server.lib 20,4 MB / client.lib 3,1 MB, manifiestos
de 660 y 71 entradas); motor reconfigurado contra ellos y XBE enlazado
(**5.017.600 B**, guardado en `~/opposing-force-x/default-op4.xbe`), con las
tablas verificadas conteniendo símbolos que solo existen en Op4 (`CDisplacer`,
`CShockrifle`, `CVoltigoreEnergyBall`). La consola sigue con el XBE de Half-Life
y sus assets intactos.

Lo siguiente son tus puntos 3 y 4, que ya no tienen incógnitas de mecanismo:
subir `gearbox/` sin la basura del CD, y `xash.cmd` con `-game gearbox +map
of0a0` — o `ofboot0` si se quiere empezar por el campo de entrenamiento, que es
además el sitio más barato para ver si el HUD de siete ranuras se dibuja.

## 70. Paso 4b — Pantalla limpia sin perder el log, y un bloqueo que no es de código: la consola no tiene sitio

**Las dos piezas de software están hechas y verificadas dentro del XBE: la
pantalla se queda sin una sola línea de texto y el log de disco **no pierde
nada**, que es más de lo que pedías. Pero el despliegue está parado por algo que
no se arregla compilando: **la partición E: de la consola está llena**. Está
medido, no supuesto: una escritura por FTP de CINCO BYTES contesta `451 Error
writing file`. La subida de gearbox entró 6,5 MB y murió ahí. No he borrado nada
tuyo, y no pienso hacerlo sin que lo digas: E: tiene tu Gentoox, ClassiCube,
mugen, XBMC4Gamers y demás. Hay salida limpia y sin borrar nada, abajo.**

### 70.1 Separar «registrar» de «pintar encima»: por qué no bastaba quitar `-dev 2`

Tu diagnóstico del síntoma era exacto — `-dev 2` es lo que enciende las líneas de
notify sobre el juego (`console.c:1838`, gateadas por `host.allow_console`, que
`-dev` pone a true). Pero quitarlo se llevaba por delante algo que sí querías
conservar. En este motor las dos cosas son el mismo interruptor:

- `Con_Printf` no está gateado: siempre llega a `Sys_Print`.
- `Con_DPrintf` (developer ≥ 1) y `Con_Reportf` (developer ≥ 2) **no dejan de
  dibujarse: dejan de EJECUTARSE**. No llegan a `Sys_Print`.
- Y en esta consola `Sys_Print` **es** el log de disco (`sys_con.c:294`:
  `Xbox_TraceRaw( line )`).

Es decir: sin `-dev 2`, `D:\xash-boot.log` habría perdido toda la conversación del
motor —carga de mapas, subidas de textura, `Spawn Server`— que es justo lo que
lleva quince secciones resolviendo bugs. Tu «el log en disco sigue igual» y tu
«quita el -dev 2» eran incompatibles, y el que manda es el primero.

`-noconsole` (nuevo, `host.c`) corta **solo la mitad de pantalla**: se ejecuta
después de los overrides de Quake y dedicado, pone `host.allow_console = false` y
deja intacto el nivel de developer. Resultado: el log conserva cada línea desde la
primera, y todo lo que lee `allow_console` —notify, consola, netgraph— no dibuja
nada. Comprobado que los once consumidores de esa variable son dibujo o entrada de
consola, ninguno lógica de juego.

### 70.2 El overlay del trazador pasa a opt-in

`Xbox_TraceOverlayEnable` se llamaba incondicionalmente tras `pbgl_init`
(`vid_xbox.c:277`). Ahora va bajo `-traceoverlay`. El `Xbox_TraceOverlayDraw` de
cada swap se queda, pero sale de inmediato por `!xbox_overlay_on`, así que no
cuesta nada.

Se retira con honores: fue el único testigo cuando la consola moría antes de que
nada llegara al disco (§29-30), y por eso sigue a una bandera de distancia — si un
arranque futuro muere en negro, `-traceoverlay` en `xash.cmd` devuelve la pantalla
como testigo. El log de disco es otro sumidero (`Xbox_TraceWriteFile`) y no
depende de esto.

`xash.cmd` queda: `-noip -dev 2 -noconsole +sv_cheats 1 -game gearbox +map of0a0`.

### 70.3 El bloqueo: E: llena

| prueba | resultado |
|---|---|
| `STOR` de un fichero de 5 bytes en `E:/Apps/xash/gearbox/` | `451 Error writing file` |
| navegación FTP a ese directorio | `250 CWD command successful` (la ruta está bien) |
| subida de gearbox | entraron 6,5 MB (DECALS.WAD, OPFOR.WAD, cached.wad, delta.lst, events/, gameinfo.txt, gfx/) y el resto falló |
| escritura en `C:`, `F:`, `G:` | **OK en las tres** |

E: es la partición original de ~4,9 GB (el disco está ampliado y particionado —
hay `Xbpartitioner` instalado y F:/G: existen), y la ocupan cosas tuyas: Gentoox
(`vmlinuz`, `initrd.gz`, `sprkrd.gz`), ClassiCube, mugen, XBMC4Gamers, Insignia,
Backups, Applications… más los ~340 MB de `valve/` que ya subimos. Lo que falta
por meter son **245 MB** de gearbox.

**No he borrado nada.** Ni tuyo ni mío: en un disco lleno, sobrescribir
`default.xbe` (que pasa de 4,63 a 5,02 MB) puede dejarlo truncado a medias y
entonces el título no arranca. Así que el XBE de Op4 tampoco está desplegado.

### 70.4 Las tres salidas, y la que recomiendo

1. **Mover el título a F:** — recomendada, y no borra nada. El `Config.xml` del
   dashboard escanea `E:\Apps`, **`F:\Apps` y `G:\Apps`** (verificado leyéndolo),
   así que un título en `F:\Apps\xash` sale en el menú igual que ahora. Y el
   motor no se entera: monta `D:` sobre el directorio del XBE
   (`filesystem_engine.c`, `FS_MountXboxRootdir`), así que `valve/` y `gearbox/`
   al lado del XBE siguen funcionando sea cual sea la partición. Coste: subir de
   nuevo `valve/` (~340 MB) y `gearbox/` (245 MB) — unos 585 MB de FTP, sin
   supervisión.
2. **Liberar ~250 MB en E:** — decisión tuya, es tu contenido. De lo mío ahí solo
   hay basura menor que puedo quitar: `default.xbe.off` (4,6 MB, copia
   desactivada de una ronda vieja), el gearbox a medias (6,5 MB) y
   `xash-boot.log`. No llega ni de lejos.
3. **Recortar más gearbox** — hay poco margen honesto. Ya quité 94 MB de mapas
   multijugador (`op4_*`, `op4ctf_*`, `ps2*`: la campaña vive dentro de
   `pak0.pak`, comprobado —56 mapas, `of0a0` y `ofboot0` incluidos—) y toda la
   protección anticopia del CD. Lo que queda es `pak0.pak` 158 MB + `sound/`
   61 MB + `gfx/` 13 MB; quitar el sonido son 61 MB a cambio de jugar Op4 mudo.

### 70.5 Qué está listo y qué NO está verificado

Listo y verificado **en local**:
- XBE de Op4 con `-noconsole` y el overlay opt-in: 5.017.600 B, cadenas
  `-noconsole` y `-traceoverlay` presentes en el binario, símbolos de Op4
  (`CDisplacer`, `CShockrifle`, `CVoltigoreEnergyBall`) intactos, **0 SSE2 en 420
  objetos**.
- `gearbox/` preparado: 680 ficheros, **254.820.634 B (245 MB)**.
- `ofx-xbox-assets-op4.sh` nuevo, con el mismo cuidado que el de valve (URL
  encoding para los nombres con espacios, lotes de 100 ficheros por conexión,
  lista completa antes del bucle para que curl no se coma el `find`, y
  verificación por conteo y tamaño contra la consola).

NO verificado:
- **Nada ha corrido en la consola.** El XBE de Op4 sigue sin desplegarse y Op4 no
  ha ejecutado una sola instrucción.
- `-noconsole` y el overlay opt-in tampoco se han visto en pantalla; el cambio es
  pequeño y está leído, pero no medido.
- Cuánto sitio libre hay exactamente en F: no se puede consultar por FTP: solo
  está comprobado que **acepta escrituras**.

### 70.6 Estado

Motor recompilado y enlazado (Op4 dentro, md5 sin desplegar), `xash.cmd` escrito
pero no subido, assets preparados y no subidos. La consola sigue **exactamente
como estaba**: Half-Life base funcionando, con su XBE y sus assets intactos.

La decisión es tuya y es de una línea: **mover el título a F:** (lo hago yo
entero, son ~585 MB de FTP y te aviso al terminar) o **hacer sitio en E:** y lo
subo donde está.

## 71. Paso 4c — El título se muda a F:, con la referencia dorada intacta en E:

**El bloqueo de §70 se resuelve moviendo, no borrando: el título entero pasa a
`F:\Apps\xash\` — XBE de Op4, `valve/` y `gearbox/` — y `E:\Apps\xash\` se queda
exactamente como está, con el Half-Life que funciona. Dos títulos en el menú del
dashboard, uno de cada, y la referencia dorada a un clic si Op4 da guerra.**

### 71.1 Por qué mudarse sale gratis

Tres cosas verificadas antes de mover un byte:

1. **El dashboard escanea F:.** Su `Config.xml` (leído por FTP) lista
   `E:\Apps`, **`F:\Apps`** y `G:\Apps` entre las rutas de escaneo, junto a
   `Games`, `Juegos`, `Applications`, `Emuladores` y demás. Un título en
   `F:\Apps\xash` aparece en el menú igual que el de E:.
2. **Al motor le da igual la partición.** `FS_MountXboxRootdir`
   (`filesystem_engine.c`) monta **`D:` sobre el directorio del XBE**, y todo el
   filesystem del juego cuelga de ahí. `valve/` y `gearbox/` al lado del
   `default.xbe` resuelven igual estén donde estén. La ruta absoluta no aparece
   en ningún sitio.
3. **F: acepta escrituras** (probado con un fichero de 5 bytes, lo mismo que
   destapó que E: estaba llena).

Coste: volver a subir `valve/` entero, porque FTP no sabe copiar de partición a
partición dentro de la propia consola — todo pasa por la red.

### 71.2 Qué se sube, y el matiz de `valve/`

El `ofx-xbox-assets.sh` de §22 subía el conjunto **mínimo del menú**: sin
`pak0.pak`, porque entonces solo se quería llegar a la pantalla de inicio. El
`pak0.pak` de la consola se subió a mano después. Al replicar el título en otra
partición eso había que arreglarlo o `valve/` habría llegado incompleto y el
fallo habría aparecido tres pantallas más tarde, como carga de mapa fallida.

Ahora el script tiene `OFX_FULL=1`, que añade `pak0.pak` (299 MB) — donde viven
los mapas, modelos, sonidos y wads de Half-Life. `media/` (82 MB de AVIs de
introducción) sigue fuera: el motor no los reproduce aquí.

| | ficheros | bytes |
|---|---|---|
| `valve/` (con `pak0.pak`, sin `media/`) | 263 | 344.171.179 (329 MB) |
| `gearbox/` (sin multijugador ni protección de CD) | 680 | 254.820.634 (245 MB) |
| `default.xbe` (Op4) + `xash.cmd` | 2 | 5.017.661 |
| **total** | **945** | **~574 MB** |

`sound/` de gearbox se queda dentro por decisión explícita: son 61 MB, y las
voces del instructor del Boot Camp son parte de lo que hay que validar. Op4 mudo
no sirve como prueba.

### 71.3 Los dos títulos, y qué es cada uno

| | `E:\Apps\xash` | `F:\Apps\xash` |
|---|---|---|
| nombre en el menú | `Opposing Force X` | **`Opposing Force X - Op4`** |
| XBE | Half-Life base (4.628.480 B) | **Opposing Force** (5.017.600 B) |
| `xash.cmd` | `-noip -dev 2 +sv_cheats 1 +map c1a1` | `-noip -dev 2 -noconsole +sv_cheats 1 -game gearbox +map of0a0` |
| consola en pantalla | sí (notify de `-dev 2`) | **no** (`-noconsole`, §70.1) |
| para qué | referencia dorada: trece secciones de bugs cerrados | el objetivo del proyecto |

El nombre distinto no es cosmética: el dashboard lista los títulos por el
**título del certificado del XBE**, y los dos directorios se llaman `xash`, así
que con el mismo título habrían salido dos entradas idénticas en el menú y la
«referencia dorada a un clic» habría sido una moneda al aire. El de F: se
reenlazó con `--xbe-title 'Opposing Force X - Op4'` (verificado leyendo el
certificado del binario). El de E: no se toca, así que conserva el nombre viejo
— que dice Op4 y ejecuta Half-Life, una herencia de cuando el proyecto solo tenía
un binario. Se arregla en la limpieza, cuando F: esté validado.

E: no se toca hasta que F: esté validado. Los 6,5 MB de gearbox a medias que
quedaron ahí del intento fallido de §70 tampoco: son inertes (ningún `xash.cmd`
de E: los mira) y borrarlos entra en la limpieza posterior, no ahora.

### 71.4 La verificación, y el fichero que llegó mal

El paso de comprobación —contar ficheros y sumar bytes contra la consola— pagó su
precio en el primer intento:

```
local:    680 ficheros    254685466 bytes
remoto:   680 ficheros    255667426 bytes
DISCREPANCIA
< 66616   gfx/env/sky_blu_rt.bmp
> 1048576 gfx/env/sky_blu_rt.bmp
```

**El mismo número de ficheros, casi un mega de más, y un solo culpable**: una
textura de cielo de 66 KB que llegó como **exactamente 1 MiB** (0x100000). No es
un fichero que falte —eso se ve enseguida— sino uno que está y es basura, del
tamaño redondo que delata un búfer escrito entero en vez del contenido. Sin la
suma de bytes habría pasado por bueno y el síntoma habría sido un cielo roto en
Op4, tres sesiones más tarde, con el renderer de sospechoso.

Resubido y verificado por md5 (`92186f45…` en los dos lados), y la comprobación
completa repetida: **680 / 680 ficheros, 254.685.466 bytes exactos a los dos
lados**. `valve/` pasó a la primera: **263 / 263, 344.117.931 bytes**.

Regla que se queda: **contar ficheros no basta; hay que sumar bytes.** Un fichero
corrupto tiene el mismo nombre y ocupa una entrada igual que el bueno.

### 71.5 Estado del despliegue

| en `F:\Apps\xash\` | |
|---|---|
| `default.xbe` | 5.017.600 B, md5 `764268332f82…` round-trip |
| `xash.cmd` | `-noip -dev 2 -noconsole +sv_cheats 1 -game gearbox +map of0a0` |
| `valve/` | 263 ficheros, 344.117.931 B — verificado |
| `gearbox/` | 680 ficheros, 254.685.466 B — verificado |
| `gearbox/pak0.pak` | 165.626.683 B (los 56 mapas de campaña) |
| `valve/pak0.pak` | 313.133.320 B (la base heredada) |

`E:\Apps\xash\` sin tocar: su `default.xbe` de 4.628.480 B, su `xash.cmd` de 35
bytes y su `valve/` intactos.

### 71.6 Qué NO está verificado

- **Op4 no ha ejecutado una sola instrucción.** Todo lo de esta sección es
  transporte y comprobación de bytes.
- El `xash.cmd` de F: apunta a `+map of0a0`, el primer mapa de la campaña según
  el `liblist.gam` de Gearbox. `ofboot0` (el Boot Camp) es la alternativa, y es
  el sitio más barato para oír las voces y ver el HUD de siete ranuras.
- `-noconsole` y el overlay opt-in tampoco se han visto en pantalla todavía.
- No se ha medido cuánto sitio libre queda en F: después de esto; solo que
  aceptó los 574 MB.

## 72. HITO — Opposing Force corre en una Xbox original. Y los dos bugs, con nombre y apellidos

**El Osprey sobrevuela el desierto con los créditos encima, en hardware de 2001,
con el gamedll de Gearbox enlazado estáticamente dentro del XBE. Es la primera
vez que Opposing Force se ejecuta en una Xbox original — el objetivo con el que
empezó el proyecto, 72 secciones atrás. Los dos bugs que trajiste están los dos
diagnosticados hasta el final: el magenta es **culpa mía**, un recorte de assets
que era inofensivo para Half-Life y letal para Op4 por una razón que se puede
medir; y el cuelgue de pbkit está decodificado registro a registro — una textura
sin datos que programa una geometría imposible y el NV2A se planta. Los dos
arreglados y desplegados. Y de propina, una trampa de build que llevaba dos
rondas dándome un binario mentiroso.**

### 72.1 (1) El magenta: 89 de 94 texturas dependían de un WAD que no subí

La cadena, medida de punta a punta:

El `worldspawn` de `of0a0` declara en su clave `wad` **cinco** ficheros:
`liquids.wad`, `xeno.wad`, `halflife.wad` y `decals.wad` de `valve/`, y `opfor.wad`
de `gearbox/`. (No se reproduce aquí el contenido del mapa: viene con la copia
comprada del juego.)

Y el log dice qué cargó de verdad:

```
Adding WAD: D:/valve/cached.wad, decals.wad, fonts.wad, gfx.wad
Adding WAD: D:/gearbox/DECALS.WAD, OPFOR.WAD, cached.wad
```

**Ni `halflife.wad`, ni `xeno.wad`, ni `liquids.wad`** — los tres exactos que el
recorte de §22 dejó fuera. Resultado en el log: **88 líneas `Error: Unable to find
<textura>.mip`**, y en `mod_bmodel.c:2726` la consecuencia:

```c
// If texture is completely missed:
texture->gl_texturenum = R_GetBuiltinTexture( REF_DEFAULT_TEXTURE );
```

`*default` es el cuadriculado magenta. No es un fallo del renderer: es el motor
diciendo «esta textura no existe» de la única forma que sabe.

**Por qué el recorte parecía inofensivo, y aquí está lo que hay que aprender.**
Un BSP de GoldSrc puede llevar sus texturas **embebidas** o solo **referenciadas
por nombre**. Medido con `ofx-bsp-textures.py`, nuevo:

| mapa | embebidas | referenciadas |
|---|---|---|
| `c1a1` (Half-Life) | **116 de 116** | 0 |
| `of0a0` (Opposing Force) | **0 de 94** | **94** |

Valve compiló sus mapas con las texturas dentro; Gearbox no. Por eso Half-Life
llevaba quince secciones jugándose sin un solo wad y nadie lo notó — **el recorte
tenía un modo de fallo invisible hasta que otro mod pidiera los wads**. El reparto
exacto de las 94 de `of0a0`: 82 de `halflife.wad`, 7 de `xeno.wad`, 5 de
`OPFOR.WAD`. Las cinco que se veían bien eran las de Op4; el 94% restante,
magenta. Cuadra al detalle con lo que viste.

Arreglado subiendo `halflife.wad` (37 MB), `xeno.wad` (6,5 MB), `liquids.wad` y
`spraypaint.wad`, con md5 verificado uno a uno. Y arreglado **en el script**, no
solo en la consola: `OFX_FULL=1` los incluye siempre, con el porqué escrito al
lado para que el siguiente que lea la lista no vuelva a pensar que sobran.

### 72.2 (2) El cuelgue: `1b04` es `NV097_SET_TEXTURE_FORMAT`, y el valor es imposible

`0x1B04` está en `nv_regs.h` de nxdk sin ambigüedad: **`NV097_SET_TEXTURE_FORMAT`**
de la unidad 0 (las demás en `+ n*64`). Y el volcado de pbkit dice que
`trapped_address` es **el método que la GPU estaba ejecutando** y `DataLow` **el
valor que se le escribía**. Así que el dato exacto: la GPU se atragantó
programando el formato de una textura con `0x00003a0a`. Decodificado campo a
campo con las máscaras de la cabecera:

| campo | valor | veredicto |
|---|---|---|
| `CONTEXT_DMA` | 2 | correcto (canal B) |
| `BORDER_SOURCE` | 1 = COLOR | correcto, pbgl siempre lo pone |
| `COLOR` | **0x3A** = `SZ_A8B8G8R8` | correcto: es lo que `intfmt_gl_to_nv` devuelve para `GL_RGB`/`GL_RGBA` |
| `DIMENSIONALITY` | **0** | **imposible** (1=1D, 2=2D, 3=3D) |
| `MIPMAP_LEVELS` | **0** | **imposible** (mínimo 1) |
| `BASE_SIZE_U/V/P` | **0** | textura de **1×1** |

Las partes **constantes** están bien y todo lo que se **deriva de la geometría**
está a cero. Eso no es memoria corrupta ni un puntero perdido: es
`tex_gl_to_nv()` corriendo sobre un `texture_t` al que nunca le llegó
`glTexImage2D` — el formato se calcula igual, y sale una textura de una dimensión
inexistente. El NV2A levanta el error y pbkit para la máquina.

Y `flush_texunit` de pbgl **empuja los registros sin comprobar que la textura
sirva**: ni `allocated`, ni dimensiones, ni nada.

Arreglo (`state.c`, no upstream): si la unidad va a programarse con una textura
sin datos, **no se programa** — se desactiva la unidad y se deja una línea en el
log con lo que la delata (alloc, dims, tamaño, mips, formato y dirección). Un
polígono sin textura en vez de una consola muerta, y el culpable con nombre en el
log la próxima vez.

**Sobre la carga de combate**: la correlación es real pero no es la causa. El log
muere justo tras subir `{yblood3`, `{yblood1` y `{yblood5` — decals de sangre
alienígena, es decir, texturas que se suben **por primera vez y bajo demanda** en
mitad del combate. Lo que el combate aporta no es carga, es **texturas nuevas**:
más oportunidades de que una llegue a la GPU a medio construir. Descartado
explícitamente que sea presión de memoria: **cero** líneas `pbgl mem: FAILED` en
todo el log, y el pool de 64 MB de §40 no se quejó ni una vez.

**Sobre `pb_trace_mode=1`**: no hace falta de momento, y conviene saber lo que
cuesta. Es un `static int` de `pbkit.c` bajo `#ifdef DBG`, así que encenderlo es
recompilar el pbkit de nxdk, y hace que pbkit **espere a que la GPU consuma cada
bloque** — mucho más lento. Su utilidad es garantizar que el registro reportado
es el último enviado; aquí el informe ya se sostiene solo, porque `0x3A` es una
constante que solo produce `intfmt_gl_to_nv` y los campos constantes salieron
exactos. Queda como carta a jugar si la guarda nueva no basta.

### 72.3 La trampa: `touch` no fuerza nada bajo waf

Casi cuela otro binario mentiroso, y es la hermana mayor de §66.6. La receta que
escribí allí para forzar el enlace era `rm default.xbe` + `touch` de un fuente. Y
es **falsa**: waf decide por **hash del contenido**, no por fecha. Un `touch`
cambia la mtime y nada más, así que no recompila, no reenlaza, y como el `.xbe` sí
faltaba, se limita a **re-exportarlo del `xash.exe` viejo**. El resultado tiene
fecha nueva, tamaño idéntico y `rc=0`.

Se cazó porque la comprobación de §66.6 —buscar la cadena nueva dentro del
binario— dio **0**, y el testigo que lo explicó fue la fecha de `build/engine/xash.map`:
las 20:29 cuando el XBE decía 21:24. **El mapa de enlace es de la última vez que
el enlazador corrió de verdad.**

La receta buena, y va a la lista: **borrar el binario enlazado**, no el XBE.

```
rm -f build/engine/xash.exe build/engine/xash.map build/engine/default.xbe
```

En §66 funcionó de casualidad: allí el fuente había cambiado de verdad.

### 72.4 Qué NO está verificado

- **Ninguno de los dos arreglos ha corrido.** Los wads están en la consola y la
  guarda en el XBE, pero nadie ha visto todavía ni el escenario con sus texturas
  ni el combate sin cuelgue.
- La guarda **no arregla la causa**: impide que una textura sin datos mate la
  consola y la delata en el log. Quién crea esa textura sigue sin saberse, y es
  la caza de la ronda siguiente — con el nombre que la propia guarda escriba.
- Que el magenta desaparezca del todo depende de que los 94 nombres estén
  cubiertos: medido en local que sí (82+7+5, cero sin cubrir), no en máquina.
- Otros mapas de Op4 pueden pedir wads distintos. El método para comprobarlo es
  `ofx-bsp-textures.py` y el careo del worldspawn, no la intuición.

### 72.5 Estado

`libpbgl.lib` reconstruida con la guarda (parche regenerado, 454 líneas), motor
**reenlazado de verdad** (mapa de enlace nuevo, cadena verificada dentro del XBE),
0 SSE2 en 420 objetos, desplegado en `F:\Apps\xash\` con md5 round-trip
(`4faa998c…`). `valve/` en la consola: **267 ficheros, 389.110.363 bytes,
verificado** — los cuatro wads dentro. `E:\Apps\xash\` sigue intacta con el
Half-Life de referencia.

La partida: arranca Op4 otra vez. Lo que hay que mirar es si el escenario tiene
sus texturas, y luego buscar pelea a propósito. Si el cuelgue vuelve, ya no
matará la consola: dejará su línea en el log diciendo qué textura era.

## 73. Paso 4d — Tres flecos: el cielo que nunca se bindeó, el grafo que pierde una carrera de fechas, y un sonido que sí carga

**Los tres diagnosticados sin necesidad de la consola, dos de ellos hasta la línea
exacta. El cielo lleva mal desde §61 y por fin tiene nombre: el motor da a las seis
caras nombres de textura **fijos** que pbgl rechaza, así que ninguna llega a
bindearse — y de paso explica el `GL_INVALID_OPERATION` que dejamos como deuda
hace doce secciones. El grafo de nodos se reconstruye siempre porque el gamedll
compara la fecha del BSP con la del `.nod`, la del BSP es **la del pak**, y mi
subida alfabética dejó el pak más nuevo: una carrera perdida por orden de letras.
Y el sonido del Osprey **no es un fichero que falte ni un WAV que el motor
rechace** — lo he simulado byte a byte y el parser lo acepta; queda por decidir en
el camino de reproducción, y ahí sí hace falta el log.**

### 73.1 (2) El skybox: nombres fijos contra un array que no llega

`R_SetupSky` carga cada cara con `GL_LoadTexture( ..., TF_CLAMP|TF_SKY )`, y
`TF_SKY` incluye `TF_SKYSIDE` (`ref_api.h:67`). Eso activa el *skyboxhack* de
`GL_AllocTexture` (`gl_image.c:1448`):

```c
if( !skyboxhack ) {
    do { pglGenTextures( 1, &texnum ); }          // nombre normal
    while( texnum >= SKYBOX_BASE_NUM && ... );
}
else texnum = tr.skyboxbasenum;                    // 5800..5805, SIN generar
```

Es un truco de compatibilidad GoldSrc: en GL de sobremesa puedes bindear
cualquier nombre no usado y el objeto se crea solo. **pbgl no funciona así**:

```c
GL_API void glBindTexture(GLenum target, GLuint tex) {
  if (pbgl.imm.active || tex >= tex_cap) {
    pbgl_set_error(GL_INVALID_OPERATION);
    return;                                        // el bind NO ocurre
  }
```

y `tex_cap` empieza en `TEX_ALLOC_STEP` = **256** y **solo crece dentro de
`glGenTextures`**. Este juego sube unas 900 texturas en todo `of0a0`, así que
`tex_cap` jamás se acerca a 5800: **las seis caras fallan al bindear, siempre**.
Los seis `glTexImage2D` posteriores van a parar a lo que hubiera bindeado antes, y
el cielo se dibuja con eso. De ahí los «colores rarísimos».

Y explica la deuda de §61.3, anotada y archivada: `GL_INVALID_OPERATION while
uploading gfx/env/desertrt.bmp`. Era este mismo fallo, en el cielo del desierto de
Half-Life, doce secciones antes. No se investigó porque el cielo de HL «se veía
más o menos» y el foco estaba en los decals.

**Arreglo** (`gl_image.c`, bajo `#if XASH_XBOX`): el hack se apaga y las caras
pasan por `glGenTextures` como cualquier textura. El camino normal ya evita el
rango reservado, así que no hay colisión posible. La alternativa —hacer que pbgl
haga crecer su array en cualquier bind, que es lo que manda el estándar— es más
correcta pero reserva ~1,2 MB de `texture_t` para nombres que nadie más usa.

Descartado por medida que fuera corrupción de ficheros, que era la sospecha
razonable después del `sky_blu_rt.bmp` de §71.4: las seis caras `cliff2*` están en
la consola con el tamaño exacto del origen (66614/66616/65780 B), y la
verificación de `gearbox/` cerró en 680 ficheros y 254.685.466 bytes exactos.

### 73.2 (1) El grafo de nodos: una carrera de fechas que perdí al subir

El gamedll decide si el `.nod` sirve en `CGraph::CheckNODFile` (`nodes.cpp:2737`):

```c
if( COMPARE_FILE_TIME( szBspFilename, szGraphFilename, &iCompare ) )
    if( iCompare > 0 ) { /* el BSP es mas nuevo */ retValue = FALSE; }
```

Y el motor resuelve esas fechas así (`common.c:754` → `FS_FileTime`):

```c
static int FS_FileTime_PAK( searchpath_t *search, const char *filename ) {
    return search->pack->handle->filetime;         // la fecha DEL PAK
}
```

`maps/of0a0.bsp` vive **dentro** de `gearbox/pak0.pak`, así que su «fecha» es la
del pak entero. Y mi script sube los ficheros con `find | sort`, es decir en orden
alfabético: **`maps/graphs/of0a0.nod` entra antes que `pak0.pak`**. El pak queda
más nuevo que el grafo, `iCompare > 0`, y el juego reconstruye el grafo en cada
carga — para siempre, porque las fechas ya están fijadas en disco. No es que falle
al escribir: es que decide, correctamente según su criterio, que el grafo está
caducado.

Lo demás de la cadena está sano y comprobado: el `of0a0.nod` está subido (13.797
B), su versión es **16** = `GRAPH_VERSION_RETAIL`, que `FLoadGraph` acepta
explícitamente, y se carga por `LOAD_FILE_FOR_ME` (el filesystem del motor), no
por `fopen`. El `fopen` solo aparece en el guardado, con `GET_GAME_DIR`, y ese
camino no se ha ejercitado todavía.

**Arreglo**: subir los grafos **los últimos**, para que su fecha gane al pak.
Corregido en `ofx-xbox-assets-op4.sh` (dos `find`, grafos al final) y aplicable a
lo que ya está en la consola resubiendo solo esos 15 ficheros.

Nota: solo 15 de los 56 mapas de la campaña traen grafo de fábrica. Los demás lo
reconstruirán la primera vez y —cuando el guardado funcione— lo cachearán.

### 73.3 (3) El Osprey: ni falta el fichero, ni el motor lo rechaza

Dos hipótesis razonables, las dos **descartadas por medida**:

- **¿Falta el fichero?** No. `ambience/osprey_rotors.wav` está en lo subido. De
  los 25 wav que `of0a0` nombra, los 10 de `intro/` y el del Osprey son de
  gearbox y están todos; el resto vive en los paks.
- **¿Lo rechaza el cargador?** Tampoco. El WAV es distinto de los que sí suenan
  —**22050 Hz y con chunk `cue`, o sea en bucle**, contra 11025 Hz sin bucle— y
  `snd_wav.c` tiene una salida que **descarta el sonido entero** si el bucle no
  cuadra (`"has a bad loop length"`). Parecía la explicación perfecta, así que
  reprodujo el parser del motor byte a byte sobre el fichero real
  (`tmp-wavsim.py`): los chunks son `fmt data cue LIST LIST`, el `LIST` que sigue
  al `cue` **no** lleva la marca `mark` que dispara el cálculo, `sound.samples`
  se queda en 0 y la comprobación no salta. **El parser lo acepta**, con
  `loopstart` 0 y 83.973 muestras.

Luego el sonido se carga y no se oye: el fallo está **después**, en el camino de
reproducción de un `ambient_generic` en bucle. Los candidatos que quedan, en
orden de sospecha, y todos decidibles con el log:

1. **Canales estáticos.** `of0a0` tiene **33 `ambient_generic`**. La dieta del
   fork encoge `MAX_CHANNELS` de 256+60 a **128+20** (`sound.h`), y en la
   auditoría de §68 lo dejé en «vigilar, sin síntoma medido». Ahora hay síntoma.
2. **El bucle en sí**: que el mixer del port no repita un sonido marcado como
   `SOUND_LOOPED` y lo corte en el primer frame.
3. **La cadena de scripts**: que el `ambient_generic` no llegue a dispararse
   (las líneas `Firing:` del log lo dirían).

No he tocado nada por esto: elegir entre las tres con el log cuesta un minuto y
adivinar cuesta una ronda.

### 73.4 Qué NO está verificado

- **Nada ha corrido.** El XBE con el arreglo del cielo está construido y
  verificado en local, no desplegado: la consola llevaba toda la sesión dentro del
  título y su FTP es del dashboard.
- El arreglo del grafo no se ha aplicado a la consola (hay que resubir los 15
  ficheros) ni se ha comprobado que las fechas queden como se espera.
- El bug 3 no tiene causa, solo dos hipótesis eliminadas y tres vivas.
- El camino de **guardado** del grafo (`fopen` con `GET_GAME_DIR`) sigue sin
  ejercitarse. Si al arreglar las fechas los mapas sin grafo de fábrica siguen
  reconstruyendo cada vez, el sospechoso es ése.

### 73.5 Estado

Motor reenlazado de verdad (mapa de enlace y XBE con la misma marca de tiempo,
`gl_image.o` recompilado), con el arreglo del cielo dentro y todo lo de §72 en su
sitio (`SIN DATOS`, `-noconsole`, `-traceoverlay`, símbolos de Op4), 0 SSE2 en 420
objetos. `ofx-xbox-assets-op4.sh` corregido para subir los grafos al final.
Pendiente de que la consola vuelva al dashboard: desplegar el XBE, resubir los 15
grafos y bajar el log para cerrar el bug 3.

### 73.6 El log: confirma uno, corrige otro y añade una causa que faltaba

La consola volvió al dashboard y el log (44.964 líneas) reordena las conclusiones.

**El grafo de nodos: mi diagnóstico confirmado por el propio motor, y una SEGUNDA
causa encima.** El mensaje exacto de `CheckNODFile` está ahí:

```
[con 33849] .NOD File will be updated
```

Esa línea es la rama `iCompare > 0` — «el BSP es más nuevo» —, es decir, la
carrera de fechas de 73.2, escrita por el juego. Pero justo después:

```
[con 48374] Couldn't create gearbox/maps/graphs/of0a0.nrp!
[con 403415] Graph not ready!
```

**`gearbox/maps/graphs/...` es una ruta RELATIVA.** `FSaveGraph` la compone con
`GET_GAME_DIR`, y este motor devuelve por defecto solo el nombre de la carpeta
(`sv_game.c:4727`, `Q_strncpy( out, GI->gamefolder, 256 )`). En una consola sin
directorio de trabajo, `fopen` de una ruta relativa no puede crear nada. Así que
había **dos fallos apilados**: uno decide reconstruir el grafo, el otro impide
guardarlo. Arreglar solo el primero habría servido para los 15 mapas que traen
grafo de fábrica y habría dejado a los otros 41 reconstruyendo eternamente.

Y el segundo no necesita tocar código: el motor ya trae la bandera
`BUGCOMP_GET_GAME_DIR_FULL_PATH` (`host.c:110`, argumento `get_game_dir_full`),
que hace que `GET_GAME_DIR` devuelva la ruta completa. Con `rootdir=D:\` medido en
el log y el `COM_StripDirectorySlash` que el filesystem aplica, la ruta sale
limpia: **`D:/gearbox`**. Añadido a `xash.cmd`.

**El cielo: el mecanismo se confirma, pero el log obliga a matizar.** La línea
que faltaba:

```
[con 45003] SKY:  upload: gfx/env/cliff2rt.bmp ... uploaded ok, 64 KB
failed
```

`R_SetupSky` imprime el nombre de cada cara **después** de cargarla y solo si
`GL_LoadTexture` devolvió algo; aquí no imprime ninguna y salta directo a
`failed`. O sea: la primera cara **se subió** (64 KB, «ok») y aun así la función
devolvió 0, el bucle rompió en `i == 0` y el mapa se quedó sin cielo — dibujando
lo que hubiera.

El matiz: más tarde, en otro mapa, el cielo `desert` de Half-Life **sí carga sus
seis caras** (`desertrt, desertbk, desertlf, desertft, desertup…`). Así que el
camino especial del skybox no falla siempre, y mi explicación de 73.1 —nombres
5800-5805 contra el `tex_cap` de pbgl— no puede ser la historia completa: hay una
segunda variable (el `MAX_TEXTURES` de 2048 del fork hace que
`GL_AllocTexture` caiga en su rama de «buscar hueco libre» para cualquier nombre
≥ 2048, y `tr.skyboxbasenum` va incrementándose entre intentos). El arreglo
desplegado sigue siendo el correcto por la razón de siempre: **quita el caso
especial entero** y deja que las seis caras se asignen como cualquier otra
textura, que es el camino que lleva 900 texturas funcionando. Pero se anota como
hipótesis con una parte sin explicar, no como causa cerrada.

**El sonido del Osprey: la cadena de scripts, descartada — y aparece un
sospechoso con números.** El `ambient_generic` **sí se dispara**:

```
[con 49449] Firing: (rotors)
[con 49451] Found: ambient_generic, firing (rotors)
```

Son dos entidades `rotors` (3344 −1864 664 y 3344 −2008 704) y las dos reciben su
trigger. Ni el fichero, ni el parser del WAV, ni la cadena de scripts: los tres
descartados por medida. Lo que queda, y ahora con aritmética, es la **envolvente
de volumen**. Los datos de la entidad son `volstart 0`, `fadein 100`, `health 6`,
y la conversión del gamedll (`sound.cpp:474`) es:

```c
m_dpv.fadein = ( 101 - m_dpv.fadein ) * 64;    // 100 -> 64, el valor MAS LENTO
m_dpv.volrun *= 10;                             // health 6 -> 60
```

`RampThink` suma `fadein` a `volfrac` cinco veces por segundo y el volumen es
`volfrac >> 8`. Para llegar a 60 hacen falta `60*256 = 15.360` de `volfrac`, es
decir **240 thinks ≈ 48 segundos** desde silencio absoluto. El sonido arranca a
volumen 0 (spawnflag `START_SILENT` + `volstart 0`) y sube durante casi un minuto.

Eso es comportamiento **vanilla**, no del port — pero cambia la pregunta: puede
que no haya bug, sino un sonido que tarda 48 s en ser audible y una intro en la
que no da tiempo. La prueba es gratis y la puede hacer el operador: quedarse
quieto cerca del Osprey un minuto largo. Si empieza a oírse, no hay nada que
arreglar; si sigue mudo pasado ese tiempo, el siguiente sospechoso son los
canales estáticos (`MAX_CHANNELS` 128+20 contra 256+60, la deuda que §68 dejó en
vigilancia y que `of0a0` estresa con 33 `ambient_generic`).

### 73.7 Estado tras el log

Desplegado en `F:\Apps\xash\`: XBE con el arreglo del cielo (md5 `d2aeb78e…`),
`xash.cmd` = `-noip -dev 2 -noconsole -bugcomp get_game_dir_full +sv_cheats 1
-game gearbox +map of0a0` (88 B, md5 verificado). Los 15 ficheros de grafos
resubidos: ahora marcan **04:39-04:40** contra las **02:58** del `pak0.pak`, así
que la comparación de fechas ya cae del lado bueno; `of0a0.nod` verificado por
md5. `ofx-xbox-assets-op4.sh` corregido para que los grafos suban siempre los
últimos.

La partida siguiente responde tres cosas de una vez: si el cielo sale bien, si
desaparece el «Node graph out of date» (y si el `.nod` se guarda ya, mirando que
no vuelva `Couldn't create`), y si el Osprey se oye esperando un minuto.
