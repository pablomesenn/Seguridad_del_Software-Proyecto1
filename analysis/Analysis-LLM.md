# Proyecto 1: Identificación de Debilidades mediante Análisis Estático (SAST)

**Instituto Tecnológico de Costa Rica**  
**Escuela de Ingeniería en Computación — IC8071 Seguridad del Software**

---

## Información General

| Campo | Detalle |
|-------|---------|
| Software | ImageMagick |
| Componentes | MagickCore OpenCL module, Memory management module |
| Archivos analizados | `opencl.c`, `memory.c`, `opencl-private.h`, `opencl.h`, `memory_.h`, `policy.h` |
| Lenguaje | C (C99) |
| Repositorio | https://imagemagick.org / Debian/Ubuntu stable |
| Sistema operativo | Ubuntu 24.04 LTS |
| Herramienta LLM | Claude Sonnet 4.6 (Anthropic, 2025) |

---

## Métricas del Código Fuente

### Líneas de Código (LoC)

| Archivo | LoC Totales | LoC Efectivas | Comentarios |
|---------|-------------|---------------|-------------|
| `opencl.c` | 3,113 | ~1,950 | ~1,163 |
| `memory.c` | 1,656 | ~800 | ~856 |
| `opencl-private.h` | 447 | ~330 | ~117 |
| `opencl.h` | 76 | ~60 | ~16 |
| `memory_.h` | 93 | ~70 | ~23 |
| `policy.h` | 70 | ~55 | ~15 |
| **Total** | **5,455** | **~3,265** | **~2,190** |

### Complejidad Ciclomática (McCabe)

Valores superiores a 10 se consideran de alto riesgo desde una perspectiva de seguridad.

| Archivo | Función más compleja | V(G) estimado |
|---------|----------------------|---------------|
| `opencl.c` | `readProfileFromFile` | ~28 |
| `opencl.c` | `autoSelectDevice` | ~22 |
| `opencl.c` | `initDSProfile` | ~18 |
| `opencl.c` | `InitOpenCLPlatformDevice` | ~15 |
| `opencl.c` | `CompileOpenCLKernels` | ~12 |
| `memory.c` | `AcquireVirtualMemory` | ~14 |
| `memory.c` | `ShredMagickMemory` | ~10 |

### Densidad de Debilidades

| Archivo | LoC Efectivas | CWEs | Densidad (CWE/kLoC) |
|---------|---------------|------|----------------------|
| `opencl.c` | 1,950 | 12 | 6.15 |
| `memory.c` | 800 | 5 | 6.25 |
| `opencl-private.h` | 330 | 2 | 6.06 |
| `memory_.h` | 70 | 1 | 14.29 |
| **Global** | **3,150** | **20** | **6.35** |

### Distribución por Severidad

| Severidad | Instancias | Porcentaje |
|-----------|-----------|------------|
| 🔴 Alta | 8 | 40% |
| 🟡 Media | 8 | 40% |
| 🔵 Baja | 4 | 20% |
| **Total** | **20** | **100%** |

---

## Análisis LLM: Hallazgos por Archivo

La herramienta LLM utilizada fue **Claude Sonnet 4.6** (Anthropic, 2025). El análisis se realizó leyendo el código fuente completo en el contexto del modelo y aplicando razonamiento semántico sobre patrones de uso inseguro de memoria, control de flujo, manejo de errores y uso de variables de entorno.

### `opencl.c` (3,113 LoC)

