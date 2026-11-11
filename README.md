# API de Gestión de Pólizas (Spring Boot)

Implementación mínima funcional con Spring Boot 3, Java 17, Spring Data JPA y base de datos H2 en memoria.

## 1. Requisitos

- JDK 17 o superior.
- Maven 3.8 o superior.

## 2. Ejecutar

Desde la carpeta raíz del proyecto:

```bash
mvn spring-boot:run
```

La API queda disponible en `http://localhost:8080`. La base H2 se crea en memoria y carga dos pólizas de ejemplo al iniciar.

## 3. Seguridad

Todas las rutas requieren el header:

```http
x-api-key: 123456
```

La clave está configurada en `src/main/resources/application.properties` mediante `app.api-key`. Para otro entorno, cambiarla por variable/secret seguro; no dejar claves fijas en producción.

## 4. Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/polizas?tipo=COLECTIVA&estado=ACTIVA` | Lista y filtra pólizas por tipo y/o estado |
| GET | `/polizas/{id}/riesgos` | Lista riesgos de una póliza |
| POST | `/polizas/{id}/renovar` | Incrementa canon y prima según IPC y marca RENOVADA |
| POST | `/polizas/{id}/cancelar` | Cancela póliza y todos sus riesgos |
| POST | `/polizas/{id}/riesgos` | Agrega riesgo solo a póliza COLECTIVA |
| POST | `/riesgos/{id}/cancelar` | Cancela un riesgo |
| POST | `/core-mock/evento` | Registra en log el intento de actualización hacia CORE |

## 5. Ejemplos cURL

### Listar pólizas

```bash
curl -H "x-api-key: 123456" "http://localhost:8080/polizas?tipo=COLECTIVA&estado=ACTIVA"
```

### Consultar riesgos

```bash
curl -H "x-api-key: 123456" "http://localhost:8080/polizas/555/riesgos"
```

### Renovar con IPC del 5,2 %

El IPC se envía como porcentaje (5.2 significa 5,2 %):

```bash
curl -X POST "http://localhost:8080/polizas/555/renovar" \
  -H "x-api-key: 123456" -H "Content-Type: application/json" \
  -d '{"ipc": 5.2}'
```

La fórmula aplicada es `nuevoValor = valorActual * (1 + IPC / 100)`. Por ejemplo, un canon de 1.500.000 con IPC 5,2 % pasa a 1.578.000.

### Agregar riesgo

```bash
curl -X POST "http://localhost:8080/polizas/555/riesgos" \
  -H "x-api-key: 123456" -H "Content-Type: application/json" \
  -d '{"descripcion":"Apartamento 302","valorAsegurado":180000000}'
```

### Cancelar póliza

```bash
curl -X POST "http://localhost:8080/polizas/555/cancelar" \
  -H "x-api-key: 123456"
```

### Cancelar riesgo

```bash
curl -X POST "http://localhost:8080/riesgos/1001/cancelar" \
  -H "x-api-key: 123456"
```

### Probar el CORE mock directamente

```bash
curl -X POST "http://localhost:8080/core-mock/evento" \
  -H "x-api-key: 123456" -H "Content-Type: application/json" \
  -d '{"evento":"ACTUALIZACION","polizaId":555}'
```

La respuesta del mock confirma únicamente que el intento quedó registrado en el log de la aplicación. No representa una actualización real del CORE.

## 6. Reglas implementadas

- Una póliza INDIVIDUAL no admite agregar riesgos; el registro inicial de ejemplo tiene un único riesgo.
- Solo una póliza COLECTIVA admite agregar riesgos.
- No se renueva una póliza CANCELADA.
- Renovar incrementa canon y prima con el porcentaje IPC recibido y cambia estado a RENOVADA.
- Cancelar una póliza cambia su estado a CANCELADA y todos sus riesgos a CANCELADO.
- Cancelar un riesgo no elimina físicamente el registro.
- Se valida la existencia de pólizas y riesgos.
- Todas las rutas, incluido el mock, exigen `x-api-key`.

## 7. Estructura

```text
api-gestion-polizas/
├── pom.xml
├── README.md
└── src/
    └── main/
        ├── java/com/ejemplo/polizas/
        │   ├── ApiGestionPolizasApplication.java
        │   ├── config/
        │   │   ├── ApiKeyInterceptor.java
        │   │   └── WebConfig.java
        │   ├── controller/
        │   │   ├── PolizaController.java
        │   │   ├── RiesgoController.java
        │   │   └── CoreMockController.java
        │   ├── dto/
        │   │   ├── RenovacionRequest.java
        │   │   └── RiesgoRequest.java
        │   ├── entity/
        │   │   ├── Poliza.java
        │   │   └── Riesgo.java
        │   ├── repository/
        │   │   ├── PolizaRepository.java
        │   │   └── RiesgoRepository.java
        │   └── service/
        │       ├── PolizaService.java
        │       └── CoreMockClient.java
        └── resources/
            ├── application.properties
            └── data.sql
```

## 8. Consideraciones

- H2 es una base temporal de demostración. Para producción se debe configurar el motor corporativo y migraciones versionadas.
- El mock CORE solo deja evidencia en logs. No integra ni actualiza un CORE real.
- El incremento de IPC usa el porcentaje recibido en el body. Si el IPC debe obtenerse de una fuente oficial o de una tabla parametrizada, debe sustituirse ese dato de entrada.
- Esta versión es una base funcional simplificada; antes de producción deben añadirse pruebas de integración, control de concurrencia, auditoría persistente, autorización por roles y gestión segura de secretos.
