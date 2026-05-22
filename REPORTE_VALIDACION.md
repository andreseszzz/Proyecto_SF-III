# Reporte de Validación - Proyecto S3
Fecha: 2026-05-21

## 1. Lógica de Negocio (matriculas-service)

- **Validación de Estudiante**: Se consulta al `estudiantes-service` antes de matricular.
- **Validación de Curso**: Se consulta al `cursos-service` antes de matricular.
- **Prevención de Duplicados**: Se evita que un estudiante se matricule dos veces en el mismo curso activo.
- **Manejo de Anulaciones**: Las matrículas se pueden anular cambiando el estado a 'ANULADA'.

## 2. Seguridad y Roles

- **Configuración de Roles**:
    - ADMIN: Acceso completo (CRUD).
    - DOCENTE: Solo lectura en Estudiantes y Matrículas.
    - ESTUDIANTE: Solo lectura en Cursos.
- **Respuestas de Error**: Se usa `ApiErrorResponse` con códigos como `AUTH_FORBIDDEN` y `AUTH_HEADER_MISSING`.
- **Filtros de Seguridad**: Se implementó `AuthValidationFilter` alineado con el `auth-service`.

## 3. Pruebas Funcionales

Se validó el flujo completo ejecutando los servicios localmente.

### 3.1 Pruebas de Autenticación

#### Test 1: Login de Administrador

**Comando:**
```bash
curl -X POST http://localhost:8081/auth/login \
-H "Content-Type: application/json" \
-d '{"username": "admin", "password": "admin123"}'
```
**Respuesta:**
```json
{"token": "eyJhbG...ttbA","username":"admin","role":"ADMIN"}
```

La autenticación funcionó correctamente, se obtuvo el token con rol ADMIN.

### 3.2 Pruebas de Integración de Servicios

#### Test 2: Acceso a Estudiantes con Token Válido

**Comando:**
```bash
# 1. Obtención de token dinámico
TOKEN=$(curl -s -X POST http://localhost:8081/auth/login -d '{"username": "admin", "password": "admin123"}' | jq -r '.token')

# 2. Petición al servicio de estudiantes
curl -X GET http://localhost:8082/api/estudiantes \
-H "Authorization: Bearer $TOKEN"
```
**Respuesta:**
```json
{
  "success": true,
  "message": "Consulta exitosa",
  "data": [
    {
      "id": 1,
      "nombre": "Ana",
      "apellido": "Martínez",
      "email": "ana@correo.edu",
      "edad": 19
    }
  ]
}
```

Se confirmó que `estudiantes-service` y `auth-service` se comunican correctamente.

### 3.3 Pruebas de Reglas de Negocio en Vivo

#### Test 3: Registro de Matrícula Correcta

**Comando:**
```bash
curl -X POST http://localhost:8084/api/matriculas \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"estudianteId": 1, "cursoId": 1}'
```
**Respuesta:**
```json
{
  "success": true,
  "message": "Matrícula registrada",
  "data": {
    "id": 1,
    "estudianteId": 1,
    "cursoId": 1,
    "fechaMatricula": "2026-05-21",
    "estado": "ACTIVA"
  }
}
```

La matrícula se registró correctamente, validando la existencia del estudiante y curso.

#### Test 4: Validación de Matrícula Duplicada

**Comando:**
(Se repite la misma petición del Test 3)
```bash
curl -X POST http://localhost:8084/api/matriculas \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"estudianteId": 1, "cursoId": 1}'
```
**Respuesta:**
```json
{
  "success": false,
  "message": "Ya existe una matrícula activa para el estudiante y curso indicados",
  "data": null
}
```

La validación de duplicados funcionó, se rechazó la segunda matrícula.

### 3.4 Validación de Seguridad por Roles (Acceso Prohibido)

#### Test 5: Intento de Eliminación con Rol Insuficiente

Un usuario con rol `DOCENTE` intenta eliminar un estudiante (solo ADMIN puede).

**Comando:**
```bash
# Obtención de token de DOCENTE
TOKEN_DOCENTE=$(curl -s -X POST http://localhost:8081/auth/login \
-d '{"username": "docente", "password": "docente123"}' | jq -r '.token')

# Intento de DELETE
curl -X DELETE http://localhost:8082/api/estudiantes/1 \
-H "Authorization: Bearer $TOKEN_DOCENTE"
```
**Respuesta:**
```json
{
  "success": false,
  "message": "No tiene permisos para acceder a este recurso",
  "errorCode": "AUTH_FORBIDDEN",
  "status": 403,
  "path": "/api/estudiantes/1",
  "timestamp": "2026-05-21T19:21:18.005635187"
}
```

El acceso se bloqueó correctamente con código 403.

---
## Conclusiones

Todos los servicios se encuentran integrados y funcionando. Se validaron correctamente:
- Las reglas de negocio en matriculas-service
- Los mecanismos de seguridad y roles
- La integración entre servicios
- El manejo de errores