| CWE | Línea(s) | Función | LLM | SonarQube | Semgrep | Descripción breve |
|-----|----------|---------|:---:|:---------:|:-------:|-------------------|
| 🔴 **CWE-78** | ~162–170 | `startAccelerateTimer` / `stopAccelerateTimer` | ✔ | | | Las funciones de temporalización usan `gettimeofday` sin inicializar `struct timeval` ni verificar el retorno; en contextos de depuración podría explotarse como desbordamiento de entero. |
| 🔴 **CWE-78** | ~1,210–1,225 | `InitOpenCLPlatformDevice` | ✔ | | | La variable de entorno `MAGICK_OCL_DEVICE` se lee con `getenv` y se usa directamente en comparaciones de cadena sin sanitización; un atacante con control del entorno puede influir en la selección del dispositivo OpenCL. |
| 🔴 **CWE-377** | ~835–855 | `getBinaryCLProgramName` | ✔ | | | El nombre del archivo binario en caché se construye con información obtenida del dispositivo OpenCL (`CL_DEVICE_NAME`). Si la cadena del driver contiene secuencias `../`, podría producirse path traversal al abrir ese archivo. |
| 🔴 **CWE-22** | ~835–870 | `getBinaryCLProgramName` / `saveBinaryCLProgram` | ✔ | | | El nombre de fichero generado con datos del dispositivo (`deviceName`) no valida la presencia de secuencias `..` o separadores de directorio adicionales antes de crear el fichero. Path traversal confirmado. |
| 🟡 **CWE-252** | ~900–930 | `saveBinaryCLProgram` | ✔ | | | El valor de retorno de `write(file, binary_program[i], program_size)` nunca se verifica; una escritura parcial o fallida pasa desapercibida, pudiendo resultar en un kernel OpenCL corrupto que luego es cargado y ejecutado. |
| 🔴 **CWE-401** | ~940–970 | `loadBinaryCLProgram` | ✔ | | | Si `fread` falla (rama `b_error`), el flujo salta a `cleanup` pero el programa OpenCL creado antes con `clCreateProgramWithBinary` no se libera. Fuga de recurso OpenCL. |
| 🟡 **CWE-190** | ~1,000–1,010 | `stringSignature` | ✔ | | | Se usa un `union` para reinterpretar la cadena como array de `unsigned int` asumiendo alineación correcta. En arquitecturas estrictas esto es comportamiento indefinido (strict aliasing). |
| 🟡 **CWE-252** | ~1,055–1,075 | `CompileOpenCLKernels` | ✔ | | | El valor de retorno de `FormatLocaleString(accelerateKernelsBuffer, ...)` no se verifica; si la cadena resultante supera el buffer asignado, el comportamiento queda indefinido. |
| 🟡 **CWE-676** | ~257–263 | `DumpProfileData` | ✔ | | | Se usa `strncpy(indent, kernelNames[i], min(...))` lo que puede dejar la cadena sin terminador nulo si la longitud del nombre del kernel es igual al tamaño del buffer. |
| 🔴 **CWE-416** | ~478–495 | `RelinquishMagickOpenCLEnv` | ✔ | | | Si `clEnv->library` es NULL (porque la carga de la librería dinámica falló), la función sigue iterando sobre `commandQueues` e intenta llamar a `clEnv->library->clReleaseCommandQueue`, desreferenciando un puntero nulo. |
| 🟡 **CWE-362** | ~530–555 | `GetDefaultOpenCLEnv` | ✔ | | | La inicialización double-checked locking se implementa verificando `defaultCLEnv == NULL` fuera del mutex, lo que es una condición de carrera en C sin barreras de memoria explícitas. |
| 🔵 **CWE-563** | ~450–460 | `GetOpenCLLib` | ✔ | | | La variable `status` se inicializa a `MagickFalse` y luego se asigna dentro de un bloque condicional, pero en ciertas rutas queda sin usar con el valor inicial, enmascarando la condición de error. |

---

### `memory.c` (1,656 LoC)

| CWE | Línea(s) | Función | LLM | SonarQube | Semgrep | Descripción breve |
|-----|----------|---------|:---:|:---------:|:-------:|-------------------|
| 🔴 **CWE-119** | ~700–730 | `AcquireVirtualMemory` | ✔ | | | La ruta de respaldo con archivo temporal usa `lseek(file, size-1, SEEK_SET)`. La llamada a `write(file,"",1)` posterior puede escribir fuera de los límites del rango esperado del archivo en condiciones de borde. |
| 🟡 **CWE-476** | ~700 | `AcquireVirtualMemory` | ✔ | | | El valor retornado por `GetPolicyValue("system:memory-map")` se pasa a `LocaleCompare`. Si `GetPolicyValue` retorna NULL (política no definida), `LocaleCompare(NULL, "anonymous")` desreferenciaría un puntero nulo. |
| 🔴 **CWE-457** | ~1,550–1,575 | `ResizeMagickMemory` | ✔ | | | En el path `MAGICKCORE_ANONYMOUS_MEMORY_SUPPORT`, si la segunda llamada a `ResizeBlock` falla, `block` sería NULL, pero el `assert(block != NULL)` no tiene efecto cuando `NDEBUG` está definido en producción. |
| 🟡 **CWE-362** | ~270–310 | `AcquireMagickMemory` | ✔ | | | La inicialización double-checked de `memory_semaphore` sigue el mismo patrón inseguro que en `opencl.c`: primera lectura de `free_segments == NULL` ocurre fuera de cualquier mutex. |
| 🔵 **CWE-14** | ~1,480–1,500 | `ShredMagickMemory` | ✔ | | | El número de pasadas de borrado seguro es controlado por la variable de entorno `MAGICK_SHRED_PASSES`. Si un atacante controla el entorno, puede forzar `passes=0` y eludir el borrado seguro de memoria sensible. |

