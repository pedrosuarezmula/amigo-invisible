# 🎁 Amigo invisible

Aplicación web para organizar **sorteos de amigo invisible** entre varios grupos de amigos o familiares. Cada grupo tiene su propio enlace, su propia lista de deseos compartida y su propio sorteo, completamente separados de los demás. Todo el sitio es **un único archivo `index.html`**, sin servidor propio ni proceso de compilación: se aloja gratis en GitHub Pages y guarda los datos en Firebase.

## Características

- **Varios grupos independientes.** Cada grupo tiene un enlace secreto, un nombre y un color propios. Un grupo no puede ver ni modificar los datos de otro.
- **Superadministrador (Root).** Una cuenta con acceso total que crea y elimina grupos, copia sus enlaces y puede entrar en cualquiera como organizador.
- **Organizador por grupo.** La primera persona que se registra en un grupo pasa a ser su organizador: crea las cuentas del resto, configura las reglas y hace el sorteo.
- **Lista de deseos común.** Cada participante añade lo que le gustaría recibir con **texto, enlace y/o foto**, y puede borrar lo suyo. Todos ven las listas de todos.
- **Varios amigos invisibles por persona.** Se puede configurar que cada uno regale a 1, 2, 3 o 4 personas distintas, y que reciba de la misma cantidad.
- **Restricciones.** Parejas, familiares u otras personas que no deben tocarse entre sí. Nadie se regala nunca a sí mismo ni repite a la misma persona.
- **Evitar regalos cruzados** (opcional, activado por defecto): si A regala a B, B no regala a A.
- **Revelación privada.** Cada persona entra con su nombre y PIN, y solo ve a quién le toca la suya, tras pulsar un botón para que nadie lo vea de reojo.
- **Datos en tiempo real.** Los cambios aparecen al instante en todos los dispositivos.
- **Adaptada a móvil y escritorio**, con modo claro y oscuro automáticos.

## Cómo funciona

| Rol | Cómo se accede | Qué puede hacer |
|---|---|---|
| **Root** | `https://TU_USUARIO.github.io/REPO/#root` con correo y contraseña (Firebase Authentication) | Crear y eliminar grupos, elegir su nombre y color, copiar sus enlaces y entrar en cualquiera como organizador |
| **Organizador** | Enlace del grupo, siendo la primera persona en registrarse | Crear usuarios, quitar participantes, configurar reglas, hacer y anular el sorteo, además de lo que hace un participante |
| **Participante** | Enlace del grupo, con el nombre y el PIN que le dio el organizador | Gestionar su lista de deseos, ver las de los demás y descubrir a quién le regala |

Flujo habitual:

1. Root crea un grupo y copia su enlace.
2. Root se lo envía a una persona de confianza del grupo, que se registra primero y se convierte en organizador.
3. El organizador crea las cuentas de sus amigos (nombre y PIN) y se las comunica.
4. Cada uno entra y rellena su lista de deseos.
5. El organizador configura las reglas (amigos por persona, parejas) y pulsa **Hacer sorteo**.
6. Cada participante entra, pulsa **Descubrir a quién le regalo** y ve a sus amigos invisibles junto con sus listas.

## Tecnología

