# Documentacion

## Entorno local

Los dos backends tienen bases PostgreSQL independientes: `catalogo` es
dueño de su base y `turnos` de la suya. No comparten tablas, credenciales
ni volúmenes. Android se ejecuta fuera de Docker, como requiere el
enunciado; no necesita una base de datos propia en un contenedor. La copia
local del catálogo pertenece al backend de catálogo.

Desde esta carpeta, en PowerShell:

```powershell
Copy-Item .env.example .env
docker compose up --build
```

La configuración levanta las API en `http://localhost:8081` y
`http://localhost:8082`, y PostgreSQL en los puertos `5433` y `5434`.
Las contraseñas de `.env.example` son únicamente valores de desarrollo;
no deben reutilizarse en otros entornos. Se puede editar `.env` para
cambiar puertos y credenciales locales. Desde el emulador Android, el
host suele ser accesible como `10.0.2.2` en lugar de `localhost`.

Los backends requieren JDK 21 para ejecutarse localmente. Para correr sus
pruebas, desde cada carpeta `catalogo/catalogo_Programacion2` y
`turnos/turnos_Programacion2`, ejecutar `.\mvnw.cmd test`.

Para detener los contenedores sin borrar datos:

```powershell
docker compose down
```

Los datos persisten en volúmenes Docker separados. `docker compose down
-v` elimina ambos volúmenes y sus datos. Para ejecutar Android, abrir
`app-KMP/app-KMP` en Android Studio, configurar el Android SDK local y
ejecutar la configuración Android; el archivo `local.properties` es
local y no se versiona.