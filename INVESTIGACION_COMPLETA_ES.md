# Investigación completa en español: Cubot KingKong ES3 / Unisoc T615

> Documento consolidado y traducido de la investigación técnica contenida en este repositorio.
>
> **Estado del documento:** 16 de julio de 2026  
> **Dispositivo analizado:** Cubot KingKong ES3  
> **Build específico:** `CUBOT_KINGKONG_ES_3_F071_V16_20260309`
>
> Este documento no convierte la investigación en una receta universal. Describe un solo dispositivo, una sola compilación de firmware y el estado real observado durante las pruebas.

## 1. Resumen ejecutivo

El Cubot KingKong ES3 analizado está rooteado con Magisk, pero continúa usando el kernel original de fábrica.

La evidencia final demuestra lo siguiente:

| Componente | Estado observado | Coincide con stock | Función en el root |
|---|---|---:|---|
| `boot_a` | Imagen stock `boot-gki.img` | Sí | Contiene el kernel stock; no lleva Magisk |
| `init_boot_a` | Modificado por Magisk | No | Es el portador principal del root |
| `vendor_boot_a` | Imagen stock `vendor_boot.img` | Sí | Conserva DTB, bootconfig, fstab, ramdisk y módulos del fabricante |
| `dtbo_a` | Imagen stock `dtbo.img` | Sí | Conserva los overlays originales |
| `vbmeta_a` | Modificación de un solo byte en flags | No | Mantiene los descriptores stock, pero su firma deja de validar |
| LK | Bootloader desbloqueado, estado naranja | No aplica | Tolera o omite la verificación AVB |

La cadena de arranque real queda así:

```text
LK desbloqueado / verified boot naranja
        |
        v
vbmeta_a con flags modificados
pero con descriptores stock
        |
        v
boot_a stock con kernel stock
        |
        +--> vendor_boot_a stock
        |
        +--> dtbo_a stock
        |
        v
init_boot_a parcheado por Magisk
        |
        v
Root de Magisk
```

La conclusión más importante es sencilla:

> **El teléfono está rooteado, pero el kernel personalizado experimental no está activo.**

Rootear el dispositivo y lograr que un kernel personalizado arranque son dos problemas distintos. La evidencia recopilada permite analizarlos por separado.

---

## 2. Identidad del dispositivo y entorno analizado

| Campo | Valor |
|---|---|
| Dispositivo | Cubot KingKong ES3 |
| SoC | Unisoc T615, familia UMS9230 |
| Android | Android 13 |
| Build | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Kernel en ejecución | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Slot activo analizado | `_a` |
| Almacenamiento | UFS |
| Evidencia UFS | `sprdboot.flash=ufs` |
| Root | Magisk |

### Mapa confirmado de particiones del slot A

| Partición | Dispositivo de bloque |
|---|---|
| `boot_a` | `/dev/block/sda36` |
| `vendor_boot_a` | `/dev/block/sda38` |
| `init_boot_a` | `/dev/block/sda40` |
| `dtbo_a` | `/dev/block/sda44` |
| `vbmeta_a` | `/dev/block/sda49` |

El uso de `/dev/block/sda*`, `/dev/block/sdb*` y `/dev/block/sdc*` es coherente con almacenamiento UFS. Este teléfono no debe tratarse como un dispositivo eMMC.

---

## 3. Alcance y metodología de la investigación

La investigación se dividió en varias fases:

1. Extracción del firmware PAC correspondiente al modelo y build.
2. Acceso BROM/FDL mediante herramientas Spreadtrum/Unisoc.
3. Desbloqueo del bootloader.
4. Obtención de dumps del slot A activo.
5. Comparaciones controladas de imágenes stock, live y candidatas.
6. Identificación exacta del componente que porta Magisk.

Las comparaciones se realizaron con hashes SHA-256, `cmp`, desempaquetado de imágenes Android, análisis AVB, extracción de ramdisks, análisis DTB/DTBO y comparación de módulos.

No se requirió modificar las imágenes originales para realizar las comparaciones.

---

## 4. Herramientas y archivos utilizados

### Herramientas principales

- `spd_dump`
- herramientas de `spreadtrum_flash`
- `adb`
- Magisk
- `unpack_bootimg`
- `avbtool`
- `sha256sum`
- `cmp`
- `file`
- `lz4cat`
- `cpio`
- `dtc`
- toolchain Android Clang

