## SonarQube — Análisis SAST en CI/CD

El proyecto integra **SonarQube Community** para análisis estático de código (SAST) mediante un **self-hosted runner** de GitHub Actions que corre como contenedor Docker en tu máquina.

### Arquitectura del flujo

```
GitHub                       Tu máquina (WSL)
┌─────────────────┐          ┌──────────────────────────────────┐
│                 │          │  Docker Compose                   │
│  push / PR a    │ ──────→ │  ┌────────────┐  ┌─────────────┐ │
│  main           │          │  │ sonarqube  │  │actions-runner│ │
│                 │          │  │ :9000      │  │(self-hosted) │ │
│  GitHub Actions │ ←────── │  │ persistente │  │             │ │
│  asigna job al  │          │  └────────────┘  └──────┬──────┘ │
│  runner         │          └─────────────────────────┼────────┘
└─────────────────┘                                    │
                   ┌────────────────────────────────────┘
                   │
                   ▼
          ┌──────────────────────┐
          │ Runner ejecuta:      │
          │ 1. git checkout      │
          │ 2. SonarQube Scan    │
          │    (conecta a        │
          │     sonarqube:9000)  │
          │ 3. Resultado se      │
          │    guarda en SQ      │
          └──────────────────────┘
```

### Prerrequisitos (condiciones para que funcione)

| Condición | Detalle |
|-----------|---------|
| **Docker en WSL** | Docker Desktop o Docker Engine corriendo en WSL2 |
| **GitHub PAT** | Token classic con permiso `repo` (configurado en `.env`) |
| **SonarQube accesible** | El contenedor `sonarqube` debe estar corriendo (se levanta con `docker compose up -d`) |
| **Runner online** | El contenedor `actions-runner` debe estar ejecutándose y mostrar `Listening for Jobs` en los logs |
| **Secrets en GitHub** | `SONAR_TOKEN` y `SONAR_HOST_URL` ya están configurados vía API (si se crea un repo desde cero, hay que configurarlos manualmente) |

### Configuración inicial (solo primera vez)

#### Paso 1 — Preparar entorno local

```bash
# 1. Clonar el repo
git clone https://github.com/JulioViche/taller-sast
cd taller-sast

# 2. Crear archivo .env con tu GitHub PAT (copiar desde .env.example)
cp .env.example .env
# Editar .env y pegar tu token: GITHUB_PAT=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 3. Iniciar todos los servicios
docker compose up -d
```

Esto levanta:
- `sonarqube` — servidor de SonarQube en http://localhost:9000
- `actions-runner` — se registra automáticamente contra el repo y queda escuchando jobs
- `auth-db`, `product-db`, `auth-service`, `product-service`, `frontend` — la aplicación

#### Paso 2 — Configurar SonarQube

1. Espera que SonarQube termine de iniciar (1-2 min) y accede a http://localhost:9000
2. Login por defecto: `admin` / `admin`
3. **Cambia la contraseña** cuando el sistema lo solicite
4. Ve a **http://localhost:9000/account/security/**
5. Genera un token con nombre `github-actions` (el que ya está configurado como `SONAR_TOKEN` en GitHub)
6. Si el token es nuevo, actualizalo en GitHub:
   - Settings → Secrets and variables → Actions → `SONAR_TOKEN`

#### Paso 3 — Verificar que el runner está online

```bash
docker logs actions-runner --tail 5
# Debería mostrar: "Listening for Jobs"
```

O desde la API de GitHub:
```
https://github.com/JulioViche/taller-sast/settings/actions/runners
```
Debería aparecer `taller-sast-runner` con estado **Idle**.

### Flujo del CI/CD (push / pull request)

Cuando hacés `git push` a `main` o abrís un PR:

1. **GitHub detecta el evento** y busca un runner con label `[self-hosted, wsl]`
2. **El runner local** (`actions-runner`) recoge el job
3. **El runner ejecuta**:
   ```
   jobs:
     sonarqube:
       runs-on: [self-hosted, wsl]
       strategy:
         matrix:
           app: [auth-service, products, frontend, analisis-vulnerabilidades]
       steps:
         - uses: actions/checkout@v4
         - name: Wait for SonarQube
           run: curl -s http://sonarqube:9000  (espera hasta 5 min)
         - name: SonarQube Scan (${{ matrix.app }})
           uses: SonarSource/sonarqube-scan-action@v5
           env:
             SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
             SONAR_HOST_URL: http://sonarqube:9000
   ```
4. **El escáner** analiza el código en `apps/<app>` y envía los resultados a SonarQube
5. **SonarQube** procesa el reporte y actualiza el Quality Gate
6. **GitHub Actions** muestra el resultado (`success` / `failure`)
7. **Podés ver los resultados** en http://localhost:9000 (historial, bugs, vulnerabilidades, code smells, cobertura)

### Ejemplo visual del resultado

En GitHub Actions, el job se ve así:

```
Workflow: SonarQube Analysis
  ├── sonarqube (auth-service)          ✅ SUCCESS
  ├── sonarqube (products)              ✅ SUCCESS
  ├── sonarqube (frontend)              ✅ SUCCESS
  └── sonarqube (analisis-vulnerabilidades)  ✅ SUCCESS
```

Y en http://localhost:9000 cada proyecto tiene su dashboard con el análisis completo.

### Solución de problemas

| Problema | Causa | Solución |
|----------|-------|----------|
| `No runner found` | El runner no está registrado o no está corriendo | `docker compose logs actions-runner` — verificar `Listening for Jobs` |
| `401 Unauthorized` al escanear | `SONAR_TOKEN` incorrecto o expirado | Regenerar token en http://localhost:9000/account/security y actualizar secret en GitHub |
| `Connection refused: sonarqube:9000` | SonarQube no está corriendo | `docker compose up -d sonarqube` |
| Runner no recibe jobs | El PAT en `.env` expiró | Regenerar PAT en GitHub y actualizar `.env` + reiniciar runner: `docker compose restart actions-runner` |
| Quality Gate falla | Código no cumple estándares | Revisar el dashboard de SonarQube en http://localhost:9000 |

### Comandos útiles

```bash
# Ver logs del runner
docker logs actions-runner -f

# Ver estado de SonarQube
curl -s http://localhost:9000/api/system/status

# Ver proyectos registrados
curl -s -H "Authorization: Bearer $(grep TOKEN .env | cut -d= -f2)" \
  http://localhost:9000/api/projects/search

# Escanear manualmente una app
docker run --rm \
  --network taller-sast_default \
  -e SONAR_HOST_URL=http://sonarqube:9000 \
  -e SONAR_TOKEN=$(grep TOKEN .env | cut -d= -f2) \
  -v $(pwd):/usr/src \
  sonarsource/sonar-scanner-cli \
  -D sonar.projectBaseDir=/usr/src/apps/auth-service \
  -D sonar.projectKey=taller-sast_auth-service

# Detener todo
docker compose down

# Detener y borrar datos (incluye análisis de SonarQube)
docker compose down -v
```

---

## Inicio rápido de la aplicación

```bash
docker compose up --build -d
```

Verifica que todos los contenedores estén en estado Up:
```
docker compose ps
```

Para crear los usuarios iniciales (operador y cliente), ejecuta la semilla:
```
docker compose exec auth-service node dist/seeds/seed.js
```

Accede al frontend en http://localhost y utiliza las credenciales:

| Rol       | Email                | Contraseña   |
|-----------|----------------------|--------------|
| Operador  | admin@example.com    | Admin12345   |
| Cliente   | cliente@example.com  | Cliente123   |
