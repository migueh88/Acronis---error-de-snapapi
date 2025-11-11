¡Listo! Acá tenés un script **todo-en-uno** para compilar/instalar SnapAPI para el kernel activo, cargar el módulo y reiniciar el servicio de Acronis. Es idempotente y te avisa si Secure Boot lo está bloqueando.

### 1) Guardá el script

```bash
vim fix-snapapi.sh
```

Pegá esto adentro y guardá:

```bash
#!/usr/bin/env bash
# fix-snapapi.sh — Recompila e instala SnapAPI para el kernel activo, lo carga,
# asegura /dev/snapapi, y deja compilado para otros kernels instalados (post-reboot).
#
# Uso:
#   sudo ./fix-snapapi.sh                 # Verboso por defecto
#   sudo ./fix-snapapi.sh --quiet         # Silencioso
#   sudo ./fix-snapapi.sh --clean-old     # Limpia DKMS de kernels que ya no existen
#   sudo ./fix-snapapi.sh --verbose       # Fuerza verboso
#   sudo ./fix-snapapi.sh --load-at-boot  # Carga snapapi en cada arranque (modules-load.d)
#
set -Eeuo pipefail
trap 'echo -e "[✗] Falló en la línea $LINENO"; exit 99' ERR
export DEBIAN_FRONTEND=noninteractive

# --------- Flags ---------
VERBOSE=1
CLEAN_OLD=0
LOAD_AT_BOOT=0
while [[ $# -gt 0 ]]; do
  case "$1" in
    --verbose|-v) VERBOSE=1;;
    --quiet|-q)   VERBOSE=0;;
    --clean-old)  CLEAN_OLD=1;;
    --load-at-boot) LOAD_AT_BOOT=1;;
    *) ;; # ignora extras
  esac
  shift
done

# Wrapper para controlar verbosidad
run(){ if [[ $VERBOSE -eq 1 ]]; then "$@"; else "$@" >/dev/null 2>&1; fi }

KERN="$(uname -r)"

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

  if [[ $VERBOSE -eq 1 ]]; then
    run apt-get update
  else
    run apt-get update -qq
  fi

  APT_INSTALL_OPTS=(-y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold)
  [[ $VERBOSE -eq 0 ]] && APT_INSTALL_OPTS+=(-qq) || APT_INSTALL_OPTS+=(-o Dpkg::Progress-Fancy=1)

  # Headers del kernel activo + herramientas
  if ! run apt-get install "${APT_INSTALL_OPTS[@]}" "linux-headers-${KERN}" dkms build-essential; then
    log "No se pudieron instalar headers exactos; probando genéricos/virtual…"
    run apt-get install "${APT_INSTALL_OPTS[@]}" linux-headers-generic dkms build-essential \
      || run apt-get install "${APT_INSTALL_OPTS[@]}" linux-headers-virtual dkms build-essential || true
  fi

  if [[ ! -e "/lib/modules/${KERN}/build/Makefile" ]]; then
    err "No están disponibles los headers para ${KERN}. Instalalos y reintentá."
    exit 1
  fi

  run dpkg --configure -a || true
  # Tener headers futuros para autocompilación
  run apt-get install "${APT_INSTALL_OPTS[@]}" linux-headers-generic || true

  ok "Dependencias listas."
}

detect_snapapi(){
  local src
  src=$(ls -1d /usr/src/snapapi*/ 2>/dev/null || true)
  if [[ -z "$src" ]]; then
    log "No hay fuentes en /usr/src/snapapi*/ — intentando instalar paquete de Acronis…"
    run apt-get install -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold acronis-snapapi || true
    run apt-get install -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold acronis-snapapi-dkms || true
  fi

  src=$(ls -1d /usr/src/snapapi*/ 2>/dev/null | sort -V | tail -n1 || true)
  if [[ -z "$src" ]]; then
    err "No se encontraron fuentes de SnapAPI en /usr/src. Verificá que el agente de Acronis esté instalado."
    exit 2
  fi

  SRC_DIR="${src%/}"
  BASENAME="$(basename "$SRC_DIR")"
  MODNAME="${BASENAME%-*}"
  VER="${BASENAME#*-}"

  ok "Detectado módulo ${MODNAME} versión ${VER} en ${SRC_DIR}"
}

build_install_dkms(){
  log "Compilando/instalando ${MODNAME}/${VER} para kernel ${KERN} con DKMS…"

  run dkms remove -m "${MODNAME}" -v "${VER}" -k "${KERN}" || true

  if ! run dkms install -m "${MODNAME}" -v "${VER}" -k "${KERN}"; then
    log "dkms install directo falló — intentando build + install…"
    run dkms build   -m "${MODNAME}" -v "${VER}" -k "${KERN}"
    run dkms install -m "${MODNAME}" -v "${VER}" -k "${KERN}"
  fi

  log "Compilando para kernels instalados faltantes (dkms autoinstall)…"
  run dkms autoinstall

  local status_pattern_slash="${MODNAME}/${VER}, ${KERN}"
  local status_pattern_comma="${MODNAME}, ${VER}, ${KERN}"
  local dkms_status status_ok=0
  dkms_status="$(dkms status || true)"
  if grep -Fq "${status_pattern_slash}" <<< "${dkms_status}"; then
    status_ok=1
  elif grep -Fq "${status_pattern_comma}" <<< "${dkms_status}"; then
    status_ok=1
  elif [[ -e "/lib/modules/${KERN}/updates/dkms/${MODNAME}.ko" ]]; then
    status_ok=1
  fi

  if [[ ${status_ok} -ne 1 ]]; then
    err "DKMS no dejó ${MODNAME} instalado para ${KERN}. Revisá el log:"
    err "  /var/lib/dkms/${MODNAME}/${VER}/build/make.log"
    exit 3
  fi
  ok "DKMS instaló ${MODNAME} para ${KERN}."
}

load_module(){
  log "Cargando módulo (${MODNAME})…"
  if ! run modprobe "${MODNAME}"; then
    [[ "${MODNAME}" == "snapapi26" ]] && run modprobe snapapi || true
    [[ "${MODNAME}" == "snapapi"   ]] && run modprobe snapapi26 || true
  fi
  local retries=5
  local delay=1
  for ((i=0; i<retries; i++)); do
    if lsmod | grep -q '^snapapi'; then
      ok "Módulo SnapAPI cargado."
      return 0
    fi
    sleep "${delay}"
  done

  err "El módulo no aparece en lsmod tras ${retries} intentos."
  if command -v mokutil >/dev/null 2>&1; then
    if mokutil --sb-state 2>/dev/null | grep -qi enabled; then
      err "Secure Boot habilitado. El kernel puede bloquear el módulo sin firmar (MOK)."
    fi
  fi
  modinfo snapapi26 2>/dev/null || modinfo snapapi 2>/dev/null || true
  dmesg | tail -n50 | sed 's/^/  /' || true
  exit 4
}

ensure_device_node(){
  log "Asegurando /dev/snapapi…"
  local maj
  maj="$(awk '/[[:space:]]snapapi$/{print $1}' /proc/devices || true)"
  [[ -z "$maj" ]] && maj="$(dmesg | tac | grep -m1 -oP 'Ctl major \K[0-9]+' || true)"

  if [[ -n "${maj:-}" && "${maj}" =~ ^[0-9]+$ ]]; then
    if [[ ! -e /dev/snapapi ]]; then
      run mknod -m 600 /dev/snapapi c "$maj" 0
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
  [[ -z "${PERSIST_MAJOR:-}" ]] && return 0
  if command -v systemd-tmpfiles >/dev/null 2>&1; then
    log "Escribiendo /etc/tmpfiles.d/snapapi.conf para persistir /dev/snapapi…"
    cat > /etc/tmpfiles.d/snapapi.conf <<EOF2
c /dev/snapapi 0600 root root - ${PERSIST_MAJOR}:0
EOF2
    run systemd-tmpfiles --create /etc/tmpfiles.d/snapapi.conf
    ok "Persistencia configurada (major=${PERSIST_MAJOR})."
  fi
}

ensure_modules_load(){
  [[ $LOAD_AT_BOOT -eq 1 ]] || return 0
  log "Forzando carga de ${MODNAME} en cada arranque (modules-load.d)…"
  echo "${MODNAME}" > /etc/modules-load.d/snapapi.conf
  ok "Creado /etc/modules-load.d/snapapi.conf"
}

restart_services(){
  log "Reiniciando servicios de Acronis…"
  local services=(acronis_mms acronis_schedule)
  local restarted=0

  for svc in "${services[@]}"; do
    if ! systemctl list-unit-files "${svc}.service" >/dev/null 2>&1 && \
       ! systemctl status "${svc}" >/dev/null 2>&1; then
      log "  Servicio ${svc} no encontrado; se omite."
      continue
    fi

    if ! run systemctl restart "${svc}"; then
      err "No pude reiniciar ${svc}. Ejecutá: systemctl status ${svc}"
      exit 5
    fi

    if ! systemctl is-active --quiet "${svc}"; then
      err "${svc} no quedó activo tras el restart. Revisá: systemctl status ${svc}"
      exit 5
    fi

    local pid
    pid="$(systemctl show -p ExecMainPID --value "${svc}" 2>/dev/null || true)"
    log "  ${svc} activo (PID ${pid:-desconocido})."
    restarted=1
  done

  if [[ $restarted -eq 1 ]]; then
    ok "Servicios de Acronis reiniciados."
  else
    ok "No había servicios de Acronis para reiniciar."
  fi
}

clean_old(){
  [[ $CLEAN_OLD -eq 1 ]] || return 0
  log "Limpieza de entradas DKMS antiguas (conservando kernels presentes en /lib/modules)…"

  mapfile -t KLIST < <(dkms status | awk -F, -v m="$MODNAME" -v v="$VER" \
    '$1 ~ m"/"v {gsub(/^[[:space:]]+|[[:space:]]+$/, "", $2); print $2}' | sort -u)

  for K in "${KLIST[@]}"; do
    if [[ ! -d "/lib/modules/$K" ]]; then
      log "Eliminando entrada DKMS huérfana para kernel $K…"
      run dkms remove -m "${MODNAME}" -v "${VER}" -k "${K}" || true
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
  dkms_line="$(dkms status | grep -F "${MODNAME}" | grep -F "${KERN}" || true)"
  echo "  DKMS:          ${dkms_line:-no encontrado}"
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
  ensure_modules_load
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
./fix-snapapi.sh --clean-old --load-at-boot
```

### 3) Qué deberías ver al final

* `DKMS instaló snapapiXX para 5.15.0-151-generic.`
* `Módulo SnapAPI cargado.`
* En el resumen: `lsmod` mostrando `snapapi26` (o `snapapi`).

Si al cargar el módulo te advierte sobre **Secure Boot habilitado**, contame y te paso el mini-procedimiento para **firmar el módulo con MOK** (así no tenés que desactivar Secure Boot).
