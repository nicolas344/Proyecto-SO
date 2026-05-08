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

El sistema opera con **un único cliente** conectado al servidor.

---

## 4. Arquitectura

### 4.1 Diagrama General

```
                    ┌─────────────────────────────────────┐
                    │             aralmac                 │
                    │         (Almacenamiento)            │
                    └──────┬──────────────┬──────┬───────┘
                           │              │      │
                    acceso │       acceso │      │ acceso
                    directo│       directo│      │ directo
                           │              │      │
                    ┌──────▼──┐    ┌──────▼──┐  ┌▼─────────┐
                    │ gesfich │    │ gesprog │  │ ejecutor │
                    │(Ficheros)│   │(Programas)│ │(Procesos)│
                    └──────┬──┘    └──────┬──┘  └────┬─────┘
                           │              │           │
                           │   Tuberías Nombradas (JSON)
                           └──────────────┬───────────┘
                                          │
                                          ▼
                                    ┌──────────┐
                                    │  ctrllt  │
                                    │ (Control)│
                                    │ Pasarela │
                                    └─────┬────┘
                                          │
                                   Tubería Nombrada (JSON)
                                          │
                                          ▼
                                    ┌──────────┐
                                    │ cliente  │
                                    └──────────┘
```

### 4.2 Patrón Arquitectónico

El sistema sigue el patrón de **microservicios**, donde:

- **`ctrllt`** actúa como **API Gateway / Pasarela**, recibiendo todas las peticiones del cliente y enrutándolas al servicio apropiado.
- Cada servicio (**`gesfich`**, **`gesprog`**, **`ejecutor`**) es independiente, con su propia responsabilidad y máquina de estados.
- Los servicios **`gesfich`**, **`gesprog`** y **`ejecutor`** acceden **directamente** a `aralmac` para leer y escribir datos. El `ejecutor` en particular no consulta a `gesfich` ni a `gesprog` a través de sus tuberías, sino que va directo al almacén.
- `ctrllt` **no accede** a `aralmac`.

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

### 6.3 Concurrencia y Modelo de Cliente

- El sistema opera con **un único cliente** conectado a `ctrllt`.
- La concurrencia relevante ocurre dentro de **`ctrllt`**, que debe manejar mensajes entrantes del cliente y respuestas de múltiples servicios usando **hilos (threads)**.
- Las escrituras en las tuberías son **atómicas** a nivel del sistema operativo, por lo que los mensajes no se mezclan entre sí.
- El `request_id` de cada mensaje permite correlacionar peticiones con sus respuestas en todo momento.

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

> **Importante:** Si el fichero está siendo utilizado por un proceso de lotes activo, la operación retorna el error `FILE_IN_USE` y el fichero **no se borra**. Se asume que el cliente es responsable de no intentar borrar ficheros que estén en uso.

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

**Respuesta de error (fichero en uso):**
```json
{
  "request_id": "uuid-v4",
  "status": "error",
  "data": null,
  "error": {
    "code": "FILE_IN_USE",
    "message": "El fichero 'f-0001' está siendo utilizado por el proceso proc-0001"
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

Ejecuta un proceso de lotes siguiendo el formato de cadena:

**`fichero_entrada → programa_1 → programa_2 → ... → fichero_salida`**

El fichero de entrada y el fichero de salida son **obligatorios**. La lista de programas define la cadena de procesos que se ejecutarán secuencialmente, donde la salida de uno es la entrada del siguiente.

La operación es **no bloqueante**: el ejecutor valida la petición, crea el proceso de lotes, y devuelve inmediatamente el identificador. Internamente lanza un hilo que espera a que el lote termine, momento en el que envía una notificación al cliente.

**Operación:** `ejecutar_proceso`

**Petición:**
```json
{
  "request_id": "uuid-v4",
  "operation": "ejecutar_proceso",
  "service": "ejecutor",
  "params": {
    "id_fichero_entrada": "f-0001",
    "programas": ["p-0001", "p-0002"],
    "id_fichero_salida": "f-0002"
  }
}
```

**Respuesta inmediata** (al recibir la petición válida):
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

**Notificación de fin** (enviada cuando el lote termina):
```json
{
  "request_id": "uuid-v4",
  "status": "success",
  "data": {
    "id_proceso": "proc-0001",
    "estado": "terminado",
    "razon_fin": "completado"
  }
}
```

> **Nota:** Si la petición es inválida (algún ID no existe), el ejecutor responde con error inmediatamente y no crea el proceso.

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

**Estados posibles:** `ejecutando`, `terminado`, `cancelado`, `error`.

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

Termina forzadamente un proceso de lotes en ejecución. El proceso queda marcado como **cancelado por agente externo**.

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
    "estado": "cancelado",
    "razon_fin": "agente_externo"
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
   │ inicio │ ─────────────►│ Corriendo│ ──────────────►│ Terminando │──► Terminado
   └────────┘               └──────────┘                └────────────┘
```

| Estado | Descripción |
|--------|-------------|
| `inicio` | Estado inicial antes de aceptar peticiones |
| `Corriendo` | Aceptando y procesando peticiones |
| `Terminando` | Enviando señal de `terminar` a gesfich, gesprog y ejecutor, esperando que finalicen |
| `Terminado` | Todos los servicios dependientes han terminado. ctrllt se apaga |

**Comportamiento al terminar:**
Al recibir la operación `terminar`, `ctrllt` sigue este orden antes de apagarse:
1. Envía `terminar` a `gesfich`, `gesprog` y `ejecutor`
2. Espera a que cada uno confirme que terminó
3. Una vez que todos han finalizado, `ctrllt` termina su propia ejecución

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
| `Suspendido` | No acepta nuevas peticiones. Los procesos de lotes **ya en ejecución continúan hasta terminar**. Las peticiones nuevas se rechazan con `SERVICE_UNAVAILABLE` |
| `Parar` | No acepta nuevos procesos. Espera a que todos los procesos activos terminen (graceful shutdown) |
| `Terminado` | Servicio finalizado |

**Diferencia entre Parar y Terminar:**
- **Parar:** Cierre ordenado. El ejecutor deja de aceptar nuevos lotes y espera a que los activos terminen naturalmente antes de apagarse.
- **Terminar:** Apagado inmediato del servicio una vez que el estado `Parar` completó su ciclo (`/Procesos = 0`).

---

## 10. Códigos de Error

| Código | Descripción |
|--------|-------------|
| `INVALID_REQUEST` | Estructura de petición inválida |
| `INVALID_PARAMS` | Parámetros inválidos o faltantes |
| `FILE_NOT_FOUND` | Fichero no encontrado en aralmac |
| `FILE_IN_USE` | Fichero en uso por un proceso activo, no puede ser borrado |
| `PROGRAM_NOT_FOUND` | Programa no encontrado en aralmac |
| `PROCESS_NOT_FOUND` | Proceso no encontrado |
| `PROCESS_CANCELLED` | Proceso terminado por agente externo (matar) |
| `INVALID_STATE` | Operación no válida en el estado actual |
| `EXECUTABLE_INVALID` | El ejecutable no es válido o no existe |
| `STORAGE_ERROR` | Error en el área de almacenamiento |
| `PIPE_ERROR` | Error de comunicación en la tubería |
| `INTERNAL_ERROR` | Error interno del servicio |
| `SERVICE_UNAVAILABLE` | Servicio suspendido o terminado, no acepta peticiones |

