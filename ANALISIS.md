# Análisis del proyecto AccesoLab / ControlSys 2.0

Auditoría completa del código actual: mapa de dependencias, hallazgos, recomendaciones priorizadas y zonas que no conviene tocar. El objetivo del documento es servir de guía antes de cualquier refactor, de forma que ningún cambio rompa funcionalidad existente.

---

## 1. Mapa de dependencias (qué llama a qué)

Esto es lo importante: si tocas A, qué se mueve.

```
index.php
  ├── App/Includes/Menu.php          (sidebar compartida)
  ├── plugins/fontawesome-free/...   (raíz)
  └── assets/plugins/.../ + assets/dist/...

App/Includes/Menu.php
  └── Detecta contexto con $_SERVER['PHP_SELF']
     (calcula $baseUrl, $viewsUrl, $logoutUrl)

App/Logout.php  -->  ../Public/Login.php  [NO EXISTE]
index.php       -->  Public/Login.php     [NO EXISTE]
App/Views/*.php -->  ../../Public/Login.php [NO EXISTE]

[Catálogo Planteles - alta]
App/Views/Planteles.php
  ├── PDO inline (port 3307, accesolab)
  ├── SELECT carreras WHERE activo=1
  ├── include_once '../Includes/Menu.php'
  └── POST AJAX --> ../../Server/GuardarPlantel.php  [NO EXISTE]

[Catálogo Carreras - alta]
App/Views/Carreras.php
  ├── PDO inline
  ├── SELECT planteles WHERE activo=1
  ├── include_once '../Includes/Menu.php'
  └── POST AJAX --> ../../Server/GuardarCarrera.php  [NO EXISTE]

[Listado de Planteles - SERVER RENDERED]
App/Views/ListadoPlanteles.php   (shell con menú y assets)
  └── include_once '../../listadoplanteles.php'   (raíz)
        └── include_once __DIR__ . '/consultas.php'
              └── Config/ConfigBD.php + Config/Conexion.php  (mysqli wrapper)
        └── Botón "Agregar"  --> administracion_planteles.php  [NO EXISTE]
        └── Botón "Modificar" --> modificarPlantel(id)         [JS NO EXISTE]

[Listado de Carreras - SERVER-SIDE DataTables AJAX]
App/Views/ListadoCarreras.php
  └── include_once '../../listadocarrera.php'
        └── DataTables AJAX --> ../../consulta.php
              └── Config/ConfigBD.php + Config/Conexion.php
        └── Botón "Agregar"  --> #                              [SIN destino]
        └── Botón "Editar"   --> modificarCarrera(id)           [JS NO EXISTE]

[Registro Uso - INCOMPLETO]
App/Views/registroUsoLabs/registo.php
  └── include_once '../../config/configbd.php'   (path con c minúscula)
  └── new configbd()   (instancia pero NO llama getconexion(), NO se usa)
  └── Form SIN action, SIN submit, SIN JS
  └── Combos (carrera/materia/lab/periodo) VACÍOS
```

---

## 2. Hallazgos (de más crítico a más cosmético)

### Bugs reales (no son opinión, están rotos)

