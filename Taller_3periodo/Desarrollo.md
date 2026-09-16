# Taller de Autenticación con FastAPI
---

## 📐 Arquitectura y Estructura del Proyecto

### 1. ¿Cuál es la responsabilidad específica de cada archivo Python (`main.py`, `rutas.py`, `seguridad.py`, `usuarios.py`) y por qué se considera una buena práctica separarlos de esta manera?

- `main.py` (29 líneas): solo crea la app de FastAPI y le "pega" las rutas con `app.include_router(router)`. No hace nada más.
- `rutas.py`: tiene las funciones que responden a cada URL (`/login`, `/`, `/perfil`, `/objetos`, `/logout`). Cada función arma qué se muestra cuando alguien entra a esa página.
- `seguridad.py`: toda la lógica de sesiones — crear token, guardarlo, comprobar si alguien tiene sesión válida.
- `usuarios.py`: la "base de datos" de usuarios (un diccionario en memoria) y sus objetos.

Separarlos así es buena práctica porque si mañana cambio cómo se guardan las sesiones (por ejemplo, pasar de un archivo JSON a una base de datos real), solo toco `seguridad.py` sin tener que tocar las rutas ni el arranque de la app. Cada archivo se puede entender, probar y arreglar por separado.

### 2. ¿Qué significa "separación de responsabilidades" en el contexto de este proyecto y cómo se manifiesta en la estructura de archivos?

Significa que cada archivo hace una sola cosa y la hace bien: uno arranca la app, otro define páginas, otro maneja seguridad, otro maneja datos. Nadie mezcla funciones que no le corresponden — por ejemplo, `rutas.py` nunca genera tokens directamente, se lo pide a `seguridad.py`.

### 3. ¿Por qué el archivo `main.py` es tan breve (apenas 29 líneas) y qué indica esto sobre el diseño de la aplicación?

Porque es solo el "punto de montaje": importa el router y lo conecta a la app. Que sea corto es buena señal — indica que el diseño está bien separado y que este archivo no carga con responsabilidades que le tocan a otros módulos.

---

## 🔐 Mecanismo de Autenticación y Sesiones

### 4. Explica el concepto de "sesión" tal como se implementa en este proyecto. ¿Por qué HTTP necesita sesiones si cada petición es independiente?

HTTP es "sin memoria": cada petición (cada clic, cada carga de página) llega al servidor como si fuera la primera vez, sin recordar nada de la anterior. Una sesión es el truco para simular esa memoria: cuando hago login, el servidor crea un identificador único (token) y me lo entrega guardado en una cookie. Como el navegador manda esa cookie en cada petición siguiente, el servidor puede "reconocerme" aunque HTTP en sí no tenga memoria.

### 5. ¿Qué es el token de sesión, cómo se genera específicamente en el código (`secrets.token_hex(16)`) y por qué es importante que sea aleatorio e impredecible?

Se genera con `secrets.token_hex(16)`, que produce 32 caracteres aleatorios en hexadecimal. Es clave que sea aleatorio e impredecible porque si alguien pudiera adivinar o calcular el token de otra persona, podría hacerse pasar por ella sin saber su contraseña — eso es literalmente robar la sesión.

### 6. Describe el flujo completo cuando un usuario ingresa credenciales válidas: desde el POST al `/login` hasta que se redirige a la página home (menciona los "PASOS" del diagrama).

- Lleno el formulario y mando `POST /login`.
- `comprobar_login()` revisa si el usuario existe y si la contraseña coincide (**PASO 2**).
- Si es correcto, `crear_sesion()` genera el token y lo guarda en el diccionario `sesiones` (**PASO 3**).
- Se arma una `RedirectResponse` hacia `/` con código 303, y se le pega la cookie con `set_cookie()` (**PASO 4**).
- El navegador sigue la redirección automáticamente con un GET a `/`.
- Ahí entra en juego la dependencia `UsuarioDep`, que lee la cookie y confirma que hay sesión (**PASO 6**), y me deja ver la home.

### 7. ¿Qué diferencia hay entre las rutas públicas (`/login`, `/logout`) y las privadas (`/`, `/perfil`, `/objetos`)? ¿Por qué el login debe ser necesariamente público?

