# 2. Autenticación y Layout de `autofix-app`

En esta guía conectaremos el frontend con el login de `autofix-api`, y armamos el panel administrativo completo: `Navbar`, `Sidebar` (con las opciones de menú y filtrado por rol), `Footer`, y `AppLayout` que integra todo. Al terminar, un usuario podrá autenticarse y ver el panel completo, responsivo, con el `Sidebar` ocultándose correctamente en resoluciones móviles.

## 2.1 Decisión de diseño: qué se guarda en `localStorage`

**Solo el token.** Nunca un objeto "usuario" por separado. Todo lo demás (nombre, rol, tipo, `empleadoId`/`clienteId`) se deriva del token con `jwt-decode` cada vez que se necesita — en el login, y al restaurar la sesión al recargar la página.

**Por qué importa esta decisión:** si guardaras el token y el usuario como dos cosas separadas en `localStorage`, existe el riesgo de que queden desincronizados (por ejemplo, si el token se reemplaza pero el objeto usuario viejo no se actualiza). Con una sola fuente de verdad (el token), eso es estructuralmente imposible — no hay dos copias que puedan desincronizarse.

## 2.2 Constante compartida para la clave de `localStorage`

```js
// src/utils/constants.js
export const CLAVE_TOKEN = "autofix_token";
```
La importan tanto `AuthContext` como `axiosClient` — si el nombre de la clave cambia algún día, se cambia en un solo lugar.

## 2.3 Cliente Axios centralizado

```js
// src/services/axiosClient.js
import axios from "axios";
import { CLAVE_TOKEN } from "../utils/constants";

const axiosClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

// Request: agrega el token a CADA petición automáticamente - ningún
// componente tiene que acordarse de hacerlo por su cuenta.

axiosClient.interceptors.request.use((config) => {
  const token = localStorage.getItem(CLAVE_TOKEN);
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response: si el backend responde 401 (token vencido o inválido), se
// limpia la sesión y se manda a /login. Esto vive fuera de un componente
// React (es un interceptor de axios), por eso usa window.location en vez
// de useNavigate - no hay contexto de React disponible aquí.
axiosClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem(CLAVE_TOKEN);
      if (window.location.pathname !== "/login") {
        window.location.href = "/login";
      }
    }
    return Promise.reject(error);
  }
);

export default axiosClient;
```

A partir de aquí, **toda** llamada a `autofix-api` en el resto del proyecto se hace con `axiosClient`, nunca con `axios` directo — así se garantiza que siempre lleve el token y que un 401 siempre se maneje igual.

## 2.4 `AuthContext` — el estado global de sesión