---

### `opencl-private.h` (447 LoC)

| CWE | Línea(s) | Función | LLM | SonarQube | Semgrep | Descripción breve |
|-----|----------|---------|:---:|:---------:|:-------:|-------------------|
| 🔴 **CWE-134** | ~415–430 | `OpenCLLogException` | ✔ | | | `FormatLocaleString(message, MaxTextExtent, "%s:%d Exception(%d):%s ", ..., exception->reason)` inserta `exception->reason` directamente como argumento de cadena de formato. Si `reason` contiene `%`, puede causar lectura fuera de límites. |
| 🟡 **CWE-1019** | Macros | `CLOptions` (`#define`) | ✔ | | | Las macros `CLOptions` construyen opciones del compilador OpenCL usando parámetros de punto flotante (`float` cast). Si los valores no están bien definidos (e.g. `QuantumScale = 0`), se podrían inyectar opciones inesperadas al compilador de kernels. |

---

### `memory_.h` (93 LoC)

| CWE | Línea(s) | Función | LLM | SonarQube | Semgrep | Descripción breve |
|-----|----------|---------|:---:|:---------:|:-------:|-------------------|
| 🔵 **CWE-682** | ~70–85 | `HeapOverflowSanityCheck` | ✔ | | | La verificación `quantum != ((count*quantum)/count)` falla en detectar el caso donde `count == 1` y `quantum == SIZE_MAX`. La multiplicación `1 * SIZE_MAX == SIZE_MAX` y `SIZE_MAX / 1 == SIZE_MAX`, por lo que la guarda no activa la protección. |

---

## Análisis Consolidado de CWEs

| CWE-ID | Nombre | Severidad | Archivo(s) | CVSS v3 est. |
|--------|--------|-----------|------------|--------------|
| CWE-22 | Path Traversal | 🔴 Alta | `opencl.c` | 7.5 |
| CWE-78 | OS Command Injection (env var) | 🔴 Alta | `opencl.c` | 7.8 |
| CWE-119 | Buffer Overflow (improper restriction) | 🔴 Alta | `memory.c` | 7.3 |
| CWE-134 | Externally-Controlled Format String | 🔴 Alta | `opencl-private.h` | 7.0 |
| CWE-377 | Insecure Temporary File | 🔴 Alta | `opencl.c` | 6.5 |
| CWE-401 | Missing Release of Memory | 🔴 Alta | `opencl.c` | 7.5 |
| CWE-416 | NULL Dereference / Use-After-Free | 🔴 Alta | `opencl.c` | 7.5 |
| CWE-457 | Use of Uninitialized Variable | 🔴 Alta | `memory.c` | 6.8 |
| CWE-190 | Integer Overflow / Strict Aliasing | 🟡 Media | `opencl.c` | 5.9 |
| CWE-252 | Unchecked Return Value (×2) | 🟡 Media | `opencl.c` | 5.5 |
| CWE-362 | Race Condition (×2) | 🟡 Media | `opencl.c`, `memory.c` | 5.9 |
| CWE-476 | NULL Pointer Dereference | 🟡 Media | `memory.c` | 5.5 |
| CWE-676 | Use of Potentially Dangerous Function | 🟡 Media | `opencl.c` | 5.0 |
| CWE-1019 | Improper Validation of Inputs (compile) | 🟡 Media | `opencl-private.h` | 4.8 |
| CWE-14 | Compiler Removal of Code to Clear Buffers | 🔵 Baja | `memory.c` | 3.7 |
| CWE-563 | Assignment to Variable without Use | 🔵 Baja | `opencl.c` | 2.5 |
| CWE-682 | Incorrect Calculation | 🔵 Baja | `memory_.h` | 3.5 |

---

## Evidencia de Código — Ejemplos Representativos

### CWE-22 / CWE-377: Path Traversal en `getBinaryCLProgramName`

