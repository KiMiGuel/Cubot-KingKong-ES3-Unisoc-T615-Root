**For English, click [here](./README.md).**

# 🔓 Cubot KingKong ES 3 — Guía de Desbloqueo y Root

**Unisoc T615 (UMS9230_6h10) · UFS · slots A/B · build `CUBOT_KINGKONG_ES_3_F071_V16_20260309`**

Sigue el mismo flujo de 5 pasos del clásico script `unlock_autopatch_9230` — más la preparación de herramientas (Paso 0) y el root con Magisk (Paso 6).

> ⚠️ **Lee esto primero**
> - 🟡 Dificultad: media/avanzada. Si no sabes lo que estás haciendo, detente aquí.
> - 💥 **Esto borra todo el dispositivo** — respalda tus datos primero.
> - 🔌 **Solo puerto y cable USB 2.0** — el USB 3.0 causa fallos de conexión en este dispositivo.
> - 🐧 Usa **Linux** (Kali). Ni macOS, ni Windows.
> - 🎯 Verificado únicamente en el build indicado arriba — vuelve a verificar antes de usarlo en cualquier otro build.
> - 🧯 ¿Se brickeó a la mitad? El modo BROM siempre responde — siempre puedes recuperarlo.

---

## 🧰 Paso 0 — Prepara las herramientas

Descarga el paquete `spl_unlock_linux` desde [Releases](https://github.com/KiMiGuel/Cubot-KingKong-ES3-Unisoc-T615-Root/releases), descomprímelo y compila:

```
cd spl_unlock_linux
make
gcc gen_spl-unlock.c -o gen_spl-unlock
gcc chsize.c -o chsize
```

Verifica que compilaste lo correcto — ambas comprobaciones deben pasar:

```
./spd_dump --help          # NO debe listar un comando "baudrate"
grep reopen_port common.c  # debe encontrar el fix de reconexión
```

Un archivo NO viene en el paquete: `init_boot.img` — extráelo de **tu propio** PAC stock (el build indicado arriba). Trabaja siempre desde dentro de la carpeta del paquete — los respaldos se guardan en el directorio actual.

---

## 💾 Paso 1 — Respalda y parchea tus loaders

Entra a **BROM**: apaga el teléfono → mantén presionado **Volumen Abajo** → conecta el USB → sigue presionando. Confirma:

```
lsusb | grep -i '1782:4d00'
```

Extrae `splloader` + `uboot`:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset
```

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

Vuelve a entrar a BROM (Paso 1) y luego:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
```

Espera ~10 segundos después del reinicio.

---

## 🔓 Paso 3 — Ejecuta el desbloqueo

Vuelve a entrar a BROM y luego:

```
spd_dump exec_addr 0x65015f08 fdl spl-unlock.bin 0x65000800
```

Lo que vas a ver, y por qué todo va bien:
- 🔄 `CHECK_BAUD FAIL` — es normal. El build parcheado se reconecta solo.
- ✅ `CHECK_BAUD FDL1` → `BSL_REP_VER: "SPRD3"` → `CMD_CONNECT FDL1` — **esta ES la señal de éxito.** El desbloqueo ya se aplicó.
- ⏱️ `timeout reached` después de eso — **es exactamente lo que quieres ver.** No es un error. **No** lo vuelvas a ejecutar; pasa al Paso 4. (Solo si nunca ves `CMD_CONNECT FDL1`, ejecútalo una vez más.)

---

## ✅ Paso 4 — Verifica el desbloqueo

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec verbose 2 read_part miscdata 8192 64 m.bin reset
```

Revisa `m.bin`: 64 bytes en cero = sigue bloqueado (repite el Paso 3) · una cadena de 32 bytes + dos hashes de 16 bytes = **desbloqueado** ✅. Un timeout al final de este paso es normal.

---

## 🧹 Paso 5 — Restaura y borra

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

Deja que el teléfono haga el restablecimiento de fábrica y arranque. 🟠 **La pantalla naranja de advertencia (unlocked) = éxito.**

---

## 🪄 Paso 6 — Root con Magisk

Arranca Android → Ajustes → Acerca del teléfono → toca **Número de compilación** ×7 → Opciones de desarrollador → **Depuración USB**. Después:

```
adb push init_boot.img /sdcard/Download/
```

En la app de Magisk: **Install → Select and Patch a File → `init_boot.img`**, trae de vuelta la imagen parcheada y flasheala — en **fastbootd**, no en el bootloader normal:

```
adb pull /sdcard/Download/magisk_patched-*.img .
adb reboot fastboot
fastboot flash init_boot magisk_patched-XXXXX.img
fastboot reboot
```

Abre Magisk → root activo. 🎉 **Listo — desbloqueado y rooteado.**

---

## 🩹 Solución de problemas

| Síntoma | Solución |
|---|---|
| `LIBUSB_ERROR_BUSY` | `sudo systemctl stop ModemManager`, reconecta |
| `LIBUSB_ERROR_NO_DEVICE` | Inicia `spd_dump` primero, **después** conecta el teléfono |
| `find port failed` | Cierra y vuelve a ejecutar el mismo comando |
| Dos `CHECK_BAUD FAIL` → `device removed, exiting...` | A tu build de `spd_dump` le falta el fix de reconexión — recompila desde el código fuente del paquete (`common.c` debe contener `reopen_port`) |
| Boot loop / soft brick en cualquier punto | BROM (Paso 1) siempre responde — reflashea el firmware stock y repite la guía |
| ☠️ En `spd_dump` | **Nunca** ejecutes `repartition`, `erase_all` ni `ufs repart/rawerase/setattr` — truncan la GPT |

---

## 📚 ¿Quieres saber el porqué? La investigación completa está aquí

- [`ROOTING_GUIDE.md`](ROOTING_GUIDE.md) — guía extendida con la evidencia detrás de cada paso
- [`BOOT_CHAIN_FINDINGS.md`](BOOT_CHAIN_FINDINGS.md) — análisis de la cadena de arranque / AVB
- [`INVESTIGACION_COMPLETA_ES.md`](INVESTIGACION_COMPLETA_ES.md) — investigación completa en español
- [`REPORT_INDEX.md`](REPORT_INDEX.md) — serie de comparaciones de particiones stock vs. en vivo

Este repositorio guarda **solo documentos y hashes** — nada de imágenes de firmware, PACs, volcados de particiones ni archivos `.ko`. Documenta el estado de un dispositivo/build específico; no lo tomes como una receta universal de rooteo.

🙏 Construido sobre el [toolkit de CVE-2022-38694 de TomKing062](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader) (`spd_dump`/`spreadtrum_flash`, `gen_spl-unlock`, `chsize`) y [Magisk](https://github.com/topjohnwu/Magisk) de topjohnwu.
