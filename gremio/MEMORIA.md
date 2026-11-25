# Memoria · Gremio de Aventureros

**Código de alumno/a:** ______  (no pongas tu nombre)

## 1. Mi aventurero
- Usuario que he añadido a la tabla: `raul`
- Comando o pasos que he usado para generar el hash:
  Generado mediante PHP CLI en el contenedor con `password_hash`:
  ```bash
  docker compose exec php php -r "echo password_hash('Aventurero2026', PASSWORD_DEFAULT), PHP_EOL;"
  ```
  El hash generado (`$2y$10$xdAnS.AKJ7zqsrBLR22/jOKjWQuOocMY/LYtQvVr5rjmUyjcKZGCC`) se ha incorporado en el script `sql/mi_aventurero.sql` con nombre `Raul <Cazador de Sombras>` y clase `Pícaro`, e insertado en la tabla `aventureros`.

## 2. Parte 1 · match
¿Has puesto `default` en `etiquetaRango()`? ¿Qué pasa si llega el rango `'Z'`? ¿Por qué lo prefieres así?

Sí, he incluido un brazo `default => '¿?'` en `etiquetaRango()` (y también `default => ''` en `claseRango()`).
En PHP 8, la estructura `match` es estricta: si llega un valor como `'Z'` que no coincide con ninguno de los casos definidos y no existe un brazo `default`, PHP lanza una excepción fatal `UnhandledMatchError` que detiene la ejecución del script. Al poner `default`, si llega un rango no reconocido o manipulado, la función devuelve un valor de sustitución seguro sin que la aplicación se rompa. Prefiero este comportamiento defensivo porque evita caídas inesperadas ante datos imprevistos.

## 3. Parte 2 · Validación de la URL
Escribe a mano en la barra `tablon.php?rango=Z&oro=hola`. ¿Qué ocurre y qué línea de tu código lo evita?

Ocurre que la página carga con normalidad mostrando las 10 misiones sin ningún error, aviso ni excepción de PHP, ignorando los parámetros manipulados.
Las líneas que lo evitan en `inc/funciones.php` son:
- Para el rango: en `leerRangoFiltro()`, la línea:
  ```php
  return in_array($rango, RANGOS, true) ? $rango : null;
  ```
  Como `'Z'` no está dentro de la constante `RANGOS`, la función retorna `null` y el filtro de rango no descarta ninguna misión.
- Para el oro: en `leerOroMinimo()`, la línea:
  ```php
  $val = filter_var($get['oro'], FILTER_VALIDATE_INT, ['options' => ['min_range' => 0, 'max_range' => 10000]]);
  return $val !== false ? $val : 0;
  ```
  Al ser `'hola'` un texto no convertible a entero en el rango [0, 10000], `filter_var` devuelve `false` y la función asigna `0` como recompensa mínima.

## 4. Parte 3 · Sesión frente a cookie
Explica con tus palabras qué guardas en `$_SESSION` y qué guardas en cookies, y por qué no al revés.

- **En `$_SESSION`:** Guardo los datos del aventurero autenticado (`$_SESSION['aventurero']`), las misiones aceptadas en el zurrón (`$_SESSION['zurron']`), la hora de entrada (`$_SESSION['entrada']`), el contador de visitas (`$_SESSION['visitas']`) y los mensajes flash (`$_SESSION['flash']`). Estos datos viven de forma segura en el servidor y están ligados al ciclo de vida de la sesión activa; el cliente únicamente custodia una cookie con el identificador opaco (`PHPSESSID`).
- **En cookies (`setcookie`):** Guardo únicamente preferencias de cliente no críticas y que deben sobrevivir al cierre de sesión o cierre del navegador: el tema visual (`gremio_tema`, caducidad a 1 año) y el nombre de usuario para recordarlo en el formulario (`gremio_usuario`, caducidad a 30 días).
- **Por qué no al revés:** Las cookies se almacenan en el navegador del cliente y se envían en cada petición HTTP. Si guardáramos el zurrón o los datos de sesión en cookies, el usuario podría manipularlas fácilmente mediante las herramientas de desarrollo (F12) para asignarse recompensas o suplantar a otro aventurero, además de sobrecargar innecesariamente las cabeceras HTTP. Por otro lado, la sesión se destruye al cerrar el navegador o pulsar "Salir", por lo que no permitiría recordar preferencias duraderas como el tema preferido o el usuario recordado.

