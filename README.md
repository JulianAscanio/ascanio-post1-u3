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
