# 🗺️ ORS Local — OpenRouteService para Isócronas a Pie

Plantilla lista para levantar un servicio local de [OpenRouteService (ORS)](https://openrouteservice.org/) con Docker, preconfigurado para calcular **isócronas caminando** (`foot-walking`) sobre datos de OpenStreetMap.

> Pensado para México, pero funciona con cualquier archivo `.osm.pbf` del mundo.

---

## 📋 Tabla de Contenidos

- [Requisitos Previos](#-requisitos-previos)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [Instalación y Configuración](#-instalación-y-configuración)
- [Uso](#-uso)
- [Endpoints Disponibles](#-endpoints-disponibles)
- [Configuración Avanzada](#-configuración-avanzada)
- [Solución de Problemas](#-solución-de-problemas)
- [Recursos Útiles](#-recursos-útiles)
- [Licencia](#-licencia)

---

## ✅ Requisitos Previos

| Herramienta | Versión mínima | Instalación |
|-------------|---------------|-------------|
| **Docker** | 20.10+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| **Docker Compose** | 2.0+ | Incluido en Docker Desktop |
| **RAM disponible** | 6 GB+ | Configurable vía `XMX` en `docker-compose.yml` |
| **Disco** | ~5 GB | Para mapa de México + elevación + grafos |

> [!IMPORTANT]
> La RAM necesaria depende del tamaño del archivo `.pbf` y los perfiles activos.
> Fórmula aproximada: `tamaño_pbf × num_perfiles × 2`.
> Para México (~643 MB) con 1 perfil: **mínimo 2 GB**, recomendado **6 GB**.

---

## 📁 Estructura del Proyecto

```
osr_local/
├── docker-compose.yml              # Configuración del servicio Docker
├── .gitignore                       # Archivos excluidos del repositorio
├── README.md                        # Este archivo
└── ors-docker/
    ├── config/
    │   ├── ors-config.yml           # ⚙️ Configuración principal de ORS
    │   ├── example-ors-config.yml   # Ejemplo completo de configuración
    │   └── example-ors-config.env   # Ejemplo de variables de entorno
    ├── files/                       # 📥 Aquí va tu archivo .osm.pbf
    │   └── .gitkeep
    ├── graphs/                      # 🔧 Grafos compilados (auto-generado)
    │   └── .gitkeep
    ├── elevation_cache/             # 🏔️ Caché de elevación (auto-generado)
    │   └── .gitkeep
    └── logs/                        # 📝 Logs del servicio (auto-generado)
        └── .gitkeep
```

> [!NOTE]
> Los directorios `graphs/`, `elevation_cache/`, `logs/` y los archivos `.pbf` **no se incluyen en el repositorio** por su tamaño (varios GB). Se generan automáticamente al iniciar el servicio.

---

## 🚀 Instalación y Configuración

### 1. Clonar el repositorio

```bash
git clone https://github.com/<tu-usuario>/osr_local.git
cd osr_local
```

### 2. Descargar el archivo de mapa (.pbf)

Descarga el archivo `.osm.pbf` de la región que necesitas desde [Geofabrik](https://download.geofabrik.de/):

```bash
# Ejemplo: descargar el mapa de México
wget -P ors-docker/files/ https://download.geofabrik.de/north-america/mexico-latest.osm.pbf
```

**Algunas regiones populares:**

| Región | URL | Tamaño aprox. |
|--------|-----|--------------|
| México | `north-america/mexico-latest.osm.pbf` | ~640 MB |
| Colombia | `south-america/colombia-latest.osm.pbf` | ~250 MB |
| España | `europe/spain-latest.osm.pbf` | ~1 GB |
| Heidelberg (test) | `europe/germany/baden-wuerttemberg/heidelberg-latest.osm.pbf` | ~2.5 MB |

### 3. Configurar el archivo fuente

Edita `ors-docker/config/ors-config.yml` y actualiza la ruta del archivo PBF:

```yaml
ors:
  engine:
    profile_default:
      build:
        source_file: /home/ors/files/mexico-latest.osm.pbf  # ← tu archivo
    profiles:
      foot-walking:
        enabled: true
```

> [!TIP]
> El nombre del archivo debe coincidir exactamente con el que descargaste en `ors-docker/files/`.

### 4. Levantar el servicio

```bash
# Primera vez: construir grafos (puede tardar 30 min - 2 horas según el PBF)
REBUILD_GRAPHS=true docker compose up -d

# Veces posteriores (usa grafos en caché, inicia en segundos)
docker compose up -d
```

### 5. Verificar que funciona

Espera a que los grafos se construyan y luego verifica:

```bash
# Ver los logs en tiempo real
docker compose logs -f ors-app

# Verificar el estado del servicio
curl http://localhost:8080/ors/v2/health
```

Respuesta esperada:
```json
{
  "status": "ready"
}
```

---

## 🔧 Uso

### Calcular una Isócrona

Una isócrona muestra el área accesible desde un punto en un tiempo dado caminando.

```bash
curl -X POST "http://localhost:8080/ors/v2/isochrones/foot-walking" \
  -H "Content-Type: application/json" \
  -d '{
    "locations": [[-99.1332, 19.4326]],
    "range": [300, 600, 900],
    "range_type": "time"
  }'
```

**Parámetros:**
| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| `locations` | Coordenadas `[longitud, latitud]` | `[[-99.1332, 19.4326]]` (CDMX) |
| `range` | Tiempos en **segundos** | `[300, 600, 900]` (5, 10, 15 min) |
| `range_type` | Tipo: `time` (segundos) o `distance` (metros) | `"time"` |

### Calcular una Ruta

```bash
curl -X POST "http://localhost:8080/ors/v2/directions/foot-walking/geojson" \
  -H "Content-Type: application/json" \
  -d '{
    "coordinates": [[-99.1332, 19.4326], [-99.1400, 19.4350]]
  }'
```

---

## 📡 Endpoints Disponibles

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/ors/v2/health` | GET | Estado del servicio |
| `/ors/v2/status` | GET | Información detallada del servicio |
| `/ors/v2/isochrones/{profile}` | POST | Cálculo de isócronas |
| `/ors/v2/directions/{profile}` | POST | Cálculo de rutas |
| `/ors/v2/matrix/{profile}` | POST | Matriz de distancias/tiempos |
| `/swagger-ui` | GET | Documentación interactiva (Swagger UI) |

> Todos los endpoints están disponibles en `http://localhost:8080`.

---

## ⚙️ Configuración Avanzada

### Cambiar la memoria asignada

En `docker-compose.yml`, ajusta las variables `XMS` y `XMX`:

```yaml
XMS: 1g   # RAM inicial
XMX: 6g   # RAM máxima
```

### Agregar más perfiles

Edita `ors-docker/config/ors-config.yml`:

```yaml
ors:
  engine:
    profiles:
      foot-walking:
        enabled: true
      cycling-regular:
        enabled: true
      driving-car:
        enabled: true
```

> [!WARNING]
> Cada perfil adicional incrementa significativamente el uso de RAM y el tiempo de construcción de grafos. Ajusta `XMX` acorde.

### Cambiar el puerto

En `docker-compose.yml`, modifica el mapeo de puertos:

```yaml
ports:
  - "3000:8082"  # Cambiar 8080 por el puerto deseado
```

### Reconstruir grafos

Si cambias el archivo `.pbf` o la configuración de perfiles:

```bash
# Opción 1: variable de entorno
REBUILD_GRAPHS=true docker compose up -d

# Opción 2: eliminar grafos y reiniciar
rm -rf ors-docker/graphs/*
docker compose restart ors-app
```

---

## 🐛 Solución de Problemas

### El contenedor se reinicia constantemente

```bash
docker compose logs ors-app | tail -50
```

**Causa común:** memoria insuficiente. Incrementa `XMX` en `docker-compose.yml`.

### "Graphs not found" o status "not ready"

Los grafos aún se están construyendo. Revisa los logs:

```bash
docker compose logs -f ors-app
```

El proceso puede tardar desde minutos (archivos pequeños) hasta horas (países grandes).

### Error de permisos en volúmenes

```bash
# Asegurar permisos correctos
sudo chown -R 1000:1000 ors-docker/
```

### Resetear todo y empezar de cero

```bash
docker compose down
rm -rf ors-docker/graphs/* ors-docker/elevation_cache/* ors-docker/logs/*
REBUILD_GRAPHS=true docker compose up -d
```

---

## 📚 Recursos Útiles

- 📖 [Documentación oficial de ORS](https://giscience.github.io/openrouteservice/)
- 🗺️ [Descargas de mapas - Geofabrik](https://download.geofabrik.de/)
- 🐳 [ORS Docker Hub](https://hub.docker.com/r/openrouteservice/openrouteservice)
- 📘 [API Reference](https://openrouteservice.org/dev/#/api-docs)
- 💬 [GitHub de ORS](https://github.com/GIScience/openrouteservice)

---

## 📄 Licencia

Este proyecto es una plantilla de configuración basada en [OpenRouteService](https://github.com/GIScience/openrouteservice), que se distribuye bajo la [Licencia LGPL 3.0](https://www.gnu.org/licenses/lgpl-3.0.html).

Los datos de mapas provienen de [OpenStreetMap](https://www.openstreetmap.org/) © contribuidores de OSM.
