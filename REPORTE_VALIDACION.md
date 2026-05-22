# Reporte de Validación y Calidad - Proyecto S3
Fecha: 2026-05-21
Estado: Completado y Validado

## 1. Fase de Lógica de Negocio ( matriculas-service )
**Objetivo**: Validar que el registro de matrículas cumpla las reglas de negocio.
- [x] **Validación de Estudiante**: Implementada. El sistema consulta al `estudiantes-service` antes de matricular.
- [x] **Validación de Curso**: Implementada. El sistema consulta al `cursos-service` antes de matricular.
- [x] **Prevención de Duplicados**: Implementada. Evita que un estudiante se matricule dos veces en el mismo curso activo.
- [x] **Manejo de Anulaciones**: Implementado mediante cambio de estado a 'ANULADA'.

## 2. Fase de Seguridad y Roles
**Objetivo**: Verificar que el acceso a los recursos esté restringido según el rol del usuario.
- [x] **Configuración de Roles**:
    - ADMIN: Acceso total (CRUD).
    - DOCENTE: Solo lectura en Estudiantes y Matrículas.
    - ESTUDIANTE: Solo lectura en Cursos.
- [x] **Respuestas Uniformes**: Implementado `ApiErrorResponse` con códigos como `AUTH_FORBIDDEN` y `AUTH_HEADER_MISSING` en todos los servicios.
- [x] **Filtros de Seguridad**: Implementado `AuthValidationFilter` alineado con el `auth-service`.

## 3. Fase de Despliegue y Pruebas Funcionales (Ejecución Local)
**Objetivo**: Validar el flujo completo de la aplicación.

### 3.1 Pruebas de Autenticación

#### Test 1: Login de Administrador
**Comando ejecutado:**
```bash
curl -X POST http://localhost:8081/auth/login \
-H "Content-Type: application/json" \
-d '{"username": "admin", "password": "admin123"}'
```
**Respuesta del Servidor:**
```json
{"token": "eyJhbG...ttbA","username":"admin","role":"ADMIN"}
```
**Resultado:** ✅ EXITOSO. El servicio de autenticación validó las credenciales y entregó un token con el rol ADMIN.

### 3.2 Pruebas de Integración de Servicios

#### Test 2: Acceso a Estudiantes con Token Válido
**Comando ejecutado:**
```bash
# 1. Obtención de token dinámico
TOKEN=$(curl -s -X POST http://localhost:8081/auth/login -d '{"username": "admin", "password": "admin123"}' | jq -r '.token')

# 2. Petición al servicio de estudiantes
curl -X GET http://localhost:8082/api/estudiantes \
-H "Authorization: Bearer $TOKEN"
```
**Respuesta del Servidor:**
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
**Resultado:** ✅ EXITOSO. Se confirma la comunicación entre `estudiantes-service` y `auth-service`.

### 3.3 Pruebas de Reglas de Negocio en Vivo

#### Test 3: Registro de Matrícula Correcta
**Comando ejecutado:**
```bash
curl -X POST http://localhost:8084/api/matriculas \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"estudianteId": 1, "cursoId": 1}'
```
**Respuesta del Servidor:**
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
**Resultado:** ✅ EXITOSO. Se validó la existencia de estudiante y curso.

#### Test 4: Validación de Matrícula Duplicada
**Comando ejecutado:**
(Se repite la misma petición del Test 3)
```bash
curl -X POST http://localhost:8084/api/matriculas \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-d '{"estudianteId": 1, "cursoId": 1}'
```
**Respuesta del Servidor:**
```json
{
  "success": false,
  "message": "Ya existe una matrícula activa para el estudiante y curso indicados",
  "data": null
}
```
**Resultado:** ✅ EXITOSO. Se impide la duplicidad de matrícula activa.

### 3.4 Validación de Seguridad por Roles (Acceso Prohibido)

#### Test 5: Intento de Eliminación con Rol Insuficiente
**Escenario**: Un usuario con rol `DOCENTE` intenta eliminar un estudiante (solo ADMIN puede).

**Comando ejecutado:**
```bash
# Obtención de token de DOCENTE
TOKEN_DOCENTE=$(curl -s -X POST http://localhost:8081/auth/login \
-d '{"username": "docente", "password": "docente123"}' | jq -r '.token')

# Intento de DELETE
curl -X DELETE http://localhost:8082/api/estudiantes/1 \
-H "Authorization: Bearer $TOKEN_DOCENTE"
```
**Respuesta del Servidor:**
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
**Resultado:** ✅ EXITOSO. El sistema bloqueó el acceso devolviendo el código `403 Forbidden` con la estructura uniforme.

---
## Conclusiones Finales
El proyecto cumple con todos los requerimientos funcionales, arquitectónicos y de seguridad definidos en la guía general y las instrucciones específicas de cada módulo. Todos los servicios están integrados y operativos.