**A. Bug en el resaltado del menú** — [App/Includes/Menu.php:12](App/Includes/Menu.php#L12)

```php
$isListadoCarreras = ($currentFile === 'listadocarreras.php');
```

El archivo en disco es **`ListadoCarreras.php`** (con `s` final) en `App/Views/`, pero también existe `listadocarrera.php` (sin `s`) en la raíz. La comparación está hecha contra `'listadocarreras.php'`, y como [App/Views/ListadoCarreras.php:1](App/Views/ListadoCarreras.php) sí termina en `s`, este caso sí funciona. Sin embargo, si alguna vez se entrara directo a `/listadocarrera.php` (raíz) el ítem no se marcaría activo. Es frágil — `Menu.php` compara nombres de archivo sin saber rutas.

**B. `App/Views/registroUsoLabs/registo.php` está incompleto y mal nombrado** — [registo.php:2-3](App/Views/registroUsoLabs/registo.php#L2)

- Nombre mal escrito: `registo.php` → debería ser `registro.php` (faltó la `r`).
- Incluye `'../../config/configbd.php'` con `c` minúscula. En Windows pasa, en Linux/producción truena.
- Instancia `$conexion = new configbd();` pero nunca llama `$conexion->getconexion()` y nunca usa la variable. Sirve de adorno.
- No tiene `session_start()` ni protección de sesión, a diferencia de las demás vistas.
- No tiene `<form action>`, ni botón submit, ni script JS — el form no se puede enviar.
- Los combos `cbcarrera`, `cbMateria`, `cbLaboratorio`, `cbperiodo` están vacíos, no hay JS que los pueble.
- Carga `plugins/select2/css/select2.css` con ruta relativa sin `../../`. Y `select2` ni siquiera está en el repositorio (`assets/plugins/select2` y `plugins/select2` no aparecen en el listado).

**C. Referencias a archivos que no existen**

- `Public/Login.php` — referenciado por `index.php`, `App/Logout.php` y todas las vistas.
- `Server/GuardarPlantel.php` — POST de `Planteles.php`.
- `Server/GuardarCarrera.php` — POST de `Carreras.php`.
- `administracion_planteles.php` — botón "Agregar Plantel" en [listadoplanteles.php:13](listadoplanteles.php#L13).
- Funciones JS `modificarPlantel(id)` y `modificarCarrera(id)` — no están definidas en ningún `.js` ni inline.

**D. `noacceso.php` y `.htaccess` están vacíos** (0 bytes). El nombre sugiere "página de no-acceso" pero no se referencia desde ningún lado.

### Vulnerabilidades de seguridad

**E. SQL Injection en [consulta.php:14-22](consulta.php#L14)** — endpoint de DataTables server-side.

```php
$txtBuscar = trim($_REQUEST['search']['value'] ?? '');
$where = '';
if ($txtBuscar !== '') {
    $txtBuscar = addslashes($txtBuscar);
    $where = " WHERE id_Carrera LIKE '%{$txtBuscar}%'"
        . " OR nombre LIKE '%{$txtBuscar}%'"
        ...
}
```

`addslashes()` no protege contra SQL injection (puede burlarse con multi-byte y con secuencias específicas). Cualquiera que abra el listado de carreras puede inyectar SQL en el buscador.

**F. Contraseñas en texto plano** en la tabla `usuarios` (seed con `admin/admin`). Cuando se construya el login, hay que decidir cómo migrar.

**G. Credenciales de DB hard-coded y duplicadas** en [Config/ConfigBD.php:5-7](Config/ConfigBD.php#L5) y vueltas a poner inline en [App/Views/Planteles.php:13](App/Views/Planteles.php#L13) y [App/Views/Carreras.php:13](App/Views/Carreras.php#L13). Si mañana cambia el puerto o la contraseña de `root`, hay que recordar tres lugares.

**H. La clase `conexion`** ([Config/Conexion.php](Config/Conexion.php)) recibe la query como propiedad pública asignable: `$conexion->query = "..."`. Cualquier código puede asignar SQL crudo. No hay método con prepared statements. Por diseño, fomenta el patrón inseguro que ya se ve en `consulta.php`.

**I. `session_start()` se llama en cada archivo individualmente.** Si un día se quiere endurecer la sesión (cookies HttpOnly, SameSite, regeneración de ID al login), hay que tocar 8 archivos.

### Inconsistencias arquitectónicas

**J. Dos formas de hablar con la DB en el mismo proyecto.** No es necesariamente malo — pero hoy:

- `consultas.php` y `consulta.php` usan la clase `conexion` (mysqli, concatenación).
- `App/Views/Planteles.php` y `Carreras.php` usan PDO inline con un `try/catch` raro:

  ```php
  try { $conexion = new PDO(..., 'root', null); }
  catch { $conexion = new PDO(..., 'root', ''); }
  ```

  Esto está intentando manejar la diferencia entre "password = null" y "password = ''". Es un workaround que indica que alguien topó con un error en algún ambiente. Innecesario y confuso.

**K. Dos modelos de paginación/búsqueda para listados muy parecidos.**

- Planteles: renderiza filas en PHP, DataTables client-side (carga TODO de un golpe). Está bien para 6 registros, escala mal si crece.
- Carreras: DataTables server-side con `consulta.php`. Más escalable, pero con el bug de SQL injection.

Inconsistente: dos catálogos prácticamente idénticos resueltos con dos arquitecturas distintas.

**L. Rutas de assets mezcladas.** Los CSS/JS se cargan unas veces desde `plugins/` (raíz), otras desde `assets/plugins/`. Ejemplo en [index.php:18-20](index.php#L18) FontAwesome viene de `plugins/`, pero AdminLTE viene de `assets/dist/`. Funciona, pero es trampa: tienes plugins duplicados en dos lugares y nunca sabes cuál se está usando.

**M. Layout duplicado en cada View.** [Planteles.php](App/Views/Planteles.php), [Carreras.php](App/Views/Carreras.php), [Dashboard.php](App/Views/Dashboard.php), [ListadoPlanteles.php](App/Views/ListadoPlanteles.php), [ListadoCarreras.php](App/Views/ListadoCarreras.php), [index.php](index.php) — todas repiten el mismo cascarón (head, navbar, sidebar wrapper, footer, scripts). Si quieres cambiar el navbar, lo cambias en 6 archivos. El sidebar sí está parcialmente extraído en `Menu.php`, pero el resto no.

**N. La función `realizarupdatedelete()`** en [Config/Conexion.php:26](Config/Conexion.php#L26) está mal nombrada (camelCase roto, hace dos cosas) y devuelve 1/0 sin distinguir si afectó filas o no. `$conexionBD->query(...)` devuelve `TRUE` para un UPDATE incluso si no afectó nada — así que ese "1" no quiere decir "se actualizó algo".

### Cosas menores

**O.** [accesolab (1).sql](accesolab%20(1).sql) — el nombre con espacio y "(1)" es ruido (parece descargado dos veces). Renombrarlo a `accesolab.sql`.

**P.** [.htaccess](.htaccess) vacío y [noacceso.php](noacceso.php) vacío — borrarlos o usarlos.

**Q.** Carpeta `dist/` en la raíz parece huérfana — el código siempre carga AdminLTE desde `assets/dist/`. Verificar antes de borrar.

**R.** En la tabla `usuarios` del seed hay `idperfil=1` ("TICs") para los usuarios "este/este" pero con `activo = NULL`. La columna `activo` en general permite NULL — debería ser `NOT NULL DEFAULT 1` para evitar ambigüedad.

**S.** [consulta.php:57](consulta.php#L57) construye `$elemento` como array indexado de 4 columnas, pero la tabla del frontend tiene 5 columnas — la columna 5 (`id_Carrera` para el botón Editar) se está renderizando con el `id_Carrera` que viene en `elemento[0]`. En [listadocarrera.php:55](listadocarrera.php#L55) el render del botón hace `'modificarCarrera(' + data + ')'` y `data` será lo que tenga la columna 4 (índice 4 desde 0)... pero no hay nada en el índice 4. La línea `//$elemento[]= (int)$row['id_Carrera'];` está comentada. **El botón Editar siempre llamará `modificarCarrera(undefined)`.** Bug funcional.

---

## 3. Recomendaciones priorizadas

En orden, de más urgente a más opcional. Ninguna implica perder funcionalidad existente, porque mucha de esa funcionalidad o no existe o está rota.

### Nivel 0 — Tapar lo que está roto antes de avanzar

1. Decidir qué hacer con `Public/Login.php`, `Server/Guardar*.php`. Sin esto el sistema no es usable. (Lo más práctico: crear `Public/Login.php` que valide contra `usuarios`, y los dos `Guardar*.php` con prepared statements.)
2. Arreglar [consulta.php:57](consulta.php#L57) para incluir `$row['id_Carrera']` como quinto elemento del array, si no, el botón Editar de carreras está muerto.
3. Renombrar `registo.php` → `registro.php` y completarlo, o moverlo a un branch / archivarlo si no estás trabajando en él aún.

### Nivel 1 — Seguridad (cambios localizados, sin perder funcionalidad)

4. Reemplazar `addslashes` en [consulta.php](consulta.php) por prepared statements con `mysqli` o PDO. Es un cambio en un solo archivo y nadie depende de la implementación interna, solo del JSON de salida.
5. Añadir un método `consultarConParametros($sql, $params)` a la clase `conexion` para que el patrón inseguro deje de ser el único disponible. No borrar `consultarDatos()` — los llamadores actuales siguen funcionando.
6. Centralizar credenciales en un solo archivo (`Config/ConfigBD.php` ya existe, ese debe ser el único). Quitar las conexiones PDO inline de [Planteles.php](App/Views/Planteles.php) y [Carreras.php](App/Views/Carreras.php) y reusar la conexión central. Antes de tocar: verificar que las vistas solo hacen SELECTs sencillos (sí, lo hacen).

### Nivel 2 — Calidad y DRY (sin tocar comportamiento)

7. Extraer el cascarón HTML repetido (navbar + aside + footer + scripts) a `App/Includes/Layout.php` con un patrón "antes/después" (`Layout_Top.php` y `Layout_Bottom.php`, por ejemplo). Las 6 vistas pasan a:

   ```php
   <?php include 'App/Includes/Layout_Top.php'; ?>
       <!-- contenido específico -->
   <?php include 'App/Includes/Layout_Bottom.php'; ?>
   ```

   Cambio mecánico, no rompe nada si se hace una vista a la vez y se prueba.
8. Centralizar `session_start()` y la protección de sesión en un `App/Includes/Auth.php` que cada vista `require` al inicio.
9. Decidir un solo origen de assets (recomiendo `assets/`). Borrar `plugins/` y `dist/` de la raíz después de hacer un grep y actualizar las referencias. (Con cuidado, no de golpe.)

### Nivel 3 — Consistencia arquitectónica

10. Unificar listados — usar DataTables server-side en ambos (con prepared statements). El de planteles también debería ir por AJAX, no renderizar inline.
11. Crear una capa fina de "modelos" (`App/Models/Plantel.php`, `App/Models/Carrera.php`) que encapsule las queries. Cada vista deja de saber SQL. Bajo costo, alto beneficio para mantenimiento.

### Nivel 4 — Pendientes funcionales

12. Implementar los módulos vacíos del menú (Registro, Reportes) con la arquitectura ya limpia.
13. Implementar edición/baja en los catálogos (`modificarPlantel`, `modificarCarrera`).
14. Decidir esquema de roles (`perfiles` y `datosusuarios` están definidos pero nadie los consume).

---

## 4. Lo que NO conviene tocar

- **El esquema de la DB.** Está coherente con FKs declaradas. Tocar nombres de columnas (`id_Carrera`, `Contrasena`) rompería muchos archivos.
- **La separación raíz / `App/Views/`** mientras no haya tiempo de hacer un router. El sistema de `Menu.php` que calcula rutas relativas funciona; un cambio aquí cae en efecto dominó.
- **AdminLTE / Bootstrap 4 / jQuery.** Es el stack del front. Actualizar a Bootstrap 5 sería un rediseño completo, no una mejora.

---

## 5. Orden sugerido de trabajo

Tres bloques independientes; se pueden hacer en cualquier orden:

1. **Nivel 0** — los handlers que faltan y el bug del botón Editar. Deja un sistema usable de extremo a extremo.
2. **Nivel 1** — seguridad (SQL injection, credenciales centralizadas).
3. **Nivel 2** — DRY de layout (deja de editar 6 archivos para cualquier cambio visual).

Antes de cada cambio conviene listar exactamente qué archivos se tocan y por qué, para no romper dependencias que aún no estén documentadas.
