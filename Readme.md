## SonarQube

El proyecto incluye SonarQube Community para análisis estático de código (SAST).

### Configuración inicial (solo primera vez)

1. Arranca SonarQube junto con los demás servicios:
   ```
   docker-compose up --build -d
   ```
2. Espera que SonarQube termine de iniciar (puede tomar 1-2 minutos) y accede a http://localhost:9000
3. Login por defecto: `admin` / `admin`
4. **Cambia la contraseña** cuando el sistema lo solicite
5. Ve a **http://localhost:9000/account/security/** y genera un token con nombre `local-ci`
6. Copia el token — lo necesitarás para el paso siguiente

### GitHub Actions

El workflow `.github/workflows/sonarqube.yml` levanta SonarQube automáticamente como servicio dentro del runner y genera un token temporal para cada ejecución. No requiere configuración adicional de secrets.

Si en el futuro quisieras apuntar a tu instancia local en vez de usar el servicio del workflow, configura estos **secrets en GitHub**:
- `SONAR_HOST_URL`: `http://<tu-ip>:9000`
- `SONAR_TOKEN`: el token generado en el paso anterior

Y cambia en el workflow `SONAR_HOST_URL: http://localhost:9000` por `SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}`.

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