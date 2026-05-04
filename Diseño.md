# Diseño del Sistema - Ejecutor de Lotes

**Curso:** Sistemas Operativos (ST0257)
**Práctica:** Ejecutor de Lotes
**Entrega:** Primera Entrega - Documentación de la API
**Integrantes del grupo:** 
- Nicolas Rico Montesino 
- Santiago Alvarez Diaz

---

## Tabla de Contenidos
1. [Anexos](#1-anexos)
2. [Introducción](#2-introducción)
3. [Descripción General del Sistema](#3-descripción-general-del-sistema)
4. [Arquitectura](#4-arquitectura)
5. [Componentes del Sistema](#5-componentes-del-sistema)
6. [Comunicación entre Componentes](#6-comunicación-entre-componentes)
7. [Formato de Mensajes (JSON)](#7-formato-de-mensajes-json)
8. [API - Operaciones Detalladas](#8-api---operaciones-detalladas)
9. [Máquinas de Estado](#9-máquinas-de-estado)
10. [Códigos de Error](#10-códigos-de-error)
11. [Flujos de Ejemplo](#11-flujos-de-ejemplo)
12. [Consideraciones de Implementación](#12-consideraciones-de-implementación)


---

## 1. Anexos

### 1.1 Anexo A: Convenciones de Nomenclatura

| Elemento | Formato | Ejemplo |
|----------|---------|---------|
| ID Fichero | `f-XXXX` | `f-0001` |
| ID Programa | `p-XXXX` | `p-0001` |
| ID Proceso | `proc-XXXX` | `proc-0001` |
| Request ID | UUID v4 | `550e8400-e29b-41d4-a716-446655440000` |

## 2. Introducción

El presente documento describe el diseño del sistema **Ejecutor de Lotes**, una simulación de un sistema de ejecución similar al encontrado en sistemas operativos de mainframe. El sistema permite registrar imágenes de programas (ejecutables, argumentos, ambiente) y ficheros, para luego ejecutarlos como procesos de lotes que leen datos de una entrada estándar, los procesan y retornan el resultado por la salida estándar.

El sistema está diseñado siguiendo una **arquitectura de microservicios**, donde cada componente tiene una responsabilidad específica y se comunica con los demás a través de **tuberías nombradas (named pipes)** utilizando mensajes en formato **JSON**.

---

## 3. Descripción General del Sistema

El sistema simula la ejecución de procesos por lotes en un mainframe. El flujo general es:

1. Un **cliente** se conecta al controlador (`ctrllt`).
2. El cliente registra **programas** (ejecutables) usando `gesprog`.
3. El cliente registra **ficheros** (entrada/salida) usando `gesfich`.
4. El cliente solicita la **ejecución** de procesos de lotes a través del `ejecutor`.
5. El `ejecutor` toma los programas y ficheros del área de almacenamiento (`aralmac`) para crear los procesos.
6. El cliente puede consultar el estado o terminar los procesos en ejecución.

El sistema soporta **múltiples clientes ejecutándose simultáneamente** conectándose al servidor.

---

## 4. Arquitectura

### 4.1 Diagrama General

```
                              ┌──────────────────┐
                              │    aralmac       │
                              │  (Almacenamiento)│
                              └────────┬─────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
              ▼                        ▼                        ▼
        ┌──────────┐             ┌──────────┐             ┌──────────┐
        │ gesfich  │             │ gesprog  │             │ ejecutor │
        │(Ficheros)│             │(Programas)│            │(Procesos)│
        └─────┬────┘             └─────┬────┘             └────┬─────┘
              │                        │                        │
              │  Tuberías Nombradas    │  Tuberías Nombradas    │
              │     (JSON)             │     (JSON)             │
              └────────────┬───────────┴────────────────────────┘
                           │
                           ▼
                     ┌──────────┐
                     │  ctrllt  │
                     │ (Control)│
                     │ Pasarela │
                     └─────┬────┘
                           │
                           │ Tuberías Nombradas (JSON)
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
          ┌──────────┐          ┌──────────┐
          │ cliente  │   ...    │ cliente  │
          │    1     │          │    N     │
          └──────────┘          └──────────┘
```

### 4.2 Patrón Arquitectónico

El sistema sigue el patrón de **microservicios**, donde:

- **`ctrllt`** actúa como **API Gateway / Pasarela**, recibiendo todas las peticiones del cliente y enrutándolas al servicio apropiado.
- Cada servicio (**`gesfich`**, **`gesprog`**, **`ejecutor`**) es independiente, con su propia responsabilidad y máquina de estados.
- Los servicios comparten una región común de almacenamiento llamada **`aralmac`**.

---

## 5. Componentes del Sistema

### 5.1 cliente

**Responsabilidad:** Interfaz de usuario que envía peticiones al sistema.

**Operaciones que realiza:**
- CRUD de programas (Crear, Leer, Actualizar, Borrar)
- CRUD de ficheros (Crear, Leer, Actualizar, Borrar)
- Lanzar, consultar y terminar procesos de lotes

**Sinopsis:**
```
cliente -c <tuberia-nombrada> [-a <tuberia-nombrada>]
```

| Parámetro | Descripción | Obligatorio |
|-----------|-------------|-------------|
| `-c <tuberia-nombrada>` | Tubería para enviar peticiones a `ctrllt` | Sí |
| `-a <tuberia-nombrada>` | Tubería para recibir respuestas (sólo en sistemas con tuberías half-duplex) | Opcional |


---

### 5.2 ctrllt (Control de Lotes)

**Responsabilidad:** Corazón del sistema. Funciona como pasarela que recibe peticiones de los clientes, las analiza y las dirige al servicio apropiado, esperando la respuesta y redirigiéndola al cliente.

**Función principal:** Pasarela (Gateway).

**Sinopsis:**
```
ctrllt -c <tuberia-nombrada> [-a <tuberia-nombrada>] \
       -f <tuberia-nombrada> [-b <tuberia-nombrada>] \
       -p <tuberia-nombrada> [-c <tuberia-nombrada>] \
       -e <tuberia-nombrada> [-d <tuberia-nombrada>]
```

| Parámetro | Descripción |
|-----------|-------------|
| `-c` | Tubería para recibir peticiones del cliente |
| `-a` | Tubería para enviar respuestas al cliente (half-duplex) |
| `-f` | Tubería para enviar peticiones a `gesfich` |
| `-b` | Tubería para recibir respuestas de `gesfich` (half-duplex) |
| `-p` | Tubería para enviar peticiones a `gesprog` |
| `-c` | Tubería para recibir respuestas de `gesprog` (half-duplex) |
| `-e` | Tubería para enviar peticiones a `ejecutor` |
| `-d` | Tubería para recibir respuestas de `ejecutor` (half-duplex) |

---

### 5.3 gesfich (Gestor de Ficheros)

**Responsabilidad:** Crear, actualizar, borrar y leer ficheros almacenados en `aralmac`.

**Sinopsis:**
```
gesfich -f <tuberia-nombrada> [-b <tuberia-nombrada>] -x <info-aralmac>
```

| Parámetro | Descripción |
|-----------|-------------|
| `-f <tuberia-nombrada>` | Tubería para recibir peticiones (o respuestas si es full-duplex) |
| `-b <tuberia-nombrada>` | Tubería para enviar respuestas (half-duplex, opcional) |
| `-x <info-aralmac>` | Configuración del almacenamiento (ruta, parámetros DB, etc.) |

**Identificadores:** Los ficheros se identifican con el formato `f-XXXX` donde X es un dígito (ej: `f-0001`, `f-0002`).

---

### 5.4 gesprog (Gestor de Programas)

**Responsabilidad:** Guardar, actualizar, borrar y mostrar programas almacenados en `aralmac`.

**Sinopsis:**
```
gesprog -p <tuberia-nombrada> [-c <tuberia-nombrada>] -x <info-aralmac>
```

| Parámetro | Descripción |
|-----------|-------------|
| `-p <tuberia-nombrada>` | Tubería para recibir peticiones (o respuestas si es full-duplex) |
| `-c <tuberia-nombrada>` | Tubería para enviar respuestas (half-duplex, opcional) |
| `-x <info-aralmac>` | Configuración del almacenamiento |

**Identificadores:** Los programas se identifican con el formato `p-XXXX` (ej: `p-0001`, `p-0002`).

---

### 5.5 ejecutor

**Responsabilidad:** Ejecutar procesos de lotes a partir de programas y ficheros almacenados en `aralmac`.

**Sinopsis:**
```
ejecutor -e <tuberia-nombrada> [-d <tuberia-nombrada>] -x <info-aralmac>
```

| Parámetro | Descripción |
|-----------|-------------|
| `-e <tuberia-nombrada>` | Tubería para recibir peticiones (o respuestas si es full-duplex) |
| `-d <tuberia-nombrada>` | Tubería para enviar respuestas (half-duplex, opcional) |
| `-x <info-aralmac>` | Configuración del almacenamiento |

**Identificadores:** Los procesos en ejecución se identifican con el formato `proc-XXXX` (ej: `proc-0001`).

---

### 5.6 aralmac (Área de Almacenamiento)

**Responsabilidad:** Región de almacenamiento compartida donde se persisten programas y ficheros.

**Tipos de almacenamiento posibles:**
- Sistema de ficheros (directorio)
- Base de datos relacional
- Almacenamiento clave-valor

---

## 6. Comunicación entre Componentes

### 6.1 Tipo de Comunicación

La comunicación entre componentes se realiza mediante **tuberías nombradas (named pipes)**:

- **Sistemas con tuberías full-duplex:** Se requiere **una sola tubería** por conexión.
- **Sistemas con tuberías half-duplex** (Linux, por ejemplo): Se requieren **dos tuberías nombradas** por conexión (una para envío, otra para recepción).

> **Importante:** Cada tubería utilizada debe tener un **nombre único** dentro del sistema.

### 6.2 Flujo de Comunicación

```
cliente  ─petición─►  ctrllt  ─petición─►  servicio (gesfich/gesprog/ejecutor)
                                                            │
cliente  ◄─respuesta─  ctrllt  ◄─respuesta─                 ┘
```

### 6.3 Concurrencia

- El sistema soporta **múltiples clientes simultáneos**.
- `ctrllt` gestiona la concurrencia, asignando identificadores de petición únicos para distinguir las respuestas.

---

## 7. Formato de Mensajes (JSON)

Todos los mensajes intercambiados entre componentes utilizan el formato **JSON**.

### 7.1 Estructura de Petición (Request)

```json
{
  "request_id": "uuid-v4",
  "operation": "<nombre_operacion>",
  "service": "<gesfich|gesprog|ejecutor>",
  "params": {
    ...
  },
  "timestamp": "2026-05-05T10:30:00Z"
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `request_id` | string (UUID) | Identificador único de la petición |
| `operation` | string | Nombre de la operación a ejecutar |
| `service` | string | Servicio destino (`gesfich`, `gesprog`, `ejecutor`) |
| `params` | object | Parámetros de la operación |
| `timestamp` | string (ISO 8601) | Marca de tiempo de la petición |

### 7.2 Estructura de Respuesta (Response)

```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    ...
  },
  "error": null,
  "timestamp": "2026-05-05T10:30:01Z"
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "success",
  "data": {
    "id": "p-0001"
  },
  "error": null,
  "timestamp": "2026-05-05T10:30:01Z"
}
```

**Respuesta de error:**
```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "error",
  "data": null,
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "El fichero con id f-9999 no existe en aralmac"
  },
  "timestamp": "2026-05-05T10:30:01Z"
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `request_id` | string | Mismo ID que la petición |
| `status` | string | `success` o `error` |
| `data` | object/null | Datos de respuesta (null si hay error) |
| `error` | object/null | Detalle del error (null si fue exitoso) |
| `timestamp` | string (ISO 8601) | Marca de tiempo de la respuesta |

---

## 8. API - Operaciones Detalladas

### 8.1 Operaciones de gesfich

#### 8.1.1 Crear Fichero

Crea un fichero vacío en `aralmac`.

**Operación:** `crear_fichero`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "crear_fichero",
  "service": "gesfich",
  "params": {}
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_fichero": "f-0001"
  }
}
```

---

#### 8.1.2 Leer Fichero (uno específico)

Retorna el contenido de un fichero existente.

**Operación:** `leer_fichero`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "leer_fichero",
  "service": "gesfich",
  "params": {
    "id_fichero": "f-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_fichero": "f-0001",
    "contenido": "...",
    "tamaño_bytes": 1024
  }
}
```

---

#### 8.1.3 Listar Ficheros

Retorna la información de todos los ficheros registrados.

**Operación:** `listar_ficheros`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "listar_ficheros",
  "service": "gesfich",
  "params": {}
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "ficheros": [
      {"id_fichero": "f-0001", "tamaño_bytes": 1024},
      {"id_fichero": "f-0002", "tamaño_bytes": 2048}
    ]
  }
}
```

---

#### 8.1.4 Actualizar Fichero

Copia el contenido de un fichero del sistema dentro de `aralmac`.

**Operación:** `actualizar_fichero`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "actualizar_fichero",
  "service": "gesfich",
  "params": {
    "id_fichero": "f-0001",
    "ruta_origen": "/home/usuario/datos.txt"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_fichero": "f-0001",
    "actualizado": true
  }
}
```

---

#### 8.1.5 Borrar Fichero

Elimina un fichero de `aralmac`.

**Operación:** `borrar_fichero`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "borrar_fichero",
  "service": "gesfich",
  "params": {
    "id_fichero": "f-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_fichero": "f-0001",
    "borrado": true
  }
}
```

---

#### 8.1.6 Operaciones de Control de gesfich

| Operación | Descripción |
|-----------|-------------|
| `suspender` | Suspende el servicio (estado: Suspendido) |
| `reasumir` | Reanuda el servicio (estado: Corriendo) |
| `terminar` | Termina la ejecución del servicio (estado: Terminado) |

**Ejemplo - Suspender:**
```json
{
  "request_id": "uuid-v4",
  "operation": "suspender",
  "service": "gesfich",
  "params": {}
}
```

---

### 8.2 Operaciones de gesprog

#### 8.2.1 Guardar Programa

Registra un nuevo programa en `aralmac`.

**Operación:** `guardar_programa`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "guardar_programa",
  "service": "gesprog",
  "params": {
    "ejecutable": "/ruta/al/binario_o_script",
    "argumentos": ["arg1", "arg2", "--opcion"],
    "ambiente": {
      "PATH": "/usr/bin:/bin",
      "LANG": "es_ES.UTF-8"
    }
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_programa": "p-0001"
  }
}
```

---

#### 8.2.2 Leer Programa

Retorna la información de un programa existente.

**Operación:** `leer_programa`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "leer_programa",
  "service": "gesprog",
  "params": {
    "id_programa": "p-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_programa": "p-0001",
    "ejecutable": "/ruta/al/binario",
    "argumentos": ["arg1", "arg2"],
    "ambiente": {"PATH": "/usr/bin"}
  }
}
```

---

#### 8.2.3 Listar Programas

Retorna la lista de todos los programas registrados.

**Operación:** `listar_programas`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "listar_programas",
  "service": "gesprog",
  "params": {}
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "programas": [
      {"id_programa": "p-0001", "ejecutable": "/bin/echo"},
      {"id_programa": "p-0002", "ejecutable": "/usr/bin/python3"}
    ]
  }
}
```

---

#### 8.2.4 Actualizar Programa

Actualiza la información de un programa registrado.

**Operación:** `actualizar_programa`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "actualizar_programa",
  "service": "gesprog",
  "params": {
    "id_programa": "p-0001",
    "ejecutable": "/nueva/ruta/al/binario",
    "argumentos": ["nuevo_arg"],
    "ambiente": {"PATH": "/usr/local/bin"}
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_programa": "p-0001",
    "actualizado": true
  }
}
```

---

#### 8.2.5 Borrar Programa

Elimina un programa de `aralmac`.

**Operación:** `borrar_programa`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "borrar_programa",
  "service": "gesprog",
  "params": {
    "id_programa": "p-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_programa": "p-0001",
    "borrado": true
  }
}
```

---

#### 8.2.6 Operaciones de Control de gesprog

| Operación | Descripción |
|-----------|-------------|
| `suspender` | Suspende el servicio |
| `reasumir` | Reanuda el servicio |
| `terminar` | Termina el servicio |

---

### 8.3 Operaciones de ejecutor

#### 8.3.1 Ejecutar Proceso de Lotes

Ejecuta un proceso de lotes a partir de un programa y ficheros registrados.

**Operación:** `ejecutar_proceso`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "ejecutar_proceso",
  "service": "ejecutor",
  "params": {
    "id_programa": "p-0001",
    "id_fichero_entrada": "f-0001",
    "id_fichero_salida": "f-0002"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_proceso": "proc-0001",
    "estado": "ejecutando"
  }
}
```

---

#### 8.3.2 Estado de un Proceso

Consulta el estado actual de un proceso específico.

**Operación:** `estado_proceso`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "estado_proceso",
  "service": "ejecutor",
  "params": {
    "id_proceso": "proc-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_proceso": "proc-0001",
    "estado": "ejecutando",
    "id_programa": "p-0001",
    "tiempo_inicio": "2026-05-05T10:30:00Z"
  }
}
```

**Estados posibles:** `ejecutando`, `suspendido`, `terminado`, `error`.

---

#### 8.3.3 Listar Procesos

Retorna el estado de todos los procesos de lotes.

**Operación:** `listar_procesos`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "listar_procesos",
  "service": "ejecutor",
  "params": {}
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "procesos": [
      {"id_proceso": "proc-0001", "estado": "ejecutando"},
      {"id_proceso": "proc-0002", "estado": "terminado"}
    ]
  }
}
```

---

#### 8.3.4 Matar Proceso

Termina forzadamente un proceso de lotes en ejecución.

**Operación:** `matar_proceso`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "matar_proceso",
  "service": "ejecutor",
  "params": {
    "id_proceso": "proc-0001"
  }
}
```

