# CIS RHEL 9 Hardening - Banco Solidario

Playbook Ansible para aplicar el **CIS Red Hat Enterprise Linux 9 Benchmark v2.0.0**,
**Nivel 1 + Nivel 2**, perfil **Server**, de forma idempotente y auditable.

Desarrollado a medida para Banco Solidario (sin dependencias de roles community de
terceros; solo colecciones oficiales `ansible.posix` y `community.general`).

> Playbook 1 de 18 de la serie de hardening CIS del banco.

## Contenido del repositorio

```
ansible.cfg                          Configuracion de Ansible (inventario, ssh, become)
requirements.yml                     Colecciones requeridas (ansible.posix, community.general, community.crypto)
site.yml                             Playbook principal (entry point)
inventory/hosts.example.ini          Ejemplo de inventario -> copiar a inventory/hosts.ini
group_vars/all.yml                   Variables de entorno del grupo "rhel9"
docs/coverage_matrix.csv             Matriz de cobertura: los 297 controles del benchmark, uno por fila
roles/rhel9_cis_hardening/
  defaults/main.yml                  Variables por defecto: nivel CIS + un toggle booleano por control
  vars/section*.yml                  Datos estructurados (listas) que consumen las tasks
  tasks/main.yml                     Orquestador: incluye cada tasks/sectionN_*.yml
  tasks/section1_initial_setup.yml           CIS 1  - Initial Setup
  tasks/section2_services.yml                CIS 2  - Services
  tasks/section3_network.yml                 CIS 3 (Network) + CIS 4 (Firewall Configuration)
  tasks/section5_access_authentication.yml   CIS 5  - Access, Authentication and Authorization
  tasks/section4_logging_auditing.yml        CIS 6  - Logging and Auditing (AIDE, journald, rsyslog, auditd)
  tasks/section6_system_maintenance.yml      CIS 7  - System Maintenance (permisos, cuentas)
  handlers/main.yml                  Handlers (restart/reload de servicios, reinicio requerido, etc.)
  templates/                         Jinja2: banner de advertencia, nftables, GDM/dconf
  meta/main.yml                      Metadatos del rol
```

Los nombres de archivo `tasks/sectionN_*.yml` siguen una numeracion interna de 6 archivos
que **no coincide 1 a 1** con la numeracion de secciones del benchmark CIS (que tiene 7
secciones de primer nivel). El mapeo exacto esta documentado como comentario al inicio de
`tasks/main.yml`.

## Requisitos

- **Control node**: `ansible-core >= 2.15`, Python 3, y las colecciones listadas en
  `requirements.yml` (instalar con `ansible-galaxy collection install -r requirements.yml`).
- **Hosts gestionados**: RHEL 9 (o derivados compatibles Rocky/AlmaLinux/Oracle Linux 9),
  acceso SSH con un usuario que tenga `sudo`/`become` habilitado, y `python3` instalado
  (requerido por el modulo Ansible; en RHEL 9 viene por defecto).
- Conectividad SSH desde el control node hacia cada host objetivo (clave publica
  recomendada; el playbook no gestiona el despliegue de claves).

## Configurar el inventario

```bash
cp inventory/hosts.example.ini inventory/hosts.ini
$EDITOR inventory/hosts.ini   # ajustar hosts reales del grupo [rhel9]
```

`ansible.cfg` ya apunta a `inventory/hosts.ini` por defecto.

## Instalar dependencias

```bash
ansible-galaxy collection install -r requirements.yml
```

## Ejecutar el playbook

Ejecucion completa (Nivel 1 + Nivel 2, segun `rhel9cis_level` en `group_vars/all.yml`):

```bash
ansible-playbook -i inventory/hosts.ini site.yml --ask-become-pass
```

### Modo simulacion (recomendado antes de cualquier corrida real)

```bash
ansible-playbook -i inventory/hosts.ini site.yml --check --diff --ask-become-pass
```

`--check` no aplica cambios; `--diff` muestra el detalle de lo que se modificaria. Algunas
tareas que dependen de comandos no idempotentes en modo `--check` (p. ej. `grubby`,
`authselect`) pueden reportarse como "changed" en check mode sin poder confirmar el diff
real; revisar la salida con atencion.

### Aplicar solo una seccion o un control puntual (tags)

Cada tarea esta etiquetada con su ID CIS exacto (`"5.1.20"`), su seccion (`"section5"`),
su nivel (`"level1"`/`"level2"`) y `"server"`. Ejemplos:

```bash
# Solo el endurecimiento de SSH (seccion 5.1)
ansible-playbook -i inventory/hosts.ini site.yml --tags "5.1" --ask-become-pass

# Solo un control puntual
ansible-playbook -i inventory/hosts.ini site.yml --tags "5.1.20" --ask-become-pass

# Todo excepto auditd (puede ser una ventana de cambio separada)
ansible-playbook -i inventory/hosts.ini site.yml --skip-tags "section6" --ask-become-pass

# Solo Nivel 1 (aunque rhel9cis_level=2, limita la EJECUCION; ver tambien el toggle de nivel abajo)
ansible-playbook -i inventory/hosts.ini site.yml --tags "level1" --ask-become-pass
```

### Aplicar solo Nivel 1 (sin tocar Nivel 2) via variable