Las públicas no piden `UsuarioDep`. Las privadas sí lo piden como parámetro. El login tiene que ser público por lógica: si para entrar al login ya necesitara tener sesión iniciada, sería un candado sin llave — nadie podría loguearse nunca.

### 8. ¿Cómo funciona la dependencia `UsuarioDep` en FastAPI y qué hace exactamente la función `get_current_user` cuando se ejecuta en cada ruta protegida?

Es un alias de tipo (`Annotated[Usuario, Depends(get_current_user)]`). Cuando una ruta lo pide como parámetro, FastAPI ejecuta `get_current_user()` antes de correr la función de la ruta. Esa función lee la cookie `sesion`, busca el token en el diccionario `sesiones`; si no encuentra sesión válida, lanza una excepción HTTP 303 que redirige a `/login`; si la encuentra, devuelve el objeto `Usuario` correspondiente y la ruta sigue normal.

---

## 🍪 Cookies y Gestión de Estado

### 9. ¿Cuál es el propósito de la cookie `COOKIE_SESION` y qué parámetros se configuran cuando se establece (`max_age=VIDA_SESION_SEGUNDOS`)? ¿Qué pasaría si no se configurara `max_age`?

La cookie guarda el token que identifica la sesión del usuario en su navegador. Cuando se crea, se le pasa `max_age=VIDA_SESION_SEGUNDOS` (una semana en segundos), lo que le da fecha de caducidad. Si no se configurara `max_age`, la cookie sería "de sesión de navegador": moriría en cuanto cerrara el navegador, en vez de durar una semana.

### 10. El proyecto guarda las sesiones en un archivo `sesiones.json` en disco. Explica el flujo de lectura y escritura de este archivo: ¿cuándo se lee, cuándo se escribe y qué sucede si el disco es de solo lectura (como en Vercel)?

Se lee una sola vez, al arrancar el servidor (`cargar_sesiones()` llena el diccionario `sesiones` en memoria). Se escribe cada vez que se crea o se cierra una sesión (`guardar_sesiones()`), para que la libreta sobreviva a un reinicio del servidor. En un disco de solo lectura, el `try/except OSError` hace que la escritura falle en silencio y la app siga funcionando, solo que ya no persiste nada en disco — trabaja únicamente con lo que tiene en memoria.

### 11. Compara el comportamiento de las sesiones en desarrollo local vs. despliegue en Vercel. ¿Por qué en Vercel las sesiones "se pierden" cuando la función se enfría (cold start)?

En local, el archivo `sesiones.json` se escribe de verdad, entonces las sesiones sobreviven aunque reinicie el servidor. En Vercel, cada función es "serverless": cuando nadie la usa por un rato, se apaga (cold start) y al volver a activarse arranca desde cero, con la memoria vacía. Como en Vercel la escritura en disco falla, esa "libreta" nunca persiste entre arranques — por eso las sesiones se pierden ahí.

### Curiosidad: ¿Por qué se les llama cookies en el mundo de la informática a esos fragmentos de texto?

El nombre viene de "magic cookie", un término viejo de programación para un paquetito de datos que un programa le pasa a otro sin que el receptor necesite entender su contenido, solo devolverlo intacto cuando se le pida. Se aplicó a la web porque es exactamente lo que hace el navegador: recibe la cookie del servidor y se la devuelve tal cual en cada petición.

---

## 🛡️ Seguridad y Mejores Prácticas

### 12. El README advierte que este proyecto deliberadamente no usa OAuth2, JWT ni hashes de contraseña. Explica qué son estos tres conceptos y por qué NO se usan en este proyecto didáctico.

- **Hash de contraseña:** transformar la contraseña en un código irreversible antes de guardarla, para que si alguien roba la base de datos no vea las contraseñas reales.
- **JWT:** un token que lleva la info del usuario firmada digitalmente dentro de sí mismo, sin necesitar guardar nada en el servidor.
- **OAuth2:** un estándar para dejar que otro servicio (Google, GitHub) confirme la identidad del usuario.

No se usan acá porque el objetivo del proyecto es que se entienda el mecanismo de sesiones "a mano", sin la complejidad extra de esos sistemas. Meterlos desde el inicio habría tapado la lógica base con conceptos avanzados.

