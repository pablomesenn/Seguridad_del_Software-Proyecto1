

## How to Run

```bash
python3 -m venv ~/security-venv
source ~/security-venv/bin/activate
pip install --upgrade pip
pip install semgrep
semgrep --version

semgrep --config "p/security-audit" --config "p/c" src/imagemagick-6.9.12.98+dfsg1/magick --json > semgrep_output.json

```



## Results

``` bash
┌─────────────┐
│ Scan Status │
└─────────────┘
  Scanning 6 files tracked by git with 276 Code rules:
                                                                                                                        
  Language      Rules   Files          Origin      Rules                                                                
 ─────────────────────────────        ───────────────────                                                               
  c                60       6          Community     225                                                                
  <multilang>       2       6          Pro rules      51                                                                
  cpp              51       4                                                                                           
                                                                                                                        
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:06                                                                                                                        
                
                
┌──────────────┐
│ Scan Summary │
└──────────────┘
✅ Scan completed successfully.
 • Findings: 3 (3 blocking)
 • Rules run: 62
 • Targets scanned: 6
 • Parsed lines: ~99.3%
 • Scan was limited to files tracked by git
 • For a detailed list of skipped files and lines, run semgrep with the --verbose flag
Ran 62 rules on 6 files: 3 findings.
```


# Analisis

## Alertas 1 y 2 — CWE-416: Use After Free in RelinquishAlignedMemory (memory.c lines 1133, 1135)
Estas dos alertas señalan el uso de la variable de memoria dentro de `RelinquishAlignedMemory` después de que supuestamente se haya liberado.
Extracto del código:

```c
MagickExport void *RelinquishAlignedMemory(void *memory)
{
  if (memory == (void *) NULL)
    return((void *) NULL);
  if (memory_methods.relinquish_aligned_memory_handler != ...)
    {
      memory_methods.relinquish_aligned_memory_handler(memory);
      return(NULL);                          
    }
#if defined(MAGICKCORE_HAVE_ALIGNED_MALLOC) || defined(MAGICKCORE_HAVE_POSIX_MEMALIGN)
  free(memory);                              
#elif defined(MAGICKCORE_HAVE__ALIGNED_MALLOC)
  _aligned_free(memory);                     
#else
  RelinquishMagickMemory(actual_base_address(memory));
#endif
  return(NULL);
}
```

La observación crucial es que estas tres ramas son bloques #ifdef mutuamente excluyentes. Solo una de ellas se compila en el binario, dependiendo de la plataforma. Es posible Semgrep no expande completamente las condicionales del preprocesador, es decir, analizó parcialmente este archivo (como lo demuestran los numerosos errores de PartialParsing en el array de errores), y su análisis de contaminación trata incorrectamente las tres ramas como si fueran secuenciales. Interpreta free(memory) en la línea 1131 como una liberación de memoria, y luego cree erróneamente que las líneas 1133 y 1135 ocurren después de dicha liberación.

Ambos son falsos positivos. La plataforma solo puede compilar una de las tres ramas. No existe una ruta en el código donde se libere memoria y luego se vuelva a pasar a _aligned_free o actual_base_address.

## Alerta 2 — CWE-676: Use of Potentially Dangerous Function (strncpy) in opencl.c line 259

La línea marcada dentro de DumpProfileData() es:

`strncpy(indent, kernelNames[i], min(strlen(kernelNames[i]), strlen(indent) - 1));`

Esto es un verdadero positivo, pero tiene dos problemas que se agravan. Primero, se está usando `strncpy`: la preocupación planteada por CWE-676 es legítima porque `strncpy` no garantiza la terminación nula cuando la fuente es más larga o igual al límite. Segundo, y más crítico, el cálculo del límite en sí es incorrecto: `strlen(indent) - 1` lee el contenido actual de `indent` en lugar de su capacidad (es decir, `sizeof(indent) - 1)`. Esto significa que el límite pasado a `strncpy` depende de la cantidad de bytes que haya en `indent` en ese momento, lo que lo hace prácticamente impredecible. Si `indent` está vacío, `strlen(indent) - 1` se desborda a un valor enorme en un tipo sin signo, creando un desbordamiento de búfer. La llamada correcta usaría `sizeof(indent) - 1` y luego terminaría explícitamente con un valor nulo, o reemplazaría el patrón con `strlcpy` / `snprintf` / `CopyMagickString` (que ImageMagick ya usa en otra parte del mismo archivo).
