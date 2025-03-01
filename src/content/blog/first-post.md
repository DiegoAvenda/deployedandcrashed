---
title: "Sveltekit + Google Oauth 2"
description: "Lorem ipsum dolor sit amet"
pubDate: "28/Feb/2025"
heroImage: "/oauth_google.png"
---

Quiero mostrar la manera mas simple de usar google oauth 2. Omitiendo varios pasos que hacen nuestro proceso mas seguro para que sea los mas facil de entender. Definitivamente no debes usar esto en una aplicacion real, es solo con el proposito de aprender el flujo de oauth. En un siguiente post, mostrare todos los pasos que se deben agregar. Me estoy basando principalmente en Lucia Auth https://lucia-auth.com/tutorials/google-oauth/sveltekit

Oauth 2.0 se compone principalmente de 3 pasos:

1. Creamos un endpoint en donde se generará la url de autorizacion que nos lleva a esta ventana:

![Alt text](/google-consent-screen.png)

2. Creamos el endpoint al que google redirigira al usuario despues que éste acceda a su cuenta de google desde la url de autorizacion. En donde recibiremos un code.

3. enviamos el code de vuelta a google, junto con el google client ID y el client secret, para recibir un JWT (JSON Web Token), el cual contiene la informacion del usuario, como su nombre, email e imagen.

### Crear una aplicación OAuth

Primero tenemos que crear un cliente de Google OAuth. Puedes seguir estos pasos de ayuda de google:
https://support.google.com/googleapi/answer/6158849?hl=en.
o puedes ver este video: https://youtu.be/AOp5TzRM5RY?si=3fZ9BbbQtVC663EY&t=773.
Establece la URI de redireccionamiento en http://localhost:5173/api/oauth/google Copia y pega el client ID y el client secret en tu archivo .env.

```markdown
# .env

GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
```

### Botón de inicio de sesión

Crea un botón de inicio de sesión, que debe ser un enlace a /api/oauth/google

```html
<!-- routes/+page.svelte -->
<a href="/api/oauth/google">Sign in with Google</a>
```

### Crear URL de autorización

Crea una ruta API en routes/api/oauth/google/+server.js y crea una nueva URL de autorización. Agrega el alcance openid y profile para tener acceso al perfil del usuario más adelante. Redirige al usuario a la URL de autorización. El usuario será redirigido a la página de inicio de sesión de Google.

```javascript
import { GOOGLE_CLIENT_ID } from "$env/static/private"

export async function GET() {
  const baseUrl = "https://accounts.google.com/o/oauth2/v2/auth"

  const options = {
    redirect_uri: "http://localhost:5173/api/oauth/google/callback",
    client_id: GOOGLE_CLIENT_ID,
    response_type: "code",
    scope: ["openid", "profile"].join(" "),
  }

  const queryParams = new URLSearchParams(options)

  return new Response(null, {
    status: 302,
    headers: {
      Location: `${baseUrl}?${queryParams.toString()}`,
    },
  })
}
```

### Validar URI de redireccionamiento

Crea una ruta de API routes/api/oauth/google/callback/+server.js para manejar la devolución de llamada. Si pasaste el alcance openidy profile, Google devolverá un ID de token con el perfil del usuario. Comprueba si el usuario ya está registrado; si no, crea un nuevo usuario.

```javascript
import { GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET } from "$env/static/private"
//import { getUserFromGoogleId, createUser } from '$lib/server/user.js';
import { error } from "@sveltejs/kit"

export async function GET(event) {
  const code = event.url.searchParams.get("code")
  if (!code) error(400, "Authorization code missing")

  let tokens
  const tokenParams = new URLSearchParams({
    redirect_uri: "http://localhost:5173/api/oauth/google/callback",
    client_id: GOOGLE_CLIENT_ID,
    client_secret: GOOGLE_CLIENT_SECRET,
    code,
    grant_type: "authorization_code",
  })
  try {
    const response = await fetch("https://oauth2.googleapis.com/token", {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: tokenParams,
    })

    tokens = await response.json()
  } catch (e) {
    // Invalid code or client credentials
    error(400, e)
  }
  const claims = decodeIdToken(tokens.id_token)
  const { name } = claims

  const encodedName = encodeURIComponent(name)

  return new Response(null, {
    status: 302,
    //redirigimos a nuestra pagina principal
    headers: { Location: `/?name=${encodedName}` },
  })
}

function decodeIdToken(idToken) {
  try {
    const payloadBase64 = idToken
      .split(".")[1]
      .replace(/-/g, "+")
      .replace(/_/g, "/")
    return JSON.parse(Buffer.from(payloadBase64, "base64").toString("utf-8"))
  } catch {
    return null
  }
}
```

Creamos routes/+page.js para leer el nommbre de la url y enviarlo a /routes/+page.svelte

```javascript
export async function load({ url }) {
  const name = url.searchParams.get("name")

  return {
    name,
  }
}
```

Y listo!, deberias ser redireccionado a '/' y ver tu nombre si usaste tu cuenta de google.