- **Frontend:** HTML, CSS y JavaScript sin dependencias ni compilación, todo en `index.html`.
- **Base de datos:** [Cloud Firestore](https://firebase.google.com/docs/firestore) (SDK `compat` 10.12.2 cargado desde `gstatic.com`).
- **Autenticación de Root:** [Firebase Authentication](https://firebase.google.com/docs/auth) con correo y contraseña.
- **Alojamiento:** [GitHub Pages](https://pages.github.com/).

## Puesta en marcha

### 1. Firebase

1. En [console.firebase.google.com](https://console.firebase.google.com) crea un proyecto (Analytics no es necesario).
2. **Firestore Database → Crear base de datos.** Elige una región europea, la edición Standard y el modo producción. La región no se puede cambiar después.
3. En la pestaña **Reglas** de Firestore pega lo siguiente, cambiando el correo por el tuyo (en minúsculas), y publica:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       function root() {
         return request.auth != null && request.auth.token.email == 'TU_CORREO@ejemplo.com';
       }
       match /groups/{gid} {
         allow get: if true;
         allow list, write: if root();
         match /{col}/{doc} {
           allow read, write: if col in ['members', 'wishes', 'draw'];
         }
       }
     }
   }
   ```

4. **Authentication → Sign-in method:** activa *Correo electrónico/Contraseña*. En la pestaña **Users** añade un usuario con **el mismo correo** que pusiste en las reglas. Esa será tu cuenta Root. Si la consola lo permite, desactiva el registro de usuarios nuevos en *Settings → User actions*.
5. **Configuración del proyecto → Tus apps → icono web `</>`:** registra una app web (sin Firebase Hosting) y copia el objeto `firebaseConfig`.

### 2. Configurar el código

Abre `index.html` y sustituye el contenido de `FIREBASE_CONFIG`, cerca del inicio del `<script>`, por tu configuración:

```js
const FIREBASE_CONFIG={
  apiKey:"...",
  authDomain:"...",
  projectId:"...",
  storageBucket:"...",
  messagingSenderId:"...",
  appId:"..."
};
```

La `apiKey` de una web de Firebase no es un secreto: la seguridad la imponen las reglas del paso anterior.

### 3. GitHub Pages

1. Crea un repositorio **público** y sube `index.html` a la raíz. GitHub Pages es gratuito en repositorios públicos con una cuenta Free.
2. **Settings → Pages → Build and deployment:** *Deploy from a branch*, rama `main`, carpeta `/ (root)`.
3. Tras uno o dos minutos la web estará en `https://TU_USUARIO.github.io/NOMBRE_DEL_REPO/`.

### 4. Primer uso

Abre la URL añadiendo `#root`, entra con tu correo y contraseña, crea un grupo y reparte su enlace.

## El sorteo

El organizador configura tres cosas en el apartado **Organizador → Reglas del sorteo**:

- **Amigos invisibles por persona** (1 a 4).
- **Restricciones entre dos personas** (mutuas), tantas como haga falta.
- **Evitar regalos cruzados**, activado o no.

El reparto se calcula así:

1. Se plantea como un problema de **flujo máximo** en un grafo bipartito (quien regala → quien recibe). Cada persona da exactamente *K* regalos y recibe exactamente *K*, cada pareja posible aparece como máximo una vez y las parejas prohibidas no existen como opción. Así se garantiza que, si existe alguna solución válida, el sorteo la encuentra.
2. Sobre esa primera solución válida se hacen miles de **intercambios aleatorios** que mantienen todas las reglas, para que el resultado sea impredecible. Si "evitar cruces" está activo, solo se aceptan intercambios que no aumenten el número de regalos cruzados.
3. Si no hay solución, la app indica qué personas se quedan sin candidatos suficientes, para que el organizador quite restricciones o reduzca *K*.

Mínimo de participantes: el mayor entre 3 y *K* + 1. Si cambias las reglas después de sortear, hay que repetir el sorteo.

## Estructura de datos en Firestore

```
groups/{codigo}                 → { name, color, at }
  ├─ members/{id}               → { name, h, at }
  ├─ wishes/{autoId}            → { m, text, link, img?, at }
  └─ draw/
       ├─ main                  → { pairs: { idPersona: [idAmigo, …] }, k, at }
       └─ config                → { k, avoid, pairs: ["idA|idB", …] }
```

- `codigo`: cadena aleatoria de 12 caracteres, que es lo que mantiene los grupos separados.
- `members.id`: el nombre normalizado (sin tildes, en minúsculas y con guiones). Por eso no puede haber dos personas con el mismo nombre en un grupo.
- `members.h`: hash SHA-256 de `id:PIN`. El PIN nunca se guarda en claro.
- `members.at`: fecha de registro. El organizador es el miembro con la fecha más antigua.
- `wishes.img`: foto reducida a un máximo de 640 px, en JPEG y en base64, guardada dentro del propio documento, así que no hace falta Cloud Storage.
- La sesión de cada persona se recuerda en el navegador (`localStorage`), por grupo.

## Seguridad y limitaciones

Es una aplicación pensada para **grupos de confianza**. Conviene conocer sus límites:

- **Root es seguro de verdad.** Solo la cuenta de Firebase Authentication cuyo correo figura en las reglas puede crear, listar o borrar grupos.
- **Los grupos están aislados** porque nadie puede listarlos y el código del enlace no se puede adivinar. Aun así, **quien tenga el enlace de un grupo puede, con conocimientos técnicos, leer los datos de ese grupo**, incluidos los emparejamientos del sorteo y los hashes de los PIN. Trata el enlace como una contraseña y compártelo solo con quien debe participar.
- **Los permisos del organizador se aplican en la interfaz**, no en las reglas de Firebase. Un participante con conocimientos técnicos podría saltárselos dentro de su propio grupo.
- **No hay recuperación de PIN.** Si alguien lo pierde, el organizador quita su usuario, lo que borra también sus deseos, y se lo vuelve a crear.
- **El repositorio es público**, así que la configuración de Firebase quedará visible. Es normal en webs de Firebase. No subas nunca contraseñas ni claves privadas.
- **Cuotas de Firebase.** El plan gratuito (Spark) incluye aproximadamente 50.000 lecturas y 20.000 escrituras al día, además de 1 GiB de almacenamiento, de sobra para varios grupos de amigos. Consulta los límites vigentes en la documentación de Firebase.

## Solución de problemas

| Síntoma | Causa probable |
|---|---|
| «No se ha podido conectar con la base de datos» | `FIREBASE_CONFIG` mal pegado o con los valores `PEGA_AQUI` sin sustituir |
| `Missing or insufficient permissions` en la consola del navegador | Reglas sin publicar, o el correo de las reglas no coincide con el de Authentication |
| «Correo o contraseña incorrectos» en `#root` | El usuario no existe en Authentication o el método de correo y contraseña no está activado |
| «Este grupo no existe o el enlace no es correcto» | Enlace incompleto o grupo eliminado por Root |
| Error 404 en GitHub Pages | `index.html` no está en la raíz del repositorio, o Pages aún está publicando |
| El sorteo dice que no es posible | Demasiadas restricciones para el número de amigos por persona: quita alguna o baja *K* |

## Actualizar la aplicación

Edita `index.html` desde GitHub (icono del lápiz) o sube una versión nueva con el mismo nombre, y confirma el cambio con *Commit*. GitHub Pages republica el sitio automáticamente. Si sustituyes el archivo completo, recuerda volver a pegar tu `FIREBASE_CONFIG`.
