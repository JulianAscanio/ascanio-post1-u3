# Laboratorio Práctico: Manejo del DEBUG — Exploración, Ensamblado y Ejecución Paso a Paso en DOSBox

**Asignatura:** Arquitectura de Computadores  
**Unidad 3:** Post-contenido (Laboratorio Guiado)  
**Entorno de Ejecución:** DOSBox 0.74-3 sobre emulación x86 (MS-DOS DEBUG)  

---

## 1. Introducción y Contexto

El presente informe documenta la experimentación práctica realizada dentro del entorno emulado DOSBox utilizando la herramienta de bajo nivel `DEBUG`. A través de este laboratorio se analiza el comportamiento interno del procesador Intel 8086 en modo real, interactuando de forma directa con los registros de la CPU, los segmentos de memoria y las instrucciones de máquina sin abstracciones de compilador.

El desarrollo se divide en dos fases fundamentales: la **Parte 1**, orientada a la inspección de registros, la manipulación de bloques de memoria mediante volcados hexadecimales y la verificación de modos de direccionamiento; y la **Parte 2**, centrada en el ensamblado directo en memoria, la ejecución controlada paso a paso, el análisis de instrucciones de control de flujo (`LOOP` vs. `DEC`/`JNZ`) y la optimización de trazas mediante puntos de interrupción (`G`).

---

## 2. Parte 1: Exploración con DEBUG en DOSBox

### 2.1 Configuración e Inspección Inicial de Registros (Checkpoint 1)
Al iniciar la sesión de `DEBUG` tras montar la unidad virtual en DOSBox (`MOUNT C ~/doswork` -> `C:\LAB3POST1> DEBUG`), se procedió a invocar el comando `R` para capturar el estado por defecto del microprocesador:

```text
-R
AX=0000  BX=0000  CX=0000  DX=0000  SP=FFFE  BP=0000  SI=0000  DI=0000
DS=1357  ES=1357  SS=1357  CS=1357  IP=0100   NV UP EI PL NZ NA PO NC
1357:0100 CD20          INT 20
```

* **Registros de Propósito General:** Los registros `AX`, `BX`, `CX` y `DX` inician en `0000h`.
* **Punteros y Segmentos:** `SP` se localiza en `FFFEh` (extremo superior del segmento de pila), mientras que `CS`, `DS`, `SS` y `ES` apuntan unívocamente a la base asignada por el sistema operativo (`1357h`), compartiendo el mismo segmento de 64 KB típico de las estructuras `.COM`.
* **Instruction Pointer:** `IP` se posiciona en `0100h`, correspondiente al primer byte ejecutable tras los 256 bytes reservados para el Program Segment Prefix (PSP).
* **Modificación selectiva:** Mediante el comando `R AX` se cargó el valor `1234h` (`4660` decimal), confirmando que la alteración es atómica y no perturba el estado de los demás registros ni de las banderas.

> **Evidencia:** `capturas/CP1_registros.png`

---

### 2.2 Manipulación de Bloques de Memoria y Área del PSP (Checkpoint 2)
Se empleó el comando `F` (*Fill*) para inicializar un bloque de 64 bytes (`40h`) a partir de la dirección `DS:0200` con la secuencia cíclica `AB CD EF`:

```text
-F 200 L40 AB CD EF
-D 200 L40
1357:0200  AB CD EF AB CD EF AB CD-EF AB CD EF AB CD EF AB  ................
1357:0210  CD EF AB CD EF AB CD EF-AB CD EF AB CD EF AB CD  ................
1357:0220  EF AB CD EF AB CD EF AB-CD EF AB CD EF AB CD EF  ................
1357:0230  AB CD EF AB CD EF AB CD-EF AB CD EF AB CD EF AB  ................
```

**Estructura y Anatomía del Volcado Hexadecimal (Comando D):**
La salida generada por el comando `D` se compone de tres zonas estructurales claramente diferenciadas:
1. **Columna de Direcciones (Extremo Izquierdo):** Muestra la dirección lógica en formato `Segmento:Offset` (por ejemplo, `1357:0200`), indicando la posición en memoria del primer byte de la fila.
2. **Cuerpo Hexadecimal Central:** Presenta 16 bytes consecutivos por línea expresados en base 16, divididos en dos bloques de 8 bytes mediante un guion central (`-`) para facilitar la localización visual rápida.
3. **Panel ASCII Lateral (Extremo Derecho):** Representa la traducción textual de cada byte. Aquellos valores fuera del rango estándar imprimible (código ASCII menor a `20h` o mayor a `7Eh`, como `ABh`, `CDh` y `EFh`) se representan mediante puntos (`.`).

