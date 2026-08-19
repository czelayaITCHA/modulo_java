# 1. Creación y configuración del proyecto `autofix-app`

En esta guía crearemos el proyecto React desde cero: creación con Vite (React 19), Tailwind CSS 3.x, PrimeReact 10.x (sin necesidad de licencia), y la estructura inicial de carpetas.

## 1.1 Prerrequisitos

- Node.js 20 o superior instalado (`node -v` para confirmar).
- `autofix-api` corriendo en `http://localhost:8080` (lo vamos a necesitar más adelante para probar peticiones reales).

## 1.2 Crear el proyecto con Vite

```bash
npm create vite@latest autofix-app -- --template react
cd autofix-app
npm install
```

Esto genera un proyecto con React 19 (la plantilla `react` de Vite ya trae la versión actual de React por defecto — verifica en tu `package.json` que quede como `"react": "^19.2.0"` y `"react-dom": "^19.2.0"`; si Vite instaló una versión distinta, ajústala a mano y corre `npm install` de nuevo).

Verifica que arranca correctamente antes de seguir:

```bash
npm run dev
```

Debería ver la página de bienvenida por defecto de Vite + React en `http://localhost:5173`.
<img width="1341" height="614" alt="image" src="https://github.com/user-attachments/assets/39d586eb-a4d7-4495-b770-c1afdc5deb96" />

## 1.3 Estructura inicial generada

```
autofix-app/
├── index.html
├── package.json
├── vite.config.js
├── eslint.config.js
├── public/
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── App.css
    └── index.css
```

## 1.4 Instalar Tailwind CSS 3.x (no 4.x)

Tailwind 4 cambió por completo su forma de configurarse (ya no usa `tailwind.config.js` por defecto), fijamos la versión 3 explícitamente:

```bash
npm install -D tailwindcss@^3.4.17 postcss autoprefixer
npx tailwindcss init -p
```

Esto genera `tailwind.config.js` y `postcss.config.js`. Edita `tailwind.config.js` para que `content` apunte a los archivos JSX (si no, Tailwind no genera el CSS de las clases que use):

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
}
```

`postcss.config.js` queda tal como Vite/Tailwind lo generó:

```js
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

## 1.5 Instalar PrimeReact 10.x + PrimeIcons

```bash
npm install primereact@^10.9.7 primeicons@^7.0.0
```

**Por qué la versión 10 y no la más reciente:** PrimeReact 11+ cambió su sistema de theming (adiós a los archivos `.css` de tema clásicos, ahora usa *design tokens* vía `@primereact/core`) y además requiere una licencia PrimeUI (gratuita "Community" o paga "Commercial") — sin esa licencia, la app funciona pero deja un warning en consola. La versión 10.x sigue siendo 100% MIT, gratuita para siempre, sin ninguna clave que gestionar — la elección correcta para este proyecto.

## 1.6 Configurar `src/index.css` — capas de cascada (Tailwind + PrimeReact sin conflictos)

Reemplaza el contenido de `src/index.css` por esto (mismo patrón exacto que ya funciona en `sife-app`):

```css
@layer tailwind-base, primereact, tailwind-utilities;

@import 'primereact/resources/themes/lara-light-blue/theme.css' layer(primereact);
@import 'primeicons/primeicons.css';

@layer tailwind-base {
  @tailwind base;
}

@layer tailwind-utilities {
  @tailwind components;
  @tailwind utilities;
}
```

**Por qué este orden:** `@layer tailwind-base, primereact, tailwind-utilities;` declara la prioridad de menor a mayor. El reset de Tailwind se aplica primero (`tailwind-base`), el tema de PrimeReact se pinta encima (`primereact`), y las utilidades de Tailwind ganan al final (`tailwind-utilities`) — así `<Button className="bg-red-500" />` sí se ve rojo, sin pelear por especificidad de selectores CSS. `primeicons.css` va sin capa asignada a propósito, para que los glifos de los íconos nunca queden obstruídos.

