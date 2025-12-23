# Lecciones Aprendidas: Ambientes Docker para Desarrollo

**Documento interno para Claude - Referencia obligatoria antes de armar cualquier ambiente Docker**

---

## Filosofia Fundamental

### El objetivo es CERO dependencias en la maquina del developer

El desarrollador solo debe tener Docker instalado. Nada mas:
- NO instalar Node.js
- NO instalar Java/Maven
- NO instalar MySQL
- NO instalar npm globalmente
- NO ejecutar builds localmente

**Todo se hace DENTRO de los contenedores.**

El codigo fuente se monta desde el host, pero la compilacion, instalacion de dependencias y ejecucion ocurren dentro de Docker.

---

## Proceso Correcto (En Este Orden)

### Paso 1: Extraer Informacion del Codigo Fuente

**ANTES de preguntar nada, analizar el proyecto yo mismo.**

#### Frontend - Buscar y extraer:

```bash
# Version de Node requerida (si esta especificada)
cat package.json | grep -A5 "engines"

# Dependencias - buscar paquetes que parezcan internos (no estan en npm publico)
cat package.json | grep -i "dependencies" -A50 | head -60

# Si existe package-lock.json, verificar si tiene URLs a registries internos
grep -E "http://[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" package-lock.json | head -5

# Como se inicia el servidor de desarrollo
cat package.json | grep -A5 '"scripts"'

# En que puerto escucha (buscar en server.js, webpack.config.js, etc)
grep -rE "port|listen|\.listen\(" *.js webpack*.js 2>/dev/null | head -10

# Verificar si escucha en localhost o 0.0.0.0
grep -E "localhost|127\.0\.0\.1|0\.0\.0\.0" server.js webpack*.js 2>/dev/null
```

#### Backend Java - Buscar y extraer:

```bash
# Version de Java
grep -E "<source>|<target>|<java.version>|<maven.compiler" pom.xml | head -5

# Repositorios Maven (pueden ser internos)
grep -B2 -A5 "<repository>" pom.xml

# Nombre del artefacto y tipo de empaquetado
grep -E "<artifactId>|<packaging>|<version>" pom.xml | head -6

# Dependencias que parezcan internas
grep -E "<groupId>" pom.xml | grep -v "org\.\|com\.sun\|javax\.\|junit\|apache\|hibernate\|spring\|jersey\|mysql\|slf4j\|log4j"

# Archivos de configuracion
find . -name "*.properties" -o -name "persistence.xml" -o -name "log4j*.xml" 2>/dev/null

# Configuracion de base de datos
grep -rE "jdbc:|hibernate\.dialect|mysql|postgresql|oracle" src/main/resources/ 2>/dev/null
```

#### Base de Datos - Determinar:

```bash
# Buscar dialect de Hibernate o driver JDBC
grep -rE "dialect|Driver" src/main/resources/ 2>/dev/null

# Nombre de la base de datos
grep -rE "database|schema|catalog" src/main/resources/ 2>/dev/null
```

### Paso 2: Preguntar lo que NO Pude Obtener

**Solo despues de analizar el codigo, preguntar al usuario:**

#### Preguntas sobre versiones (si no pude determinarlas):
- "No encontre la version de Node especificada en package.json. Cual debo usar?"
- "El pom.xml indica Java 8, es correcto o hay alguna restriccion adicional?"
- "El proyecto usa MySQL segun el dialect de Hibernate. Que version especifica necesito? (5.7, 8.0, etc)"

#### Preguntas sobre repositorios internos:
- "Detecte estas dependencias que parecen internas: [lista]. Hay un Nexus u otro repositorio donde estan publicadas? Cual es la URL?"
- "El package-lock.json referencia este registry: [URL]. Sigue siendo valido o cambio?"

#### Preguntas sobre configuracion:
- "Hay un modo de desarrollo sin autenticacion LDAP? Cual es la propiedad para activarlo?"
- "Donde esta el dump de la base de datos para desarrollo?"

#### Preguntas sobre referencias:
- "Tenes algun proyecto Docker similar ya funcionando que pueda usar como referencia?"

### Paso 3: Validar Dudas

Si algo me genera dudas aunque haya encontrado la informacion, **preguntar para validar:**

- "Encontre que el frontend escucha en el puerto 7000, pero esta en localhost. Debo cambiarlo a 0.0.0.0 para que sea accesible desde fuera del contenedor, correcto?"
- "El driver JDBC es mysql-connector-java version X. Esto es compatible con MySQL 8 o necesito usar 5.7?"
- "Veo que el frontend tiene la URL del backend hardcodeada. La modifico para usar una variable de entorno (ej: REACT_APP_BACKEND_URL)?"

---

## Configuraciones Criticas (Aplican a Casi Todos los Proyectos)

### Frontend - El servidor debe escuchar en 0.0.0.0

