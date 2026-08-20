# 3. CRUD de Marca (tabla independiente)

En esta guía construimos el primer CRUD real de `autofix-app`: `Marca`, la entidad más simple del backend (sin relaciones, `id` + `nombre`). Sirve como base reutilizable para los catálogos que siguen (`Modelo`, `RepuestoServicio`), y de paso conecta por primera vez la pantalla con `autofix-api` end-to-end: listar, crear, editar y eliminar. También cerramos aquí un problema que aparece justo en este punto - CORS - porque es la primera vez que el navegador hace una petición real cruzando de un origen (`localhost:5173`) a otro (`localhost:8080`).

## 3.1 Servicio genérico de catálogos

En vez de escribir las mismas 5 operaciones (`getAll`, `getById`, `create`, `update`, `delete`) para cada catálogo, se arma **una sola vez** como una función *factory* que genera el servicio a partir del endpoint:

```js
// src/services/catalogosService.js
import axiosClient from "./axiosClient";

const catalogoService = (endpoint) => ({

    //función para obtener todos los registros - para un DropDown o dt
    getAll: async () => {
        const response = await axiosClient.get(endpoint);
        return response.data;
    },
    //obtener un registro por id
    getById: async (id) => {
        const response = await axiosClient.get(`${endpoint}/${id}`);
        return response.data;
    },

    //crear un registro
    create: async (dto) => {
        const response = await axiosClient.post(endpoint,dto);
        return response.data;
    },
    //actualizar un registro
    update: async (id,dto) => {
        const response = await axiosClient.put(`${endpoint}/${id}`,dto);
        return response.data;
    },
    delete: async (id) => {
        const response = await axiosClient.delete(`${endpoint}/${id}`);
        return response.data;
    },

});

export default catalogoService;
```
## 3.2 Servicio específico de Marca

```js
// src/services/marcaService.js
import catalogoService from "./catalogosService";

export const marcaService = catalogoService("/marcas");
```
## 3.3 La forma exacta de cada respuesta del backend

| Operación | Forma de la respuesta |
|---|---|
| `GET /marcas` | Arreglo directo: `[{ id, nombre }, ...]` |
| `GET /marcas/{id}` | Objeto directo: `{ id, nombre }` |
| `POST /marcas` | `{ "message": "...", "marca": { id, nombre } }` |
| `PUT /marcas/{id}` | `{ "message": "...", "marca": { id, nombre } }` |
| `DELETE /marcas/{id}` | `{ "message": "..." }` (sin dato adicional) |
| Cualquier error (400/404/409/500) | `{ "message": "..." }` |

## 3.4 Habilitar CORS en el backend (antes de probar nada desde el navegador)

Sin esto, cualquier petición desde `autofix-app` (`http://localhost:5173`) hacia `autofix-api` (`http://localhost:8080`) es bloqueada por el navegador con un error de CORS en consola - **aunque en Postman funcione perfecto**, porque Postman no aplica la política de mismo origen que sí aplican los navegadores.

**Por qué `@CrossOrigin` en los Controllers NO alcanza:** con Spring Security activo, la petición `OPTIONS` de *preflight* que manda el navegador pasa primero por el filtro de seguridad. Si ahí no está permitida, se rechaza antes de que Spring MVC llegue siquiera a leer la anotación `@CrossOrigin` del Controller.

En `SecurityConfig.java`, se agrega un bean de configuración de CORS y se conecta al `SecurityFilterChain`, agrega el siguiente beans al final de la clase:

```java
import org.springframework.http.HttpMethod;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

// ...

/**
 * Configuración de CORS real, integrada al filtro de Spring Security.
 * @CrossOrigin en los Controllers NO es suficiente por sí solo.
 */
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();

    // Ajusta esta lista al/los puerto(s) real(es) donde corre autofix-app.
    // Vite puede tomar otro puerto si el 5173 ya está ocupado (5174, 5175...).
    configuration.setAllowedOrigins(List.of("http://localhost:5173", "http://localhost:5174"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    configuration.setAllowedHeaders(List.of("*"));

    // false porque el frontend manda el token vía header Authorization
    // (axiosClient), no vía cookies - no se necesita el modo "credentials" de CORS.
    configuration.setAllowCredentials(false);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

Y dentro de `securityFilterChain(HttpSecurity http)`, dos cambios sobre lo que ya tenías:

```java
http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))   // Primera línea de la cadena
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
                // El navegador NUNCA manda el header Authorization en una
                // petición OPTIONS de preflight - sin esta línea, Spring
                // Security la rechazaría como "no autenticada" antes de
                // que el preflight pudiera completarse.
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()   // Primera regla
                .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                .requestMatchers("/images/**").permitAll()
                .anyRequest().authenticated()
        )
        // ...el resto de la cadena queda exactamente igual
