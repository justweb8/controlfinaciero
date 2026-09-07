# 💰 Control Financiero — Sistema de Gestión para Constructora

> **"Entre decisiones, construimos resultados"**

Sistema web PWA de control financiero diseñado específicamente para gestionar los aportes, saldos, préstamos bancarios y movimientos de los socios de una constructora. Funciona en PC y celular, se instala como app nativa y se conecta a Supabase como base de datos en la nube.

---

## 🗂️ ESTRUCTURA DE ARCHIVOS

```
constructora-pwa/
├── index.html              → App completa (HTML + CSS + JS en un solo archivo)
├── manifest.json           → Configuración PWA (nombre, íconos, colores)
├── sw.js                   → Service Worker (caché offline)
├── logo-full.png           → Logo "Control Financiero" (cabecera y login)
├── favicon.png             → Ícono pestaña navegador (32x32)
├── icon-192.png            → Ícono app Android (192x192)
├── icon-512.png            → Ícono app grande (512x512)
├── icon-maskable-512.png   → Ícono adaptable Android con fondo
├── apple-touch-icon.png    → Ícono pantalla inicio iOS (180x180)
└── README.md               → Este archivo
```

---

## 🔐 ACCESO AL SISTEMA

El sistema tiene login con usuario y contraseña. Los usuarios están guardados en la tabla `usuarios` de Supabase.

| Usuario | Contraseña | Rol |
|---|---|---|
| admin@gmail.com | 1234 | Administrador |
| socio@constructora.com | socio123 | Socio |

> Para agregar más usuarios: ve a **Supabase → Table Editor → usuarios → Insert row**

---

## ☁️ BASE DE DATOS (SUPABASE)

El sistema usa **Supabase** como base de datos en la nube. URL del proyecto:
```
https://afhnepfgvutfcozsbqct.supabase.co
```

### Tablas creadas:

#### `socios`
Almacena a cada socio y su saldo actual.
| Campo | Tipo | Descripción |
|---|---|---|
| id | SERIAL | ID único auto-incremental |
| nombre | TEXT | Nombre del socio |
| saldo | NUMERIC(12,2) | Saldo actual (+ empresa le debe, - socio le debe) |
| created_at | TIMESTAMPTZ | Fecha de creación |

#### `prestamos`
Almacena los financiamientos externos (bancos, proveedores, etc.)
| Campo | Tipo | Descripción |
|---|---|---|
| id | SERIAL | ID único |
| entidad | TEXT | Nombre del banco o acreedor |
| monto_original | NUMERIC(12,2) | Monto total del préstamo |
| saldo_pendiente | NUMERIC(12,2) | Lo que falta por pagar |
| cuota_mensual | NUMERIC(12,2) | Cuota mensual |
| proxima_cuota | DATE | Fecha de próximo vencimiento |
| descripcion | TEXT | Notas adicionales |
| created_at | TIMESTAMPTZ | Fecha de registro |

#### `historial`
Todos los movimientos del sistema. Es la tabla principal.
| Campo | Tipo | Descripción |
|---|---|---|
| id | SERIAL | ID único |
| socio_id | INT | FK → socios.id (puede ser NULL) |
| prestamo_id | INT | FK → prestamos.id (puede ser NULL) |
| tipo | TEXT | Tipo de movimiento (ver tabla abajo) |
| descripcion | TEXT | Descripción del movimiento |
| monto | NUMERIC(12,2) | Monto en soles |
| fecha | DATE | Fecha del movimiento |
| created_at | TIMESTAMPTZ | Fecha de registro |

#### `usuarios`
Usuarios con acceso al sistema.
| Campo | Tipo | Descripción |
|---|---|---|
| id | SERIAL | ID único |
| email | TEXT | Correo (único) |
| password_hash | TEXT | Contraseña (texto plano por ahora) |
| nombre | TEXT | Nombre completo |
| rol | TEXT | 'admin' o 'socio' |

---

## 📋 TIPOS DE MOVIMIENTO

Estos son los valores del campo `tipo` en la tabla `historial`:

| Tipo | Significado | Efecto en saldo del socio |
|---|---|---|
| `GASTO_BOLSILLO` | El socio puso dinero de su bolsillo para la obra | **Saldo sube** (empresa le debe más) |
| `PAGO_CUOTA` | Un socio pagó una cuota del banco | **Saldo sube** (empresa le debe más) |
| `PAGO_CUOTA_DIST` | Varios socios pagan la cuota cada uno con su monto | **Saldo sube** para cada uno |
| `DEVOLUCION` | La empresa le devolvió dinero al socio | **Saldo baja** (empresa le debe menos) |
| `PRESTAMO_EMPRESA` | El socio retiró dinero de la empresa | **Saldo baja** (socio le debe a empresa) |
| `FINANCIAMIENTO` | Ingreso de un préstamo externo (banco) | No afecta saldo de socios |

---

## 📱 SECCIONES DEL SISTEMA

### 1. 🏠 BALANCES (Resumen general)
Pantalla principal. Muestra:
- **Balance Neto** = Total aportes de socios − Deuda externa con bancos
- **4 KPIs:** Total aportes socios / Empresa debe a socios / Socios deben a empresa / Deuda externa
- **Cuentas de Socios:** Muestra máximo 6. Si hay más → botón "Ver todas" va a sección Socios
- **Financiamientos:** Muestra máximo 3 tarjetas oscuras con progreso de pago. Si hay más → chip "Ver X más" va a página Financiamientos

### 2. ➕ NUEVO MOVIMIENTO
Formulario para registrar cualquier transacción:
- Selecciona **tipo** de movimiento (cambia el formulario automáticamente)
- Selecciona **socio** afectado
- Si es `PAGO_CUOTA` → aparece selector de préstamo para descontar del saldo pendiente
- Si es `PAGO_CUOTA_DIST` → aparece campo de monto individual para cada socio
- Campos: descripción, monto, fecha
- Al guardar → actualiza saldo del socio en Supabase y registra en historial

### 3. 🏦 NUEVO PRÉSTAMO
Formulario para registrar un nuevo financiamiento externo:
- Entidad (banco o acreedor)
- Monto original
- Cuota mensual
- Próxima fecha de cuota
- Descripción opcional
- Al guardar → crea registro en `prestamos` y registra en `historial` como FINANCIAMIENTO

### 4. 📊 HISTORIAL
Tabla completa de todos los movimientos:
- **Filtros:** por socio, por tipo de movimiento, búsqueda por texto
- **Ordenar** por fecha o monto (click en cabecera)
- **Paginación:** 10 registros por página
- **Acciones por fila:** Editar (cambia monto/descripción/fecha y recalcula saldo) / Eliminar (revierte el saldo)
- **Exportar Excel** → descarga `.xlsx` con todos los movimientos
- **Exportar PDF** → abre ventana de impresión con reporte completo
- En móvil muestra cards en lugar de tabla

### 5. 📈 GRÁFICOS
Reportes visuales con Chart.js:
- **Saldos por socio:** Barras de color por socio (verde = positivo, rojo = negativo)
- **Aportes por socio:** Cuánto ha aportado cada uno en total
- **Distribución por tipo:** Gráfico donut con % de cada tipo de movimiento
- **Avance de préstamos:** Barras de progreso de cada financiamiento
- 3 KPIs resumen: Saldo total / Total ingresos / Total egresos
- Botones Exportar Excel y PDF

### 6. 👥 SOCIOS
Administración completa de socios:
- **Desktop:** Tabla con columnas (nombre, saldo, estado, fecha, acciones)
- **Mobile:** Cards individuales
- **Filtro dropdown:** Todos / Saldo positivo / Saldo negativo / Sin movimientos
- **Búsqueda** en tiempo real por nombre
- **Botones por socio:**
  - 👁 Ver detalles → modal con historial del socio
  - ➕ Nueva transacción → va al formulario preseleccionando al socio
  - 🗑 Eliminar → solo si saldo = 0
- **Agregar nuevo socio** → modal con campo de nombre

### 7. 💳 FINANCIAMIENTOS (página completa)
Vista dedicada a todos los préstamos:
- Header azul oscuro con KPIs: Total préstamos / Deuda pendiente / Pagado (%)
- **Filtros:** Todos / Activos / Pagados
- **Tarjetas expandidas** con:
  - Nombre del banco
  - Donut de progreso (% pagado)
  - Cuota mensual y saldo pendiente
  - Socios que han aportado a ese préstamo
  - Próxima fecha de cuota
  - Botones: Ver detalle → modal / Pagar → formulario