Adicionalmente, la inspección de `DS:0000` (`D 0 L20`) evidenció los primeros bytes del PSP, donde destacan los valores `CD 20`, correspondientes a la instrucción de interrupción DOS `INT 20h` para terminaciones seguras de programa.

> **Evidencia:** `capturas/CP2_volcado_memoria.png`

---

### 2.3 Ensamblado Directo y Desensamblado (Checkpoint 3)
Se introdujo un bloque aritmético elemental en memoria mediante el comando `A 100` y se contrastó inmediatamente con el comando `U 100 109`:

```text
-A 100
1357:0100 MOV AX, 0005
1357:0103 MOV BX, 0003
1357:0106 ADD AX, BX
1357:0108 INT 20
1357:010A

-U 100 109
1357:0100 B80500        MOV AX,0005
1357:0103 BB0300        MOV BX,0003
1357:0106 03C3          ADD AX,BX
1357:0108 CD20          INT 20
```

Se comprueba la correspondencia binaria directa del repertorio x86:
* `MOV AX, 0005` genera el opcode `B8` seguido del dato inmediato en orden little-endian (`05 00`).
* `MOV BX, 0003` genera el opcode `BB 03 00`.
* `ADD AX, BX` codifica como `03 C3` (2 bytes, especificando registros mediante byte ModR/M).
* `INT 20` se traduce en `CD 20`.

> **Evidencia:** `capturas/CP3_ensamblado_desensamblado.png`

---

### 2.4 Modificación Puntual y Direccionamiento Directo (Checkpoint 4)

#### Decisión Técnica 1: Verificación No Destructiva de una Escritura en Memoria (Paso 12)
> **Dictamen Técnico:** Se determina de forma concluyente que el comando **`D` (Dump)** es la única herramienta técnica válida y segura para verificar el contenido de la memoria sin alterar el estado del sistema. El comando `E` ejecutado en modo interactivo (`E 300`) presenta un riesgo operativo inaceptable, dado que mantiene abierto el búfer de entrada esperando modificaciones por teclado; cualquier pulsación accidental de la barra espaciadora o de caracteres alfanuméricos sobreescribe los bytes en curso de manera irreversible. Por su parte, el comando `F` es destructivo por definición, pues su función consiste en escribir y sobrescribir rangos completos de memoria con patrones prefijados. En contraste estricto, el comando `D` garantiza una operación de **solo lectura absoluta** en el bus de datos: emite ciclos de lectura hacia la memoria RAM y formatea los datos directamente hacia la consola estándar sin emitir ciclos de escritura, preservando la integridad del contenido verificado y asegurando la fidelidad del estado del procesador.

---

#### Procedimiento de Escritura y Carga Directa
Se limpió el bloque `DS:0300` con `F 300 L10 00` y se ingresaron dos bytes mediante `E 300 78 56`. Posteriormente se ensambló en `CS:0320` la instrucción de direccionamiento directo a memoria:

```text
-A 320
1357:0320 MOV AX, [0300]
1357:0323 INT 20
1357:0325

-U 320 324
1357:0320 A10003        MOV AX,[0300]
1357:0323 CD20          INT 20
```

Al forzar el puntero con `R IP` a `0320` y ejecutar `T`, el procesador resolvió el desplazamiento `0300h`, leyendo el byte bajo `78h` y el alto `56h`. El registro `AX` pasó a contener exactamente el valor **`5678h`**.

---

#### Decisión Técnica 2: Modo de Direccionamiento Inmediato vs. Directo a Memoria (Paso 14)
> **Dictamen Técnico:** Aunque `MOV AX, 0005` (`B8 05 00`) y `MOV AX, [0300]` (`A1 00 03`) ocupan ambas exactamente 3 bytes en memoria, su comportamiento microarquitectural es radicalmente distinto. La instrucción con direccionamiento directo `MOV AX, [0300]` requiere **un ciclo de bus de memoria adicional** durante su fase de ejecución (*Execute/Memory*). Mientras que en el modo inmediato el operando fuente viene embebido en el propio flujo de bytes decodificado en la etapa de búsqueda (*Fetch*), en el direccionamiento directo la CPU decodifica el offset `0300h` y debe emitir una transacción de lectura complementaria al bus de datos para extraer los 16 bits residentes en la RAM. No obstante, el direccionamiento directo resulta imprescindible en escenarios donde los datos son dinámicos y mutables en tiempo de ejecución (como buffers de entrada, tablas de variables o datos suministrados externamente con `E`), donde una constante inmediata fijada durante el ensamblado resultaría inoperante. La herramienta `U` permite confirmar esta divergencia sin ejecutar código: inspeccionando el opcode inicial, `B8` identifica de inmediato la carga de un literal, mientras que `A1` delata la instrucción de lectura directa desde memoria hacia el acumulador.

