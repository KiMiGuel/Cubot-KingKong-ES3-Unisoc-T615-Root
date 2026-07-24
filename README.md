# Cubot KingKong ES3 Unisoc T615 Root Research

**For English, click [here](https://github.com/KiMiGuel/Cubot-KingKong-ES3-Unisoc-T615-Root/blob/main/README.md).**

Este repositorio documenta la investigación sobre la cadena de arranque (boot chain), AVB, el estado de Magisk/root, NetHunter y la viabilidad de módulos de kernel para el Cubot KingKong ES3.

Es un repositorio de análisis y documentación. Las imágenes de firmware crudas, los archivos PAC, los volcados de particiones, las salidas de kernel generadas y los árboles de firmware propietario extraídos no se almacenan aquí intencionadamente.

## Documento completo en español

La investigación consolidada y traducida al español está disponible en:

- [`INVESTIGACION_COMPLETA_ES.md`](INVESTIGACION_COMPLETA_ES.md)

Incluye la cadena de arranque, el proceso de root, las comparaciones de particiones, la investigación del kernel personalizado, el entorno ABI de módulos y los resultados de NetHunter con AR9271.

## Dispositivo

| Campo | Valor |
|---|---|
| Dispositivo | Cubot KingKong ES3 |
| SoC | Unisoc T615 / familia ums9230 |
| Build de Android | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Kernel en ejecución | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Slot activo analizado | slot A |
| Almacenamiento | UFS |
| Evidencia UFS en la línea de comandos | `sprdboot.flash=ufs` |
| Estado de root | Rooteado con Magisk, con NetHunter Lite |

## Estado Actual

El teléfono está rooteado y ejecuta el kernel stock.

La cadena de arranque activa es mayormente stock:

| Partición | Hallazgo |
|---|---|
| `boot_a` | `boot-gki.img` del PAC stock, byte por byte |
| `init_boot_a` | Parcheada con Magisk y portando el root |
| `vendor_boot_a` | `vendor_boot.img` del PAC stock, byte por byte |
| `dtbo_a` | `dtbo.img` del PAC stock, byte por byte |
| `vbmeta_a` | Campo de flags modificado en un byte; los descriptores coinciden con stock |

La fuente del root es el `init_boot_a` parcheado, no `boot_a`.

### Hito de NetHunter / Wi-Fi USB

Ya se ha demostrado un entorno de compilación de módulos externos sobre el kernel stock para este dispositivo.

Objetivo confirmado:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

Entorno confirmado:

- commit exacto del common-kernel: `4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7`
- Android Clang 14.0.7 / `clang-r450784e`
- base de `/proc/config.gz` en vivo
- `Module.symvers` coincidente
- `CONFIG_MODVERSIONS=y`
- CRC de `module_layout` coincidente: `0x0222dd63`

Se logró el soporte del Netgear WNA1100 / AR9271 sobre el kernel Cubot stock rooteado mediante módulos compilados externamente.

Validado en un entorno de laboratorio autorizado:

- la pila de módulos `ath9k_htc` carga
- el firmware AR9271 carga
- aparece `wlan1`
- el monitor mode funciona
- la prueba de inyección de paquetes funciona
- Wifite puede usar el adaptador

Advertencia importante: el empaquetado automático/cargador-Magisk actual no se considera seguro para producción. El resultado verificado es una prueba manual de módulo externo sobre kernel stock, no una versión completa de kernel personalizado.

Consulta `NETHUNTER_NETGEAR_AR9271_RESULTS.md`.

## Modelo de Rooteo Confirmado por la Evidencia

El estado rooteado analizado se ve así:

```text
unlocked/orange LK bootloader
  |
  v
modified vbmeta_a flags byte, stock descriptors
  |
  v
stock boot_a kernel
  |
  v
stock vendor_boot_a and stock dtbo_a
  |
  v
Magisk-patched init_boot_a
  |
  v
Magisk root
```

Esto significa:

- El root de Magisk funciona desde `init_boot_a`.
- El kernel activo sigue siendo stock.
- Un teléfono rooteado no es prueba de que un candidato de kernel personalizado funcione.
- El trabajo futuro sobre el kernel debe coincidir con el ABI del kernel stock en vivo o proporcionar un conjunto coherente de kernel/módulos de reemplazo.

Para una guía más clara del proceso y de la evidencia que lo respalda, consulta `ROOTING_GUIDE.md`.

## Hallazgos Principales

### boot_a

El `boot_a` en vivo es byte por byte idéntico al `boot-gki.img` del PAC stock.

Hash del kernel stock/en vivo desempaquetado:

```text
8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532
```

Hash del kernel candidato personalizado:

```text
773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a
```

### init_boot_a

El `init_boot_a` en vivo está parcheado con Magisk. La evidencia incluye:

```text
.backup/.magisk
.backup/init.xz
.backup/.rmlist
overlay.d/sbin/magisk.xz
overlay.d/sbin/init-ld.xz
overlay.d/sbin/stub.xz
```

### vendor_boot_a

El `vendor_boot_a` en vivo es stock. Contiene el DTB stock, bootconfig, el ramdisk de vendor, el fstab de primera etapa y 157 módulos de vendor stock.

Bootconfig:

```text
androidboot.hardware=ums9230_6h10
```

Entre los módulos críticos de arranque que coinciden con stock se incluyen:

```text
ufs_sprd.ko
sc2730-regulator.ko
sprd-pmic-spi.ko
printk_cpuid.ko
rpmb.ko
regmap-hook.ko
```

### dtbo_a

El `dtbo_a` en vivo es stock. Contiene 13 entradas DTBO, todas coincidentes con stock tras la extracción y descompilación.

El DTBO confirma evidencia de overlays relacionados con UFS y reguladores, incluyendo:

```text
20200000.ufs
regulator-name = "vddvbus"
regulator-always-on
avdd12-supply
vddwcn
pmic_regulator
```

### vbmeta_a

El `vbmeta_a` en vivo difiere del stock en exactamente un byte del campo de flags de la cabecera AVB firmada.

| Campo | Stock | En vivo |
|---|---:|---:|
| Flags | `0` | `33554432` |

Todos los descriptores, claves, metadatos de rollback, cadena de versión y propiedades de build coinciden con stock. El `vbmeta_a` en vivo falla la verificación de firma de `avbtool`, lo cual encaja con el estado de bootloader LK desbloqueado/naranja (unlocked/orange).

## Índice de Reportes

Empieza aquí:

- `BOOT_CHAIN_FINDINGS.md`
- `REPORT_INDEX.md`
- `ROOTING_GUIDE.md`
- `NEXT_PHASE_HEADERS_MODULES_PLAN.md`
- `NETHUNTER_NETGEAR_AR9271_RESULTS.md`

Reportes por componente:

- `REPORT_3WAY_BOOT_COMPARISON.md`
- `REPORT_INIT_BOOT_STOCK_VS_LIVE.md`
- `REPORT_VENDOR_BOOT_STOCK_VS_LIVE.md`
- `REPORT_DTBO_STOCK_VS_LIVE.md`
- `REPORT_VBMETA_STOCK_VS_LIVE.md`

Resúmenes de evidencia:

- `IMAGE_METADATA.txt`
- `EXTRACTED_TREE_SUMMARY.txt`
- `SEARCH_HITS_BOOT_CRITICAL.txt`
- `ACTIVE_SLOT_A_SHA256SUMS.txt`

## Política de Firmware Crudo

Lo siguiente se excluye intencionadamente de git:

- volcados crudos de particiones `.img`
- binarios `.bin` de bootloader o firmware
- paquetes de firmware `.pac`
- árboles de firmware extraídos
- directorios de extracción de comparación generados
- salidas de módulos `.ko` compilados
- blobs de firmware
- archivos comprimidos y descargas parciales

Los reportes incluyen hashes, metadatos y resultados de análisis sin almacenar contenido crudo de firmware propietario.

## Siguiente Fase

La fase de identidad de la cadena de arranque está completa.

La fase de cabeceras-en-vivo/compilación-de-módulos también ha producido un resultado confirmado: el Netgear WNA1100 / AR9271 puede hacerse funcionar sobre el kernel Cubot stock rooteado con módulos compilados externamente.

Siguiente fase:

- clasificar el adaptador TP-Link Realtek por su ID USB exacto
- clasificar el adaptador ALFA por su ID USB exacto
- evitar cargar el módulo Magisk experimental del AR9271 hasta rediseñar su cargador
- evaluar rutas de drivers de vendor Realtek que puedan evitar un `mac80211` personalizado
- continuar la investigación coherente de la pila inalámbrica antes de tratar el AR9271 como listo para producción

Plan y evidencia:

- `NEXT_PHASE_HEADERS_MODULES_PLAN.md`
- `NETHUNTER_NETGEAR_AR9271_RESULTS.md`

No trates este repositorio como una receta universal de rooteo. Documenta el estado de un dispositivo/build concreto y la evidencia recopilada de él.
