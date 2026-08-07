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
