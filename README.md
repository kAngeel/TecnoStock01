# TecnoStock S.A.C. — Sistema de inventario

Aplicación completa de gestión de inventario: base de datos SQL Server con
procedimientos almacenados transaccionales, API REST en FastAPI y una SPA en
Angular 22 con Angular Material y Signal Forms.

## Stack

| Capa | Tecnología |
|---|---|
| Base de datos | SQL Server (probado en 2025 Express 17.0), T-SQL, sin ORM |
| Backend | Python 3.12, FastAPI, pyodbc, Pydantic v2, JWT, Argon2id |
| Frontend | Angular 22 (SPA, sin SSR), Angular Material 22, Signal Forms |
| Pruebas | pytest (unitarias e integración real contra SQL Server), Vitest + jsdom |

## Estructura

```
TecnoStock/
├── backend/            # API FastAPI (uv)
│   ├── src/backend/
│   │   ├── api/        # rutas HTTP y dependencias
│   │   ├── core/       # configuración, errores, seguridad, pool pyodbc
│   │   ├── repositories/  # acceso a datos exclusivamente vía SP
│   │   ├── schemas/    # modelos Pydantic de entrada/salida
│   │   └── services/   # reglas de negocio y orquestación
│   └── tests/          # 40 pruebas unitarias y de integración
├── database/           # scripts SQL numerados (01..07) y su README
└── frontend/           # SPA Angular 22 (npm)
```

## Requisitos

- SQL Server 2019 o superior (la base `TecnoStock` se crea en el paso 1).
- [ODBC Driver 18 for SQL Server](https://learn.microsoft.com/sql/connect/odbc/download-odbc-driver-for-sql-server).
- Python 3.12 y [uv](https://docs.astral.sh/uv/).
- Node.js 22+ y npm 10 (el proyecto fija `npm@10.9.2`).

## Puesta en marcha

### 1. Base de datos

Ejecutar los scripts en orden con `sqlcmd` (o SSMS). En Windows con autenticación
integrada y certificado autofirmado:

```powershell
sqlcmd -S ".\SQLEXPRESS" -E -C -f 65001 -b -i database\01_database.sql -W -s "|"
# repetir para 02, 03, 04, 05, 06 y 07
```

| Script | Contenido |
|---|---|
| `01_database.sql` | Crea la base si no existe y fija el nivel de compatibilidad. |
| `02_tables.sql` | Crea `Usuarios`, `Categorias`, `Productos`, `MovimientosInventario`. |
| `03_constraints_indexes.sql` | PK, UNIQUE, FK, CHECK e índices (con verificación previa). |
| `04_procedures.sql` | 24 procedimientos almacenados del sistema. |
| `05_permissions.sql` | Rol `TecnoStockApp` con `EXECUTE` y sin DML directo. |
| `06_seed.sql` | 5 categorías iniciales (idempotente). |
| `07_verification.sql` | Verificación de solo lectura; termina con `RESULTADO GLOBAL: OK`. |

Detalle completo en [`database/README.md`](database/README.md).

### 2. Backend (FastAPI)

```powershell
cd backend
Copy-Item .env.example .env
# editar .env: conexión a SQL Server, JWT_SECRET y BOOTSTRAP_ADMIN_PASSWORD
uv sync
uv run backend
```

`uv run backend` ejecuta `backend.main:run()` y utiliza la fábrica de event loop
`backend.core.loop:event_loop_factory`. En Windows selecciona
`SelectorEventLoop`, que evita que un `WinError 64` cierre el socket de escucha
cuando un cliente aborta la conexión. Si se arranca uvicorn directamente, hay
que indicar la misma fábrica:

```powershell
uv run uvicorn backend.main:app --reload --loop backend.core.loop:event_loop_factory --port 8000
```

La documentación interactiva queda en <http://localhost:8000/docs> y la
alternativa ReDoc en <http://localhost:8000/redoc>.

`BOOTSTRAP_ADMIN_*` crea el administrador inicial **solo si no existe**, porque
las contraseñas se almacenan con Argon2id y el hash debe generarse en Python
(no en SQL). Cambiar la contraseña por defecto antes de usar el sistema.

### 3. Frontend (Angular)
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process
```powershell
cd frontend
npm install
npm start
```

SPA disponible en <http://localhost:4200>. La URL de la API se configura en
`frontend/src/environments/environment.ts` (por defecto
`http://localhost:8000/api`) y debe coincidir con `CORS_ORIGINS` del backend.

Para generar la compilación de producción:

```powershell
npm run build
# salida: frontend/dist/frontend/browser
```

### Direcciones locales

| Servicio | URL |
|---|---|
| Frontend (SPA) | <http://localhost:4200> |
| API FastAPI | <http://localhost:8000> |
| Swagger UI | <http://localhost:8000/docs> |
| ReDoc | <http://localhost:8000/redoc> |
| Salud de la API | <http://localhost:8000/api/health> |

## Pruebas

```powershell
# Backend: 40 pruebas aprobadas (unitarias e integración real)
cd backend
uv sync
uv run pytest

# Frontend: 37 pruebas unitarias aprobadas (Vitest + jsdom)
cd frontend
npm install
npm test -- --watch=false

# Compilación de producción
cd frontend
npm run build
```

Las pruebas del backend incluyen pruebas unitarias, como la fábrica del event
loop y su cableado en `run()`, y pruebas de integración reales contra la base
`TecnoStock`. Estas últimas crean datos con prefijos `TST-` / `TSTCAT-`, se
ejecutan contra los procedimientos almacenados y limpian todo al terminar. La
prueba de concurrencia lanza 20 salidas simultáneas y verifica que nunca quede
stock negativo.

## Reglas de negocio implementadas

- Todo el acceso a datos pasa por procedimientos almacenados; no hay ORM ni DML
  directo desde la aplicación.
- El stock de un producto nuevo sempre es `0`; el alta y la edición no aceptan
  ni modifican `Stock`. Solo cambia con entradas y salidas.
- Entradas y salidas se ejecutan en transacción, registran `StockAnterior` y
  `StockPosterior` y no permiten stock negativo ni en concurrencia.
- Los productos inactivos no admiten movimientos y no aparecen en stock bajo;
  sí se incluyen en el valor del inventario si conservan existencias.
- Valor del inventario: `SUM(Stock * PrecioCompra)`; stock bajo: productos
  activos con `Stock <= StockMinimo`.
- `FechaActualizacion` se asigna con `SYSDATETIME()` en el procedimiento de
  actualización.
- El nombre `MovimientosInventario` se usa sin variantes en tabla, SP,
  repositorios, esquemas y API.

## Seguridad

- JWT firmado con HS256; el token se guarda en `sessionStorage` (no en
  `localStorage`) y el interceptor lo adjunta a cada petición.
- Contraseñas con Argon2id (implementación en Python) y validación de estado
  activo del usuario.
- Secretos solo en variables de entorno (`backend/.env`, ignorado por Git).
  `.env.example` documenta todas las variables sin valores reales.
- Rol de aplicación `TecnoStockApp`: `GRANT EXECUTE ON SCHEMA::dbo` y
  `DENY SELECT/INSERT/UPDATE/DELETE ON SCHEMA::dbo`.
- Errores de negocio de SQL Server (500xx) se traducen a respuestas HTTP 4xx
  con códigos estables; nunca se exponen detalles internos de SQL.

## Documentación por componente

- [`backend/README.md`](backend/README.md)
- [`frontend/README.md`](frontend/README.md)
- [`database/README.md`](database/README.md)