> **Evidencia:** `capturas/CP4_memoria_direccionamiento.png`

---

## 3. Parte 2: Ensamblado y Ejecución Paso a Paso

### 3.1 Programa de Suma con Traza Completa (Checkpoint 1)
Se ensambló en `CS:0100` una rutina para sumar tres valores en memoria:

```assembly
MOV AX, 000A    ; AX = 10d
MOV BX, 0005    ; BX = 5d
MOV CX, 0003    ; CX = 3d
ADD AX, BX      ; AX = 15d (000Fh)
ADD AX, CX      ; AX = 18d (0012h)
INT 20
```

#### Tabla de Traza Completa (Valores Observados con T)

| Paso | Instrucción Ejecutada | AX | BX | CX | IP Sig. | ZF | CF | SF | Observación |
| :---: | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **0** | *Estado Inicial* | `0000` | `0000` | `0000` | `0100` | 0 (NZ) | 0 (NC) | 0 (PL) | Registros limpios antes de iniciar |
| **1** | `MOV AX, 000A` | `000A` | `0000` | `0000` | `0103` | 0 (NZ) | 0 (NC) | 0 (PL) | Carga inmediata en AX (10 decimal) |
| **2** | `MOV BX, 0005` | `000A` | `0005` | `0000` | `0106` | 0 (NZ) | 0 (NC) | 0 (PL) | Carga inmediata en BX (5 decimal) |
| **3** | `MOV CX, 0003` | `000A` | `0005` | `0003` | `0109` | 0 (NZ) | 0 (NC) | 0 (PL) | Carga inmediata en CX (3 decimal) |
| **4** | `ADD AX, BX` | `000F` | `0005` | `0003` | `010B` | 0 (NZ) | 0 (NC) | 0 (PL) | 10 + 5 = 15 (`000Fh`). Banderas inalteradas |
| **5** | `ADD AX, CX` | `0012` | `0005` | `0003` | `010D` | 0 (NZ) | 0 (NC) | 0 (PL) | 15 + 3 = 18 (`0012h`). Resultado positivo |
| **6** | `INT 20` | `0012` | `0005` | `0003` | `0000` | 0 (NZ) | 0 (NC) | 0 (PL) | Terminación controlada del programa |

> **Evidencia:** `capturas/CP1_traza_suma.png`

---

### 3.2 Programa con Bucle usando LOOP y CX (Checkpoint 2)
Se analizó un bucle para acumular cuatro sumas de `0002h`:

```assembly
1357:0100 MOV AX, 0000    ; Acumulador en 0
1357:0103 MOV CX, 0004    ; Contador de iteraciones
1357:0106 ADD AX, 0002    ; Inicio de bucle
1357:0109 LOOP 0106       ; Decrementa CX; si CX != 0 salta
1357:010B INT 20
```

La codificación de `LOOP 0106` arrojó los bytes **`E2 FB`**. El byte `E2` representa el código de operación de la instrucción `LOOP`, mientras que `FB` es el desplazamiento relativo firmado en complemento a 2 ($-5$ decimal):
$$\text{Offset destino} = 0106\text{h} - 010B\text{h} = -5 = \text{FBh}$$

#### Tabla de Traza Completa del Bucle con LOOP

| Iteración | Instrucción Ejecutada | AX Después | CX Después | IP Siguiente | ¿LOOP Salta? | Estado de Banderas Relevantes |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- |
| **Init 1** | `MOV AX, 0000` | `0000` | `0000` | `0103` | N/A | Banderas en estado base |
| **Init 2** | `MOV CX, 0004` | `0000` | `0004` | `0106` | N/A | CX cargado con el límite de vueltas |
| **1** | `ADD AX, 0002` | `0002` | `0004` | `0109` | N/A | Primera adición: AX = 2 |
| **1** | `LOOP 0106` | `0002` | `0003` | `0106` | **SÍ** | CX se decrementa a 3 ($3 \neq 0$) |
| **2** | `ADD AX, 0002` | `0004` | `0003` | `0109` | N/A | Segunda adición: AX = 4 |
| **2** | `LOOP 0106` | `0004` | `0002` | `0106` | **SÍ** | CX se decrementa a 2 ($2 \neq 0$) |
| **3** | `ADD AX, 0002` | `0006` | `0002` | `0109` | N/A | Tercera adición: AX = 6 |
| **3** | `LOOP 0106` | `0006` | `0001` | `0106` | **SÍ** | CX se decrementa a 1 ($1 \neq 0$) |
| **4** | `ADD AX, 0002` | `0008` | `0001` | `0109` | N/A | Cuarta adición: AX = 8 |
| **4** | `LOOP 0106` | `0008` | `0000` | `010B` | **NO** | CX llega a 0. Termina el bucle |
| **Fin** | `INT 20` | `0008` | `0000` | `0000` | N/A | Termina ejecución. AX = 8 |

