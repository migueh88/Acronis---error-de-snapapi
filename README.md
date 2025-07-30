Perfecto. Acá tenés el **paso a paso claro y ordenado** para que **SnapAPI siempre funcione después de una actualización de kernel** en tus VPS con Acronis.

---

# ✅ Paso a Paso para evitar fallos de SnapAPI tras actualizar el kernel

---

## 🔹 1. Asegurate de tener las herramientas necesarias

Ejecutá esto una sola vez en cada VPS:

```bash
sudo apt update
```
```bash
sudo apt install dkms build-essential linux-headers-$(uname -r) -y
```

> Esto asegura que el sistema pueda **compilar módulos automáticamente** con DKMS.

---

## 🔹 2. Verificá que SnapAPI está registrado en DKMS

```bash
dkms status | grep snapapi26
```

Si **no aparece**, agregalo manualmente:

```bash
sudo dkms add -m snapapi26 -v 0.8.54
```

> Esto le dice a DKMS que debe **compilar SnapAPI para cada nuevo kernel** automáticamente.

---

## 🔹 3. Cuando el kernel se actualice (automático o manual)

Después de un reinicio con nuevo kernel (por ejemplo `5.4.0-218-generic`):

### Ejecutá este script para recompilar SnapAPI:

```bash
sudo dkms build -m snapapi26 -v 0.8.54
sudo dkms install -m snapapi26 -v 0.8.54
sudo modprobe snapapi26
sudo systemctl restart acronis_mms
```

---

## 🔹 4. Verificá que todo esté cargado

```bash
uname -r                 # Confirma el kernel actual
dkms status              # Debe decir "installed" para ese kernel
lsmod | grep snapapi     # Debe aparecer "snapapi26"
```

---

## 🔹 5. Ejecutá el backup normalmente

Desde el portal Acronis (botón “Run now”) o desde línea de comando si usás backup local.

---

# 🛠 Tip: Script dinámico que detecta la versión automáticamente

Podés guardar este script en `/usr/local/bin/check-snapapi.sh`:
```bash
vim /home/check-snapapi.sh
```
```bash
#!/bin/bash
VERSION=$(basename $(ls -d /usr/src/snapapi26-* | sort -V | tail -1) | cut -d '-' -f2)
KERNEL=$(uname -r)

dkms status | grep "snapapi26/$VERSION, $KERNEL" > /dev/null
if [ $? -ne 0 ]; then
  echo "🛠 Instalando snapapi26 v$VERSION para kernel $KERNEL"
  dkms add -m snapapi26 -v $VERSION 2>/dev/null
  dkms build -m snapapi26 -v $VERSION
  dkms install -m snapapi26 -v $VERSION
  modprobe snapapi26
  systemctl restart acronis_mms
else
  echo "✅ SnapAPI ya está instalado para este kernel y versión."
fi
```
```bash
chmod +x /home/check-snapapi.sh
```
```bash
./home/check-snapapi.sh
```
Esto detecta automáticamente la versión de SnapAPI que tenés en tu sistema y compila para el kernel actual.
### Para activarlo:

```bash
sudo chmod +x /usr/local/bin/check-snapapi.sh
sudo /usr/local/bin/check-snapapi.sh
```

📌 Lo podés agregar a `@reboot` en `crontab` o ejecutarlo manualmente después de un reboot.

---

¿Querés que te lo empaquete como archivo `.sh` ya listo para descargar y usar?
