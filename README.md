# Proyecto #2 – Sistema de Reserva de Recursos (Frontend – Backend – Base de datos)

EIF206 Programación 3 (2026-II). Nueva versión del Proyecto #1 con **arquitectura distribuida**:
dos procesos independientes (Frontend y Backend) que se comunican por **sockets** y usan **hilos**;
el Backend es el único que accede a **MySQL**. Puede haber **varios Frontends conectados a un único
Backend**, cada uno en una máquina distinta de la red.

```
┌────────────────────┐   sockets (TCP)   ┌──────────────────────────┐   JDBC   ┌───────┐
│ Frontend (Swing)   │ ───────────────►  │ Backend (sin UI)         │ ───────► │ MySQL │
│  MVC + ServiceProxy│ ◄───────────────  │  ServerSocket + hilos    │          └───────┘
│  Área de Alertas   │   respuestas y    │  Service + DAOs          │
└────────────────────┘   notificaciones  └──────────────────────────┘
        ▲  ▲  ▲  (varios Frontends)                  │
        └──┴──┴──────── alerta asincrónica de cada nueva reserva a los demás Frontends activos
```

## Módulos Maven

| Módulo | Contenido |
|---|---|
| `common` | Modelo serializable (`Reserva`, `Recurso`…), contrato `ReservaService`, protocolo de red (`Solicitud`, `Respuesta`, `NotificacionReserva`, `Canal`) y filtro de deserialización. Sin Swing ni JDBC. |
| `backend` | Capas **Lógica y Datos**: `ServidorReservas` (un hilo por cliente), `Despachador` (sesión y roles), `ReservaServiceImpl` (lógica de negocio), `PersistenciaService` + DAOs (JDBC/MySQL), importador XML→MySQL. **Sin interfaz de usuario.** |
| `frontend` | Capa de **Presentación** (MVC, Swing). `ServiceProxy` (patrón **Proxy**), `ConexionBackend` (socket + hilo lector), área de alertas. **Sin ninguna referencia a la base de datos.** |
| `integration-tests` | Pruebas de integración de punta a punta (Backend real + varios Frontends reales + MySQL). Solo pruebas. |

### Patrón Proxy
El Frontend nunca llama a lógica local: los controllers usan `ServiceProxy`, que implementa el mismo contrato
`ReservaService` que el Service real (`ReservaServiceImpl`, en el Backend). Cada llamada se convierte en una
`Solicitud` enviada por el socket; el Backend la despacha al Service real y devuelve una `Respuesta`
(o un error con mensaje listo para el usuario, que el Proxy convierte en `ServicioException`).

### Alertas asincrónicas
1. Un usuario registra una reserva → el Backend la valida y guarda en MySQL.
2. El Backend envía una `NotificacionReserva` a **todos los demás Frontends con sesión iniciada** (no al que reservó).
3. En cada Frontend, el hilo lector del socket entrega la notificación al `ServiceProxy` → `AlertasController`
   la agrega (en el hilo de Swing) al `AlertasModel` y aparece en el **Área de Alertas** (derecha de la
   pantalla principal, visible sin importar la pestaña activa).
4. **Ver** (ícono carpeta) abre el detalle (Encargado, Fecha, Desde, Hasta, Recursos). **Descartar**
   (ícono rojo) pide confirmación y quita la alerta **solo de la lista local**; no se envía nada al Backend.

Las alertas son por sesión: solo se muestran las reservas hechas por otros mientras la sesión está activa.

## Requisitos
- JDK 17 o superior y Maven 3.9+
- MySQL 8 (o compatible)

## 1. Base de datos
1. Ejecutar `database/esquema_reservas_mysql.sql` (crea la base `reservas_recursos` y sus 7 tablas).
2. Crear un usuario para el Backend y darle permisos, por ejemplo:
   ```sql
   CREATE USER 'reservas_backend'@'%' IDENTIFIED BY 'una_clave_segura';
   GRANT ALL PRIVILEGES ON reservas_recursos.* TO 'reservas_backend'@'%';
   ```
3. (Opcional) Cargar los datos del Proyecto #1 con el importador, sobre una base vacía:
   `java -cp backend/target/backend-all.jar cr.ac.una.reservas.backend.migration.XmlToMySqlImporter data/datos.xml`

Variables de entorno del **Backend** (no publicar contraseñas en GitHub):

| Variable | Valor |
|---|---|
| `RESERVAS_DB_URL` | `jdbc:mysql://localhost:3306/reservas_recursos` (por defecto) |
| `RESERVAS_DB_USER` | usuario de MySQL |
| `RESERVAS_DB_PASSWORD` | contraseña |
| `RESERVAS_PORT` | puerto de escucha (por defecto `5050`) |
| `RESERVAS_ADMIN_ID` / `RESERVAS_ADMIN_PASSWORD` | cuenta de administración que se crea en la primera ejecución (por defecto `admin` / `admin`; **cámbiela** al ingresar) |

## 2. Compilar
```
mvn clean package -DskipTests      # genera backend/target/backend-all.jar y frontend/target/frontend-all.jar
```

## 3. Ejecutar
**Backend** (en la máquina del servidor):
```
java -jar backend/target/backend-all.jar [puerto]
```
**Frontend** (en cada máquina cliente; puede ser otra máquina de la red):
```
java -jar frontend/target/frontend-all.jar <ip-o-host-del-backend> [puerto]
```
También se puede indicar con `RESERVAS_HOST` y `RESERVAS_PORT`. Si no se logra conectar, la aplicación pregunta
la dirección del servidor. Recuerde abrir el puerto (por defecto 5050) en el firewall del servidor.

Desde el IDE: ejecutar `cr.ac.una.reservas.backend.Main` y luego `cr.ac.una.reservas.Application`
(varias veces, para simular varios usuarios).

Usuarios: el administrador (`admin`) crea funcionarios; la clave inicial de cada funcionario es su propio ID
(puede cambiarla desde *Cuenta → Cambiar clave* o desde el botón *Cambiar* del login).

## 4. Pruebas (JUnit Jupiter)
- **Unitarias (Surefire)**, clases `*Test`, sin base de datos: `mvn test`
- **Integración (Failsafe)**, clases `*IT`: requieren MySQL y las variables `RESERVAS_DB_*`:
  ```
  mvn verify
  ```
  - `backend`: DAOs contra MySQL y `ServidorReservasIT` (sockets reales, alertas, roles, concurrencia).
  - `integration-tests`: `SistemaDistribuidoIT` (Backend real + 3 Frontends con `ServiceProxy` y área de alertas).
  Al ejecutar estas pruebas desde IntelliJ, configure `RESERVAS_DB_USER` y `RESERVAS_DB_PASSWORD`
  en la configuración de ejecución; sin ellas, las pruebas `*IT` que usan MySQL se omiten.

## Reglas de negocio principales (validadas en el Backend)
- Reservas: fecha/hora futura, hora final > inicial, una o más categorías; se asigna el primer recurso libre de
  cada categoría; si falta disponibilidad se informa en qué categorías y no se guarda nada.
- Solo se cancelan reservas futuras activas y solo por su dueño.
- No se elimina un funcionario con reservas pendientes, ni una categoría con recursos, ni un recurso en una
  reserva activa.
- Estadísticas: recursos usados por categoría (de mayor a menor) y actividades por semana (clave = lunes de la
  semana, incluye semanas en 0).
- Seguridad: sesión obligatoria y permisos por rol en el servidor; las claves se guardan con PBKDF2; la
  deserialización de mensajes acepta únicamente las clases del protocolo.