```jsx
import { createContext, useContext, useState, useEffect } from "react";
import { jwtDecode } from "jwt-decode";
import { CLAVE_TOKEN } from "../utils/constants";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [token, setToken] = useState(null);
  const [usuario, setUsuario] = useState(null);

  const [cargando, setCargando] = useState(true);

  useEffect(() => {
    restaurarSesion();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  const restaurarSesion = () => {
    const tokenGuardado = localStorage.getItem(CLAVE_TOKEN);

    if (!tokenGuardado) {
      setCargando(false);
      return;
    }

    try {
      const payload = jwtDecode(tokenGuardado);

      // "exp" en el JWT viene en segundos desde epoch; Date.now() da milisegundos.
      const yaExpiro = payload.exp * 1000 < Date.now();
      if (yaExpiro) {
        localStorage.removeItem(CLAVE_TOKEN);
        setCargando(false);
        return;
      }

      setToken(tokenGuardado);
      setUsuario(mapearPayload(payload));
    } catch {
      // Token corrupto o ilegible - se descarta en vez de dejar la app en
      // un estado a medias donde "hay token" pero "no hay usuario".
      localStorage.removeItem(CLAVE_TOKEN);
    } finally {
      setCargando(false);
    }
  };

  const login = (tokenNuevo) => {
    const payload = jwtDecode(tokenNuevo);
    localStorage.setItem(CLAVE_TOKEN, tokenNuevo);
    setToken(tokenNuevo);
    setUsuario(mapearPayload(payload));
  };

  const logout = () => {
    localStorage.removeItem(CLAVE_TOKEN);
    setToken(null);
    setUsuario(null);
  };

  const tienePermiso = (rolesPermitidos) => {
    if (!rolesPermitidos || rolesPermitidos.length === 0) return true;
    return usuario ? rolesPermitidos.includes(usuario.rol) : false;
  };

  const value = {
    usuario,
    token,
    cargando,
    estaAutenticado: !!token,
    login,
    logout,
    tienePermiso,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

// Único punto donde se traduce el payload del token a la forma que usa
// el resto de la app - si el backend agrega/renombra un claim, solo se
// ajusta aquí.
const mapearPayload = (payload) => {
  return {
    username: payload.sub,
    nombre: payload.nombre,
    tipo: payload.tipo,           // "EMPLEADO" o "CLIENTE"
    rol: payload.rol,
    empleadoId: payload.empleadoId,
    clienteId: payload.clienteId,
  };
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth debe usarse dentro de un AuthProvider");
  }
  return context;
}
```
## 2.5 Protección de Rutas
* crear el archivo RutaProtegida.jsx en src/auth
```jsx
import { Navigate } from "react-router-dom";
import { useAuth } from "./AuthContext";

export default function RutaProtegida ({children, rolesPermisos}) {
    const {estaAutenticado, cargando, tienePermiso} = useAuth();

    if(cargando){
        return (
            <div className="h-screen flex items-center justify-center text-slate-500">
                Cargando...
            </div>
        );
    }

    if(!estaAutenticado){
        return <Navigate to="/login" replace />
    }

    if(rolesPermisos && !tienePermiso(rolesPermisos)){
        return <Navigate to="/" replace />
    }

    return children;

}
```
## 2.6 Crear el componente Login.jsx en la carpeta auth
```jsx
import { useState } from "react";
import { useNavigate } from "react-router-dom";
import Swal from "sweetalert2";
import axiosClient from "../services/axiosClient";
import { useAuth } from "./AuthContext";

export default function LoginPage() {
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [enviando, setEnviando] = useState(false);

  const { login, estaAutenticado } = useAuth();
  const navigate = useNavigate();

  // Si ya hay sesión activa, lo mandamos directo al panel en vez de
  // mostrar el formulario.
  if (estaAutenticado) {
    navigate("/", { replace: true });
    return null;
  }

  const manejarEnvio = async (evento) => {
    evento.preventDefault();
    setEnviando(true);

    try {
      const respuesta = await axiosClient.post("/auth/login", { username, password });
      login(respuesta.data.auth.token);
      navigate("/", { replace: true });
    } catch (error) {
        console.log(error)
      const mensaje = error.response?.data?.message || "No se pudo iniciar sesión";
      Swal.fire({ icon: "error", title: "Error", text: mensaje });
    } finally {
      setEnviando(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50 p-4">
      <form onSubmit={manejarEnvio} className="bg-white shadow-md rounded-lg p-8 w-full max-w-sm">
        <h1 className="text-2xl font-bold text-slate-900 mb-6 text-center">AutoFix</h1>

        <label className="block text-sm font-medium text-slate-700 mb-1">Usuario</label>
        <input
          type="text"
          value={username}
          onChange={(evento) => setUsername(evento.target.value)}
          className="w-full border border-slate-300 rounded-md px-3 py-2 mb-4 focus:outline-none focus:ring-2 focus:ring-blue-500"
          required
          autoFocus
        />

        <label className="block text-sm font-medium text-slate-700 mb-1">Contraseña</label>
        <input
          type="password"
          value={password}
          onChange={(evento) => setPassword(evento.target.value)}
          className="w-full border border-slate-300 rounded-md px-3 py-2 mb-6 focus:outline-none focus:ring-2 focus:ring-blue-500"
          required
        />

        <button
          type="submit"
          disabled={enviando}
          className="w-full bg-blue-600 hover:bg-blue-700 disabled:opacity-50 text-white font-medium py-2 rounded-md transition-colors"
        >
          {enviando ? "Ingresando..." : "Ingresar"}
        </button>
      </form>
    </div>
  );
}
```
## 2.7 Crear el componente Navbar.jsx en la carpeta layout
```jsx
import { useAuth } from "../auth/AuthContext";

export default function Navbar({ onToggleSidebar }) {
  const { usuario, logout } = useAuth();

  return (
    <header className="sticky top-0 z-30 h-16 flex items-center justify-between px-4 md:px-6 bg-white border-b border-slate-200 shadow-sm shrink-0">
      <div className="flex items-center gap-3">
        <button
          onClick={onToggleSidebar}
          className="p-2 rounded-md text-slate-600 hover:bg-slate-100 transition-colors"
          aria-label="Alternar menú"
        >
          <i className="pi pi-bars text-xl" />
        </button>
        <span className="text-lg font-bold text-slate-900">Sistema de Gestión de Taller Automotríz</span>
      </div>

      <div className="flex items-center gap-4">
        <div className="hidden sm:flex flex-col items-end leading-tight">
          <span className="text-sm font-medium text-slate-900">{usuario?.nombre}</span>
          <span className="text-xs text-slate-500">{usuario?.rol}</span>
        </div>
        <button
          onClick={logout}
          className="p-2 rounded-md text-slate-600 hover:bg-red-50 hover:text-red-600 transition-colors"
          aria-label="Cerrar sesión"
          title="Cerrar sesión"
        >
          <i className="pi pi-sign-out text-xl" />
        </button>
      </div>
    </header>
  );
}
```

