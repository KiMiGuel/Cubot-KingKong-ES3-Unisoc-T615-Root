# Investigación completa en español: Cubot KingKong ES3 / Unisoc T615

> Documento consolidado y traducido de la investigación técnica contenida en este repositorio.
>
> **Estado del documento:** 16 de julio de 2026  
> **Dispositivo analizado:** Cubot KingKong ES3  
> **Build específico:** `CUBOT_KINGKONG_ES_3_F071_V16_20260309`
>
> Este documento no convierte la investigación en una receta universal. Describe un solo dispositivo, una sola compilación de firmware y el estado real observado durante las pruebas.

## 1. Resumen ejecutivo

El Cubot KingKong ES3 analizado está rooteado con Magisk y puede ejecutar NetHunter Lite, pero continúa usando el kernel original de fábrica.

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
Root de Magisk + NetHunter Lite
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
| NetHunter | NetHunter Lite / Kali chroot |

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
7. Investigación del fallo del kernel personalizado.
8. Reconstrucción de un entorno externo de compilación de módulos.
9. Prueba de Wi-Fi USB para NetHunter con AR9271.
10. Definición de los siguientes pasos para Realtek, MediaTek y una pila inalámbrica coherente.

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

El camino funcional utilizó `loadexec`:

```sh
sudo ./spd_dump --verbose 2 --wait 300 \
  loadexec custom_exec_no_verify_65015f08.bin \
  fdl fdl1-dl.bin 0x65000800 \
  fdl fdl2-dl.bin 0x9efffe00 \
  exec
```

Resultado esperado:

```text
FDL2 >
```

Desde `FDL2>` existe acceso de bajo nivel a lectura y escritura de particiones. Una escritura equivocada puede impedir el arranque y requerir recuperación mediante firmware stock.

---

## 6. Desbloqueo del bootloader

El flujo funcional de desbloqueo escribió:

```text
w splloader splloader.bin
w misc misc-wipe.bin
```

`misc-wipe.bin` activa el borrado requerido por el proceso de desbloqueo.

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
- NetHunter Lite funciona sobre este estado rooteado.

### Lo que no está comprobado

- Que el kernel personalizado llegue a iniciar Android.
- Que el slot B haya sido utilizado correctamente durante todos los intentos.
- Que el fallo del kernel tenga una única causa.
- Que todos los adaptadores Wi-Fi compatibles con NetHunter funcionen.
- Que el cargador automático de módulos AR9271 sea seguro.
- Que la pila inalámbrica híbrida sea estable para uso continuo.

---

## 14. Investigación del kernel personalizado

Un candidato de kernel personalizado fue construido y probado, pero el teléfono terminó regresando al estado stock o falló durante el arranque.

El error observado fue:

```text
Attempted to kill init! exitcode=0x00007f00
```

### Teoría de fallo con mayor respaldo

```text
kernel personalizado
        |
        v
vendor_boot stock con módulos, fstab y DTB stock
        |
        v
dtbo stock
        |
        v
incompatibilidad ABI o dependencia UFS/reguladores
        |
        v
fallo de UFS o ausencia de /dev/block/sda*
        |
        v
fallo de first-stage mount
        |
        v
init termina
        |
        v
Attempted to kill init! exitcode=0x00007f00
```

### Riesgos técnicos específicos

- `CONFIG_SCSI_UFS_SPRD=y` integra el controlador UFS en el kernel.
- Si `CONFIG_REGULATOR_SC2730` no está disponible, el UFS integrado puede no obtener sus fuentes de alimentación.
- El nodo UFS depende de rieles como:
  - `vcc`
  - `vddgen0`
  - `avdd12`
  - `avdd18`
  - `vddcore`
  - `vddmodem`
- Los módulos stock usan versionado de símbolos.
- `CONFIG_MODVERSIONS=y` hace relevantes los CRC de símbolos.
- Una diferencia en `module_layout` puede impedir que carguen módulos críticos.
- El fallo anterior alrededor de `printk_cpuid.ko` es compatible con una discrepancia de CRC.
- `ufs_sprd.ko`, `rpmb.ko`, `sprd-pmic-spi.ko` y `sc2730-regulator.ko` forman parte del camino crítico.

### Nivel de certeza

Esta cadena es una teoría técnica fuertemente respaldada por la evidencia de `vendor_boot`, DTB, DTBO y módulos. No es todavía una traza runtime completa que pruebe una única causa.

---

## 15. Reconstrucción del entorno ABI para módulos externos

La fase de headers y módulos dejó de ser teórica. Se reconstruyó un entorno que produjo módulos capaces de cargar en el kernel stock real.

### Entorno confirmado

