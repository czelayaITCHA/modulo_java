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

export default RutaProtegida = ({children, rolesPermisos}) => {
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
