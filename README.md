# Dashboard de Infraestructura — Documentacion para presentacion

---

## 1. Resumen del proyecto

Aplicacion web que consulta ~281,000 registros de infraestructura desde **Google BigQuery** y los muestra en un dashboard con graficas interactivas (barras, pastel) y tabla de datos. Los filtros permiten segmentar por tipo de infraestructura, tenencia y caja de compensacion.

---

## 2. Arquitectura general (flujo de datos)

```
NAVEGADOR                    BACKEND                    BIGQUERY
┌──────────┐    HTTP/JSON    ┌──────────────┐    SQL     ┌──────────┐
│ React +  │ ──────────────> │  FastAPI     │ ─────────> │  Google  │
│ Vite     │ <────────────── │  (Python)    │ <───────── │ BigQuery │
│ (5173)   │    JSON         │  (8000)      │   filas    │          │
└──────────┘                 └──────────────┘            └──────────┘
     │                             │
     │ proxy de Vite               │ google-cloud-bigquery
     │ /api -> localhost:8000      │ + service account (RSA)
     │                             │
```

**Flujo completo:**

1. El usuario abre `http://localhost:5173` en el navegador
2. React (App.jsx) llama a `getResumen()`, `getDatos()` y `getFiltros()` en `api/bigquery.js`
3. Axios hace peticiones `GET /api/resumen`, `GET /api/datos`, `GET /api/filtros`
4. Vite (en desarrollo) intercepta `/api/*` y lo redirige (proxy) a `http://localhost:8000`
5. FastAPI recibe la peticion, construye consultas SQL, las ejecuta contra BigQuery
6. BigQuery devuelve las filas, FastAPI las convierte a JSON y responde al frontend
7. React renderiza las graficas (Chart.js) y la tabla con los datos recibidos

---

## 3. Estructura de carpetas y funcionamiento de cada archivo

```
PRESENTACIONBD/
│
├── base/                                  # MATERIAL BASE (solo referencia)
│   ├── conexion2.py                       # Script Python original de conexion a BigQuery
│   └── 2-004A Infraestructura.csv         # Dataset original en CSV (~281k registros)
│
├── api/                                   # ENTRY POINT PARA VERCEL
│   ├── index.py                           # Importa la app de FastAPI desde dashboard-api/main.py
│   └── requirements.txt                   # Dependencias para Vercel
│
├── dashboard-api/                         # BACKEND (Python + FastAPI)
│   ├── main.py                            # Servidor API con 3 endpoints
│   ├── conexion_bq.py                     # Conexion a BigQuery con service account
│   └── requirements.txt                   # fastapi, uvicorn, google-cloud-bigquery
│
├── dashboard-ui/                          # FRONTEND (React + Vite + Chart.js)
│   ├── index.html                         # Pagina HTML principal
│   ├── vite.config.js                     # Config de Vite (proxy /api a localhost:8000)
│   ├── package.json                       # Dependencias: react, chart.js, axios, vite
│   ├── src/
│   │   ├── main.jsx                       # Punto de entrada de React (renderiza App)
│   │   ├── App.jsx                        # Componente principal con toda la logica
│   │   ├── App.css                        # Estilos del dashboard
│   │   ├── constants.js                   # Colores y funcion truncar()
│   │   ├── api/
│   │   │   └── bigquery.js                # 3 funciones para llamar al backend
│   │   └── components/
│   │       ├── KPIcards.jsx               # 4 tarjetas numericas de resumen
│   │       ├── BarChartTipos.jsx          # Barras: top 15 tipos de infraestructura
│   │       ├── PieChartTenencia.jsx       # Pastel: distribucion por tenencia
│   │       ├── TopCapacidades.jsx         # Barras: top 10 por capacidad total
│   │       ├── BarChartCaja.jsx           # Barras: registros por caja de compensacion
│   │       └── DataTable.jsx              # Tabla con los datos paginados
│   │
│   ├── dist/                              # Compilacion para produccion (npm run build)
│   └── node_modules/                      # Dependencias de Node.js
│
├── vercel.json                            # Configuracion para deploy en Vercel
├── README.md                              # Instrucciones rapidas
├── DOCUMENTACION.md                       # Documentacion detallada tecnica
└── EXPOSICION.md                          # Este archivo
```