En `group_vars/all.yml` o `-e` en la linea de comandos:

```bash
ansible-playbook -i inventory/hosts.ini site.yml -e "rhel9cis_level=1" --ask-become-pass
```

Cada tarea valida internamente `rhel9cis_level >= <nivel del control>`, de modo que con
`rhel9cis_level=1` ningun control de Nivel 2 se ejecuta (independientemente de los tags).

### Exceptuar un control puntual (toggle individual)

Cada uno de los 274 controles automatizados tiene su propia variable booleana
`rhel9cis_rule_<id_con_guiones_bajos>` en `roles/rhel9_cis_hardening/defaults/main.yml`
(p. ej. `rhel9cis_rule_1_1_1_8` para el control `1.1.1.8`). **No editar ese archivo
directamente**: sobreescribir la excepcion puntual en `group_vars/<grupo>.yml` o
`host_vars/<host>.yml`, documentando siempre el ticket/CR asociado:

```yaml
# group_vars/rhel9.yml (ejemplo)
rhel9cis_rule_1_1_1_8: false   # usb-storage requerido en srv-rhel9-app01 para backup a cinta USB (CR-1234)
```

### Otras variables relevantes (`group_vars/all.yml` / `roles/.../defaults/main.yml`)

| Variable | Proposito |
|---|---|
| `rhel9cis_level` | `1` = solo Nivel 1, `2` = Nivel 1 + Nivel 2 (recomendado banco) |
| `rhel9cis_firewall_utility` | `firewalld` (recomendado) o `nftables`; solo una debe estar activa (CIS 4.1.2) |
| `rhel9cis_grub_password` | Password de GRUB2 (CIS 1.4.1); vacio = tarea se omite. Definir via Ansible Vault, nunca en texto plano |
| `rhel9cis_su_group` | Grupo autorizado a usar `su` (CIS 5.2.7) |
| `rhel9cis_sshd_allow_groups` / `_allow_users` / `_deny_groups` / `_deny_users` | Control de acceso SSH (CIS 5.1.7); `AllowGroups` tiene prioridad si no esta vacio |
| `rhel9cis_remote_log_host` | Host remoto de `systemd-journal-upload` (CIS 6.2.2.1.3); vacio = tarea se omite |
| `rhel9cis_is_container` | Poner en `true` si el host es un contenedor (omite tareas de kernel/GRUB/particiones no aplicables) |

## Advertencia importante

**Probar siempre primero en un entorno no productivo.** Este playbook modifica SSH,
firewall, PAM, GRUB, el modulo de auditoria del kernel y politicas criptograficas del
sistema; un error de configuracion puede dejar un host inaccesible remotamente. Se
recomienda:

1. Ejecutar primero con `--check --diff` y revisar el resultado.
2. Ejecutar en un host de laboratorio/no productivo con consola de emergencia disponible
   (fuera de banda, iLO/iDRAC/consola de hipervisor) antes de tocar SSH o firewall.
3. Coordinar ventana de mantenimiento: varias tareas (GRUB, `audit=1`, crypto-policies)
   requieren **reinicio** para quedar completamente activas; el playbook lo señala al
   final de la corrida (`rhel9cis_reboot_required`).
4. Revisar `docs/coverage_matrix.csv`, columna `notes`, para los controles que se
   implementan como **auditoria** (reportan hallazgos sin modificar automaticamente, por
   ejemplo cuentas con `NOPASSWD` en sudo, archivos world-writable, o cuentas UID 0
   adicionales) — estos requieren remediacion manual tras revisar el resultado.

## Resumen de cobertura

Total de controles del CIS RHEL 9 Benchmark v2.0.0, perfil Server (Nivel 1 + Nivel 2): **297**

| | Nivel 1 | Nivel 2 | Total |
|---|---|---|---|
| Automated (segun CIS) | 217 | 58 | 275 |
| Manual (segun CIS) | 18 | 4 | 22 |
| **Total** | **235** | **62** | **297** |

Estado de implementacion en este rol (ver detalle completo por ID en
`docs/coverage_matrix.csv`):

- **274 controles** marcados `Automated` por CIS estan **implementados** en
  `roles/rhel9_cis_hardening/tasks/*.yml` (remediacion activa e idempotente, o
  auditoria no destructiva cuando la remediacion automatica implicaba riesgo de negocio
  — ver notas en la matriz, p. ej. 5.2.4, 5.4.2.1, 7.1.11, 7.1.12).
- **1 control** (`5.4.2.4` - Ensure root account access is controlled) esta marcado
  `Automated` por CIS pero se deja como **manual** en este rol: la remediacion (fijar
  password a root o bloquear la cuenta) depende de la politica de acceso de emergencia
  ("break-glass") del banco y debe ejecutarse manualmente tras acordarla.
- **22 controles** estan marcados `Manual` por el propio CIS Benchmark (requieren
  politica organizacional, revision humana o decisiones de seguridad fisica) y **no se
  automatizan**; quedan documentados en `docs/coverage_matrix.csv` para seguimiento de
  cumplimiento.

Ningun control queda sin rastro: todo ID del benchmark aparece exactamente una vez en
`docs/coverage_matrix.csv` con su estado (`implemented` o `manual`) y, cuando aplica, el
tag/ID de la tarea Ansible que lo implementa.