### Archivos esperados del PAC stock

```text
fdl1-dl.bin
fdl2-dl.bin
init_boot.img
vbmeta-sign.img
boot-gki.img
vendor_boot.img
dtbo.img
```

### Archivos utilizados durante el flujo de desbloqueo

```text
custom_exec_no_verify_65015f08.bin
splloader.bin
misc-wipe.bin
spd_dump
```

### Respaldos recomendados

```text
boot_a.img
vendor_boot_a.img
init_boot_a.img
dtbo_a.img
vbmeta_a.img
splloader_original.bin
```

Este repositorio no almacena imágenes PAC, dumps crudos, binarios propietarios ni imágenes parcheadas.

---

## 5. Acceso BROM y FDL2

### Entrada a BROM

1. Apagar completamente el teléfono.
2. Mantener presionado Volumen Abajo.
3. Conectar USB sin soltar el botón.
4. Confirmar que el host detecte el dispositivo Spreadtrum/Unisoc.

Comprobación típica:

```sh
lsusb | grep -i '1782:4d00\|spreadtrum\|unisoc'
```

### Compilación de `spd_dump`

```sh
git clone https://github.com/TomKing062/spreadtrum_flash.git
cd spreadtrum_flash
make

mkdir -p ~/cubot_unlock
cp spd_dump ~/cubot_unlock/
chmod +x ~/cubot_unlock/spd_dump
```

Linux fue el entorno utilizado para BROM/FDL. El comportamiento USB/libusb de macOS no fue considerado suficientemente confiable para este flujo.

### Comando de acceso FDL2 que funcionó

El camino verificado utiliza `exec_addr`, no `loadexec`:

