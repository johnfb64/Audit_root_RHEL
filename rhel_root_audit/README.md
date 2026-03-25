# rhel_root_audit — Auditoría de configuración de root

Playbook de **solo lectura** para auditar configuraciones inseguras de root
en infraestructura RHEL/CentOS. No realiza ningún cambio en los sistemas.

---

## Requisitos

- Ansible >= 2.9
- Usuario `svcsatrmtupg` con acceso SSH y `sudo NOPASSWD` en los hosts objetivo
- Python en los hosts administrados (requerido por Ansible)

---

## Estructura

```
rhel_root_audit/
├── inventory/
│   ├── lab.ini                        # Inventario de laboratorio
│   └── group_vars/
│       └── laboratorio.yml            # Variables del grupo
├── playbooks/
│   ├── audit_root.yml                 # Playbook principal
│   └── consolidate_report.yml        # Genera CSV consolidado
├── roles/
│   └── root_audit/
│       ├── tasks/
│       │   ├── main.yml               # Orquestador
│       │   ├── check_uid0.yml         # Usuarios con UID 0
│       │   ├── check_sudoers.yml      # Escalación a root en sudoers
│       │   ├── check_services.yml     # Servicios systemd como root
│       │   ├── check_processes.yml    # Procesos y scripts heredados
│       │   └── check_ssh.yml         # SSH y estado cuenta root
│       ├── templates/
│       │   └── report_host.json.j2    # Template de reporte por host
│       └── vars/
│           └── usuarios_genericos.yml # Lista de usuarios válidos
└── reports/
    └── YYYY-MM-DD/
        ├── por_host/
        │   └── hostname.json
        └── consolidado_YYYY-MM-DD.csv
```

---

## Uso

### 1. Actualizar usuarios genéricos válidos

Antes de ejecutar, editar `roles/root_audit/vars/usuarios_genericos.yml`
con los usuarios reales de la organización.

### 2. Ejecutar auditoría completa en laboratorio

```bash
ansible-playbook -i inventory/lab.ini playbooks/audit_root.yml
```

### 3. Ejecutar solo un check específico

```bash
# Solo el check de UID 0 (más crítico)
ansible-playbook -i inventory/lab.ini playbooks/audit_root.yml --tags uid0

# Solo sudoers
ansible-playbook -i inventory/lab.ini playbooks/audit_root.yml --tags sudoers

# Solo servicios
ansible-playbook -i inventory/lab.ini playbooks/audit_root.yml --tags services
```

### 4. Ejecutar en un solo host para prueba

```bash
ansible-playbook -i inventory/lab.ini playbooks/audit_root.yml \
  --limit saox1l0mdctlab08
```

### 5. Generar CSV consolidado

```bash
ansible-playbook playbooks/consolidate_report.yml
```

---

## Niveles de severidad

| Nivel | Condición |
|---|---|
| CRÍTICO | UID 0 en usuario no-root, OR sudo sin restricción total |
| ALTO | Servicio systemd corriendo como root, OR proceso huérfano como root |
| ADVERTENCIA | SSH root habilitado, authorized_keys presente, password root activa |
| OK | Sin hallazgos |

---

## Escalar a otros ambientes

Cuando se requiera auditar dev/uat/prod, agregar los inventarios correspondientes:

```bash
ansible-playbook -i inventory/dev.ini playbooks/audit_root.yml
ansible-playbook -i inventory/prod.ini playbooks/audit_root.yml
```

---

## Notas importantes

- Este playbook usa `become: true` para leer archivos protegidos (`/etc/shadow`, `/etc/sudoers`)
- **No modifica ningún archivo ni configuración**
- Los reportes se generan en el nodo de control (Satellite), no en los hosts
- El parámetro `serial: 10` limita la ejecución paralela — ajustar según recursos disponibles