## 2.8 Crear el componente Sidebar.jsx en la carpeta layout
```jsx
import { useState } from "react";
import { NavLink } from "react-router-dom";
import { useAuth } from "../auth/AuthContext";

const opcionesMenu = [
  { etiqueta: "Inicio", icono: "pi pi-home", ruta: "/" },
  {
    etiqueta: "Catálogos",
    icono: "pi pi-box",
    submenu: [
      { etiqueta: "Marcas", ruta: "/catalogos/marcas" },
      { etiqueta: "Modelos", ruta: "/catalogos/modelos" },
      { etiqueta: "Repuestos y Servicios", ruta: "/catalogos/repuestos-servicios" },
    ],
  },
  { etiqueta: "Clientes", icono: "pi pi-users", ruta: "/clientes" },
  { etiqueta: "Gestión de Órdenes", icono: "pi pi-clipboard", ruta: "/ordenes-trabajo" },
  { etiqueta: "Reportes", icono: "pi pi-chart-bar", ruta: "/reportes" },
  // rolesPermitidos: si se define, la opción solo aparece para esos roles -
  // mismo criterio que ya aplicamos con @PreAuthorize del lado del backend.
  { etiqueta: "Gestión de Usuarios", icono: "pi pi-user-edit", ruta: "/usuarios", rolesPermitidos: ["ADMIN"] },
];

export default function Sidebar ({ isOpen }) {
  const { usuario, tienePermiso } = useAuth();
  const [submenuAbierto, setSubmenuAbierto] = useState(null);

  const opcionesVisibles = opcionesMenu.filter(
    (opcion) => !opcion.rolesPermitidos || tienePermiso(opcion.rolesPermitidos)
  );

  return (
    <aside
      className={`
        bg-slate-900 text-slate-100 flex flex-col shrink-0
        fixed md:static inset-y-0 left-0 z-30 h-full
        transition-all duration-300 ease-in-out
        ${isOpen
          ? "w-64 translate-x-0"
          : "w-64 -translate-x-full md:w-0 md:translate-x-0 md:overflow-hidden"}
      `}
    >
      <div className="h-16 flex items-center px-4 border-b border-slate-700 shrink-0">
        <span className="font-bold text-lg whitespace-nowrap">AutoFix</span>
      </div>

      <nav className="flex-1 overflow-y-auto py-2">
        {opcionesVisibles.map((opcion) => (
          <div key={opcion.etiqueta}>
            {opcion.submenu ? (
              <SubmenuItem
                opcion={opcion}
                abierto={submenuAbierto === opcion.etiqueta}
                onToggle={() =>
                  setSubmenuAbierto(submenuAbierto === opcion.etiqueta ? null : opcion.etiqueta)
                }
              />
            ) : (
              <EnlaceMenu opcion={opcion} />
            )}
          </div>
        ))}
      </nav>

      <div className="px-4 py-3 border-t border-slate-700 text-xs text-slate-400 shrink-0 truncate">
        {usuario?.nombre}
      </div>
    </aside>
  );
}

const EnlaceMenu = ({ opcion }) => {
  return (
    <NavLink
      to={opcion.ruta}
      end={opcion.ruta === "/"}   // sin "end", "/" quedaría "activo" en cualquier ruta
      className={({ isActive }) =>
        `flex items-center gap-3 px-4 py-3 text-sm transition-colors ${
          isActive ? "bg-blue-600 text-white" : "hover:bg-slate-800"
        }`
      }
    >
      <i className={`${opcion.icono} text-base`} />
      {opcion.etiqueta}
    </NavLink>
  );
}

const SubmenuItem = ({ opcion, abierto, onToggle }) => {
  return (
    <>
      <button
        onClick={onToggle}
        className="w-full flex items-center justify-between gap-3 px-4 py-3 text-sm hover:bg-slate-800 transition-colors"
      >
        <span className="flex items-center gap-3">
          <i className={`${opcion.icono} text-base`} />
          {opcion.etiqueta}
        </span>
        <i className={`pi pi-chevron-${abierto ? "up" : "down"} text-xs`} />
      </button>

      {abierto && (
        <div className="bg-slate-950">
          {opcion.submenu.map((sub) => (
            <NavLink
              key={sub.ruta}
              to={sub.ruta}
              className={({ isActive }) =>
                `block pl-12 pr-4 py-2 text-sm transition-colors ${
                  isActive ? "bg-blue-600 text-white" : "text-slate-300 hover:bg-slate-800"
                }`
              }
            >
              {sub.etiqueta}
            </NavLink>
          ))}
        </div>
      )}
    </>
  );
}
```
## 2.9 Crear el componente Footer.jsx en la carpeta layout
```jsx
export default function Footer(){
  return (
    <footer className="px-4 md:px-6 py-3 text-center text-xs text-slate-400 border-t border-slate-200 bg-white shrink-0">
      © {new Date().getFullYear()} AutoFix — Sistema de gestión de taller mecánico
    </footer>
  );
}
```

