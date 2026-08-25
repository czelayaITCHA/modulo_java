# 5. CRUD de RepuestoServicio (con imagen y previsualización)

En esta guía se construye el CRUD de `RepuestoServicio`, aplicando el mismo patrón ya establecido en `Marca` y `Vehiculo` (actualización local del arreglo sin volver a consultar al servidor, alertas diferenciadas por código de estado, confirmación con SweetAlert2, ocultamiento de acciones según el rol). La particularidad de esta entidad es que, cuando el tipo es `REPUESTO`, se puede adjuntar una imagen — y esa imagen debe poder previsualizarse antes de guardar o actualizar el registro, sin haberla enviado todavía al servidor.

## 5.1 Variable de entorno nueva: la URL de las imágenes

Las imágenes las sirve `autofix-api` en `/images/...`, una ruta que queda **fuera** del prefijo `/api` que usa `axiosClient`. Se agrega una variable de entorno independiente:

```
# .env
VITE_IMAGES_URL=http://localhost:8080/images
```

Y el helper correspondiente:

```js
// src/utils/constants.js - agregado
const IMAGENES_BASE_URL = import.meta.env.VITE_IMAGES_URL;

export const construirUrlImagen = (nombreArchivo) => {
  if (!nombreArchivo) return null;
  return `${IMAGENES_BASE_URL}/${nombreArchivo}`;
};
```

## 5.2 El servicio: por qué no reutiliza la factory genérica

`createCatalogoService` (usada por `Marca`, `Modelo`, `Vehiculo`, `Cliente`) asume que el cuerpo de la petición es JSON puro. El backend de `RepuestoServicio` espera `multipart/form-data`, con dos partes separadas: `dto` (el JSON, con su propio `Content-Type: application/json`) e `imagen` (el archivo, opcional). Por esa razón, este servicio se escribe aparte:

```js
// src/services/repuestoServicioService.js
import axiosClient from "./axiosClient";

const ENDPOINT = "/repuestos-servicios";

const construirFormData = (dto, archivoImagen) => {
  const formData = new FormData();

  // Blob explícito con type "application/json": sin esto, FormData trata
  // un string plano como "text/plain", y el backend fallaría al deserializarlo.
  formData.append("dto", new Blob([JSON.stringify(dto)], { type: "application/json" }));

  if (archivoImagen) {
    formData.append("imagen", archivoImagen);
  }

  return formData;
};

export const repuestoServicioService = {
  getAll: async () => {
    const response = await axiosClient.get(ENDPOINT);
    return response.data;
  },

  getById: async (id) => {
    const response = await axiosClient.get(`${ENDPOINT}/${id}`);
    return response.data;
  },

  create: async (dto, archivoImagen) => {
    const formData = construirFormData(dto, archivoImagen);
    const response = await axiosClient.post(ENDPOINT, formData);
    return response.data;
  },

  update: async (id, dto, archivoImagen) => {
    const formData = construirFormData(dto, archivoImagen);
    const response = await axiosClient.put(`${ENDPOINT}/${id}`, formData);
    return response.data;
  },

  delete: async (id) => {
    const response = await axiosClient.delete(`${ENDPOINT}/${id}`);
    return response.data;
  },
};
```

**Debe evitarse fijar manualmente el header `Content-Type` en las peticiones `create`/`update`.** Cuando el cuerpo de la petición es un `FormData`, axios (a través del navegador) arma automáticamente `multipart/form-data; boundary=...` con el valor de `boundary` correcto. Fijar el header a mano sin ese valor es una causa frecuente de que el backend no logre interpretar el cuerpo de la petición.