**Respuesta exitosa:**
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_proceso": "proc-0001",
    "estado": "terminado"
  }
}
```

---

#### 8.3.5 Operaciones de Control de ejecutor

| Operación | Descripción |
|-----------|-------------|
| `suspender` | Suspende el servicio |
| `reasumir` | Reanuda el servicio |
| `parar` | Detiene la aceptación de nuevos procesos |

---

## 9. Máquinas de Estado

### 9.1 ctrllt

```
   ┌────────┐    iniciar    ┌──────────┐    terminar    ┌────────────┐
   │ inicio │ ─────────────►│ Corriendo│ ──────────────►│ Terminado  │
   └────────┘               └──────────┘                └────────────┘
```

| Estado | Descripción |
|--------|-------------|
| `inicio` | Estado inicial antes de aceptar peticiones |
| `Corriendo` | Aceptando y procesando peticiones |
| `Terminado` | Servicio finalizado |

---

### 9.2 gesfich y gesprog

```
                       Crear/Leer/
                       Actualizar/Borrar
                            │
                            ▼
   ┌────────┐  iniciar  ┌──────────┐  suspender   ┌─────────────┐
   │ inicio │ ────────► │ Corriendo│ ───────────► │ Suspendido  │
   └────────┘           └─────┬────┘  ◄────────── └──────┬──────┘
                              │      reasumir            │
                              │                          │
                              │ terminar       terminar  │
                              ▼                          ▼
                        ┌────────────┐
                        │ Terminado  │
                        └────────────┘