```

**Verificación específica de CORS:** con `autofix-api` reiniciado, abre las DevTools del navegador (pestaña *Network*) y haz cualquier petición desde `autofix-app`. Busca la petición `OPTIONS` que precede a la petición real - en su respuesta debe aparecer el header `Access-Control-Allow-Origin` con el puerto exacto de tu frontend. Si el puerto no coincide carácter por carácter (incluyendo `http://`, sin `/` al final), el navegador sigue bloqueando aunque todo lo demás esté bien.

Con este ajuste, ya puede usar `@CrossOrigin` en los Controllers o quitarlo - a partir de aquí no hace ninguna diferencia, porque CORS se resuelve centralizado en `SecurityConfig`.

## 3.5 El componente `Marcas`

```jsx
//importaciones necesarias
import { useState, useEffect, useRef } from "react";
import { DataTable } from "primereact/datatable";
import { Column } from "primereact/column";
import { Button } from "primereact/button";
import { Toolbar } from "primereact/toolbar";
import { Dialog } from "primereact/dialog";
import { InputText } from "primereact/inputtext";
import { IconField } from "primereact/iconfield";
import { InputIcon } from "primereact/inputicon";
import { classNames } from "primereact/utils";
import { Toast } from "primereact/toast";
import Swal from "sweetalert2";

import { marcaService } from "../../services/marcaService";

export default function Marcas() {
    //lógica del componente
    const emptyMarca = { id: null, nombre: "" };
    const NOMBRE_MIN = 3;
    const NOMBRE_MAX = 50;

    //definiendo los estados - Hooks de estado
    const [marcas, setMarcas] = useState([]);
    const [marca, setMarca] = useState(emptyMarca);
    const [dialog, setDialog] = useState(false);
    const [submitted, setSubmitted] = useState(false);
    const [globalFilter, setGlobalFilter] = useState(null);
    const [loading, setLoading] = useState(false);
    const [guardando, setGuardando] = useState(false); // evita doble clic en Guardar mientras la petición está en curso

    const toast = useRef(null);

    useEffect(() => {
        fetchMarcas();
    }, []);

    //funciones para hacer peticiones al bk
    const fetchMarcas = async () => {
        setLoading(true);
        try {
            const data = await marcaService.getAll();
            setMarcas(data);
        } catch {
            toast.current.show({
                severity: "error",
                summary: "Error",
                detail: "No se pudo obtener las marcas",
            });
        } finally {
            setLoading(false);
        }
    };

    //función para abrir modal y agregar una marca
    const openNew = () => {
        setMarca(emptyMarca);
        setSubmitted(false);
        setDialog(true);
    };

    //función para abrir modal, para editar una marca
    const openEdit = (rowData) => {
        setMarca({ ...rowData });
        setSubmitted(false);
        setDialog(true);
    };

    
    const existeNombreDuplicado = (nombre) => {
        const nombreNormalizado = nombre.trim().toLowerCase();
        return marcas.some(
            (m) => m.id !== marca.id && m.nombre.trim().toLowerCase() === nombreNormalizado
        );
    };

    //función para validar datos del formulario
    const validarFormulario = () => {
        const nombre = marca.nombre?.trim() ?? "";
        if (!nombre) return `El nombre es requerido.`;
        if (nombre.length < NOMBRE_MIN) return `Nombre debe tener al menos ${NOMBRE_MIN} caracteres.`;
        if (nombre.length > NOMBRE_MAX) return `Nombre no puede superar los ${NOMBRE_MAX} caracteres.`;
        if (existeNombreDuplicado(nombre)) return `Ya existe una marca registrada con el nombre "${nombre}".`;

        return null;
    };

    const errorValidacion = submitted ? validarFormulario() : null;

    
    const saveOrUpdate = async () => {
        setSubmitted(true);
        if (validarFormulario()) return; 

        setGuardando(true);
        try {
            const datosLimpios = { ...marca, nombre: marca.nombre.trim() };

            const respuesta = marca.id
                ? await marcaService.update(marca.id, datosLimpios)
                : await marcaService.create(datosLimpios);

            toast.current.show({ severity: "success", summary: "Éxito", detail: respuesta.message, life: 3000 });
            setDialog(false);
            fetchMarcas();
        } catch (error) {
            const msj = error.response?.data?.message || "Ocurrió un error al guardar la marca";
            toast.current.show({ severity: "error", summary: "Error", detail: msj, life: 4000 });
        } finally {
            setGuardando(false);
        }
    };

    
    const confirmDelete = (rowData) => {
        Swal.fire({
            title: "¿Eliminar marca?",
            html: `Esta acción no se puede deshacer.<br/>Se eliminará <b>${rowData.nombre}</b>.`,
            icon: "question",
            showCancelButton: true,
            confirmButtonText: "Sí, eliminar",
            cancelButtonText: "Cancelar",
            confirmButtonColor: "#dc2626",
            cancelButtonColor: "#6b7280",
            reverseButtons: true,
        }).then((resultado) => {
            if (resultado.isConfirmed) {
                deleteMarca(rowData.id, rowData.nombre);
            }
        });
    };

    const deleteMarca = async (id, nombre) => {
        try {
            const respuesta = await marcaService.delete(id);
            toast.current.show({ severity: "success", summary: "Éxito", detail: respuesta.message, life: 3000 });
            fetchMarcas();
        } catch (error) {
            const msj = error.response?.data?.message || `No se pudo eliminar "${nombre}"`;
            toast.current.show({ severity: "error", summary: "Error", detail: msj, life: 4000 });
        }
    };

    const templateAcciones = (rowData) => {
        return (
            <div className="flex gap-2 justify-center md:justify-start">
                <Button icon="pi pi-pencil" rounded outlined severity="success" onClick={() => openEdit(rowData)} />
                <Button icon="pi pi-trash" rounded outlined severity="danger" onClick={() => confirmDelete(rowData)} />
            </div>
        );
    };

    const header = (
        <div className="flex flex-col md:flex-row md:items-center justify-between gap-4 p-1">
            <h4 className="m-0 text-xl font-bold text-gray-700">Mantenimiento de Marcas</h4>

            <div className="w-full md:w-72">
                <IconField iconPosition="left">
                    <InputIcon className="pi pi-search" />
                    <InputText
                        type="search"
                        onInput={(e) => setGlobalFilter(e.target.value)}
                        placeholder="Buscar por nombre..."
                        className="w-full p-inputtext-sm"
                    />
                </IconField>
            </div>
        </div>
    );

    return (
        <div className="p-2 md:p-4">
            <Toast ref={toast} />
            <div className="card shadow-md rounded-xl bg-white">
                <Toolbar
                    className="mb-4 bg-gray-50 border-none"
                    left={() => (
                        <Button label="Nueva Marca" icon="pi pi-plus" severity="primary" onClick={openNew} />
                    )}
                />
                <DataTable
                    value={marcas}
                    loading={loading}
                    paginator
                    rows={10}
                     rowsPerPageOptions={[5, 10, 25, 50]} 
                    header={header}
                    globalFilter={globalFilter}
                    responsiveLayout="stack"
                    breakpoint="768px"
                    className="p-datatable-sm"
                    emptyMessage="No se encontraron marcas"
                >
                    <Column field="nombre" header="Marca" sortable className="font-semibold" />
                    <Column header="Acciones" body={templateAcciones} exportable={false} style={{ minWidth: "8rem" }} />
                </DataTable>
            </div>

            {/* Inicio del dialog */}
            <Dialog
                visible={dialog}
                style={{ width: "32rem" }}
                breakpoints={{ "960px": "75vw", "641px": "90vw" }}
                header={marca.id ? "Actualizar Marca" : "Registrar Marca"}
                modal
                className="p-fluid"
                onHide={() => setDialog(false)}
                footer={
                    <div className="flex justify-end gap-2">
                        <Button label="Cancelar" icon="pi pi-times" outlined onClick={() => setDialog(false)} disabled={guardando} />
                        <Button
                            label={marca.id ? "Actualizar" : "Guardar"}
                            icon="pi pi-save"
                            onClick={saveOrUpdate}
                            loading={guardando}
                        />
                    </div>
                }
            >
                <div className="field">
                    <label htmlFor="nombre" className="font-bold block mb-2">
                        Nombre
                    </label>
                    <InputText
                        id="nombre"
                        value={marca.nombre}
                        onChange={(e) => setMarca({ ...marca, nombre: e.target.value })}
                        required
                        autoFocus
                        maxLength={NOMBRE_MAX}
                        className={classNames({ "p-invalid": errorValidacion })}
                    />
                    {errorValidacion && <small className="p-error">{errorValidacion}</small>}
                </div>
            </Dialog>
            {/* Inicio del dialog */}
        </div>
    );
}
```
## 3.6 Regla del proyecto: SweetAlert2 vs. Toast