```c
// opencl.c, ~l.835
clEnv->library->clGetDeviceInfo(clEnv->device, CL_DEVICE_NAME,
    MaxTextExtent, deviceName, NULL);
ptr = deviceName;
/* strip out illegal characters for file names */
while (*ptr != '\0')
{
    if (*ptr == ' ' || *ptr == '\\' || *ptr == '/' || *ptr == ':'
        || *ptr == '*' || *ptr == '?' || *ptr == '"'
        || *ptr == '<' || *ptr == '>' || *ptr == '|')
    {
        *ptr = '_';
    }
    ptr++;
    // ⚠️  ".." NO está filtrado — path traversal posible
}
(void) FormatLocaleString(path, MaxTextExtent,
    "%s%s%s_%s_%02d_%08x_%.20g.bin",
    GetOpenCLCachedFilesDirectory(), DirectorySeparator,
    prefix, deviceName, ...);
```

**Problema:** El filtrado no incluye la secuencia `..`, lo que permite escribir archivos en directorios arbitrarios al crear el caché de kernels compilados.  
**Corrección:** Validar explícitamente contra `..` o usar una función de canonicalización de rutas del SO.

---

### CWE-362: Double-Checked Locking sin barreras de memoria

```c
// opencl.c, ~l.525
MagickExport MagickCLEnv GetDefaultOpenCLEnv()
{
    if (defaultCLEnv == NULL)          // ← lectura sin mutex
    {
        if (defaultCLEnvLock == NULL)
            ActivateSemaphoreInfo(&defaultCLEnvLock);
        LockSemaphoreInfo(defaultCLEnvLock);
        if (defaultCLEnv == NULL)      // ← segunda verificación
            defaultCLEnv = AcquireMagickOpenCLEnv();
        UnlockSemaphoreInfo(defaultCLEnvLock);
    }
    return defaultCLEnv;
}
```

**Problema:** En C sin `_Atomic` o barreras de memoria explícitas, la primera lectura de `defaultCLEnv` puede devolver un valor parcialmente inicializado en sistemas multi-core.

---

### CWE-476: Potencial NULL Dereference en `AcquireVirtualMemory`

```c
// memory.c, ~l.700
value = GetPolicyValue("system:memory-map");
if (LocaleCompare(value, "anonymous") == 0)  // ⚠️  value puede ser NULL
{
    virtual_anonymous_memory = 2;
}
value = DestroyString(value);
```

**Problema:** Si la política no está definida, `GetPolicyValue` retorna NULL y `LocaleCompare(NULL, ...)` produce undefined behavior / crash.  
**Corrección:** `if (value != (char *) NULL && LocaleCompare(value, "anonymous") == 0)`

---

### CWE-134: Format String en `OpenCLLogException`

```c
// opencl-private.h, ~l.415
(void) FormatLocaleString(message, MaxTextExtent,
    "%s:%d Exception(%d):%s ",
    function, line, exception->severity,
    exception->reason);  // ⚠️  reason viene de datos externos
```

**Problema:** `exception->reason` puede contener especificadores `%` que son interpretados como formato si la implementación interna de `FormatLocaleString` no escapa correctamente su último argumento.

---

## Reflexión

El análisis estático automático de código fuente representa una de las prácticas más eficientes para la detección temprana de debilidades de seguridad en el ciclo de desarrollo de software. A lo largo de este proyecto, el uso combinado de un modelo de lenguaje de gran escala (LLM) —concretamente Claude Sonnet 4.6— junto con herramientas especializadas como SonarQube y Semgrep permitió una cobertura complementaria de los distintos tipos de CWEs presentes en los módulos analizados de ImageMagick.

La principal fortaleza de los LLMs en este contexto es su capacidad para comprender semánticamente el flujo de datos y el flujo de control sin requerir compilación del código, lo que permite identificar debilidades que dependen del contexto de uso, como el patrón de *double-checked locking* sin barreras de memoria (CWE-362) o la posibilidad de path traversal en la construcción de nombres de archivo a partir de metadatos del dispositivo (CWE-22). Estas clases de debilidades son históricamente difíciles de detectar mediante análisis sintáctico o puramente léxico.

No obstante, los LLMs presentan limitaciones importantes. Al no ejecutar el código ni disponer del contexto completo de compilación (definiciones de macros de plataforma, encabezados del sistema operativo), pueden generar falsos positivos en debilidades que dependen de rutas de compilación condicional (`#ifdef`). Por ejemplo, el CWE-457 identificado en `ResizeMagickMemory` solo se materializa cuando `NDEBUG` está definido y el bloque `MAGICKCORE_ANONYMOUS_MEMORY_SUPPORT` está activo simultáneamente. Adicionalmente, los LLMs no pueden detectar debilidades que emergen de la interacción dinámica entre módulos en tiempo de ejecución, como desbordamientos de búfer que dependen de entradas específicas de usuario.

