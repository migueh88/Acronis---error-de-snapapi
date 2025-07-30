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

# 🛠 OPCIONAL: Automatizar el chequeo con script

Podés guardar este script en `/usr/local/bin/check-snapapi.sh`:

```bash
#!/bin/bash
# Script para verificar y recompilar snapapi26 si hace falta

KERNEL=$(uname -r)
VERSION="0.8.54"

echo "🔍 Verificando SnapAPI para el kernel $KERNEL..."

if ! dkms status | grep -q "snapapi26/$VERSION, $KERNEL"; then
    echo "🛠 Compilando SnapAPI para $KERNEL..."
    dkms add -m snapapi26 -v $VERSION 2>/dev/null
    dkms build -m snapapi26 -v $VERSION
    dkms install -m snapapi26 -v $VERSION
else
    echo "✅ SnapAPI ya está instalado para $KERNEL"
fi

echo "📦 Cargando módulo..."
modprobe snapapi26

echo "🔁 Reiniciando servicio Acronis..."
systemctl restart acronis_mms
```

### Para activarlo:

```bash
sudo chmod +x /usr/local/bin/check-snapapi.sh
sudo /usr/local/bin/check-snapapi.sh
```

📌 Lo podés agregar a `@reboot` en `crontab` o ejecutarlo manualmente después de un reboot.

---

¿Querés que te lo empaquete como archivo `.sh` ya listo para descargar y usar?
