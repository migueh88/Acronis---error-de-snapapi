¡Listo! Acá tenés un script **todo-en-uno** para compilar/instalar SnapAPI para el kernel activo, cargar el módulo y reiniciar el servicio de Acronis. Es idempotente y te avisa si Secure Boot lo está bloqueando.

### 1) Guardá el script

```bash
vim fix-snapapi.sh
```

Pegá esto adentro y guardá:

```bash
#!/usr/bin/env bash
# fix-snapapi.sh — Recompila e instala SnapAPI para el kernel activo, lo carga,
# asegura /dev/snapapi y reinicia servicios de Acronis.
# Uso: sudo /usr/local/sbin/fix-snapapi.sh [--clean-old]
set -Eeuo pipefail

KERN="$(uname -r)"
CLEAN_OLD="${1:-}"

log(){ echo -e "[*] $*"; }
ok(){  echo -e "[✓] $*"; }
err(){ echo -e "[✗] $*" >&2; }

require_root(){
  if [[ $EUID -ne 0 ]]; then
    err "Ejecutá como root: sudo $0"
    exit 1
  fi
}

ensure_build_tools(){
  log "Instalando dependencias (headers exactos, dkms, build-essential)…"
  apt-get update -qq || true

  # Intento 1: headers exactos del kernel
  if ! apt-get install -y "linux-headers-${KERN}" dkms build-essential >/dev/null 2>&1; then
    log "No se pudieron instalar headers exactos; probando genéricos…"
    # Intento 2: headers genéricos (útil en kernels de nubes)
    apt-get install -y linux-headers-generic dkms build-essential >/dev/null 2>&1 \
      || apt-get install -y linux-headers-virtual dkms build-essential >/dev/null 2>&1 || true
  fi

  if [[ ! -e "/lib/modules/${KERN}/build/Makefile" ]]; then
    err "No están disponibles los headers para ${KERN}. Instalalos y reintentá."
    exit 1
  fi
  ok "Dependencias listas."
}

detect_snapapi(){
  local src
  src=$(ls -1d /usr/src/snapapi*/ 2>/dev/null || true)
  if [[ -z "$src" ]]; then
    log "No se encuentran fuentes en /usr/src/snapapi*/ — instalando paquete de Acronis…"
    # Según instalación puede ser uno u otro
    if ! apt-get install -y acronis-snapapi >/dev/null 2>&1; then
      apt-get install -y acronis-snapapi-dkms >/dev/null 2>&1 || true
    fi
  fi

  src=$(ls -1d /usr/src/snapapi*/ 2>/dev/null | sort -V | tail -n1 || true)
  if [[ -z "$src" ]]; then
    err "No se encontraron fuentes de SnapAPI en /usr/src. Verificá que el agente de Acronis esté instalado."
    exit 2
  fi

  SRC_DIR="${src%/}"
  BASENAME="$(basename "$SRC_DIR")"          # ej: snapapi26-0.8.57
  MODNAME="${BASENAME%-*}"                   # ej: snapapi26
  VER="${BASENAME#*-}"                       # ej: 0.8.57

  ok "Detectado módulo ${MODNAME} versión ${VER} en ${SRC_DIR}"
}

build_install_dkms(){
  log "Compilando/instalando ${MODNAME}/${VER} para kernel ${KERN} con DKMS…"
  # Limpiar restos para este kernel si existieran
  dkms remove -m "${MODNAME}" -v "${VER}" -k "${KERN}" --all >/dev/null 2>&1 || true

  if ! dkms install -m "${MODNAME}" -v "${VER}" -k "${KERN}" >/dev/null 2>&1; then
    log "dkms install directo falló — intentando build + install…"
    dkms build   -m "${MODNAME}" -v "${VER}" -k "${KERN}" >/dev/null
    dkms install -m "${MODNAME}" -v "${VER}" -k "${KERN}" >/dev/null
  fi

  # Verificación
  if ! dkms status | grep -Fq "${MODNAME}/${VER}, ${KERN}"; then
    err "DKMS no dejó ${MODNAME} instalado para ${KERN}. Revisá el log:"
    err "  /var/lib/dkms/${MODNAME}/${VER}/build/make.log"
    exit 3
  fi
  ok "DKMS instaló ${MODNAME} para ${KERN}."
}

load_module(){
  log "Cargando módulo (${MODNAME})…"
  if ! modprobe "${MODNAME}" 2>/dev/null; then
    # Intento alternativo por si el nombre difiere
    [[ "${MODNAME}" == "snapapi26" ]] && modprobe snapapi 2>/dev/null || true
    [[ "${MODNAME}" == "snapapi"   ]] && modprobe snapapi26 2>/dev/null || true
  fi

  if lsmod | grep -q '^snapapi'; then
    ok "Módulo SnapAPI cargado."
  else
    err "El módulo no aparece en lsmod."
    if command -v mokutil >/dev/null 2>&1; then
      if mokutil --sb-state 2>/dev/null | grep -qi enabled; then
        err "Secure Boot está habilitado. El kernel puede estar bloqueando el módulo sin firmar."
        err "Opciones: desactivar Secure Boot o firmar el módulo con MOK."
      fi
    fi
    # Mostrar pistas del .ko y del kernel
    modinfo snapapi26 2>/dev/null || modinfo snapapi 2>/dev/null || true
    dmesg | tail -n50 | sed 's/^/  /' || true
    exit 4
  fi
}

ensure_device_node(){
  log "Asegurando /dev/snapapi…"
  local maj
  maj="$(awk '/[[:space:]]snapapi$/{print $1}' /proc/devices || true)"
  [[ -z "$maj" ]] && maj="$(dmesg | tac | grep -m1 -oP 'Ctl major \K[0-9]+' || true)"

  if [[ -n "${maj:-}" && "${maj}" =~ ^[0-9]+$ ]]; then
    if [[ ! -e /dev/snapapi ]]; then
      mknod -m 600 /dev/snapapi c "$maj" 0
      ok "/dev/snapapi creado (major=${maj})."
    else
      ok "/dev/snapapi ya existe (major=${maj})."
    fi
    PERSIST_MAJOR="$maj"
  else
    err "No pude determinar el major; revisá dmesg."
  fi
}

persist_device_node(){
  # Persistencia via systemd-tmpfiles (si está disponible y tenemos un major válido)
  [[ -z "${PERSIST_MAJOR:-}" ]] && return 0
  if command -v systemd-tmpfiles >/dev/null 2>&1; then
    log "Escribiendo /etc/tmpfiles.d/snapapi.conf para persistir /dev/snapapi…"
    cat > /etc/tmpfiles.d/snapapi.conf <<EOF
c /dev/snapapi 0600 root root - ${PERSIST_MAJOR}:0
EOF
    systemd-tmpfiles --create /etc/tmpfiles.d/snapapi.conf >/dev/null 2>&1 || true
    ok "Persistencia configurada (major=${PERSIST_MAJOR})."
  fi
}

restart_services(){
  log "Reiniciando servicios de Acronis…"
  systemctl restart acronis_mms  2>/dev/null || true
  systemctl restart acronisagent 2>/dev/null || true
  systemctl restart acronis_agent 2>/dev/null || true
  ok "Servicios reiniciados (si existían)."
}

clean_old(){
  [[ "$CLEAN_OLD" == "--clean-old" ]] || return 0
  log "Limpieza de entradas DKMS antiguas (conservando ${KERN})…"
  mapfile -t KLIST < <(dkms status | awk -F, -v m="$MODNAME" -v v="$VER" \
    '$1 ~ m"/"v {gsub(/^[[:space:]]+|[[:space:]]+$/, "", $2); print $2}' | sort -u)
  for K in "${KLIST[@]}"; do
    if [[ "$K" != "${KERN}" ]]; then
      dkms remove -m "${MODNAME}" -v "${VER}" -k "${K}" --all >/dev/null 2>&1 || true
    fi
  done
  ok "Limpieza DKMS completada."
}

show_summary(){
  echo
  ok "Resumen:"
  echo "  Kernel activo: ${KERN}"
  echo "  Módulo:        ${MODNAME} ${VER}"
  local dkms_line
  dkms_line="$(dkms status | grep -F "${MODNAME}/${VER}, ${KERN}" || true)"
  echo "  DKMS:          ${dkms_line}"
  local lsmod_line
  lsmod_line="$(lsmod | (grep -E '^snapapi' || echo 'no cargado'))"
  echo "  lsmod:         ${lsmod_line}"
  [[ -e /dev/snapapi ]] && echo "  device:        $(ls -l /dev/snapapi)"
}

main(){
  require_root
  ensure_build_tools
  detect_snapapi
  build_install_dkms
  load_module
  ensure_device_node
  persist_device_node
  restart_services
  clean_old
  show_summary
  ok "SnapAPI listo para backups en ${KERN}."
}

main "$@"
```

Dale permisos:

```bash
chmod +x fix-snapapi.sh
```

### 2) Ejecutalo

```bash
sudo fix-snapapi.sh
# opcional: limpiar entradas viejas de DKMS
sudo fix-snapapi.sh --clean-old
```

### 3) Qué deberías ver al final

* `DKMS instaló snapapiXX para 5.15.0-151-generic.`
* `Módulo SnapAPI cargado.`
* En el resumen: `lsmod` mostrando `snapapi26` (o `snapapi`).

Si al cargar el módulo te advierte sobre **Secure Boot habilitado**, contame y te paso el mini-procedimiento para **firmar el módulo con MOK** (así no tenés que desactivar Secure Boot).
