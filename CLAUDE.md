# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**AccesoLab / ControlSys 2.0** — laboratory access/usage management system for TECNM / ITESI (Instituto Tecnológico Superior de Irapuato). PHP + MariaDB application served from XAMPP, with an AdminLTE 3 dashboard UI.

UI text and identifiers are in Spanish (e.g. `planteles`, `carreras`, `Usuario`). Preserve Spanish in user-facing strings, schema names, and column names.

## Runtime environment

- **Stack:** PHP (no framework, no composer), MariaDB / MySQL, vanilla jQuery + Bootstrap 4 + AdminLTE.
- **Server:** XAMPP. Docroot is the parent (`c:\xampp\htdocs`); this app lives under `Desarrollos/login/` and is accessed at `http://localhost/Desarrollos/login/`.
- **Database:** MariaDB on `127.0.0.1:3307` (non-default port), DB name `accesolab`, user `root`, **empty password**. Credentials are hard-coded in [Config/ConfigBD.php](Config/ConfigBD.php) and duplicated inline in some view files ([App/Views/Planteles.php](App/Views/Planteles.php), [App/Views/Carreras.php](App/Views/Carreras.php)).
- **Schema seed:** [accesolab (1).sql](accesolab%20(1).sql) — load via phpMyAdmin to create the `accesolab` DB.
- There are no build/lint/test commands; PHP files are edited and served directly. Reload the page in the browser to see changes.

## Architecture

### Auth & session flow

Every page is gated by `$_SESSION['Usuario']`. The entry point [index.php](index.php) and each view in `App/Views/` redirects to `Public/Login.php` when the session is missing. **`Public/Login.php` and the `Server/Guardar*.php` form handlers referenced by the views do not exist in the repo yet** — they are expected endpoints that need to be created. When adding auth/server-side handlers, place them at the paths the views already POST to (e.g. `Server/GuardarPlantel.php`, `Server/GuardarCarrera.php`) and create `Public/Login.php` rather than changing the redirects.

`App/Logout.php` clears the session and redirects back to `Public/Login.php`.

### Page shell + shared menu

[App/Includes/Menu.php](App/Includes/Menu.php) is the shared sidebar. It detects whether it's being included from the project root (`index.php`) or from `App/Views/...` by inspecting `$_SERVER['PHP_SELF']` and computes `$baseUrl` / `$viewsUrl` / `$logoutUrl` accordingly. **When adding a new view, mirror the include pattern used by existing views** (`include_once '../Includes/Menu.php';` from `App/Views/`, `include_once "App/Includes/Menu.php";` from root) so the relative-path logic keeps working.

### Two DB-access patterns coexist

Don't unify these unless asked — both are in active use:

1. **mysqli wrapper** ([Config/Conexion.php](Config/Conexion.php) + [Config/ConfigBD.php](Config/ConfigBD.php)): instantiate `new conexion()`, assign a SQL string to `$conexion->query`, then call `consultarDatos()` / `realizarInsert()` / `realizarupdatedelete()`. Used by root-level scripts ([consultas.php](consultas.php), [consulta.php](consulta.php)).
2. **PDO inline**: views like [App/Views/Planteles.php](App/Views/Planteles.php) and [App/Views/Carreras.php](App/Views/Carreras.php) open their own `PDO` connection at the top of the file, with a fallback `try/catch` for empty-vs-null password.

Both connect to `accesolab` on port 3307.

### Two view paradigms coexist

- **Self-contained views** under `App/Views/` (e.g. `Planteles.php`, `Carreras.php`, `Dashboard.php`) render their own full `<html>` document, include the menu, and POST to `Server/Guardar*.php` via AJAX.
- **Listing views** (`App/Views/ListadoPlanteles.php`, `App/Views/ListadoCarreras.php`) render the AdminLTE shell, then `include_once` a root-level partial (`../../listadoplanteles.php`, `../../listadocarrera.php`) for the table body. Those partials in turn `include_once __DIR__ . '/consultas.php'` (server-rendered rows) or hit `consulta.php` (DataTables server-side JSON endpoint).

When editing a listing, check both the shell in `App/Views/` and the partial at the repo root — they're tightly coupled by relative asset paths (`../../plugins/...`, `../../assets/...`).

### Asset paths

`plugins/` is duplicated at the root **and** under `assets/plugins/` (likewise `dist/` and `assets/dist/`). Existing pages mix both prefixes. When adding script/style tags, follow the surrounding file's convention rather than picking one globally.

### Database schema (from `accesolab (1).sql`)

Core tables:
- `planteles` (id_Plantel, plantel, lplantel, activo) — campuses
- `carreras` (id_Carrera, nombre, lcarrera, activo) — degree programs
- `plantel_carrera` — junction table for which careers each campus offers
- `usuarios` / `perfiles` / `datosusuarios` — accounts and roles (TICs, Administrador, Docente, Alumno, Tecnico Laboratorio)

Passwords in `usuarios.Contrasena` are stored in **plain text** in the seed (`admin/admin`). If asked to add login functionality, flag this and ask before introducing hashing — it will break the seed accounts.

## Known fragilities to watch

- [consulta.php](consulta.php) builds its `WHERE` clause with `addslashes()` and string concatenation (DataTables server-side endpoint). Don't copy this pattern into new code — use prepared statements.
- DB credentials are duplicated across `Config/ConfigBD.php` and individual view files; changing the port or password requires updating every site.
- The hard-coded `Public/` and `Server/` paths in redirects and form actions assume directories that don't exist yet — verify they're present (or create them) before claiming a flow works end-to-end.