- Botones Exportar Excel y PDF
- Botón ← Balances para volver

---

## 🔔 MODAL DE SOCIO
Al hacer click en cualquier socio (en Balances o en Socios):
- Header de color según estado (verde = al día, amarillo = empresa le debe, rojo = debe a empresa)
- Avatar con iniciales del socio
- Saldo actual destacado
- Historial completo de movimientos de ese socio
- Botón "Nueva transacción" → va al formulario

## 🔔 MODAL DE PRÉSTAMO
Al hacer click en cualquier tarjeta de préstamo:
- KPIs: Pendiente / Cuota mensual / Pagado
- Barra de progreso animada
- Lista de aportes por socio
- Historial de pagos del préstamo
- Botón "💳 Registrar pago" → va al formulario con el préstamo preseleccionado
- Botón "🗑 Eliminar" → con confirmación

---

## 🧮 LÓGICA DE CÁLCULOS

### Saldo de cada socio:
```
Saldo = Σ(GASTO_BOLSILLO + PAGO_CUOTA + PAGO_CUOTA_DIST) − Σ(DEVOLUCION + PRESTAMO_EMPRESA)
```
- Saldo **positivo** = La empresa le debe al socio
- Saldo **negativo** = El socio le debe a la empresa
- Saldo **cero** = Cuenta al día

### Balance Neto (hero card):
```
Balance Neto = Total aportes de socios − Deuda externa total
```

### Progreso de préstamo:
```
% Pagado = ((Monto original − Saldo pendiente) / Monto original) × 100
```

---

## 📲 INSTALACIÓN COMO APP (PWA)

### En Android (Chrome):
1. Abre la URL del sistema en Chrome
2. Aparece banner azul automático con botón **"Instalar"**
3. Confirma → queda en pantalla de inicio como app nativa

### En iPhone/iPad (Safari):
1. Abre la URL en Safari
2. Toca el botón **Compartir** ⬆
3. Selecciona **"Añadir a pantalla de inicio"**
4. Se instala con el ícono CF dorado

### En PC (Chrome/Edge):
1. Abre la URL
2. Click en el ícono `⊕` en la barra de dirección
3. Click "Instalar" → queda como app de escritorio

### Funciona offline:
Una vez instalada, la app funciona sin internet gracias al Service Worker (`sw.js`). Los datos de Supabase requieren conexión para sincronizarse.

---

## 🚀 DESPLIEGUE EN GITHUB PAGES

1. Crear repositorio en [github.com](https://github.com) → New repository
2. Subir **todos los archivos** de esta carpeta
3. Ir a **Settings → Pages → Branch: main → / (root) → Save**
4. En 2-3 minutos la app estará en:
   ```
   https://TU-USUARIO.github.io/NOMBRE-REPO/
   ```
5. Esa URL soporta HTTPS → PWA funciona al 100%

---

## 🛠️ TECNOLOGÍAS USADAS

| Tecnología | Uso |
|---|---|
| HTML5 / CSS3 / JavaScript | Frontend completo (sin frameworks) |
| Supabase | Base de datos PostgreSQL en la nube + API REST |
| Chart.js | Gráficos de barras y donut |
| SheetJS (XLSX) | Exportación a Excel |
| Service Worker API | Cache offline y PWA |
| Web App Manifest | Instalación como app nativa |

---

## ⚠️ NOTAS IMPORTANTES

- Las **contraseñas** están en texto plano en Supabase. Para producción real se recomienda implementar hash (bcrypt) o usar Supabase Auth.
- El **Row Level Security (RLS)** está configurado con acceso público. Para mayor seguridad configurar políticas por usuario.
- Los **saldos** se recalculan en tiempo real al guardar/editar/eliminar movimientos.
- Si se **elimina un movimiento**, el saldo del socio se revierte automáticamente.
- Si se **edita el monto** de un movimiento, la diferencia se aplica al saldo del socio.

---

## 📞 SOPORTE

Sistema desarrollado con Claude (Anthropic).
Base de datos: Supabase (afhnepfgvutfcozsbqct)