*(Puede cambiar `lara-light-blue` por cualquier otro tema del catálogo de PrimeReact 10 si prefiere otra paleta — el mecanismo de capas funciona igual con cualquiera.)*
## 1.7 Instalar dependencias adicionales

Estas las vamos a necesitar casi de inmediato (llamadas a `autofix-api`, rutas, confirmaciones, y leer el JWT en el frontend):

```bash
npm install axios react-router-dom sweetalert2 jwt-decode
```

| Paquete | Para qué |
|---|---|
| `axios` | Peticiones HTTP a `autofix-api` |
| `react-router-dom` | Rutas y navegación entre pantallas |
| `sweetalert2` | Confirmaciones y alertas |
| `jwt-decode` | Leer el payload del JWT (nombre, rol, clienteId/empleadoId) en el frontend sin necesidad de llamar al backend para saber quién está logueado |

*(`chart.js` / `react-chartjs-2` para el Dashboard y `react-icons` los instalamos más adelante, cuando lleguemos a esa parte — no hace falta cargarlos desde el inicio.)*

## 1.8 Estructura de carpetas

Crear la siguiente estructura de carpetas (`App.jsx` importando `./app/Router`, `main.jsx` envolviendo con un `AuthProvider`):

```
src/
├── main.jsx
├── App.jsx
├── App.css
├── index.css
├── app/
│   └── Router.jsx          (se crea en la siguiente guía)
├── auth/
│   └── AuthContext.jsx     (se crea en la siguiente guía)
├── components/
│   ├── catalogos/
│   └── ordenes-trabajo/
├── layout/
│   
|── services/
|
└── utils/

```

Por ahora, `Router.jsx` y `AuthContext.jsx` **todavía no existen** — los construimos en la próxima guía. Deje `App.jsx` con un contenido temporal simple mientras tanto:

```jsx
function App() {
  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold">AutoFix</h1>
      <p>Proyecto configurado correctamente.</p>
    </div>
  );
}

export default App;
```

Y `main.jsx` sin el `AuthProvider` todavía (lo agregamos cuando exista):

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

## 1.9 Variables de entorno — URL de la API

En vez de escribir `http://localhost:8080` a mano en cada archivo, crea un `.env` en la raíz del proyecto:

```
VITE_API_URL=http://localhost:8080/api
```

Uso en cualquier archivo del proyecto:
```js
const API_URL = import.meta.env.VITE_API_URL;
```

**Importante:** Vite solo expone al navegador las variables que empiezan con el prefijo `VITE_` — cualquier otro nombre queda oculto por seguridad. Verifica que `.env` esté en tu `.gitignore` (Vite ya lo agrega ahí por defecto) para no subir configuración de entorno al repositorio.

## 1.10 Verificar que todo quedó bien configurado

Con el contenido temporal de `App.jsx` de arriba, agrega momentáneamente un componente de PrimeReact para probar la integración completa:

```jsx
import { Button } from "primereact/button";

function App() {
  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-4">AutoFix</h1>
      <Button label="Probar" icon="pi pi-check" className="bg-red-500" />
    </div>
  );
}

export default App;
```

```bash
npm run dev
```
<img width="1354" height="283" alt="image" src="https://github.com/user-attachments/assets/9467f6af-2334-4640-9474-aa5fd32b4a52" />

| # | Qué revisar | Resultado esperado |
|---|---|---|
| 1 | Color del botón | Rojo (`bg-red-500` de Tailwind), no el azul por defecto de PrimeReact — confirme que las capas están en el orden correcto |
| 2 | Forma del botón | Con el estilo redondeado/padding propio de PrimeReact, no un `<button>` sin estilo — confirme que el tema sí cargó |
| 3 | El ícono `pi pi-check` | Se ve el ✓ real, no un cuadro vacío |
| 4 | Consola del navegador (F12) | Sin errores de "Failed to resolve import", sin warning de licencia PrimeUI |

Si los 4 puntos se cumplen, el proyecto está listo para empezar a construir sobre él. Quite el `<Button>` de prueba (o déjelo, ya no importa) y sigue con la siguiente guía: `AuthContext` + `Router`, para conectar `autofix-app` con el login de `autofix-api`.