```

| Estado | Descripción |
|--------|-------------|
| `inicio` | Estado inicial |
| `Corriendo` | Procesando peticiones (CRUD) |
| `Suspendido` | Servicio pausado, no procesa peticiones |
| `Terminado` | Servicio finalizado |

**Transiciones:**
- `Corriendo → Suspendido`: con operación `suspender`
- `Suspendido → Corriendo`: con operación `reasumir`
- `Corriendo/Suspendido → Terminado`: con operación `terminar`

---

### 9.3 ejecutor

```
                  Ejecutar/Estado/Matar
                          │
                          ▼
   ┌────────┐  ┌────────────────┐  suspender   ┌──────────────┐
   │ inicio │─►│   Ejecutar     │ ───────────► │ Suspendidos  │
   └────────┘  └────┬───────────┘  ◄────────── └──────────────┘
                    │     reasumir
                    │ parar
                    ▼   /Procesos = 0
              ┌─────────┐  terminar  ┌────────────┐
              │  Parar  │ ─────────► │ Terminado  │
              └─────────┘            └────────────┘
```

| Estado | Descripción |
|--------|-------------|
| `inicio` | Estado inicial |
| `Ejecutar` | Aceptando y ejecutando procesos de lotes |
| `Suspendidos` | Procesos pausados |
| `Parar` | No acepta nuevos procesos, espera a que los actuales terminen |
| `Terminado` | Servicio finalizado |

---

## 10. Códigos de Error

| Código | Descripción |
|--------|-------------|
| `INVALID_REQUEST` | Estructura de petición inválida |
| `INVALID_PARAMS` | Parámetros inválidos o faltantes |
| `FILE_NOT_FOUND` | Fichero no encontrado en aralmac |
| `PROGRAM_NOT_FOUND` | Programa no encontrado en aralmac |
| `PROCESS_NOT_FOUND` | Proceso no encontrado |
| `INVALID_STATE` | Operación no válida en el estado actual |
| `EXECUTABLE_INVALID` | El ejecutable no es válido o no existe |
| `STORAGE_ERROR` | Error en el área de almacenamiento |
| `PIPE_ERROR` | Error de comunicación en la tubería |
| `INTERNAL_ERROR` | Error interno del servicio |
| `SERVICE_UNAVAILABLE` | Servicio no disponible (suspendido o terminado) |

**Ejemplo de respuesta con error:**
```json
{
  "request_id": "uuid-v4",
  "status": "error",
  "data": null,
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "El fichero con id 'f-9999' no existe en aralmac",
    "detalles": {
      "id_solicitado": "f-9999"
    }
  },
  "timestamp": "2026-05-05T10:30:01Z"
}
```

---

## 11. Flujos de Ejemplo

### 11.1 Flujo: Registrar y Ejecutar un Proceso de Lotes

**Paso 1:** Cliente registra un programa.

```
cliente → ctrllt → gesprog
```

```json
// Petición del cliente
{
  "request_id": "req-001",
  "operation": "guardar_programa",
  "service": "gesprog",
  "params": {
    "ejecutable": "/usr/bin/wc",
    "argumentos": ["-l"],
    "ambiente": {}
  }
}