### 13. Las contraseñas en `usuarios.py` están en texto plano (`"password": "1234"`). ¿Qué riesgo de seguridad representa esto en producción y qué solución se usaría en un proyecto real?

El riesgo es enorme: si alguien accede al archivo o base de datos, ve todas las contraseñas reales de una. En un proyecto real se guardarían solo los hashes (con `bcrypt` o `Argon2`, por ejemplo), así aunque roben la base de datos no pueden recuperar la contraseña original.

### 14. ¿Qué es el "sesion hijacking" (secuestro de sesión) y qué medidas adicionales podrían implementarse para prevenirlo que este proyecto no incluye?

Es cuando alguien roba el token de sesión de otra persona (por ejemplo, interceptándolo en una red insegura) y lo usa para hacerse pasar por ella sin necesitar su contraseña. Este proyecto no implementa medidas extra como: marcar la cookie como `Secure` (solo viaja por HTTPS) y `HttpOnly` (JavaScript no puede leerla), regenerar el token después del login, expirar la sesión por inactividad, o ligar el token a la IP/dispositivo.

---

## 🌐 Flujo HTTP y Códigos de Estado

### 15. El código usa `status_code=303` en las redirecciones después del login y logout. ¿Por qué se usa específicamente 303 (See Other) en lugar de 302 (Found) o 301 (Moved Permanently)?

303 (See Other) le dice al navegador explícitamente: "lo que sigue lo pedís con GET", sin importar qué método usaste antes. Es justo lo que se necesita después de un POST de login: no querés que el navegador repita el POST. El 302 es ambiguo en su significado histórico (algunos navegadores lo reinterpretan distinto), y el 301 es para redirecciones permanentes de una URL a otra, no para este caso puntual.

### 16. Explica la diferencia semántica entre usar `GET /login` (mostrar formulario) y `POST /login` (enviar credenciales). ¿Por qué no se envían las credenciales por GET?

`GET /login` solo pide mostrar el formulario vacío — no manda datos sensibles, y por eso puede ir en la URL sin problema. `POST /login` manda las credenciales en el cuerpo de la petición, no visibles en la URL. Si el login fuera por GET, el usuario y la contraseña quedarían escritos en la URL, visibles en el historial del navegador, en logs del servidor y en cualquier lugar por donde pase esa URL — una fuga de seguridad total.

---

## 🚀 Despliegue y Configuración

### 17. El proyecto utiliza `Path(__file__).parent` para ubicar las plantillas y el archivo de sesiones. ¿Por qué es importante usar rutas absolutas basadas en la ubicación del archivo en lugar de rutas relativas a la carpeta de trabajo actual?

Usa la ubicación del propio archivo de código como referencia, en vez de la carpeta desde donde se ejecuta el comando. Si usara una ruta relativa normal, la app fallaría en encontrar `templates/` o `sesiones.json` según desde dónde se lance el servidor (por ejemplo en Vercel, donde el directorio de trabajo no es necesariamente la raíz del proyecto). Con `Path(__file__).parent` siempre encuentra esas carpetas sin importar desde dónde se arranque.

---

## 💡 Pensamiento Crítico

### 18. Bonus Extra: Si tuvieras que convertir este proyecto didáctico en una aplicación real para producción, enumera al menos 5 cambios que harías y justifica cada uno desde el punto de vista de seguridad, escalabilidad y mantenibilidad.

1. **Hashear contraseñas** (bcrypt/Argon2) — seguridad, para no exponer contraseñas reales si hackean la base.
2. **Base de datos real** (PostgreSQL, por ejemplo) en vez de diccionarios en memoria + JSON — escalabilidad, porque el archivo JSON no soporta múltiples usuarios escribiendo a la vez ni crece bien.
3. **Cookies `Secure` y `HttpOnly`** — seguridad, para que la cookie solo viaje por HTTPS y no se pueda leer desde JavaScript malicioso.
4. **JWT o sesiones en un almacén externo** (Redis) — escalabilidad, porque si la app corre en varios servidores, cada uno necesita ver las mismas sesiones, y un diccionario en memoria local no sirve para eso.
5. **Tests automatizados y logging de errores** — mantenibilidad, para detectar fallos rápido y no tener que probar todo a mano cada vez que se cambia algo.