```sh
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

`exec_addr 0x65015f08` es el punto de entrada del exploit CVE-2022-38694 — es lo que logra que BROM acepte y ejecute los payloads FDL1/FDL2 sin firmar que vienen después. Una vez cargados FDL1 y FDL2, la herramienta tiene acceso de bajo nivel a lectura y escritura de particiones. Una escritura equivocada puede impedir el arranque y requerir recuperación mediante firmware stock.

---

## 6. Desbloqueo del bootloader

El flujo verificado no escribe `splloader` directamente. Funciona neutralizando la verificación de bloqueo en una copia del splloader, y dejando que BROM ejecute esa copia neutralizada — el simple hecho de correrla es lo que voltea la bandera de desbloqueo en `miscdata`. No existe un paso de "restauración" separado para esa bandera.

Resumen de la secuencia verificada (ver `README_ES.md`, Pasos 1–5, para los comandos exactos):

1. Extraer el `splloader` y `uboot` originales del teléfono (`r splloader`, `r uboot`) y respaldarlos.
2. Generar `spl-unlock.bin` a partir del `splloader.bin` extraído usando `gen_spl-unlock` — es el splloader original con la verificación de bloqueo eliminada.
3. Reemplazar temporalmente la partición `uboot` con `fdl2-cboot.bin`, una copia cooperativa de FDL2 que ejecutará lo que se cargue después (el `uboot` stock se negaría).
4. Cargar y ejecutar `spl-unlock.bin` a través de BROM (`fdl spl-unlock.bin`). Ejecutarlo — no escribirlo en ninguna partición — es lo que voltea la bandera de desbloqueo en `miscdata`.
5. Verificar la bandera directamente leyendo `miscdata` (`read_part miscdata`).
6. Restaurar los respaldos originales de `splloader` y `uboot`, y luego escribir `misc-wipe.bin` para forzar el restablecimiento de fábrica que requiere el desbloqueo.

Después del reinicio se observó:

- Restablecimiento de fábrica.
- Estado de bootloader desbloqueado.
- Estado de verified boot naranja.
- Advertencia del bootloader durante el arranque.
- Tolerancia u omisión de la verificación AVB.

### Evidencia encontrada en LK

```text
INFO: LOCK FLAG IS : UNLOCK!!!
WARNNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY!!!
bootloader has been unlocked
androidboot.verifiedbootstate=orange
read_is_device_unlocked()
device unlocked so not initializing digest
```

La advertencia visible proviene de la etapa LK del bootloader. No proviene de Android, Magisk, `init_boot` ni del kernel Linux.

---

## 7. Modelo correcto de root con Magisk

Para este build Android 13/GKI, Magisk debe aplicarse a `init_boot.img`, no a `boot.img`.

### Flujo confirmado

1. Extraer `init_boot.img` del PAC exacto.
2. Transferirlo al teléfono.
3. Parchearlo mediante la aplicación Magisk.
4. Copiar la imagen parcheada de regreso al host.
5. Escribirla en `init_boot_a`.
6. Reiniciar y verificar root.

Transferencia al teléfono:

```sh
adb push init_boot.img /sdcard/Download/
```

En Magisk:

```text
Install
Select and Patch a File
Seleccionar init_boot.img
```

Recuperar la imagen parcheada:

```sh
adb pull /sdcard/Download/magisk_patched-*.img .
```

Desde `FDL2>`:

```text
w init_boot_a magisk_patched-XXXXX.img
```

Debe utilizarse el nombre real generado por Magisk.

### Regla crítica

```text
NO escribir la imagen parcheada de Magisk en boot_a
para este build específico.
```

La evidencia del dispositivo rooteado muestra:

- `boot_a` stock.
- `init_boot_a` parcheado.
- `vendor_boot_a` stock.
- `dtbo_a` stock.

---

## 8. Comparación de `boot_a`

### Resultado principal

El `boot_a` activo coincide byte por byte con `boot-gki.img` del PAC stock.

| Campo | Stock PAC | Live slot A | Candidato custom | Custom + Magisk |
|---|---|---|---|---|
| Tamaño bruto | 67,108,864 | 67,108,864 | 34,181,120 | 34,545,664 |
| SHA-256 bruto | `3113ec9523f41a8a667c594c8590731876233b82d80c9c13a551f697f240f1da` | `3113ec9523f41a8a667c594c8590731876233b82d80c9c13a551f697f240f1da` | `de91093263b7c229c8efea1cb4fa7d1dc1ed5bdfd3238f05aeb8fd944bfd91af` | `7670cfb93a8b53e78aa2af4e99994500312bc57b28ddf69f6f2b2f0e4424e8a2` |
| Header | v4 | v4 | v4 | v4 |
| Kernel | 47,442,432 bytes | 47,442,432 bytes | 34,175,488 bytes | 34,175,488 bytes |
| Ramdisk | Vacío | Vacío | Vacío | 360,840 bytes |
| Footer AVB | Presente | Presente | No detectado | No detectado |

Hashes del kernel desempaquetado:

```text
Stock/live:
8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532

Custom/custom+Magisk:
773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a
```

### Conclusión

- El kernel activo es el kernel stock.
- `boot_a` no porta Magisk.
- El candidato custom no está activo en el slot A analizado.
- El candidato custom+Magisk tampoco está activo.
- El éxito del root no demuestra que el kernel personalizado haya arrancado.

---

## 9. Comparación de `init_boot_a`

### Resultado principal

`init_boot_a` no coincide con el `init_boot.img` stock y contiene evidencia directa de Magisk.

| Campo | Stock | Live |
|---|---:|---:|
| Tamaño bruto | 8,388,608 bytes | 8,388,608 bytes |
| SHA-256 bruto | `7d57db8a3eb7e7544bdb18a313eec6dbe03d09a2e19dd962cc56f620904e5a92` | `4a6563154596ab31558ffbda94523c251ab2292567c1384fcffd74f9c02362b6` |
| Header | v4 | v4 |
| Kernel | Vacío | Vacío |
| Tamaño ramdisk | 2,606,941 bytes | 2,316,314 bytes |
| SHA-256 ramdisk | `793a92bb18dd886bef7d87812ccbb84234a8eaeaea2443ca28621b4f685d9184` | `098834ff153c5ddb53a996e9889b6f92dc19c83dbb0c28355fef8194ae47f878` |
| Entradas cpio | 21 | 29 |

Archivos exclusivos encontrados en el ramdisk live:

```text
.backup/.magisk
.backup/.rmlist
.backup/init.xz
overlay.d/sbin/init-ld.xz
overlay.d/sbin/magisk.xz
overlay.d/sbin/stub.xz
```

También se encontraron cadenas como:

```text
/.magisk
/data/magisk
${MAGISKTMP}
overlay.d
```

### Conclusión

`init_boot_a` es el portador confirmado de Magisk y del root.

---

## 10. Comparación de `vendor_boot_a`

### Resultado principal

`vendor_boot_a` coincide byte por byte con el `vendor_boot.img` stock.

| Campo | Stock y live |
|---|---|
| Tamaño bruto | 104,857,600 bytes |
| SHA-256 bruto | `0df796cca5a5a80b1d93caeca17acbcd075262ab3e2ff73bbe6e99fa075efadf` |
| Header | v4 |
| Page size | `0x1000` |
| Bootconfig | `androidboot.hardware=ums9230_6h10` |
| DTB | 132,833 bytes |
| Ramdisk vendor | 33,809,764 bytes |
| Fragmentos | `vendor_ramdisk00`, tipo `0x1` |
| Archivos regulares extraídos | 838 |
| Módulos | 157 |

Hashes normalizados:

```text
DTB:
65dfe08c027d2c502e5a1fc3c8e2a76f22dd6e9ac8b63362f08ded9b7b929776