// Respuesta
{
  "request_id": "req-001",
  "status": "success",
  "data": {"id_programa": "p-0001"}
}
```

**Paso 2:** Cliente registra fichero de entrada.

```json
// Petición
{
  "request_id": "req-002",
  "operation": "crear_fichero",
  "service": "gesfich",
  "params": {}
}

// Respuesta
{
  "request_id": "req-002",
  "status": "success",
  "data": {"id_fichero": "f-0001"}
}
```

**Paso 3:** Cliente actualiza el fichero con datos.

```json
{
  "request_id": "req-003",
  "operation": "actualizar_fichero",
  "service": "gesfich",
  "params": {
    "id_fichero": "f-0001",
    "ruta_origen": "/tmp/datos.txt"
  }
}
```

**Paso 4:** Cliente registra fichero de salida.

```json
{
  "request_id": "req-004",
  "operation": "crear_fichero",
  "service": "gesfich",
  "params": {}
}
// → respuesta con f-0002
```

**Paso 5:** Cliente ejecuta el proceso de lotes.

```json
{
  "request_id": "req-005",
  "operation": "ejecutar_proceso",
  "service": "ejecutor",
  "params": {
    "id_programa": "p-0001",
    "id_fichero_entrada": "f-0001",
    "id_fichero_salida": "f-0002"
  }
}
// → respuesta con id_proceso "proc-0001"
```

**Paso 6:** Cliente consulta el estado.

```json
{
  "request_id": "req-006",
  "operation": "estado_proceso",
  "service": "ejecutor",
  "params": {"id_proceso": "proc-0001"}
}
```

**Paso 7:** Cliente lee el resultado del fichero de salida.

```json
{
  "request_id": "req-007",
  "operation": "leer_fichero",
  "service": "gesfich",
  "params": {"id_fichero": "f-0002"}
}
```

---

### 11.2 Flujo: Manejo de Error

**Caso:** El cliente intenta ejecutar un programa con un fichero inexistente.

```json
// Petición
{
  "request_id": "req-100",
  "operation": "ejecutar_proceso",
  "service": "ejecutor",
  "params": {
    "id_programa": "p-0001",
    "id_fichero_entrada": "f-9999"
  }
}