Se aplica de forma consistente en todo el CRUD, y se repite en cada catálogo que sigue:

- **SweetAlert2** → solo cuando hay que **decidir algo** (confirmar una acción destructiva). En este componente, únicamente `confirmDelete`.
- **Toast de PrimeReact** → solo para **informar un resultado ya ocurrido** (éxito o error de una petición que ya se ejecutó). Nunca se usa para pedir confirmación.
- **Los errores de validación del formulario** (campo vacío, muy corto, duplicado) no usan ninguno de los dos - se muestran inline, justo bajo el campo (`<small className="p-error">`), porque es más rápido de leer que esperar un popup.

## 3.7 Clases que parecen de Tailwind pero son de PrimeFlex (no instalado en este proyecto)

Vale la pena tenerlas identificadas, porque no generan ningún error - simplemente no aplican ningún estilo, y ese tipo de bug es el más difícil de notar:

| Clase que NO funciona aquí | Equivalente real de Tailwind |
|---|---|
| `justify-content-center` / `justify-content-start` / `justify-content-end` | `justify-center` / `justify-start` / `justify-end` |
| `align-items-center` | `items-center` |
| `justify-items-start` (válida en Tailwind, pero para *grid*, no *flex*) | `justify-start` |
| `shadow-2` | `shadow-md` (o el nivel de sombra de Tailwind que prefieras) |
| `border-round-xl` | `rounded-xl` |