## 5.3 Programar el componente `RepuestosServicios`
```jsx
import { useState, useEffect, useRef } from "react";
import { DataTable } from "primereact/datatable";
import { Column } from "primereact/column";
import { Button } from "primereact/button";
import { Toolbar } from "primereact/toolbar";
import { Dialog } from "primereact/dialog";
import { InputText } from "primereact/inputtext";
import { InputNumber } from "primereact/inputnumber";
import { InputTextarea } from "primereact/inputtextarea";
import { Dropdown } from "primereact/dropdown";
import { Tag } from "primereact/tag";
import { IconField } from "primereact/iconfield";
import { InputIcon } from "primereact/inputicon";
import { classNames } from "primereact/utils";
import { Toast } from "primereact/toast";
import Swal from "sweetalert2";

import { repuestoServicioService } from "../../services/repuestoServicioService";
import { mostrarErrorApi, mostrarExitoApi } from "../../utils/alertasApi";
import { construirUrlImagen } from "../../utils/constants";
import { useAuth } from "../../auth/AuthContext";

const emptyItem = { id: null, nombre: "", descripcion: "", precio: null, stock: null, tipo: null, foto: null };
const NOMBRE_MAX = 100;
const DESCRIPCION_MAX = 250;
const TAMANIO_MAX_MB = 5;
const TIPOS = [
    { label: "Repuesto", value: "REPUESTO" },
    { label: "Servicio", value: "SERVICIO" },
];

export default function RepuestosServicios() {
    const [items, setItems] = useState([]);
    const [item, setItem] = useState(emptyItem);
    const [archivoImagen, setArchivoImagen] = useState(null);
    const [previewUrl, setPreviewUrl] = useState(null);
    const [dialog, setDialog] = useState(false);
    const [submitted, setSubmitted] = useState(false);
    const [globalFilter, setGlobalFilter] = useState(null);
    const [loading, setLoading] = useState(false);
    const [guardando, setGuardando] = useState(false);

    const toast = useRef(null);
    const inputImagenRef = useRef(null);
    const { tienePermiso } = useAuth();

    const puedeEditar = tienePermiso(["ADMIN", "RECEPCIONISTA"]);
    const puedeEliminar = tienePermiso(["ADMIN"]);

    useEffect(() => {
        fetchItems();
    }, []);

    useEffect(() => {
      return () => {
        if (previewUrl?.startsWith("blob:")) {
          URL.revokeObjectURL(previewUrl);
        }
      };
    }, []);

    const fetchItems = async () => {
        setLoading(true);
        try {
            const data = await repuestoServicioService.getAll();
            setItems(data);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los repuestos y servicios");
        } finally {
            setLoading(false);
        }
    };

    const limpiarSeleccionImagen = () => {
        if (previewUrl?.startsWith("blob:")) {
            URL.revokeObjectURL(previewUrl);
        }
        setArchivoImagen(null);
        setPreviewUrl(null);
        if (inputImagenRef.current) inputImagenRef.current.value = "";
    };

    const openNew = () => {
        setItem(emptyItem);
        limpiarSeleccionImagen();
        setSubmitted(false);
        setDialog(true);
    };

    const openEdit = (rowData) => {
        setItem({ ...rowData });
        setArchivoImagen(null);
        setPreviewUrl(rowData.tipo === "REPUESTO" ? construirUrlImagen(rowData.foto) : null);
        setSubmitted(false);
        setDialog(true);
    };

    const manejarCambioTipo = (nuevoTipo) => {
        setItem((prev) => ({ ...prev, tipo: nuevoTipo, stock: nuevoTipo === "REPUESTO" ? prev.stock : null }));
        if (nuevoTipo !== "REPUESTO") {
            limpiarSeleccionImagen();
        }
    };

    const manejarSeleccionImagen = (evento) => {
        const archivo = evento.target.files?.[0];
        if (!archivo) return;

        if (!archivo.type.startsWith("image/")) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "El archivo debe ser una imagen", life: 4000 });
            return;
        }
        if (archivo.size > TAMANIO_MAX_MB * 1024 * 1024) {
            toast.current.show({
                severity: "warn",
                summary: "Atención",
                detail: `La imagen no puede superar ${TAMANIO_MAX_MB} MB`,
                life: 4000,
            });
            return;
        }

        if (previewUrl?.startsWith("blob:")) {
            URL.revokeObjectURL(previewUrl);
        }
        setArchivoImagen(archivo);
        setPreviewUrl(URL.createObjectURL(archivo));
    };

    const validarFormulario = () => {
        const nombre = item.nombre?.trim() ?? "";
        if (!nombre) return "El nombre es obligatorio.";
        if (nombre.length > NOMBRE_MAX) return `El nombre no puede superar los ${NOMBRE_MAX} caracteres.`;

        if (!item.tipo) return "Debe seleccionar el tipo.";

        if (item.precio === null || item.precio === undefined) return "El precio es obligatorio.";
        if (item.precio <= 0) return "El precio debe ser mayor a 0.";

        if (item.tipo === "REPUESTO") {
            if (item.stock === null || item.stock === undefined) return "El stock es obligatorio para un repuesto.";
            if (item.stock < 0) return "El stock no puede ser negativo.";
        }

        if (item.descripcion && item.descripcion.length > DESCRIPCION_MAX) {
            return `La descripción no puede superar los ${DESCRIPCION_MAX} caracteres.`;
        }

        return null;
    };

    const errorValidacion = submitted ? validarFormulario() : null;

    const saveOrUpdate = async () => {
        setSubmitted(true);
        if (validarFormulario()) return;

        setGuardando(true);
        try {
            const datosLimpios = {
                ...item,
                nombre: item.nombre.trim(),
                descripcion: item.descripcion?.trim() || null,
                stock: item.tipo === "REPUESTO" ? item.stock : 0,
            };

            const respuesta = item.id
                ? await repuestoServicioService.update(item.id, datosLimpios, archivoImagen)
                : await repuestoServicioService.create(datosLimpios, archivoImagen);

            const guardado = respuesta.repuestoServicio;

            if (item.id) {
                setItems((prev) => {
                    const indice = prev.findIndex((i) => i.id === guardado.id);
                    if (indice === -1) return prev;
                    const copia = [...prev];
                    copia[indice] = guardado;
                    return copia;
                });
            } else {
                setItems((prev) => [guardado, ...prev]);
            }

            mostrarExitoApi(toast, respuesta.message);
            setDialog(false);
        } catch (error) {
            mostrarErrorApi(toast, error, "Ocurrió un error al guardar el registro");
        } finally {
            setGuardando(false);
        }
    };

    const confirmDelete = (rowData) => {
        Swal.fire({
            title: "¿Eliminar registro?",
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
                deleteItem(rowData.id, rowData.nombre);
            }
        });
    };

    const deleteItem = async (id, nombre) => {
        try {
            const respuesta = await repuestoServicioService.delete(id);
            setItems((prev) => prev.filter((i) => i.id !== id));
            mostrarExitoApi(toast, respuesta.message);
        } catch (error) {
            mostrarErrorApi(toast, error, `No se pudo eliminar "${nombre}"`);
        }
    };

    const templateFoto = (rowData) => {
        const url = construirUrlImagen(rowData.foto);
        if (!url) {
            return (
                <div className="w-10 h-10 flex items-center justify-center bg-gray-100 rounded">
                    <i className="pi pi-image text-gray-400" />
                </div>
            );
        }
        return <img src={url} alt={rowData.nombre} className="w-10 h-10 object-cover rounded" />;
    };

    const templateDescripcion = (rowData) => (
      <span className="block truncate" title={rowData.descripcion || ""}>
        {rowData.descripcion || "—"}
      </span>
    );

    const templateTipo = (rowData) => (
        <Tag value={rowData.tipo} severity={rowData.tipo === "REPUESTO" ? "info" : "success"} />
    );

    const templatePrecio = (rowData) =>
        rowData.precio?.toLocaleString("es-SV", { style: "currency", currency: "USD" });

    const templateStock = (rowData) => (rowData.tipo === "REPUESTO" ? rowData.stock : "—");

    const templateAcciones = (rowData) => {
        if (!puedeEditar && !puedeEliminar) return null;

        return (
            <div className="flex gap-2 justify-center md:justify-start">
                {puedeEditar && (
                    <Button icon="pi pi-pencil" rounded outlined severity="success" onClick={() => openEdit(rowData)} />
                )}
                {puedeEliminar && (
                    <Button icon="pi pi-trash" rounded outlined severity="danger" onClick={() => confirmDelete(rowData)} />
                )}
            </div>
        );
    };

    const headerDT = (
        <div className="flex flex-col md:flex-row md:items-center justify-between gap-4 p-1">
            <h4 className="m-0 text-xl font-bold text-gray-700">Repuestos y Servicios</h4>

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
                {puedeEditar && (
                    <Toolbar
                        className="mb-4 bg-gray-50 border-none"
                        start={() => (
                            <Button label="Nuevo Registro" icon="pi pi-plus" severity="primary" onClick={openNew} />
                        )}
                    />
                )}

                <div className="overflow-x-auto">
                    <DataTable
                        value={items}
                        loading={loading}
                        paginator
                        rows={10}
                        rowsPerPageOptions={[5, 10, 25, 50]}
                        header={headerDT}
                        globalFilter={globalFilter}
                        globalFilterFields={["nombre"]}
                        className="p-datatable-sm"
                        emptyMessage="No se encontraron registros."
                    >
                        <Column header="Foto" body={templateFoto} style={{ width: "5rem" }} />
                        <Column field="nombre" header="Nombre" sortable className="font-semibold" />
                        <Column
                            header="Descripción"
                            body={templateDescripcion}
                            style={{ maxWidth: "16rem" }}
                        />
                        <Column header="Tipo" body={templateTipo} sortable field="tipo" />
                        <Column header="Precio" body={templatePrecio} sortable field="precio" />
                        <Column header="Stock" body={templateStock} />
                        {(puedeEditar || puedeEliminar) && (
                            <Column header="Acciones" body={templateAcciones} exportable={false} style={{ minWidth: "8rem" }} />
                        )}
                    </DataTable>
                </div>
            </div>

            <Dialog
                visible={dialog}
                style={{ width: "36rem" }}
                breakpoints={{ "960px": "75vw", "641px": "90vw" }}
                header={item.id ? "Actualizar Registro" : "Registrar Repuesto o Servicio"}
                modal
                className="p-fluid"
                onHide={() => setDialog(false)}
                footer={
                    <div className="flex justify-end gap-2">
                        <Button label="Cancelar" icon="pi pi-times" outlined onClick={() => setDialog(false)} disabled={guardando} />
                        <Button
                            label={item.id ? "Actualizar" : "Guardar"}
                            icon="pi pi-save"
                            onClick={saveOrUpdate}
                            loading={guardando}
                        />
                    </div>
                }
            >
                <div className="field">
                    <label htmlFor="tipo" className="font-bold block mb-2">Tipo</label>
                    <Dropdown
                        id="tipo"
                        value={item.tipo}
                        options={TIPOS}
                        placeholder="Seleccione un tipo"
                        onChange={(e) => manejarCambioTipo(e.value)}
                        className={classNames({ "p-invalid": errorValidacion && !item.tipo })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="nombre" className="font-bold block mb-2">Nombre</label>
                    <InputText
                        id="nombre"
                        value={item.nombre}
                        onChange={(e) => setItem({ ...item, nombre: e.target.value })}
                        maxLength={NOMBRE_MAX}
                        autoFocus
                        className={classNames({ "p-invalid": errorValidacion && !item.nombre?.trim() })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="precio" className="font-bold block mb-2">Precio</label>
                    <InputNumber
                        id="precio"
                        value={item.precio}
                        onValueChange={(e) => setItem({ ...item, precio: e.value })}
                        mode="currency"
                        currency="USD"
                        locale="es-SV"
                        className={classNames({ "p-invalid": errorValidacion && !item.precio })}
                    />
                </div>

                {item.tipo === "REPUESTO" && (
                    <div className="field">
                        <label htmlFor="stock" className="font-bold block mb-2">Stock</label>
                        <InputNumber
                            id="stock"
                            value={item.stock}
                            onValueChange={(e) => setItem({ ...item, stock: e.value })}
                            useGrouping={false}
                            min={0}
                            className={classNames({ "p-invalid": errorValidacion && item.stock === null })}
                        />
                    </div>
                )}

                <div className="field">
                    <label htmlFor="descripcion" className="font-bold block mb-2">Descripción (opcional)</label>
                    <InputTextarea
                        id="descripcion"
                        value={item.descripcion}
                        onChange={(e) => setItem({ ...item, descripcion: e.target.value })}
                        rows={3}
                        maxLength={DESCRIPCION_MAX}
                        autoResize
                    />
                </div>

                {item.tipo === "REPUESTO" && (
                    <div className="field">
                        <label className="font-bold block mb-2">Foto (opcional)</label>

                        {previewUrl && (
                            <div className="mb-3">
                                <img
                                    src={previewUrl}
                                    alt="Vista previa"
                                    className="w-32 h-32 object-cover rounded border border-gray-200"
                                />
                            </div>
                        )}

                        <div className="flex items-center gap-2">
                            <input
                                ref={inputImagenRef}
                                type="file"
                                accept="image/*"
                                onChange={manejarSeleccionImagen}
                                className="text-sm"
                            />
                            {previewUrl && (
                                <Button
                                    icon="pi pi-times"
                                    rounded
                                    outlined
                                    severity="secondary"
                                    type="button"
                                    onClick={limpiarSeleccionImagen}
                                    tooltip="Quitar imagen"
                                />
                            )}
                        </div>
                    </div>
                )}

                {errorValidacion && <small className="p-error block mt-1">{errorValidacion}</small>}
            </Dialog>
        </div>
    );
}
```