bootconfig:
d5179fc1f67658a6b024ef84f5d847e83b9d33b644b7dc8cd61c290803faf166

vendor ramdisk:
3f40b369306e77b5c4223a8cdfa0fd65038bcac471273d6ce57be0db5d94f496

fstab.ums9230_6h10:
efef71e3305a94fff71bde0f70725051e5ab8cc5c17e048beecbe89ba68fdc44
```

### Módulos críticos verificados

```text
ufs_sprd.ko
sc2730-regulator.ko
sprd-pmic-spi.ko
printk_cpuid.ko
rpmb.ko
regmap-hook.ko
```

Todos coincidieron con stock.

Vermagic observado en esos módulos:

```text
5.15.178-android13-8-g5b7f481f7f44-dirty SMP preempt mod_unload modversions aarch64
```

### Evidencia UFS y reguladores en el DTB

```text
ufs@20200000
/soc/ap-apb/ufs@20200000
sprd,sc2730-regulator
vdd-vddgen0-supply
vdd-avdd12-supply
vdd-avdd18-supply
vdd-vddcore-supply
vdd-vddmodem-supply
```

Etiquetas de dispositivo UFS encontradas:

```text
/dev/block/sda[0-9]*
/dev/block/sdb[0-9]*
/dev/block/sdc[0-9]*
```

### Conclusión

El root no modificó `vendor_boot_a`. Este componente continúa siendo crítico para cualquier intento de kernel personalizado porque contiene:

- El DTB base.
- El bootconfig.
- El fstab de first stage.
- Los módulos del fabricante.
- Dependencias UFS, RPMB, PMIC y reguladores.

---

## 11. Comparación de `dtbo_a`

### Resultado principal

`dtbo_a` coincide byte por byte con el `dtbo.img` stock.

| Campo | Valor |
|---|---|
| Tamaño bruto | 8,388,608 bytes |
| SHA-256 | `f16238c3ddc31dc5015d42523d29e5fa74ca2b3b43df6fbaed9aad11c18f5672` |
| Magic | `0xd7b7ab1e` |
| Tamaño DTBO real | 1,462,504 bytes |
| Entradas | 13 |
| Page size | 2,048 |
| Versión | 0 |
| Footer AVB | Presente |

Se verificó:

- Igualdad del encabezado y tabla DTBO.
- Igualdad de las 13 entradas extraídas.
- Igualdad de los 13 archivos DTS decompilados.
- Igualdad de todos los conteos de cadenas UFS y reguladores.

Referencias encontradas:

```text
20200000.ufs
regulator-name = "vddvbus"
regulator-always-on
avdd12-supply
vddwcn
pmic_regulator
```

Rutas UFS para módem y GNSS:

```text
/dev/block/platform/soc/soc:ap-apb/20200000.ufs/by-name/wcnmodem
/dev/block/platform/soc/soc:ap-apb/20200000.ufs/by-name/gnssmodem
```

Las cadenas `sc2730`, `vddgen0`, `avdd18`, `vddcore` y `vddmodem` no aparecieron directamente en los overlays decompilados. Esas dependencias sí aparecen en el DTB base de `vendor_boot`.

### Conclusión

El root no modificó `dtbo_a`. Los overlays confirman que un kernel personalizado debe respetar la topología UFS y las dependencias de reguladores esperadas por el hardware.

---

## 12. Comparación de `vbmeta_a`

### Resultado principal

`vbmeta_a` no coincide byte por byte con stock, pero solamente difiere en un byte.

| Campo | Stock | Live |
|---|---|---|
| Tamaño | 1,048,576 bytes | 1,048,576 bytes |
| SHA-256 | `a30d4d8bc891320613f973153bf226fb1e580d9943fa1bc65025282584fbb6b5` | `6935f498ac76f8f175e64f3d51fa6db8aa398cdf2ac1d5b0a190ad2dbca74715` |
| Algoritmo | `SHA256_RSA4096` | `SHA256_RSA4096` |
| Public key SHA-1 | `7b595403a7796dc3c96d57d79583a21721893d20` | Igual |
| Rollback index | 0 | 0 |
| Flags | `0` | `33554432` |
| Release | `avbtool 1.3.0` | Igual |
| Firma | Válida | Falla |

Diferencia exacta:

```text
Byte 121:
stock = 0x00
live  = 0x02
```

Hexdump:

```diff
-00000070: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
+00000070: 00 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00
```

Los descriptores y claves de estas particiones coinciden con stock:

```text
boot
init_boot
vendor_boot
dtbo
vbmeta_system
vbmeta_vendor
vbmeta_product
vbmeta_odm
vbmeta_system_ext
l_modem
l_ldsp
l_gdsp
pm_sys
l_agdsp
```

### Interpretación

- No se reconstruyó un mapa AVB completamente distinto.
- Se modificó el campo de flags firmado.
- Esa modificación invalida la firma de `vbmeta_a`.
- LK está desbloqueado y reporta que omite la verificación.
- Por eso puede arrancar el `init_boot_a` modificado.

### Conclusión

El modelo más preciso es:

```text
vbmeta_a mínimamente modificado
+ descriptores stock
+ firma no válida
+ LK desbloqueado que omite verificación
```

---

## 13. Estado comprobado de la cadena de arranque

### Hechos comprobados

- `boot_a` es stock.
- El kernel activo es stock.
- `init_boot_a` está parcheado por Magisk.
- `vendor_boot_a` es stock.
- `dtbo_a` es stock.
- `vbmeta_a` difiere en un byte.
- Los descriptores AVB coinciden con stock.
- La firma de `vbmeta_a` live falla.
- LK está desbloqueado y en estado naranja.

---


## 14. Política del repositorio

Este repositorio sí debe almacenar:

- Documentación.
- Hashes.
- Logs sanitizados.
- Notas de compilación.
- Informes de dependencias.
- Hallazgos de seguridad y estabilidad.
- Procedimientos reproducibles sin material propietario.

Este repositorio no debe almacenar:

- PAC completos.
- Dumps `.img`.
- Binarios `.bin` propietarios.
- Árboles vendor extraídos.
- Imágenes Magisk parcheadas.
- Módulos `.ko` compilados.
- Firmware blobs.
- Claves privadas.
- Datos personales del dispositivo.

---

## 15. Estado actual del proyecto

### Completado

- Desbloqueo del bootloader.
- Root con Magisk desde `init_boot_a`.
- Identificación completa de la cadena del slot A.
- Confirmación de kernel stock activo.
- Comparación de `boot_a`.
- Comparación de `init_boot_a`.
- Comparación de `vendor_boot_a`.
- Comparación de `dtbo_a`.
- Comparación de `vbmeta_a`.
- Identificación de la advertencia LK.

---

## 16. Conclusión final

El Cubot KingKong ES3 analizado tiene un root funcional y reproducible basado en:

```text
bootloader LK desbloqueado
+ vbmeta_a mínimamente modificado
+ boot_a stock
+ vendor_boot_a stock
+ dtbo_a stock
+ init_boot_a parcheado por Magisk
```

---

## 17. Informes originales en inglés

Este documento consolida y traduce hallazgos de:

- [`README.md`](README.md)
- [`ROOTING_GUIDE.md`](ROOTING_GUIDE.md)

Los reportes técnicos detallados por componente (comparaciones stock-vs-live, hashes, y demás evidencia cruda) se conservan en el archivo interno del proyecto, no en este repositorio público.
