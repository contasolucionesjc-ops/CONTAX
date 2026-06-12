# CONTAX — Sistema Contable y Tributario Multi-Tenant (Ecuador)

## ✅ Sesión 1 completada: Base del proyecto

### Lo que se construyó

**1. Arquitectura multi-tenant**
- Schema `public`: tablas globales (contadores/licencias, usuarios, auditoría)
- Schema `contador_<id>`: un schema POR CADA contador licenciatario, con sus
  propias tablas de clientes, facturas, asientos, etc.
- Aislamiento total entre licenciatarios — tu colega con 50 clientes nunca
  ve ni comparte datos contigo.

**2. Autenticación con 3 roles (JWT)**
- `SUPERADMIN` — tú, dueño de CONTAX. Gestiona licencias (contadores).
- `CONTADOR` — licenciatario (tú mismo o tu colega). Gestiona sus clientes.
- `CLIENTE` — usuario final, ve solo su propia información.

**3. Panel de licencias** (`/contadores`)
- Crear nuevas licencias (contador + schema dedicado se crean automáticamente)
- Planes: BÁSICO (10 clientes), PROFESIONAL (50), ESTUDIO (ilimitado)
- Activar/desactivar, cambiar de plan, ver cantidad de clientes por licencia

**4. Gestión de clientes** (`/clientes`)
- CRUD de contribuyentes dentro del tenant del contador
- Cálculo automático del 9no dígito del RUC (clave para vencimientos SRI)
- Credenciales SRI **encriptadas** con Fernet (nunca en texto plano)
- Configuración de frecuencia de sync por cliente (manual/semanal/quincenal/mensual)
- Control de límite de clientes según plan de licencia

**5. Modelo de datos completo para las próximas sesiones**
- `documentos_electronicos` — facturas/NC/retenciones emitidas y recibidas
- `proveedores_recurrentes` — clasificación automática por proveedor conocido
- `reglas_clasificacion` — clasificación por palabra clave
- `asientos_contables` / `lineas_asiento` — diario contable NIIF
- `alertas_tributarias` — calendario y avisos
- `logs_sync` — auditoría de sincronizaciones con el SRI

---

## Estructura del proyecto

```
contax/backend/
├── app/
│   ├── main.py                  # FastAPI app
│   ├── init_db.py               # Script de inicialización (crear superadmin)
│   ├── core/
│   │   ├── config.py            # Configuración (.env)
│   │   ├── database.py          # Multi-tenant: schemas dinámicos
│   │   ├── security.py          # JWT + bcrypt + encriptación Fernet
│   │   └── deps.py               # Dependencias: auth, roles, tenant
│   ├── models/
│   │   ├── global_models.py     # Contador, Usuario (schema public)
│   │   └── tenant_models.py     # Cliente, Documentos, Asientos (por schema)
│   ├── schemas/                 # Pydantic (request/response)
│   └── routers/
│       ├── auth.py              # /auth/login, /auth/refresh
│       ├── contadores.py        # /contadores (panel de licencias)
│       └── clientes.py          # /clientes (CRUD contribuyentes)
├── requirements.txt
├── railway.toml
└── .env.example
```

---

## Cómo desplegar (Railway)

### 1. Crear proyecto en Railway
- Conecta tu cuenta de GitHub
- Sube esta carpeta `contax/backend` a un repositorio
- En Railway: "New Project" → "Deploy from GitHub repo"

### 2. Agregar PostgreSQL
- En el proyecto Railway: "+ New" → "Database" → "PostgreSQL"
- Railway genera automáticamente `DATABASE_URL` y la inyecta como variable

### 3. Configurar variables de entorno
En la pestaña "Variables" del servicio backend, agrega:

```
SECRET_KEY=<generar con: python -c "import secrets; print(secrets.token_urlsafe(32))">
FERNET_KEY=<generar con: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())">
ENVIRONMENT=production
```

`DATABASE_URL` ya la pone Railway automáticamente al conectar el plugin de Postgres.

### 4. Inicializar la base de datos
Desde la terminal de Railway (o localmente con la DATABASE_URL de producción):

```bash
python -m app.init_db
```

Esto crea las tablas globales y te pide crear tu usuario SUPERADMIN
(tu cuenta, la que administra todo CONTAX).

### 5. Crear las dos primeras licencias (tú y tu colega)

Con tu token de superadmin, llamar:

```bash
POST /contadores
{
  "nombre_completo": "Joel Chica",
  "razon_social": "Soluciones Contables JC",
  "email": "tu_email@ejemplo.com",
  "plan": "basico",
  "password_inicial": "..."
}
```

```bash
POST /contadores
{
  "nombre_completo": "Tu colega",
  "razon_social": "Estudio Contable XYZ",
  "email": "colega@ejemplo.com",
  "plan": "profesional",
  "password_inicial": "..."
}
```

Cada uno recibirá su propio schema (`contador_1`, `contador_2`) completamente
aislado, con su propio límite de clientes (10 vs 50).

---

## Próxima sesión (Sesión 2)

- Scraper del SRI con Playwright (login + descarga de comprobantes emitidos/recibidos)
- Endpoint `/sync` para disparar sincronización manual por cliente
- Procesamiento de XML → `documentos_electronicos` + clasificación automática
- Scheduler con Celery para sync configurable por cliente