## 5.4 Cómo funciona la previsualización, paso a paso

1. El usuario selecciona un archivo con `<input type="file" accept="image/*" onChange={manejarSeleccionImagen} />`.
2. `manejarSeleccionImagen` valida que sea una imagen y que no supere `TAMANIO_MAX_MB` — **antes** de tocar el estado.
3. Se llama `URL.createObjectURL(archivo)`, que genera una URL local (`blob:...`) que el navegador puede mostrar directamente en un `<img>`, sin haber subido nada todavía al servidor. Esa URL se guarda en `previewUrl`.
4. El `<img src={previewUrl}>` muestra la imagen de inmediato — la previsualización ocurre completamente en el navegador.
5. El archivo real (`archivoImagen`, el objeto `File`) se guarda aparte, para enviarlo solo cuando el usuario confirme con "Guardar" o "Actualizar".
6. Al guardar exitosamente, o al cerrar el diálogo, o al desmontar el componente, se llama `URL.revokeObjectURL(previewUrl)` — **debe liberarse esa URL local explícitamente**, porque el navegador no la libera solo; dejar de hacerlo acumula memoria en sesiones largas donde se abren y cierran muchos formularios.

## 5.5 Por qué la imagen y el stock son condicionales al tipo

Un `SERVICIO` no tiene inventario que descontar (el propio backend, al procesar el detalle de una orden de trabajo, solo descuenta stock cuando el tipo es `REPUESTO`) ni una fotografía que mostrar en el catálogo. Por eso:

- El campo `Stock` solo se muestra cuando `item.tipo === "REPUESTO"`; para `SERVICIO`, se envía `stock: 0` de forma automática, sin pedírselo al usuario.
- El bloque completo de imagen (previsualización + `<input type="file">`) solo se muestra para `REPUESTO`.
- `manejarCambioTipo` limpia cualquier imagen ya seleccionada si el usuario cambia de `REPUESTO` a `SERVICIO` a mitad de la edición — evita enviar una imagen para un registro que no debería tenerla.

## 5.6 Conectar la ruta

```jsx
// src/app/Router.jsx
import RepuestosServicios from "../components/catalogos/RepuestosServicios";
// ...
<Route path="/catalogos/repuestos-servicios" element={<RepuestosServicios />} />
```

## 5.7 Verificar que todo funciona

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Crear un `SERVICIO` | No aparece ningún campo de Stock ni de Foto en el formulario |
| 2 | Crear un `REPUESTO` sin imagen | Se guarda correctamente, la columna "Foto" de la tabla muestra el ícono de marcador de posición |
| 3 | Crear un `REPUESTO` y seleccionar una imagen | Aparece la previsualización de inmediato, **antes** de presionar "Guardar" |
| 4 | Seleccionar un archivo que no es una imagen (por ejemplo, un `.pdf`) | Advertencia (`warn`), no se acepta el archivo |
| 5 | Seleccionar una imagen de más de 5 MB | Advertencia (`warn`), no se acepta el archivo |
| 6 | Guardar un `REPUESTO` con imagen | La tabla se actualiza sin recargar, mostrando la foto ya subida en la columna correspondiente |
| 7 | Editar un `REPUESTO` que ya tiene foto | La previsualización muestra la imagen actual (la que ya está en el servidor), no un campo vacío |
| 8 | Editar un `REPUESTO` con foto y cambiar el tipo a `SERVICIO` | El campo de imagen desaparece y la selección se descarta |
| 9 | Eliminar un registro | Confirmación con SweetAlert2, luego actualización local sin nueva consulta al servidor |
| 10 | Probar con un usuario `MECANICO` | No se ve el botón "Nuevo Registro" ni las acciones de editar/eliminar |