---

## 4. Backend — dashboard-api/

### 4.1 conexion_bq.py (autenticacion con BigQuery)

```python
# Soporta DOS metodos de autenticacion:

# Metodo 1: Variable GOOGLE_CREDENTIALS_JSON (para Vercel)
# Se pasa el contenido completo del JSON como string
if "GOOGLE_CREDENTIALS_JSON" in os.environ:
    cred_info = json.loads(os.environ["GOOGLE_CREDENTIALS_JSON"])
    creds = service_account.Credentials.from_service_account_info(cred_info)
    cliente = bigquery.Client(credentials=creds, project=cred_info["project_id"])

# Metodo 2: Archivo JSON local (para desarrollo)
# Usa GOOGLE_APPLICATION_CREDENTIALS que apunta al archivo .json
else:
    cliente = bigquery.Client()
```

**Donde esta la clave:**
- **Local:** `C:\Users\ASUS\.google\credentials\mapaibague-94fefa456fd7.json`
- **Variable de entorno:** `GOOGLE_APPLICATION_CREDENTIALS` apunta a esa ruta
- **Vercel:** Se configura `GOOGLE_CREDENTIALS_JSON` en el dashboard con el JSON completo
- La clave NO esta en el repositorio por seguridad (se excluye con `.gitignore`)

### 4.2 main.py (servidor FastAPI — 3 endpoints)

```
FastAPI app
  │
  ├── GET /api/filtros
  │   └── 3 consultas paralelas: tipos, tenencias, cajas disponibles
  │
  ├── GET /api/resumen
  │   └── 6 consultas paralelas (ThreadPoolExecutor):
  │       ├── stats: total registros, capacidad total, tipos unicos, municipios
  │       ├── por_tipo: top 15 tipos de infraestructura
  │       ├── por_tenencia: distribucion de tenencia
  │       ├── capacidad_por_tipo: top 10 por capacidad
  │       ├── por_municipio: top 10 municipios
  │       └── por_caja: agrupacion por caja de compensacion
  │
  └── GET /api/datos
      └── Filas completas con paginacion (limit/offset) y filtros
```

Caracteristicas:
- **ThreadPoolExecutor:** ejecuta hasta 6 consultas SQL en paralelo, reduce tiempo de ~30s a ~5s
- **Filtros dinamicos:** construye clausula WHERE automaticamente segun parametros `?tipo=&tenencia=&caja=`
- **CORS abierto:** permite peticiones desde cualquier origen (necesario para desarrollo)
- **Sanitizacion:** funcion `escape()` para evitar inyeccion SQL en los filtros

---

## 5. Frontend — dashboard-ui/

### 5.1 Flujo de React

```
main.jsx
  └── App.jsx (estado global: resumen, datos, filtros, loading)
        ├── Carga inicial: getFiltros() + getResumen({}) + getDatos()
        ├── Filters (3 selects)
        │     └── onChange → recalcula todo con nuevo filtro
        ├── KPIcards (total registros, capacidad, tipos, municipios)
        ├── BarChartTipos (top 15 tipos)
        ├── PieChartTenencia (distribucion de tenencia)
        ├── TopCapacidades (top 10 por capacidad)
        ├── BarChartCaja (por caja de compensacion)
        └── DataTable (datos paginados)
```

### 5.2 Componentes y librerias

