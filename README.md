# Documentación - Ejecutor de Lotes

Este directorio contiene la documentación de la API del sistema **Ejecutor de Lotes**, correspondiente a la **primera entrega** de la práctica del curso de Sistemas Operativos (ST0257).

**Integrantes del grupo:** 
- Nicolas Rico Montesino 
- Santiago Alvarez Diaz

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `Diseño.md` | Documento principal con el diseño completo del sistema, arquitectura, componentes, operaciones y flujos de ejemplo. |
| `openapi.yaml` | Especificación OpenAPI 3.0 (Swagger) de la API. |
| `README.md` | Este archivo - guía para visualizar la documentación. |

---

## Cómo visualizar la documentación

### Opción 1: Swagger Editor Online

1. Abre tu navegador en: **https://editor.swagger.io/**
2. En el menú superior haz clic en **File → Import file**
3. Selecciona el archivo `openapi.yaml`
4. La documentación se renderizará automáticamente en el panel derecho.


---

### Opción 2: Swagger UI Local (Docker)

Si tienes Docker instalado, puedes correr Swagger UI localmente:

```bash
docker run -p 8080:8080 \
  -e SWAGGER_JSON=/api/openapi.yaml \
  -v $(pwd):/api \
  swaggerapi/swagger-ui
```

Luego abre tu navegador en: `http://localhost:8080`

---

### Opción 3: Extensión de VS Code

Si usas VS Code:

1. Instala la extensión **"Swagger Viewer"** (de Arjun G).
2. Abre el archivo `openapi.yaml`.
3. Presiona `Shift + Alt + P` (o `Cmd + Shift + P` en Mac) y selecciona **"Preview Swagger"**.

> Otras extensiones útiles:
> - **OpenAPI (Swagger) Editor** - de 42Crunch
> - **YAML** - de Red Hat (validación)

---


## Cómo leer la documentación

### 1. Empezar por `Diseño.md`

Este documento contiene:
- Descripción general del sistema
- Arquitectura y componentes
- Formato de mensajes JSON
- Operaciones detalladas con ejemplos
- Máquinas de estado
- Códigos de error
- Flujos de ejemplo completos

### 2. Consultar `openapi.yaml`

Para detalles técnicos específicos de cada operación:
- Endpoints disponibles
- Esquemas de request/response
- Códigos de respuesta
- Ejemplos de uso

---

## Estructura del Sistema

El sistema sigue una **arquitectura de microservicios**:

```
cliente → ctrllt (pasarela) → [gesfich | gesprog | ejecutor] → aralmac
```

| Componente | Responsabilidad |
|------------|-----------------|
| **cliente** | Realiza peticiones (CRUD de programas/ficheros, ejecución de procesos) |
| **ctrllt** | Pasarela que enruta peticiones al servicio adecuado |
| **gesfich** | CRUD de ficheros en `aralmac` |
| **gesprog** | CRUD de programas en `aralmac` |
| **ejecutor** | Ejecuta procesos de lotes |
| **aralmac** | Almacenamiento compartido |

---

## Convenciones

| Elemento | Formato | Ejemplo |
|----------|---------|---------|
| ID Fichero | `f-XXXX` | `f-0001` |
| ID Programa | `p-XXXX` | `p-0001` |
| ID Proceso | `proc-XXXX` | `proc-0001` |
| Request ID | UUID v4 | `550e8400-e29b-41d4-a716-446655440000` |

---