## 3.8 Piezas de PrimeReact usadas, en corto

| Componente | Para qué |
|---|---|
| `DataTable` + `Column` | Tabla principal, con paginación y `responsiveLayout="stack"` (en pantallas menores a `768px`, cada fila se convierte en tarjeta apilada) |
| `Toolbar` | Contenedor del botón "Nueva Marca" |
| `Dialog` | El único que queda: creación/edición (la confirmación de borrado ahora es SweetAlert2, no un `Dialog`) |
| `Toast` | Notificaciones de éxito/error - se controla con una `ref`, no con estado |
| `IconField` + `InputIcon` | El input de búsqueda con la lupa integrada |
| `classNames` (de `primereact/utils`) | Aplica `p-invalid` condicionalmente, para el borde rojo de validación |

## 3.9 Conectar la ruta

```jsx
// src/app/Router.jsx
import Marcas from "../components/catalogos/Marcas";
// ...
<Route path="/catalogos/marcas" element={<Marcas />} />
```

## 3.10 Verificar que todo funciona

Con `autofix-api` corriendo (ya con el CORS de la sección 3.4 configurado) y sesión iniciada:

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Cargar la pantalla | Sin errores de CORS en consola; la tabla muestra las marcas existentes |
| 2 | Crear una marca válida | Diálogo se cierra, `Toast` verde con el mensaje del backend, tabla se refresca sola |
| 3 | Crear con nombre vacío | Error inline "El nombre es requerido.", no se manda ninguna petición |
| 4 | Crear con nombre ya existente **en la tabla cargada** | Error inline instantáneo, sin esperar respuesta del backend (validación de frontend) |
| 5 | Crear con nombre duplicado que el frontend no detectó (ej. otra persona lo creó justo antes) | `Toast` rojo con el mensaje `409` real del backend |
| 6 | Editar una marca sin cambiar el nombre | Se guarda sin marcar "duplicado" (la validación se excluye a sí misma correctamente) |
| 7 | Eliminar una marca | Aparece el diálogo de **SweetAlert2** (no un `Dialog` de PrimeReact) pidiendo confirmar; al confirmar, `Toast` verde y desaparece de la tabla |
| 8 | Cancelar la eliminación en SweetAlert2 | La marca sigue en la tabla, sin ningún `Toast` (cancelar no es un resultado que informar) |
| 9 | Buscar por nombre | Filtra la tabla en vivo, sin pedir nada nuevo al backend |
| 10 | Responsividad | Reducir la ventana a menos de 768px convierte la tabla en tarjetas apiladas |

## 3.11 Una limitación conocida, sin resolver todavía

Si intenta eliminar una `Marca` que ya tiene `Modelo`s asociados, el backend no tiene hoy una validación de negocio para ese caso (a diferencia de `RepuestoServicio`, `Modelo` o `Vehiculo`, que sí protegen su borrado). La base de datos rechazará la operación por la restricción de llave foránea, y verás el mensaje genérico de error de base de datos en el `Toast`, no uno específico como *"no se puede eliminar, tiene modelos asociados"*. Se puede cerrar agregando la misma protección que ya tienen las otras entidades.