| Componente | Libreria | Que muestra |
|---|---|---|
| `KPIcards` | HTML + CSS | 4 tarjetas: registros, capacidad total, tipos unicos, municipios |
| `BarChartTipos` | chart.js (Bar horizontal) | Top 15 tipos de infraestructura |
| `PieChartTenencia` | chart.js (Doughnut) | Distribucion de tenencia (<1% se agrupa en "Otros") |
| `TopCapacidades` | chart.js (Bar horizontal) | Top 10 tipos con mayor capacidad total |
| `BarChartCaja` | chart.js (Bar vertical) | Registros agrupados por Caja de Compensacion |
| `DataTable` | HTML + CSS | Tabla con 5 columnas, scroll, hover |

### 5.3 api/bigquery.js (conexion backend)

```javascript
import axios from 'axios'

getResumen(filtros)   → GET /api/resumen?tipo=&tenencia=&caja=
getDatos(limit, offset, filtros) → GET /api/datos?limit=&offset=&tipo=...
getFiltros()          → GET /api/filtros
```

Todas las llamadas usan **axios** con rutas relativas (`/api/...`). Vite redirige esas peticiones al backend mediante el proxy configurado en `vite.config.js`.

### 5.4 vite.config.js (proxy)

```javascript
export default defineConfig({
  plugins: [react()],
  server: { proxy: { "/api": "http://localhost:8000" } }
})
```

Sin este proxy, el frontend en `localhost:5173` no podria hacer peticiones al backend en `localhost:8000` por politicas CORS del navegador.

---

## 6. Base de datos — Google BigQuery

- **Proyecto:** `mapaibague`
- **Dataset:** `PRESENTACION`
- **Tabla:** `PRESENTACION`
- **Registros:** ~281,605
- **Columnas principales:**

| Columna | Tipo | Ejemplo |
|---|---|---|
| `Tipo_Infraestructura` | String | "BIBLIOTECA", "COLEGIOS" |
| `Tenencia_Infraestructura` | String | "PROPIO", "ARRENDATARIO" |
| `Capacidad_Infraestructura` | Integer | 150, 80000 |
| `Caja_de_Compensacion` | String | "CAFAMAZ", "COMFENALCO" |
| `Codigo_Municipio` | String | 91001, 68001 |
| `Georreferenciacion_Latitud` | Float | 4.0, 7.2 |
| `Georreferenciacion_Longitud` | Float | -70.0, -73.5 |

---

## 7. Autenticacion con Google Cloud

### 7.1 Que es una Service Account

Es una "identidad de maquina" que Google Cloud otorga para que aplicaciones (no personas) accedan a recursos como BigQuery. Viene en un archivo JSON que contiene:

| Campo | Que es |
|---|---|
| `project_id` | ID del proyecto en Google Cloud |
| `private_key` | Llave privada RSA de 2048 bits |
| `client_email` | Correo electronico de la cuenta de servicio |
| `private_key_id` | Identificador de la llave |

### 7.2 Como se consume en el proyecto

**En local (desarrollo):**
1. El archivo JSON esta en `C:\Users\ASUS\.google\credentials\mapaibague-94fefa456fd7.json`
2. La variable `GOOGLE_APPLICATION_CREDENTIALS` apunta a esa ruta
3. `bigquery.Client()` lee automaticamente esa variable y se autentica
4. La clave esta FUERA del repositorio para no exponerla en git

**En Vercel (produccion):**
1. No se puede usar un archivo, se necesita la variable `GOOGLE_CREDENTIALS_JSON`
2. En el dashboard de Vercel → Settings → Environment Variables se pega el contenido completo del JSON
3. `conexion_bq.py` detecta la variable y crea las credenciales desde el JSON directamente

### 7.3 Seguridad

- La llave privada permite firmar tokens JWT (RS256) que Google valida
- Sin la llave, el backend NO puede consultar BigQuery
- El archivo JSON no debe compartirse ni subirse a repositorios publicos
- El `.gitignore` del proyecto excluye archivos `*.json` con credenciales

---

## 8. Tecnologias y librerias — para que sirve cada una