| Elemento | Valor |
|---|---|
| Kernel objetivo | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Commit common-kernel | `4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7` |
| Toolchain | Android Clang 14.0.7 |
| Clang | `clang-r450784e` |
| Config base | `/proc/config.gz` live |
| `CONFIG_MODULES` | Activado |
| `CONFIG_MODVERSIONS` | Activado |
| `Module.symvers` | Coincidente |
| CRC de `module_layout` | `0x0222dd63` |

### Lo que esto demuestra

- Kali 3 puede compilar módulos externos para el Cubot.
- Los módulos pueden pasar la validación de versión y símbolos.
- El kernel stock puede cargar módulos externos.
- No es obligatorio reemplazar todo el kernel para añadir determinadas funciones.
- La dependencia completa de cada driver sigue siendo crítica.

---

## 16. NetHunter y Netgear WNA1100 / AR9271

Se logró el primer soporte Wi-Fi USB confirmado sobre el kernel stock rooteado.

### Hardware probado

| Campo | Valor |
|---|---|
| Adaptador | Netgear WNA1100 |
| Chipset | Qualcomm Atheros AR9271 |
| USB ID | `0846:9030` |
| Driver | `ath9k_htc` |

### Pila de módulos utilizada

```text
mac80211.ko
ath.ko
ath9k_hw.ko
ath9k_common.ko
ath9k_htc.ko
```

Orden manual de carga:

```text
mac80211
ath
ath9k_hw
ath9k_common
ath9k_htc
```

Firmware utilizado:

```text
htc_9271.fw
ath9k_htc/htc_9271-1.4.0.fw
```

### Arquitectura híbrida utilizada

```text
cfg80211 stock del Cubot
sprd_wlan_combo stock del Cubot
mac80211 externo
ath/ath9k externo
```

No debe cargarse un `cfg80211.ko` externo encima del `cfg80211` stock. El Wi-Fi interno Unisoc depende de la combinación stock `cfg80211` + `sprd_wlan_combo`.

### Resultado validado

- El USB AR9271 fue detectado.
- El firmware cargó.
- Apareció `wlan1`.
- El modo monitor funcionó.
- La inyección de paquetes funcionó en laboratorio autorizado.
- Wifite pudo utilizar el adaptador.

Comandos de prueba:

```sh
ip link set wlan1 down
iw dev wlan1 set type monitor
ip link set wlan1 up
iw dev wlan1 info
```

Estado observado:

```text
Interface wlan1
        type monitor
        channel 1 (2412 MHz), width: 20 MHz
        txpower 20.00 dBm
```

### Lo que este logro demuestra

- El Cubot puede usar Wi-Fi USB externo con NetHunter Lite.
- La ABI de módulos fue alineada correctamente.
- AR9271 puede ofrecer modo monitor e inyección en este build.
- El resultado se obtuvo sin arrancar un kernel personalizado.

### Lo que no demuestra

- Que el módulo Magisk automático sea seguro.
- Que la pila híbrida sea estable indefinidamente.
- Que otros chipsets funcionen sin trabajo adicional.
- Que Realtek o MediaTek estén resueltos.
- Que el kernel personalizado funcione.

---

## 17. Riesgos del cargador automático AR9271

La carga manual funcionó, pero experimentos posteriores con empaquetado/carga automática se asociaron con inestabilidad del touchscreen o de entrada.

Por seguridad técnica:

- Mantener deshabilitado el módulo Magisk experimental.
- No cargar la pila automáticamente al arrancar.
- No descargar el `cfg80211` stock.
- No descargar `sprd_wlan_combo`.
- No sustituir módulos Wi-Fi vendor sin una construcción coherente.
- Probar un solo stack y un solo adaptador a la vez.

La afirmación segura es:

```text
AR9271 puede funcionar manualmente en el kernel stock rooteado
usando módulos externos compatibles.
```

La afirmación no demostrada sería:

```text
AR9271 está listo para uso permanente mediante un módulo Magisk.
```

---

## 18. Próximos adaptadores y líneas de investigación

### Prioridad actual

1. Adaptador TP-Link Realtek.
2. Adaptador ALFA MediaTek.
3. Endurecimiento del cargador AR9271.
4. Construcción de una pila inalámbrica coherente.

### Caminos de driver considerados

| Ruta | Riesgo estimado | Motivo |
|---|---|---|
| Realtek `88XXau` vendor | Verde/amarillo | Puede evitar depender de un `mac80211` externo |
| Realtek `rtw88_usb` | Amarillo/rojo | Normalmente depende de `mac80211` |
| MediaTek `mt76x0u` | Amarillo/rojo | Requiere `mt76` y probablemente `mac80211` |
| Atheros `ath9k_htc` | Rojo para producción | Funciona, pero la integración automática no es estable |

Inventario inicial recomendado:

```sh
lsusb -nn | grep -Ei \
'Realtek|MediaTek|Ralink|Atheros|Qualcomm|TP-Link|Alfa|Netgear|0bda|148f|0cf3|0846'
```

Para cada adaptador debe registrarse:

- Vendor ID.
- Product ID.
- Modelo comercial.
- Chipset real.
- Driver esperado.
- Dependencias.
- Necesidad o no de `mac80211`.
- Compatibilidad con el `cfg80211` stock.
- Efectos en touchscreen, Wi-Fi interno, USB y estabilidad.

---

## 19. Objetivo de arquitectura inalámbrica a largo plazo

La solución robusta no consiste simplemente en cambiar opciones de `=m` a `=y`.

Una pila coherente debería considerar componentes construidos desde un entorno compatible:

```text
cfg80211
mac80211
ath
ath9k_hw
ath9k_common
ath9k_htc
sprd_wlan_combo
unisoc_wcn_bsp
dependencias SIPC/WCN requeridas
```

El problema real es la compatibilidad de subsistemas, símbolos, configuraciones y ABI.

---

## 20. Condiciones para detener una prueba

Detener inmediatamente si:

- Un driver exige reemplazar `cfg80211` stock sin un plan coherente.
- Aparecen símbolos no exportados que requieren parches inseguros.
- La carga afecta touchscreen, entrada, Wi-Fi interno, USB o estabilidad.
- El firmware requerido no puede distribuirse legalmente.
- La prueba exige escribir imágenes activas sin respaldo.
- Se pierde la capacidad de regresar al kernel stock conocido.

---

## 21. Política del repositorio

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

## 22. Estado actual del proyecto

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
- Reconstrucción del entorno ABI.
- Carga de módulos externos.
- Soporte manual AR9271.
- Modo monitor.
- Prueba de inyección.
- Uso de Wifite.

### Pendiente

- Convertir el soporte AR9271 en una solución estable.
- Probar el TP-Link Realtek por su ruta adecuada.
- Probar el ALFA MediaTek por su ruta adecuada.
- Construir una pila Wi-Fi coherente.
- Resolver de forma concluyente el arranque del kernel personalizado.
- Obtener una traza runtime completa del fallo de first-stage init.
- Validar UFS, reguladores y módulos en un kernel custom coherente.

---

## 23. Conclusión final

El Cubot KingKong ES3 analizado tiene un root funcional y reproducible basado en:

```text
bootloader LK desbloqueado
+ vbmeta_a mínimamente modificado
+ boot_a stock
+ vendor_boot_a stock
+ dtbo_a stock
+ init_boot_a parcheado por Magisk
```

El kernel personalizado continúa siendo un proyecto separado. La evidencia apunta a incompatibilidades alrededor de UFS, reguladores SC2730, módulos stock y `CONFIG_MODVERSIONS`, pero todavía no existe una traza definitiva que reduzca el fallo a una sola causa.

La investigación de módulos externos sí produjo un resultado firme: el kernel stock acepta módulos construidos contra su ABI exacta, y el adaptador Netgear WNA1100 / AR9271 logró modo monitor e inyección con NetHunter Lite.

Ese resultado permite continuar el desarrollo de capacidades NetHunter sin confundirlo con la validación pendiente del kernel personalizado.

---

## 24. Informes originales en inglés

Este documento consolida y traduce los siguientes archivos:

- [`README.md`](README.md)
- [`REPORT_INDEX.md`](REPORT_INDEX.md)
- [`BOOT_CHAIN_FINDINGS.md`](BOOT_CHAIN_FINDINGS.md)
- [`ROOTING_GUIDE.md`](ROOTING_GUIDE.md)
- [`REPORT_3WAY_BOOT_COMPARISON.md`](REPORT_3WAY_BOOT_COMPARISON.md)
- [`REPORT_INIT_BOOT_STOCK_VS_LIVE.md`](REPORT_INIT_BOOT_STOCK_VS_LIVE.md)
- [`REPORT_VENDOR_BOOT_STOCK_VS_LIVE.md`](REPORT_VENDOR_BOOT_STOCK_VS_LIVE.md)
- [`REPORT_DTBO_STOCK_VS_LIVE.md`](REPORT_DTBO_STOCK_VS_LIVE.md)
- [`REPORT_VBMETA_STOCK_VS_LIVE.md`](REPORT_VBMETA_STOCK_VS_LIVE.md)
- [`NEXT_PHASE_HEADERS_MODULES_PLAN.md`](NEXT_PHASE_HEADERS_MODULES_PLAN.md)
- [`NETHUNTER_NETGEAR_AR9271_RESULTS.md`](NETHUNTER_NETGEAR_AR9271_RESULTS.md)

Los informes originales siguen siendo la fuente más detallada para tablas completas, artefactos de comparación y contexto histórico.
