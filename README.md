# Nova Librería + Editorial AR

Proyecto integral de dos aplicaciones web conectadas por una API REST y una base de datos MySQL.

## 1. Estructura

- `web/nova/`: tienda en línea Nova.
- `web/editorial/`: panel del proveedor Editorial AR.
- `web/shared/`: estilos y JavaScript compartidos.
- `api/`: API REST Node.js/Express.
- `database/init.sql`: tablas y catálogo inicial.
- `nginx/`: proxy inverso y balanceo entre dos instancias de API.
- `docker-compose.yml`: infraestructura completa.
- `certs/`: certificados TLS para el dominio.

## 2. Requisitos

Docker Desktop (Windows/Mac) o Docker Engine + Docker Compose (Linux), y puertos 80 y 443 disponibles.

## 3. Preparación local

1. Copia `.env.example` como `.env`.
2. Cambia todas las contraseñas y `JWT_SECRET`. Usa una clave aleatoria larga para JWT.
3. Ejecuta `docker compose up -d --build`.
4. Abre `http://localhost/nova/` para la tienda.
5. Abre `http://localhost/editorial/` para el panel del proveedor.
6. Entra al panel con `ADMIN_USER` y `ADMIN_PASSWORD` definidos en `.env`.

La primera inicialización de MySQL carga el catálogo de ejemplo. Para reiniciar desde cero y volver a ejecutar el script SQL: `docker compose down -v` y después `docker compose up -d --build`. **Esto elimina los datos almacenados.**

## 4. Flujo de compra

1. El cliente abre Nova y consulta `/api/books`.
2. Agrega libros al carrito.
3. Captura nombre, correo, teléfono y dirección.
4. Nova envía `POST /api/orders` a la API.
5. La API valida los datos, verifica existencias y registra el pedido y sus partidas dentro de una transacción MySQL.
6. Editorial AR consulta `GET /api/admin/orders` con autenticación.
7. El proveedor actualiza el estado con `PATCH /api/admin/orders/:id/status`.

El pedido aparece en Editorial AR porque ambas aplicaciones consultan la misma API y base de datos; no se depende de copiar datos manualmente.

## 5. API REST

| Método | Ruta | Función |
|---|---|---|
| GET | `/api/health` | Estado de API y conexión a DB |
| GET | `/api/books` | Catálogo público |
| POST | `/api/orders` | Crear pedido desde Nova |
| POST | `/api/admin/login` | Autenticación del proveedor |
| GET | `/api/admin/orders` | Consultar pedidos (JWT) |
| PATCH | `/api/admin/orders/:id/status` | Cambiar estado (JWT) |

Ejemplo de cuerpo para crear pedido:
```json
{
  "customer_name": "Cliente de prueba",
  "email": "cliente@ejemplo.com",
  "phone": "5550000000",
  "address": "Calle de prueba 123",
  "items": [{"book_id": 1, "quantity": 2}]
}
```

## 6. Infraestructura de despliegue

- **Servidor de aplicaciones:** dos contenedores Node.js (`api1` y `api2`) ejecutan la API.
- **Servidor de base de datos:** MySQL 8.4 en contenedor independiente, con volumen persistente.
- **Balanceo de cargas:** Nginx distribuye solicitudes `/api/` entre las dos instancias con `least_conn`.
- **Dominio:** configura los registros DNS A/AAAA de los dominios para apuntar a la IP pública del servidor. Ajusta `server_name` en Nginx para los dominios definitivos.
- **TLS/HTTPS:** instala certificados válidos de Let's Encrypt o de tu proveedor en `certs/` y agrega el bloque `listen 443 ssl` a Nginx. La configuración inicial se entrega en HTTP para permitir pruebas locales; HTTPS no queda activo hasta instalar un certificado real.
- **Red:** MySQL no publica puertos al exterior; solamente se conecta a la red interna de Docker.

### Publicación en un servidor Linux

1. Contrata una VM/VPS con Docker y Docker Compose.
2. Apunta los dominios al IP público y permite tráfico 80/443 en firewall.
3. Sube esta carpeta al servidor y crea `.env` con secretos únicos.
4. Configura los nombres de dominio y certificados TLS en Nginx.
5. Ejecuta `docker compose up -d --build`.
6. Verifica `https://tudominio/nova/`, `https://tudominio/editorial/` y `/api/health`.
7. Configura respaldos periódicos de MySQL y monitoreo de logs.

No publiques el panel administrativo sin HTTPS, contraseñas robustas y restricciones de acceso. Para un entorno de producción conviene añadir limitación de intentos, recuperación de contraseña, auditoría, backups probados y gestión de secretos.

## 7. Pruebas de liberación

| Prueba | Procedimiento | Resultado esperado |
|---|---|---|
| Disponibilidad | Abrir `/api/health` | Respuesta JSON `status: ok` |
| Catálogo | Abrir Nova | Libros y precios cargan desde MySQL |
| Carrito | Agregar y quitar libros | Cantidades y total correctos |
| Pedido | Enviar formulario válido | Respuesta 201 y número de pedido |
| Integración | Entrar a Editorial AR | El pedido recién creado aparece |
| Seguridad | Consultar pedidos sin token | Respuesta 401 |
| Validación | Enviar cantidad inválida | Respuesta 400 |
| Inventario | Solicitar más unidades que stock | Respuesta 409 |
| Estado | Cambiar a “En preparación” | El nuevo estado queda guardado |
| Balanceo | Consultar varias veces `/api/health` | Se observan instancias api1/api2 |
| Persistencia | Reiniciar contenedores | Los datos siguen en MySQL |

## 8. Criterios de aceptación

- Ambas interfaces abren correctamente.
- El catálogo se obtiene desde la API.
- Los pedidos se guardan en MySQL.
- El proveedor puede consultar y actualizar pedidos.
- Las rutas administrativas requieren token.
- Los servicios se levantan mediante Docker Compose.
- El servidor web distribuye las solicitudes de API entre dos instancias.

## 9. Alcance y consideraciones

Este proyecto es una base funcional para demostración académica y despliegue inicial. No integra cobros bancarios, facturación fiscal, cálculo de paquetería ni envío de correos. El pedido queda registrado como solicitud de compra, no como pago confirmado. Los certificados, dominios, DNS, firewall y servidor cloud deben configurarse con los datos reales del proveedor de hosting.