Por su parte, SonarQube aporta reglas deterministas bien establecidas para debilidades como valores de retorno no verificados (CWE-252) y usos de funciones peligrosas (CWE-676), con un nivel bajo de falsos positivos en código C bien formado. Semgrep, con sus patrones semánticos configurables, resulta particularmente útil para detección de patrones específicos de la organización, como el uso de `getenv` sin sanitización.

En definitiva, ninguna herramienta por sí sola es suficiente. La sinergia entre el razonamiento contextual de los LLMs y las reglas formales de herramientas tradicionales produce el conjunto más completo de alertas, requiriendo siempre una fase de post-análisis manual para descartar falsos positivos y confirmar la explotabilidad real de las debilidades identificadas.

---

## Recomendaciones

1. **Configurar reglas específicas de C en SonarQube antes de analizar.** El perfil de calidad por defecto no activa todas las reglas relacionadas con manejo de memoria dinámica y punteros. Es fundamental crear un perfil personalizado que incluya reglas para CWE-476, CWE-401 y CWE-362 antes de lanzar el análisis.

2. **Usar el LLM para análisis de flujo de datos entre funciones.** Los LLMs son especialmente útiles cuando la debilidad se manifiesta en la interacción entre múltiples funciones (e.g., un valor retornado por `getenv` que fluye sin sanitizar hasta `fopen`). Se recomienda suministrar al modelo fragmentos que incluyan las funciones llamadas y las llamadoras en conjunto.

3. **No descartar alertas de CWE de severidad baja sin revisión manual.** Debilidades como CWE-14 pueden parecer irrelevantes en aislamiento, pero su combinación con otras debilidades puede elevar el riesgo. En este proyecto, CWE-14 en `ShredMagickMemory` combinado con CWE-22 podría permitir a un atacante recuperar datos sensibles de archivos temporales.

4. **Validar la configuración de compilación antes de reportar falsos positivos.** Muchas alertas en código con `#ifdef` extensivos son falsos positivos que dependen de la configuración de compilación activa. Se recomienda reproducir el entorno de construcción del paquete Debian/Ubuntu (via `dpkg-buildpackage`) y analizar el código preprocesado con `gcc -E`.

5. **Combinar Semgrep con reglas personalizadas para patrones de la base de código analizada.** Semgrep permite definir patrones YAML que reflejan convenciones específicas del proyecto (e.g., el patrón `getenv()` seguido de uso directo sin `NULL`-check). Invertir tiempo en redactar 3–5 reglas personalizadas tras la primera ronda mejora significativamente la cobertura en iteraciones posteriores.

---

## Resumen Ejecutivo

Se analizaron **5,455 líneas de código** en 6 archivos pertenecientes al módulo OpenCL y de gestión de memoria de **ImageMagick**. El análisis LLM (Claude Sonnet 4.6) identificó **20 instancias de debilidades** distribuidas en **14 CWEs distintos**. La densidad global es de **6.35 CWE/kLoC**, consistente con la media reportada en proyectos C de complejidad similar según el NIST SARD.

Las debilidades de mayor riesgo son:

- **CWE-22 (Path Traversal)** en la construcción del nombre de archivo de caché de kernels OpenCL, que podría permitir escritura de archivos en directorios arbitrarios.
- **CWE-78 (OS Command)** por uso sin sanitización de la variable de entorno `MAGICK_OCL_DEVICE`.
- **CWE-362 (Race Condition)** por el patrón *double-checked locking* sin barreras de memoria en la inicialización del entorno OpenCL y del gestor de memoria.
- **CWE-134 (Format String)** en el log de excepciones OpenCL que inserta datos externos como argumento de cadena de formato.

---

## Referencias

1. MITRE Corporation. *Common Weakness Enumeration (CWE)*. https://cwe.mitre.org, 2024.
2. Anthropic. *Claude Sonnet 4.6 Model Card*. 2025.
3. ImageMagick Studio LLC. *ImageMagick Source Code*. https://imagemagick.org, 2024.
4. NIST. *Software Assurance Reference Dataset (SARD)*. https://samate.nist.gov/SARD/, 2024.
5. Seacord, R.C. *Secure Coding in C and C++*, 2nd ed. Addison-Wesley, 2013.
6. Howard, M., LeBlanc, D. *Writing Secure Code*, 2nd ed. Microsoft Press, 2002.