// Respuesta
{
  "request_id": "req-100",
  "status": "error",
  "data": null,
  "error": {
    "code": "FILE_NOT_FOUND",
    "message": "El fichero con id 'f-9999' no existe en aralmac"
  }
}
```

---

## 12. Consideraciones de Implementación

### 12.1 Sistemas Operativos Soportados

- **Linux:** Tuberías nombradas half-duplex (FIFOs vía `mkfifo`).
- **Windows 11:** Tuberías nombradas full-duplex (`CreateNamedPipe`).

### 12.2 Concurrencia

- Cada servicio debe manejar múltiples peticiones de forma concurrente.
- Se recomienda usar **hilos** o **procesos** para procesar peticiones en paralelo.
- El `request_id` permite correlacionar peticiones y respuestas en entornos concurrentes.

### 12.3 Persistencia

- `aralmac` puede implementarse como:
  - Sistema de directorios (recomendado para la primera versión).
  - Base de datos.
- Los identificadores (`f-XXXX`, `p-XXXX`, `proc-XXXX`) deben persistir entre reinicios del servicio.

### 12.4 Validaciones

Cada servicio debe validar:
- Estructura del JSON recibido.
- Existencia de campos obligatorios.
- Existencia de identificadores referenciados.
- Validez de rutas y ejecutables.
- Estado del servicio antes de procesar la operación.

### 12.5 Logging

Se recomienda implementar logging para:
- Peticiones recibidas.
- Respuestas enviadas.
- Errores y excepciones.
- Cambios de estado del servicio.

---


