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
