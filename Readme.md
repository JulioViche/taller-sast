## SonarQube

El proyecto incluye SonarQube Community para análisis estático de código (SAST).

### Configuración inicial (solo primera vez)

1. Arranca todos los servicios:
   ```
   docker-compose up --build -d
   ```
2. Espera que SonarQube termine de iniciar (puede tomar 1-2 minutos) y accede a http://localhost:9000
3. Login por defecto: `admin` / `admin`
4. **Cambia la contraseña** cuando el sistema lo solicite
5. Ve a **http://localhost:9000/account/security/** y genera un token con nombre `github-actions`
6. El workflow ya está configurado con el secret `SONAR_TOKEN` en GitHub y el runner auto-registrado como servicio en docker-compose

### Self-hosted Runner

El pipeline CI/CD usa un **self-hosted runner** (`actions-runner`) que corre como contenedor Docker dentro del mismo `docker-compose.yml`. Esto permite:

- Análisis persistente en SonarQube local (historial, quality gates, métricas)
- El runner se registra automáticamente contra el repo `JulioViche/taller-sast`
- El token de GitHub se configura vía `.env` (ver `.env.example`)

### Workflow

El archivo `.github/workflows/sonarqube.yml`:
- Se ejecuta en `push` y `pull_request` a `main`
- Corre sobre el runner `[self-hosted, wsl]`
- Escanea los 4 proyectos via matrix strategy
- Usa los secrets de GitHub `SONAR_TOKEN` y `SONAR_HOST_URL` (configurados previamente)

Para agregar este workflow a tu propio repo:
1. Configura los secrets en GitHub → Settings → Secrets and variables → Actions:
   - `SONAR_TOKEN`: token generado desde la UI de SonarQube
   - `SONAR_HOST_URL`: `http://sonarqube:9000`
2. Crea el archivo `.env` con tu `GITHUB_PAT` (copiando `.env.example`)
3. Ejecuta `docker-compose up -d`

---

Ejecuta el comando de construcción y arranque:
```
docker-compose up --build
```

O en segundo plano:
```
docker-compose up --build -d
```

Verifica que todos los contenedores estén en estado Up:
```
docker-compose ps
```

Para crear los usuarios iniciales (operador y cliente), ejecuta la semilla:
```
docker-compose exec auth-service node dist/seeds/seed.js
```
Accede al frontend en http://localhost y utiliza las credenciales:

| Rol       | Email                | Contraseña   |
|-----------|----------------------|--------------|
| Operador  | admin@example.com    | Admin12345   |
| Cliente   | cliente@example.com  | Cliente123   |

Detener servicios
```
docker-compose down
```

Detener y eliminar volúmenes (borra las bases de datos):
```
docker-compose down -v
```