## 2.10 Crear el componente AppLayout en la carpeta app
```jsx
import { useState, useEffect } from "react";
import { Outlet, useLocation } from "react-router-dom";
import Navbar from "../layout/Navbar";
import Sidebar from "../layout/Sidebar";
import Footer from "../layout/Footer";

export default function AppLayout(){
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const location = useLocation();

  useEffect(() => {
    const isDesktop = window.innerWidth >= 768;
    setSidebarOpen(isDesktop);

    const handleResize = () => {
      if (window.innerWidth < 768) setSidebarOpen(false);
      else setSidebarOpen(true);
    };

    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  useEffect(() => {
    if (window.innerWidth < 768) setSidebarOpen(false);
  }, [location]);

  return (
    <div className="h-screen flex flex-col bg-slate-50 overflow-hidden font-sans text-slate-900">
      <Navbar onToggleSidebar={() => setSidebarOpen(!sidebarOpen)} />

      <div className="flex flex-1 relative overflow-hidden h-[calc(100vh-64px)]">
        <Sidebar isOpen={sidebarOpen} />

        <div className="flex-1 flex flex-col min-w-0 overflow-hidden relative h-full">
          <main className="flex-1 p-4 md:p-6 overflow-y-auto bg-[radial-gradient(#e5e7eb_1px,transparent_1px)] [background-size:16px_16px]">
            <Outlet />
          </main>

          <Footer />
        </div>
      </div>

      {sidebarOpen && (
        <div
          className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm z-20 md:hidden transition-all"
          onClick={() => setSidebarOpen(false)}
        />
      )}
    </div>
  );
}
```
## 2.11 Definir sistema de rutas en Router.jsx
```jsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import AppLayout from "./AppLayout";
import LoginPage from "../auth/Login";
import RutaProtegida from "../auth/RutaProtegida";

export default function Router() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<LoginPage />} />

        <Route
          element={
            <RutaProtegida>
              <AppLayout />
            </RutaProtegida>
          }
        >
          <Route path="/" element={<div>Inicio</div>} />
          <Route path="/catalogos/marcas" element={<div>Marcas</div>} />
          <Route path="/catalogos/modelos" element={<div>Modelos</div>} />
          <Route path="/catalogos/repuestos-servicios" element={<div>Repuestos y Servicios</div>} />
          <Route path="/clientes" element={<div>Clientes</div>} />
          <Route path="/ordenes-trabajo" element={<div>Gestión de Órdenes</div>} />
          <Route path="/reportes" element={<div>Reportes</div>} />

          <Route
            path="/usuarios"
            element={
              <RutaProtegida rolesPermitidos={["ADMIN"]}>
                <div>Gestión de Usuarios</div>
              </RutaProtegida>
            }
          />
        </Route>

        <Route path="*" element={<Navigate to="/" replace />} />
      </Routes>
    </BrowserRouter>
  );
}
```

## 2.12 Actualizar los archivos App.jsx y main.jsx
* App.jsx
```jsx
import Router from "./app/Router";

const App = () => {
  return <Router />;
}

export default App;
```
* main.jsx
```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'
import { AuthProvider } from './auth/AuthContext.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <AuthProvider>
      <App />
    </AuthProvider>
  </StrictMode>
)
```
 