> **Evidencia:** `capturas/CP2_traza_loop.png`

---

### 3.3 Bucle Equivalente con DEC/JNZ y Comparación Estructural (Checkpoint 3)
Se implementó la misma rutina iterativa en `CS:0200` sustituyendo `LOOP` por la combinación manual `DEC CX` y `JNZ 0206`:

```text
-U 200 20D
1357:0200 B80000        MOV AX,0000
1357:0203 B90400        MOV CX,0004
1357:0206 050200        ADD AX,+02
1357:0209 49            DEC CX
1357:020A 75FA          JNZ 0206
1357:020C CD20          INT 20
```

#### Tabla de Traza Completa con DEC / JNZ

| Iteración | Instrucción | AX Después | CX Después | IP Sig. | ¿JNZ Salta? | Estado de Banderas |
| :---: | :--- | :---: | :---: | :---: | :---: | :--- |
| **Init** | `MOV AX, 0000` | `0000` | `0000` | `0203` | N/A | Estado inicial |
| **Init** | `MOV CX, 0004` | `0000` | `0004` | `0206` | N/A | Contador inicializado |
| **1** | `ADD AX, 0002` | `0002` | `0004` | `0209` | N/A | AX = 2 |
| **1** | `DEC CX` | `0002` | `0003` | `020A` | N/A | ZF=0 (NZ) |
| **1** | `JNZ 0206` | `0002` | `0003` | `0206` | **SÍ** | Salta porque ZF=0 |
| **2** | `ADD AX, 0002` | `0004` | `0003` | `0209` | N/A | AX = 4 |
| **2** | `DEC CX` | `0004` | `0002` | `020A` | N/A | ZF=0 (NZ) |
| **2** | `JNZ 0206` | `0004` | `0002` | `0206` | **SÍ** | Salta porque ZF=0 |
| **3** | `ADD AX, 0002` | `0006` | `0002` | `0209` | N/A | AX = 6 |
| **3** | `DEC CX` | `0006` | `0001` | `020A` | N/A | ZF=0 (NZ) |
| **3** | `JNZ 0206` | `0006` | `0001` | `0206` | **SÍ** | Salta porque ZF=0 |
| **4** | `ADD AX, 0002` | `0008` | `0001` | `0209` | N/A | AX = 8 |
| **4** | `DEC CX` | `0008` | `0000` | `020A` | N/A | **ZF=1 (ZR)** (llegó a cero) |
| **4** | `JNZ 0206` | `0008` | `0000` | `020C` | **NO** | No salta porque ZF=1 |
| **Fin** | `INT 20` | `0008` | `0000` | `0000` | N/A | Programa finalizado con éxito |

#### Comparativa Cuantitativa de Instrucciones Ejecutadas
* **Implementación con `LOOP`:** $2\text{ (init)} + 4 \times (\text{ADD} + \text{LOOP}) + 1\text{ (fin)} = \mathbf{11\text{ instrucciones}}$.
* **Implementación con `DEC/JNZ`:** $2\text{ (init)} + 4 \times (\text{ADD} + \text{DEC} + \text{JNZ}) + 1\text{ (fin)} = \mathbf{15\text{ instrucciones}}$.
* **Balance:** El mecanismo desacoplado `DEC/JNZ` introduce una instrucción de control adicional por iteración, elevando el tiempo de procesamiento total.

---