## 5. Parte 3 · Seguridad del login
- **¿Qué pasa si en el usuario escribes `' OR '1'='1`? ¿Por qué?**
  No se produce ningún acceso no autorizado; la aplicación muestra el mensaje `"Usuario o contraseña incorrectos."`. Esto es así porque la consulta SQL está preparada (`$pdo->prepare('SELECT ... WHERE usuario = :usuario')`) utilizando un marcador de posición nombrado `:usuario`. El motor de base de datos trata el valor `' OR '1'='1` estrictamente como un dato literal y nunca como código SQL ejecutable, anulando cualquier posibilidad de inyección SQL.

- **¿Para qué sirve `session_regenerate_id(true)`?**
  Sirve para prevenir ataques de **fijación de sesión** (*Session Fixation*). Al llamar a esta función justo después de validar las credenciales, PHP genera un nuevo identificador de sesión criptográficamente seguro y destruye el archivo/estado de la sesión previa (`true`). De este modo, si un atacante había forzado o interceptado previamente un identificador de sesión previo al login, ese ID antiguo queda invalidado y no puede ser utilizado para secuestrar la cuenta autenticada.

- **¿Por qué el mensaje de error es el mismo si falla el usuario o la contraseña?**
  Para evitar la **enumeración de usuarios** (*User Enumeration*). Si el sistema distinguiera entre *"El usuario no existe"* y *"Contraseña incorrecta"*, un atacante podría recopilar una lista de cuentas válidas del gremio mediante prueba y error, facilitando ataques dirigidos de fuerza bruta o phishing. Con un mensaje unificado, no se revela información sobre la existencia previa del usuario.

## 6. Pruebas que he hecho
| Prueba | Resultado esperado | Resultado obtenido |
|---|---|---|
| Entrar en tablon.php sin login | Redirige al login con el aviso de acceso restringido | Redirige a `login.php?motivo=privado` mostrando el mensaje "Esa sala es solo para miembros del gremio. Inicia sesión." |
| Login con contraseña mala | Muestra mensaje de error genérico sin entrar | Muestra `"Usuario o contraseña incorrectos."` y mantiene el usuario escrito |
| Aceptar la misma misión dos veces | No se duplica en el zurrón; muestra aviso | Muestra `"Esa misión ya está en tu zurrón."` y el zurrón mantiene solo un registro |
| Cerrar sesión y volver al zurrón | Redirige al login y el zurrón se vacía | Al cerrar sesión se destruye la sesión; al intentar acceder a zurron.php redirige a login |
| Cambiar el tema y cerrar sesión | El tema elegido se mantiene tras salir | Tras hacer logout, la pantalla de login sigue aplicando la clase CSS del tema guardado |

## 7. Dificultades y cómo las he resuelto
- **Implementación del patrón Post/Redirect/Get (PRG):** Al aceptar misiones en el tablón o abandonarlas en el zurrón, era fundamental evitar que la recarga de página (F5) reenviara el formulario POST. Se resolvió procesando el POST, almacenando el mensaje en `$_SESSION['flash']` y redirigiendo inmediatamente con `header('Location: ...')` seguido de `exit`.
- **Validación robusta de cookies manipuladas:** Al leer la cookie `gremio_tema`, se implementó una verificación estricta con `in_array($tema, TEMAS, true)`. Si un usuario introduce un valor arbitrario mediante las herramientas de desarrollador, el sistema vuelve limpiamente al tema `'claro'`, protegiendo el diseño frente a inyecciones de clases no deseadas.
- **Limpieza y reindexación de arrays al abandonar misiones:** Al eliminar un id del zurrón con `unset($_SESSION['zurron'][$pos])`, los índices del array asociativo quedaban discontinuos. Se utilizó `array_values()` para garantizar un array secuencial limpio.