```javascript
// INCORRECTO - solo accesible desde dentro del contenedor
.listen(7000, 'localhost', ...)

// CORRECTO - accesible desde el host
.listen(7000, '0.0.0.0', ...)
```

### Frontend - URL del backend configurable via variable de entorno

En lugar de usar proxy, modificar el archivo de configuracion (ej: `config.js`) para que use una variable de entorno:

```javascript
// INCORRECTO - URL hardcodeada
const SERVER = 'http://localhost:8080/mi-app/rest';

// CORRECTO - Configurable via variable de entorno
const SERVER = process.env.REACT_APP_BACKEND_URL || 'http://localhost:8080/mi-app/rest';
```

Luego en `docker-compose.yml` definir la variable:

```yaml
frontend:
  environment:
    - REACT_APP_BACKEND_URL=http://localhost:8080/mi-app/rest
```

**Nota:** El backend debe exponer el puerto al host (ej: `8080:8080`) para que el navegador pueda acceder directamente.

### npm - Crear .npmrc si hay registry interno

```
registry=http://[IP_NEXUS]:[PUERTO]/repository/npm-group/
```

### Maven - Crear settings.xml si hay repositorio interno

```xml
<mirrors>
    <mirror>
        <id>nexus-mirror</id>
        <mirrorOf>*</mirrorOf>
        <url>http://[IP_NEXUS]:[PUERTO]/repository/maven-public/</url>
    </mirror>
</mirrors>
```

### Volumen para node_modules (evitar problemas en macOS)

```yaml
volumes:
  - ../proyecto-frontend:/workspace:cached
  - /workspace/node_modules  # Volumen anonimo, evita conflictos
```

---

## Errores Que NUNCA Debo Repetir

### 1. Asumir Versiones Sin Verificar
**NO:** "Voy a usar MySQL 8.0 porque es la ultima"
**SI:** Verificar el driver JDBC y preguntar si hay dudas sobre compatibilidad

**Ejemplo real:** Asumi MySQL 8.0, pero el driver JDBC era viejo y fallo con `Unknown system variable 'query_cache_size'`. MySQL 5.7 era la version correcta.

### 2. Ignorar el package-lock.json
**NO:** Dejar el package-lock.json con URLs a registries viejos
**SI:** Si el registry cambio, eliminar package-lock.json para que npm genere uno nuevo

**Ejemplo real:** El lockfile tenia checksums de un Nexus en IP .200, pero el Nexus actual estaba en .205. npm fallaba con errores de integridad.

### 3. No Crear .npmrc Desde el Inicio
**NO:** Esperar a que falle npm install para darme cuenta que hay paquetes internos
**SI:** Revisar package.json buscando dependencias que no esten en npm publico y crear .npmrc preventivamente

### 4. Probar Soluciones al Azar
**NO:** Cambiar configuraciones sin entender el error
**SI:** Leer el log completo, identificar la causa raiz, aplicar una solucion especifica

**Ejemplo real:** Perdi tiempo cambiando volumenes de Docker cuando el problema real era un checksum incorrecto en package-lock.json.

### 5. No Verificar Puerto de Escucha
**NO:** Asumir que webpack-dev-server escucha en 0.0.0.0
**SI:** Verificar server.js y modificar si escucha en localhost

### 6. No Configurar la URL del Backend para Docker
**NO:** Dejar URLs hardcodeadas que solo funcionan en un ambiente
**SI:** Modificar config.js para usar variable de entorno (REACT_APP_BACKEND_URL) y definirla en docker-compose.yml

### 7. No Buscar Proyectos de Referencia
**NO:** Crear todo desde cero inventando la estructura
**SI:** Pedir al usuario un proyecto Docker similar que ya funcione y copiar la estructura

---

## Checklist Pre-Entrega

Antes de decir que el ambiente esta listo:

- [ ] `docker-compose up -d` levanta todos los servicios sin errores
- [ ] `docker-compose ps` muestra todos como "healthy" o "running"
- [ ] Frontend responde en el navegador
- [ ] Backend responde (puede ser 404 en root, pero no connection refused)
- [ ] Login funciona desde el navegador (no solo curl)
- [ ] Hot-reload del frontend funciona (modificar un archivo y ver el cambio)
- [ ] Documentacion DOCKER.md creada con instrucciones claras

---

## Tiempo Objetivo

**Un ambiente Docker debe estar funcionando en MAXIMO 1 hora.**

Distribucion:
- 15 min: Analisis del codigo fuente
- 5 min: Preguntas al usuario sobre lo que no pude determinar
- 20 min: Crear archivos (copiando de referencia si hay)
- 20 min: Testing y ajustes finales

**Si paso de 1 hora, debo:**
1. Parar
2. Revisar este documento
3. Identificar que paso por alto
4. Preguntar al usuario que informacion me falta