#### Decisión Técnica 3: Selección de Mecanismo de Control de Bucle (Paso 10)
> **Dictamen Técnico:** Para un bucle contador elemental donde el registro `CX` actúa exclusivamente como índice decremental, se recomienda categóricamente el uso de la instrucción **`LOOP`**. Desde la perspectiva de densidad de código, `LOOP` ocupa únicamente 2 bytes (`E2 FB`) frente a los 3 bytes requeridos por el par `DEC CX` y `JNZ` (`49 75 FA`). Más relevante aún es la reducción en la carga de trabajo del procesador: `LOOP` consolida el decremento y la bifurcación condicional en un único ciclo de extracción (*Fetch-Decode*), ejecutando solo 11 instrucciones en lugar de las 15 que demanda `DEC/JNZ`. Sin embargo, la combinación manual `DEC/JNZ` resulta preferible cuando se requiere desacoplar el contador (por ejemplo, iterar sobre registros distintos como `BX` o `SI` reservando `CX` para operaciones de cadena o corrimientos), o cuando la condición de ruptura no depende de una anulación directa a cero, sino de comparaciones explícitas basadas en banderas como acarreo (`CF`) u overflow (`OF`) evaluadas mediante `CMP`. El comando `U` confirma esta discrepancia sin ejecución al comparar directamente los offsets de inicio y fin: el bloque `LOOP` abarca desde `0100h` hasta `010Dh` (13 bytes totales), mientras que `DEC/JNZ` requiere desde `0200h` hasta `020Eh` (14 bytes totales).

---

#### Decisión Técnica 4: Comando de Verificación para Bucles de Gran Escala — T vs. G (Paso 12)
> **Dictamen Técnico:** Frente a un bucle con un volumen elevado de iteraciones (como $CX = 100\text{d}$ o $64\text{h}$), la ejecución manual mediante el comando `T` se vuelve inviable debido al costo operativo y al tiempo involucrado: exigiría invocar y registrar manualmente más de 300 transiciones de estado. En este escenario, la utilización del comando **`G` con punto de interrupción** (`G 20C`) es la técnica correcta. Su funcionamiento se basa en reemplazar temporalmente el primer byte de la dirección destino por el opcode `CC` (`INT 3`), permitiendo que el hardware procese las iteraciones a velocidad de reloj nativa antes de devolver el control al depurador. No obstante, `G` sacrifica por completo la visibilidad intermedia: imposibilita detectar condiciones de desbordamiento transitorio, sobreescrituras accidentales de memoria o desalineaciones de puntero producidas a mitad del ciclo. El comando `T` fue estrictamente indispensable en los Pasos 4, 7 y 11 debido a que la construcción de la tabla de traza requiere capturar la evolución de los registros paso a paso, granularidad que `G` omite deliberadamente. Finalmente, tras ejecutar `G 20C`, el estudiante debe usar el comando **`R AX`** (o `R`) para confirmar de manera aislada que el acumulador retiene el valor final esperado (`AX = 0008`) sin requerir un nuevo ciclo de ejecución.

> **Evidencia:** `capturas/CP3_traza_dec_jnz.png`

---

## 4. Estructura del Repositorio y Registro de Commits (Checkpoint 4)

El repositorio institucional cumple con la estructura requerida:

```text
ascanio-post1-u3/
├── README.md
└── capturas/
    ├── CP1_registros.png
    ├── CP2_volcado_memoria.png
    ├── CP3_ensamblado_desensamblado.png
    ├── CP4_memoria_direccionamiento.png
    ├── CP1_traza_suma.png
    ├── CP2_traza_loop.png
    └── CP3_traza_dec_jnz.png
```

### Historial de Commits
1. `commit 1`: *"Configura entorno DOSBox e implementa inspeccion de registros CP1"*
2. `commit 2`: *"Completa volcado hexadecimal CP2 y ensamblado basico CP3"*
3. `commit 3`: *"Implementa modificacion de memoria y decisiones tecnicas Parte 1 CP4"*
4. `commit 4`: *"Agrega estructura inicial de la Parte 2 y traza de suma CP1"*
5. `commit 5`: *"Completa programa con LOOP y genera tabla de traza CP2"*
6. `commit 6`: *"Completa bucle DEC/JNZ, decisiones tecnicas y finaliza laboratorio CP3"*

---

## 5. Conclusiones

1. **Transparencia Arquitectónica:** El depurador `DEBUG` evidencia que en la arquitectura x86 en modo real no existe separación física entre código y datos; cualquier región de memoria apuntada por el par `CS:IP` será decodificada y ejecutada por la CPU como código de máquina.
2. **Eficiencia en el Conjunto de Instrucciones:** La comparativa cuantitativa entre `LOOP` y `DEC/JNZ` demuestra la ventaja de las instrucciones compuestas del repertorio CISC, logrando una mayor densidad de código y eliminando ciclos redundantes de decodificación en estructuras iterativas estándar.
3. **Control y Diagnóstico por Hardware:** Se constata que la depuración mediante traza (`T`) depende del bit `TF` (*Trap Flag*) y la interrupción de paso a paso (`INT 1`), mientras que la ejecución rápida mediante puntos de interrupción (`G`) se apoya en la sustitución de código por `INT 3` (`0xCC`), fundamentando los principios que rigen a los depuradores modernos de bajo nivel.