| Tecnologia | Version | Rol en el proyecto |
|---|---|---|
| **Python 3.13** | 3.13 | Lenguaje del backend |
| **FastAPI** | 0.116 | Framework web para crear los endpoints REST |
| **Uvicorn** | 0.37 | Servidor ASGI que ejecuta FastAPI |
| **google-cloud-bigquery** | 3.41 | Cliente oficial de Google para consultar BigQuery desde Python |
| **google-auth / google.oauth2** | — | Autenticacion con service account (OAuth 2.0 + RSA) |
| | | |
| **Node.js** | 18+ | Entorno de ejecucion del frontend |
| **React** | 18.3 | Biblioteca para construir la interfaz de usuario |
| **Vite** | 5.4 | Entorno de desarrollo rapido y empaquetador (build) |
| **Chart.js** | 4.4 | Biblioteca de graficas (barras, pastel/doughnut) |
| **react-chartjs-2** | 5.2 | Adaptador de Chart.js para React |
| **Axios** | 1.7 | Cliente HTTP para hacer peticiones al backend |

---

## 9. Ejecucion local paso a paso

### Requisitos previos
- Python 3.9+
- Node.js 18+
- Credenciales de BigQuery configuradas

### 1. Configurar credenciales (una sola vez)
```powershell
$env:GOOGLE_APPLICATION_CREDENTIALS = "C:\Users\ASUS\.google\credentials\mapaibague-94fefa456fd7.json"
setx GOOGLE_APPLICATION_CREDENTIALS "C:\Users\ASUS\.google\credentials\mapaibague-94fefa456fd7.json"
```
Cerrar y abrir terminal para que `setx` tenga efecto.

### 2. Iniciar backend (terminal 1)
```powershell
cd dashboard-api
pip install -r requirements.txt
python -m uvicorn main:app --reload --port 8000
```

### 3. Iniciar frontend (terminal 2)
```powershell
cd dashboard-ui
npm install
npm run dev
```

### 4. Abrir navegador
```
http://localhost:5173
```

### Endpoints de la API (para pruebas)
```
http://localhost:8000/api/filtros
http://localhost:8000/api/resumen
http://localhost:8000/api/datos?limit=10
```

---

## 10. Deploy en Vercel

### Estructura para Vercel

```
api/index.py         → Entry point que importa la app desde dashboard-api/main.py
api/requirements.txt → Dependencias del backend para Vercel
vercel.json          → Configuracion de builds y rutas
```

### vercel.json
```json
{
  "builds": [
    { "src": "api/index.py", "use": "@vercel/python" },
    { "src": "dashboard-ui/package.json", "use": "@vercel/static-build", "config": { "distDir": "dist" } }
  ],
  "routes": [
    { "handle": "filesystem" },
    { "src": "/api/(.*)", "dest": "api/index.py" }
  ]
}
```

### Configuracion necesaria en Vercel
- Agregar variable de entorno: `GOOGLE_CREDENTIALS_JSON` con el contenido completo del JSON de service account
- El frontend compilado (de `npm run build`) se sirve automaticamente desde `dashboard-ui/dist`

---

## 11. Posibles problemas y soluciones

| Problema | Causa | Solucion |
|---|---|---|
| Pantalla en blanco / skeletons infinitos | Backend no corre o no responde | Verificar que uvicorn este ejecutandose en puerto 8000 |
| Error "Project not determined" | Faltan credenciales de BigQuery | Configurar GOOGLE_APPLICATION_CREDENTIALS o GOOGLE_CREDENTIALS_JSON |
| Error CORS en consola del navegador | Frontend no usa el proxy de Vite | Verificar vite.config.js y que el frontend acceda por localhost:5173 |
| npm run dev no funciona | Falta node_modules | Ejecutar `npm install` en dashboard-ui |
| pip install falla | Python no instalado o version incorrecta | Verificar Python 3.9+ con `python --version` |
