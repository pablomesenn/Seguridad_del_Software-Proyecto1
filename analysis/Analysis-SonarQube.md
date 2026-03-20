# Reporte de Análisis Estático con SonarQube

**Instituto Tecnológico de Costa Rica — Escuela de Ingeniería en Computación**

| Campo | Valor |
|---|---|
| Asignatura | IC8071 — Seguridad del Software |
| Profesor | Herson Esquivel Vargas |
| Herramienta | SonarQube Community Build v25.8.0.112029 |
| Plugin C/C++ | sonar-cxx v2.1.1.2864 |
| Plataforma | Docker (localhost:9000) |
| Fecha del análisis | 19 de marzo de 2026 |
| Fecha del reporte | 20 de marzo de 2026 |

> Proyecto 1: Identificación de debilidades mediante análisis estático (SAST) — Valor: 15%

---

## Tabla de contenidos

1. [Introducción](#1-introducción)
2. [Software analizado](#2-software-analizado)
3. [Configuración del entorno](#3-configuración-del-entorno)
4. [Resultados](#4-resultados)
5. [Observaciones y limitaciones](#5-observaciones-y-limitaciones)
6. [Conclusiones](#6-conclusiones)
7. [Anexo: Capturas de pantalla](#7-anexo-capturas-de-pantalla)

---

## 1. Introducción

Este reporte documenta la configuración y ejecución de SonarQube Community mediante Docker para el análisis estático de seguridad (SAST) del código fuente de **ImageMagick**, una biblioteca de código abierto para crear, editar y convertir imágenes en más de 200 formatos.

El trabajo forma parte del Proyecto 1 del curso IC8071 del ITCR, cuyo objetivo es identificar debilidades de software usando herramientas automatizadas bajo el marco de referencia Mitre CWE. Dado que SonarQube Community no incluye soporte nativo para C/C++, fue necesario instalar el plugin **sonar-cxx** antes de ejecutar el análisis.

---

## 2. Software analizado

| Campo | Valor |
|---|---|
| Nombre | ImageMagick |
| Versión | 1.0 (rama `main`) |
| Lenguaje | C / C++ |
| Líneas de código | 273,155 |
| Módulos | `coders`, `filters`, `Magick++`, `MagickCore`, `MagickWand` |
| Fuente | `apt-get source imagemagick` |

---

## 3. Configuración del entorno

### Paso 1: Iniciar SonarQube con Docker

Se levantó el contenedor usando la imagen oficial de la edición Community:

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

La interfaz quedó disponible en `http://localhost:9000`. La instancia usa una base de datos embebida, adecuada para evaluación pero no recomendada para producción (SonarQube lo indica con una advertencia en la interfaz).

### Paso 2: Instalación del plugin sonar-cxx

SonarQube Community Edition no tiene soporte nativo para C/C++; ese soporte está reservado para las ediciones Developer y Enterprise. El plugin de código abierto **sonar-cxx** cubre esta necesidad y fue necesario instalarlo antes de analizar ImageMagick.

#### Descarga

Se descargó el archivo `.jar` desde el repositorio oficial en GitHub:

```bash
wget https://github.com/SonarOpenCommunity/sonar-cxx/releases/\
download/cxx-2.1.1/sonar-cxx-plugin-2.1.1.2864.jar
```

La versión 2.1.x es compatible con SonarQube 9.x y posteriores, incluyendo la Community Build v25.8.0.112029. La tabla de compatibilidad está en el repositorio del plugin en GitHub.

#### Instalación

El `.jar` se copió al directorio de extensiones del contenedor y se reinició:

```bash
docker cp sonar-cxx-plugin-2.1.1.2864.jar \
  sonarqube:/opt/sonarqube/extensions/plugins/

docker restart sonarqube
```

Como alternativa, se puede montar el directorio como volumen al crear el contenedor para evitar repetir la copia si el contenedor se recrea:

```bash
docker run -d --name sonarqube -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  -v /ruta/local/plugins:/opt/sonarqube/extensions/plugins \
  sonarqube:community
```

#### Verificación

La carga exitosa del plugin se confirma en los logs del contenedor:

```bash
docker logs sonarqube | grep -i "cxx"
# [INFO] Deploy plugin C++ (Community) / 2.1.1.2864
```

También aparece en **Administration → Marketplace → Installed** como *C++ (Community)*.

#### Reglas que aporta sonar-cxx

El plugin registra el perfil de calidad *Sonar way for C++* con reglas en las siguientes categorías:

| Categoría | Ejemplos |
|---|---|
| Manejo de memoria | Desbordamiento de búfer, punteros nulos, use-after-free |
| Funciones inseguras | `strcpy`, `sprintf`, `gets` sin límite |
| Inicialización | Variables usadas sin inicializar |
| Aritmética | Desbordamiento de enteros, conversiones inseguras |
| Integración externa | Resultados de Cppcheck, Valgrind, Vera++, RATS |

### Paso 3: Creación del proyecto

Desde la interfaz web se creó el proyecto con nombre **ImageMagick**, se generó un token de autenticación, se fijó la rama principal como `main` y se seleccionó el perfil de calidad *Sonar way for C++* habilitado por sonar-cxx.

### Paso 4: Configuración del scanner

En el directorio raíz del código fuente se creó el archivo `sonar-project.properties`:

```properties
sonar.projectKey=imagemagick
sonar.projectName=ImageMagick
sonar.projectVersion=1.0
sonar.sources=.
sonar.host.url=http://localhost:9000
sonar.token=<TOKEN>

# Propiedades para sonar-cxx (C/C++)
sonar.language=c++
sonar.cxx.file.suffixes=.c,.cpp,.cxx,.cc,.h,.hpp,.hxx
sonar.cxx.errorRecoveryEnabled=true
```

La propiedad `errorRecoveryEnabled` permite que el analizador continúe procesando aunque encuentre errores de parseo en archivos con sintaxis no estándar, útil en proyectos grandes como ImageMagick.

### Paso 5: Ejecución del análisis

```bash
sonar-scanner \
  -Dsonar.projectKey=imagemagick \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=<TOKEN> \
  -Dsonar.language=c++ \
  -Dsonar.cxx.file.suffixes=.c,.cpp,.cxx,.cc,.h,.hpp,.hxx
```

El análisis tardó aproximadamente 22 minutos. Durante esa sesión se realizaron cinco ejecuciones incrementales entre las 5:58 PM y las 6:20 PM del 19 de marzo de 2026.

---

## 4. Resultados

### Historial de análisis y Quality Gate

El proyecto superó el Quality Gate en las cinco ejecuciones:

| Hora | Resultado | Notas |
|---|---|---|
| 5:58 PM (primer análisis) | ✅ Passed | 0 issues, 0.0% Cov., 0.0% Dup. |
| 6:03 PM | ✅ Passed | +0 issues |
| 6:07 PM | ✅ Passed | +0 issues |
| 6:13 PM | ✅ Passed | +0 issues |
| 6:20 PM (v1.0) | ✅ Passed | +0 issues, +5.0% Dup. (advertencia) |

El incremento de duplicaciones en el último análisis generó una advertencia visible en la interfaz, pero no cambió el resultado del gate.

### Métricas generales

| Métrica | Nuevo código | Total |
|---|---|---|
| Issues | 0 | 0 |
| Security Hotspots | 0 | 0 |
| Cobertura | No computada | No computada |
| Duplicaciones | No computada | 5.0% |
| Líneas de código | — | 273,155 |

La cobertura no fue calculada porque no se ejecutaron pruebas instrumentadas. Para obtenerla se requeriría integrar `gcov` y pasar los reportes al scanner.

### Distribución por módulo

| Módulo | LoC | Seguridad | Confiab. | Mant. | Dup. |
|---|---:|:---:|:---:|:---:|---:|
| coders | 92,164 | 0 | 0 | 0 | 5.0% |
| filters | 154 | 0 | 0 | 0 | 0.0% |
| Magick++ | 21,617 | 0 | 0 | 0 | 0.1% |
| MagickCore | 124,854 | 0 | 0 | 0 | 3.5% |
| MagickWand | 34,366 | 0 | 0 | 0 | 13.8% |
| **Total** | **273,155** | 0 | 0 | 0 | 5.0% |

`MagickCore` es el módulo más grande (45.7% del total). `MagickWand` tiene el mayor porcentaje de duplicación (13.8%), esperable en bibliotecas C con patrones repetitivos para múltiples formatos de imagen.

### Issues por categoría

La sección Issues confirmó cero problemas en todas las categorías ("No Issues. Hooray!"):

| Categoría | Cantidad |
|---|:---:|
| Security / Reliability / Maintainability | 0 |
| Blocker / High / Medium / Low / Info | 0 |
| Consistency / Intentionality / Adaptability | 0 |
| **Total** | **0** |

---

## 5. Observaciones y limitaciones

- SonarQube Community, incluso con el plugin sonar-cxx, no realiza *taint analysis* profundo. Ese nivel de análisis está disponible en las ediciones Developer y Enterprise. El plugin complementa con reglas específicas de C/C++ pero no es equivalente.
- Cero issues no descarta debilidades. SonarQube es susceptible a falsos negativos en código C con manejo manual de memoria (desbordamientos de búfer, use-after-free). Es necesario complementar con Semgrep y LLMs para una cobertura más amplia frente a CWEs relevantes en C/C++.
- El análisis cubre únicamente el código fuente de ImageMagick, sin dependencias externas.

---

## 6. Conclusiones

El análisis de 273,155 líneas de código de ImageMagick con SonarQube Community y el plugin sonar-cxx arrojó **0 issues** y **0 Security Hotspots**, con el Quality Gate aprobado en los cinco análisis realizados. La única observación fue el incremento de duplicaciones en el análisis final (+5%), que generó una advertencia pero no afectó el gate.

Los resultados deben interpretarse con cautela: la ausencia de issues no implica que el código esté libre de debilidades de seguridad. Este análisis es un primer paso que debe complementarse con Semgrep y revisión mediante LLMs para cobertura más amplia.

---

## 7. Anexo: Capturas de pantalla

- **Capturas 1 y 2:** Vista general del proyecto con Quality Gate aprobado, 0 nuevos issues y 0 Security Hotspots.
- **Captura 3:** Historial de actividad con los cinco análisis del 19 de marzo (5:58 — 6:20 PM).
- **Captura 4:** Sección Issues mostrando "No Issues. Hooray!" en todas las categorías.
- **Captura 5:** Vista Code con la distribución de LoC y métricas por módulo.
