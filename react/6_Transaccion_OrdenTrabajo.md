# 6. CRUD de OrdenTrabajo (maestro-detalle, filtros por fecha y cambios de estados)

Esta es la entidad más compleja del proyecto en el frontend: depende de `Vehiculo` y `Empleado`, gestiona una lista de detalleOrden (repuestos y servicios) dentro del mismo formulario, sigue una máquina de estados que debe respetarse tanto en el backend como en la interfaz, y agrega una vista de solo lectura para consultar una orden sin necesidad de editarla.

## 6.1 Crear un endpoint en el backend: `GET /api/empleados`, con filtro por role

El `Dropdown` de "mecánico asignado" necesita listar solo empleados con role `MECANICO` — no todos los empleados. El role no vive en la entidad `Empleado`, sino en la relación `Usuario → UsuarioRole → Role`, así que hace falta una consulta que cruce esas tablas.

```java
// repository/EmpleadoRepository.java
package com.devsv.autofix_api.repository;

import com.devsv.autofix_api.entities.Empleado;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface EmpleadoRepository extends JpaRepository<Empleado, Integer> {

    // El rol no vive en Empleado - se navega Usuario -> UsuarioRole -> Role.
    // ur.usuario.empleado usa navegación implícita de JPQL (evita declarar
    // joins explícitos entre entidades sin relación directa); se filtra
    // "empleado IS NOT NULL" porque un Usuario también puede estar
    // vinculado a un Cliente en vez de a un Empleado.
    @Query("SELECT ur.usuario.empleado FROM UsuarioRole ur " +
            "WHERE ur.role.nombre = :rolNombre AND ur.usuario.empleado IS NOT NULL")
    List<Empleado> findByRolNombre(@Param("rolNombre") String rolNombre);
}
```

```java
// interfaces/IEmpleadoService.java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.EmpleadoDTO;

import java.util.List;

public interface IEmpleadoService {
    // role es opcional: null o vacío -> todos los empleados;
    // con valor (ej. "MECANICO") -> solo los que tienen ese role asignado.
    List<EmpleadoDTO> findAll(String role);
    EmpleadoDTO findById(Integer id);
}
```

```java
// services/EmpleadoService.java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.EmpleadoDTO;
import com.devsv.autofix_api.entities.Empleado;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IEmpleadoService;
import com.devsv.autofix_api.mappers.EmpleadoMapper;
import com.devsv.autofix_api.repository.EmpleadoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class EmpleadoServiceImpl implements IEmpleadoService {

    private final EmpleadoRepository repository;
    private final EmpleadoMapper mapper;

    @Override
    @Transactional(readOnly = true)
    public List<EmpleadoDTO> findAll(String role) {
        List<Empleado> empleados = (role != null && !role.isBlank())
                ? repository.findByRolNombre(role)
                : repository.findAll();

        return mapper.toDtoList(empleados);
    }

    @Override
    @Transactional(readOnly = true)
    public EmpleadoDTO findById(Integer id) {
        Empleado entity = repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el empleado con ID: " + id));
        return mapper.toDTO(entity);
    }
}
```

```java
// controllers/EmpleadoController.java
package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.dto.EmpleadoDTO;
import com.devsv.autofix_api.interfaces.IEmpleadoService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@CrossOrigin
@RequestMapping("/api")
@RequiredArgsConstructor
public class EmpleadoController {
    private final IEmpleadoService empleadoService;

    // GET /api/empleados              -> todos
    // GET /api/empleados?role=MECANICO -> solo empleados con ese role
    @GetMapping("/empleados")
    public ResponseEntity<List<EmpleadoDTO>> getAll(@RequestParam(required = false) String role) {
        return ResponseEntity.ok(empleadoService.findAll(role));
    }

    @GetMapping("/empleados/{id}")
    public ResponseEntity<EmpleadoDTO> getById(@PathVariable Integer id) {
        return ResponseEntity.ok(empleadoService.findById(id));
    }
}
```

## 6.2 Los servicios del frontend

`createCatalogoService` (la *factory* compartida) acepta ahora parámetros opcionales de consulta en `getAll` — retrocompatible: llamarlo sin argumentos sigue funcionando exactamente igual en `Marca`, `Modelo`, `Vehiculo` y `Cliente`.

```js
// src/services/catalogosService.js
import axiosClient from "./axiosClient";

const createCatalogoService = (endpoint) => ({
  getAll: async (params = {}) => {
    const response = await axiosClient.get(endpoint, { params });
    return response.data;
  },
  getById: async (id) => {
    const response = await axiosClient.get(`${endpoint}/${id}`);
    return response.data;
  },
  create: async (data) => {
    const response = await axiosClient.post(endpoint, data);
    return response.data;
  },
  update: async (id, data) => {
    const response = await axiosClient.put(`${endpoint}/${id}`, data);
    return response.data;
  },
  delete: async (id) => {
    const response = await axiosClient.delete(`${endpoint}/${id}`);
    return response.data;
  },
});

export default createCatalogoService;
```

```js
// src/services/empleadoService.js
import createCatalogoService from "./catalogosService";

export const empleadoService = createCatalogoService("/empleados");
```

```js
// src/services/ordenTrabajoService.js
import axiosClient from "./axiosClient";

const ENDPOINT = "/ordenes-trabajo";

export const ordenTrabajoService = {

    getAll: async (params = {}) => {
        const response = await axiosClient.get(ENDPOINT, {params});
        return response.data;
    },
    getById: async (id) =>{
        const response = await axiosClient.get(`${ENDPOINT}/${id}`);
        return response.data;
    },
    create: async(dto) => {
        const response = await axiosClient.post(ENDPOINT, dto);
        return response.data;
    },
    update: async (id, dto) => {
        const response = await axiosClient.put(`${ENDPOINT}/${id}`, dto);
        return response.data;
    },
    cambiarEstado: async (id, estado) => {
        const response = await axiosClient.patch(`${ENDPOINT}/${id}/estado`, {estado});
        return response.data;
    },
    delete: async (id) => {
        const response = await axiosClient.delete(`${ENDPOINT}/${id}`);
        return response.data;
    }

}
```

## 6.3 El componente completo

```jsx
// src/components/ordenes-trabajo/OrdenesTrabajo.jsx
import { useState, useEffect, useRef } from "react";
import { DataTable } from "primereact/datatable";
import { Column } from "primereact/column";
import { Button } from "primereact/button";
import { Toolbar } from "primereact/toolbar";
import { Dialog } from "primereact/dialog";
import { InputNumber } from "primereact/inputnumber";
import { InputTextarea } from "primereact/inputtextarea";
import { Dropdown } from "primereact/dropdown";
import { Calendar } from "primereact/calendar";
import { Tag } from "primereact/tag";
import { IconField } from "primereact/iconfield";
import { InputIcon } from "primereact/inputicon";
import { InputText } from "primereact/inputtext";
import { classNames } from "primereact/utils";
import { Toast } from "primereact/toast";
import Swal from "sweetalert2";

import { ordenTrabajoService } from "../../services/ordenTrabajoService";
import { vehiculoService } from "../../services/vehiculoService";
import { empleadoService } from "../../services/empleadoService";
import { repuestoServicioService } from "../../services/repuestoServicioService";
import { mostrarErrorApi, mostrarExitoApi } from "../../utils/alertasApi";
import { useAuth } from "../../auth/AuthContext";

const OBSERVACIONES_MAX = 250;

const TRANSICIONES_VALIDAS = {
    PENDIENTE: ["EN_PROCESO", "CANCELADA"],
    EN_PROCESO: ["COMPLETADA", "CANCELADA"],
    COMPLETADA: ["ENTREGADA"],
    ENTREGADA: [],
    CANCELADA: [],
};

const COLORES_ESTADO = {
    PENDIENTE: "secondary",
    EN_PROCESO: "info",
    COMPLETADA: "success",
    ENTREGADA: "contrast",
    CANCELADA: "danger",
};

const emptyOrden = {
    id: null,
    vehiculoId: null,
    mecanicoId: null,
    fecha: new Date(),
    observaciones: "",
    detalleOrden: [],
};


const formatearFechaParaBackend = (fecha) => {
    if (!fecha) return null;
    const year = fecha.getFullYear();
    const month = String(fecha.getMonth() + 1).padStart(2, "0");
    const day = String(fecha.getDate()).padStart(2, "0");
    return `${year}-${month}-${day}`;
};

const parsearFechaDelBackend = (fechaStr) => {
    if (!fechaStr) return null;
    const [year, month, day] = fechaStr.split("-").map(Number);
    return new Date(year, month - 1, day);
};

const formatearMoneda = (valor) =>
    (valor ?? 0).toLocaleString("es-SV", { style: "currency", currency: "USD" });

export default function OrdenesTrabajo() {
    const [ordenes, setOrdenes] = useState([]);
    const [vehiculos, setVehiculos] = useState([]);
    const [empleados, setEmpleados] = useState([]);
    const [repuestosServicios, setRepuestosServicios] = useState([]);

    const [orden, setOrden] = useState(emptyOrden);
    const [detalleSeleccionado, setDetalleSeleccionado] = useState(null);
    const [cantidadDetalle, setCantidadDetalle] = useState(1);

    const [dialog, setDialog] = useState(false);
    const [submitted, setSubmitted] = useState(false);
    const [globalFilter, setGlobalFilter] = useState(null);
    const [rangoFechas, setRangoFechas] = useState(null);
    const [loading, setLoading] = useState(false);
    const [guardando, setGuardando] = useState(false);

    const [dialogEstado, setDialogEstado] = useState(false);
    const [ordenParaEstado, setOrdenParaEstado] = useState(null);
    const [nuevoEstado, setNuevoEstado] = useState(null);

    const [dialogVer, setDialogVer] = useState(false);
    const [ordenVista, setOrdenVista] = useState(null);

    const toast = useRef(null);
    const { tienePermiso } = useAuth();

    const puedeGestionar = tienePermiso(["ADMIN", "RECEPCIONISTA"]);
    const puedeEliminar = tienePermiso(["ADMIN"]);
    const puedeCambiarEstado = tienePermiso(["ADMIN", "RECEPCIONISTA", "MECANICO"]);

    useEffect(() => {
        fetchOrdenes();
        fetchVehiculos();
        fetchEmpleados();
        fetchRepuestosServicios();
    }, []);

    //funciones para obtener las colecciones a necesitar del backend
    const fetchOrdenes = async (params = {}) => {
        setLoading(true);
        try {
            const data = await ordenTrabajoService.getAll(params);
            setOrdenes(data);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener las órdenes de trabajo");
        } finally {
            setLoading(false);
        }
    };

    const fetchVehiculos = async () => {
        try {
            setVehiculos(await vehiculoService.getAll());
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los vehículos");
        }
    };

    const fetchEmpleados = async () => {
        try {
            //filtramos la colección de empleados para obtener solo mecánicos
            setEmpleados(await empleadoService.getAll({ role: "MECANICO" }));
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los mecánicos");
        }
    };

    const fetchRepuestosServicios = async () => {
        try {
            setRepuestosServicios(await repuestoServicioService.getAll());
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los repuestos y servicios");
        }
    };

    // Filtro por rango de fechas (equivale a "fecha exacta" cuando ambas coinciden) 
    const aplicarFiltroFecha = () => {
        if (!rangoFechas || !rangoFechas[0] || !rangoFechas[1]) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "Seleccione un rango de fechas completo" });
            return;
        }
        fetchOrdenes({
            fechaInicio: formatearFechaParaBackend(rangoFechas[0]),
            fechaFin: formatearFechaParaBackend(rangoFechas[1]),
        });
    };

    const limpiarFiltroFecha = () => {
        setRangoFechas(null);
        fetchOrdenes();
    };

    // Opciones de los Dropdown 
    const vehiculoOpciones = vehiculos.map((v) => ({
        ...v,
        etiqueta: [v.placa, v.cliente?.nombre].filter(Boolean).join(" - "),
    }));

    const mecanicoOpciones = empleados.map((e) => ({ ...e, etiqueta: e.nombre }));

    const repuestoServicioOpciones = repuestosServicios.map((r) => ({
        ...r,
        etiqueta: `${r.nombre}${r.tipo === "REPUESTO" ? ` (stock: ${r.stock})` : ""} - ${formatearMoneda(r.precio)}`,
    }));

    const openNew = () => {
        setOrden(emptyOrden);
        setDetalleSeleccionado(null);
        setCantidadDetalle(1);
        setSubmitted(false);
        setDialog(true);
    };

    const openEdit = (rowData) => {
        setOrden({
            id: rowData.id,
            
            vehiculoId: rowData.vehiculo?.id ?? rowData.vehiculoId,
            mecanicoId: rowData.mecanico?.id ?? rowData.mecanicoId ?? null,
            fecha: parsearFechaDelBackend(rowData.fecha),
            observaciones: rowData.observaciones ?? "",
            
            detalleOrden: (rowData.detalleOrden ?? []).map((d) => ({
                repuestoServicioId: d.repuestoServicio?.id ?? d.repuestoServicioId,
                nombre: d.repuestoServicio?.nombre,
                tipo: d.repuestoServicio?.tipo,
                cantidad: d.cantidad,
                precioUnitario: d.precioUnitario,
                subtotal: d.subtotal,
            })),
        });
        setDetalleSeleccionado(null);
        setCantidadDetalle(1);
        setSubmitted(false);
        setDialog(true);
    };

    // Gestión de detalleOrden dentro del formulario 
    const agregarDetalle = () => {
        if (!detalleSeleccionado) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "Seleccione un repuesto o servicio" });
            return;
        }
        if (!cantidadDetalle || cantidadDetalle <= 0) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "La cantidad debe ser mayor a 0" });
            return;
        }

        const itemCatalogo = repuestosServicios.find((r) => r.id === detalleSeleccionado);
        if (!itemCatalogo) return;

        const indiceExistente = orden.detalleOrden.findIndex((d) => d.repuestoServicioId === detalleSeleccionado);
        const cantidadYaAgregada = indiceExistente !== -1 ? orden.detalleOrden[indiceExistente].cantidad : 0;
        const cantidadTotalDeseada = cantidadYaAgregada + cantidadDetalle;

        if (itemCatalogo.tipo === "REPUESTO" && cantidadTotalDeseada > itemCatalogo.stock) {
            toast.current.show({
                severity: "warn",
                summary: "Atención",
                detail: `Solo hay ${itemCatalogo.stock} unidades disponibles de "${itemCatalogo.nombre}"`,
                life: 4000,
            });
            return;
        }

        if (indiceExistente !== -1) {
            setOrden((prev) => {
                const copia = [...prev.detalleOrden];
                copia[indiceExistente] = {
                    ...copia[indiceExistente],
                    cantidad: cantidadTotalDeseada,
                    subtotal: copia[indiceExistente].precioUnitario * cantidadTotalDeseada,
                };
                return { ...prev, detalleOrden: copia };
            });
        } else {
            setOrden((prev) => ({
                ...prev,
                detalleOrden: [
                    ...prev.detalleOrden,
                    {
                        repuestoServicioId: itemCatalogo.id,
                        nombre: itemCatalogo.nombre,
                        tipo: itemCatalogo.tipo,
                        cantidad: cantidadDetalle,
                        precioUnitario: itemCatalogo.precio,
                        subtotal: itemCatalogo.precio * cantidadDetalle,
                    },
                ],
            }));
        }

        setDetalleSeleccionado(null);
        setCantidadDetalle(1);
    };

    const quitarDetalle = (repuestoServicioId) => {
        setOrden((prev) => ({
            ...prev,
            detalleOrden: prev.detalleOrden.filter((d) => d.repuestoServicioId !== repuestoServicioId),
        }));
    };

    const totalEstimado = orden.detalleOrden.reduce((acc, d) => acc + d.subtotal, 0);

    // Validación y guardado 
    const validarFormulario = () => {
        if (!orden.vehiculoId) return "Debe seleccionar el vehículo.";
        if (!orden.fecha) return "La fecha es obligatoria.";
        if (orden.detalleOrden.length === 0) return "Debe agregar al menos un repuesto o servicio.";
        if (orden.observaciones && orden.observaciones.length > OBSERVACIONES_MAX) {
            return `Las observaciones no pueden superar los ${OBSERVACIONES_MAX} caracteres.`;
        }
        return null;
    };

    const errorValidacion = submitted ? validarFormulario() : null;

    const saveOrUpdate = async () => {
        setSubmitted(true);
        if (validarFormulario()) return;

        setGuardando(true);
        try {
            const datosEnvio = {
                vehiculoId: orden.vehiculoId,
                mecanicoId: orden.mecanicoId || null,
                fecha: formatearFechaParaBackend(orden.fecha),
                observaciones: orden.observaciones?.trim() || null,
                // El backend solo necesita repuestoServicioId + cantidad -
                // precioUnitario/subtotal los calcula él mismo con el precio
                // ACTUAL en la base de datos.
                detalleOrden: orden.detalleOrden.map((d) => ({
                    repuestoServicioId: d.repuestoServicioId,
                    cantidad: d.cantidad,
                })),
            };

            const respuesta = orden.id
                ? await ordenTrabajoService.update(orden.id, datosEnvio)
                : await ordenTrabajoService.create(datosEnvio);

            const guardada = respuesta.ordenTrabajo;

            if (orden.id) {
                setOrdenes((prev) => {
                    const indice = prev.findIndex((o) => o.id === guardada.id);
                    if (indice === -1) return prev;
                    const copia = [...prev];
                    copia[indice] = guardada;
                    return copia;
                });
            } else {
                setOrdenes((prev) => [guardada, ...prev]);
            }

            mostrarExitoApi(toast, respuesta.message);
            setDialog(false);
        } catch (error) {
            mostrarErrorApi(toast, error, "Ocurrió un error al guardar la orden");
        } finally {
            setGuardando(false);
        }
    };

    // Cambiar estado 
    const abrirCambioEstado = (rowData) => {
        setOrdenParaEstado(rowData);
        setNuevoEstado(null);
        setDialogEstado(true);
    };

    const abrirVerDetalle = (rowData) => {
        setOrdenVista(rowData);
        setDialogVer(true);
    };

    const opcionesEstadoDisponibles = ordenParaEstado
        ? (TRANSICIONES_VALIDAS[ordenParaEstado.estado] || []).map((e) => ({ label: e, value: e }))
        : [];

    const aplicarCambioEstado = async () => {
        try {
            const respuesta = await ordenTrabajoService.cambiarEstado(ordenParaEstado.id, nuevoEstado);
            const actualizada = respuesta.ordenTrabajo;
            setOrdenes((prev) => {
                const indice = prev.findIndex((o) => o.id === actualizada.id);
                if (indice === -1) return prev;
                const copia = [...prev];
                copia[indice] = actualizada;
                return copia;
            });
            mostrarExitoApi(toast, respuesta.message);
            setDialogEstado(false);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudo cambiar el estado");
        }
    };

    const confirmarCambioEstado = () => {
        if (!nuevoEstado) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "Seleccione el nuevo estado" });
            return;
        }
       
        if (nuevoEstado === "CANCELADA") {
            Swal.fire({
                title: "¿Cancelar esta orden?",
                text: "Esta acción no se puede deshacer y devuelve el stock reservado al inventario.",
                icon: "warning",
                showCancelButton: true,
                confirmButtonText: "Sí, cancelar orden",
                cancelButtonText: "No",
                confirmButtonColor: "#dc2626",
                cancelButtonColor: "#6b7280",
                reverseButtons: true,
            }).then((resultado) => {
                if (resultado.isConfirmed) aplicarCambioEstado();
            });
        } else {
            aplicarCambioEstado();
        }
    };

    const confirmDelete = (rowData) => {
        Swal.fire({
            title: "¿Eliminar orden?",
            html: `Esta acción no se puede deshacer.<br/>Se eliminará la orden <b>${rowData.numero}</b>.`,
            icon: "warning",
            showCancelButton: true,
            confirmButtonText: "Sí, eliminar",
            cancelButtonText: "Cancelar",
            confirmButtonColor: "#dc2626",
            cancelButtonColor: "#6b7280",
            reverseButtons: true,
        }).then((resultado) => {
            if (resultado.isConfirmed) deleteOrden(rowData.id, rowData.numero);
        });
    };

    const deleteOrden = async (id, numero) => {
        try {
            const respuesta = await ordenTrabajoService.delete(id);
            setOrdenes((prev) => prev.filter((o) => o.id !== id));
            mostrarExitoApi(toast, respuesta.message);
        } catch (error) {
            mostrarErrorApi(toast, error, `No se pudo eliminar la orden "${numero}"`);
        }
    };

    // Plantillas para columna 
    const templateVehiculo = (rowData) => <span>{rowData.vehiculo?.placa ?? "—"}</span>;
    const templateCliente = (rowData) => <span>{rowData.vehiculo?.cliente?.nombre ?? "—"}</span>;
    const templateMecanico = (rowData) => <span>{rowData.mecanico?.nombre ?? "Sin asignar"}</span>;
    const templateEstado = (rowData) => <Tag value={rowData.estado} severity={COLORES_ESTADO[rowData.estado]} />;
    const templateTotal = (rowData) => formatearMoneda(rowData.total);

    const templateAcciones = (rowData) => {
        const transicionesDisponibles = TRANSICIONES_VALIDAS[rowData.estado] || [];
        const puedeEditarEstaOrden = puedeGestionar && !["ENTREGADA", "CANCELADA"].includes(rowData.estado);
        const puedeEliminarEstaOrden = puedeEliminar && rowData.estado === "PENDIENTE";
        const puedeCambiarEstadoEstaOrden = puedeCambiarEstado && transicionesDisponibles.length > 0;

        if (!puedeEditarEstaOrden && !puedeEliminarEstaOrden && !puedeCambiarEstadoEstaOrden) return null;

        return (
            <div className="flex gap-2 justify-center md:justify-start">
                <Button
                    icon="pi pi-eye"
                    rounded
                    outlined
                    severity="primary"
                    onClick={() => abrirVerDetalle(rowData)}
                    tooltip="Ver detalle"
                />
                {puedeCambiarEstadoEstaOrden && (
                    <Button
                        icon="pi pi-refresh"
                        rounded
                        outlined
                        severity="info"
                        onClick={() => abrirCambioEstado(rowData)}
                        tooltip="Cambiar estado"
                    />
                )}
                {puedeEditarEstaOrden && (
                    <Button icon="pi pi-pencil" rounded outlined severity="success" onClick={() => openEdit(rowData)} />
                )}
                {puedeEliminarEstaOrden && (
                    <Button icon="pi pi-trash" rounded outlined severity="danger" onClick={() => confirmDelete(rowData)} />
                )}
            </div>
        );
    };

    const headerDT = (
        <div className="flex flex-col gap-3 p-1">
            <div className="flex flex-col md:flex-row md:items-center justify-between gap-4">
                <h4 className="m-0 text-xl font-bold text-gray-700">Órdenes de Trabajo</h4>

                <div className="w-full md:w-72">
                    <IconField iconPosition="left">
                        <InputIcon className="pi pi-search" />
                        <InputText
                            type="search"
                            onInput={(e) => setGlobalFilter(e.target.value)}
                            placeholder="Buscar por número..."
                            className="w-full p-inputtext-sm"
                        />
                    </IconField>
                </div>
            </div>

            <div className="flex flex-wrap items-center gap-2">
                <Calendar
                    value={rangoFechas}
                    onChange={(e) => setRangoFechas(e.value)}
                    selectionMode="range"
                    readOnlyInput
                    placeholder="Filtrar por rango de fechas"
                    dateFormat="dd/mm/yy"
                    className="p-inputtext-sm"
                />
                <Button label="Filtrar" icon="pi pi-filter" size="small" onClick={aplicarFiltroFecha} />
                {rangoFechas && (
                    <Button label="Limpiar" icon="pi pi-filter-slash" size="small" outlined onClick={limpiarFiltroFecha} />
                )}
            </div>
        </div>
    );

    return (
        <div className="p-2 md:p-4">
            <Toast ref={toast} />

            <div className="card shadow-md rounded-xl bg-white">
                {puedeGestionar && (
                    <Toolbar
                        className="mb-4 bg-gray-50 border-none"
                        start={() => (
                            <Button label="Nueva Orden" icon="pi pi-plus" severity="primary" onClick={openNew} />
                        )}
                    />
                )}

                <div className="overflow-x-auto">
                    <DataTable
                        value={ordenes}
                        loading={loading}
                        paginator
                        rows={10}
                        rowsPerPageOptions={[5, 10, 25, 50]}
                        header={headerDT}
                        globalFilter={globalFilter}
                        globalFilterFields={["numero"]}
                        className="p-datatable-sm"
                        emptyMessage="No se encontraron órdenes de trabajo."
                    >
                        <Column field="numero" header="Número" sortable className="font-semibold" />
                        <Column field="fecha" header="Fecha" sortable />
                        <Column header="Vehículo" body={templateVehiculo} />
                        <Column header="Cliente" body={templateCliente} />
                        <Column header="Mecánico" body={templateMecanico} />
                        <Column header="Estado" body={templateEstado} sortable field="estado" />
                        <Column header="Total" body={templateTotal} sortable field="total" />
                        <Column header="Acciones" body={templateAcciones} exportable={false} style={{ minWidth: "10rem" }} />
                    </DataTable>
                </div>
            </div>

            {/* Formulario de creación/edición */}
            <Dialog
                visible={dialog}
                style={{ width: "42rem" }}
                breakpoints={{ "960px": "85vw", "641px": "95vw" }}
                header={orden.id ? "Actualizar Orden de Trabajo" : "Registrar Orden de Trabajo"}
                modal
                className="p-fluid"
                onHide={() => setDialog(false)}
                footer={
                    <div className="flex justify-end gap-2">
                        <Button label="Cancelar" icon="pi pi-times" outlined onClick={() => setDialog(false)} disabled={guardando} />
                        <Button
                            label={orden.id ? "Actualizar" : "Guardar"}
                            icon="pi pi-save"
                            onClick={saveOrUpdate}
                            loading={guardando}
                        />
                    </div>
                }
            >
                <div className="field">
                    <label htmlFor="vehiculo" className="font-bold block mb-2">Vehículo</label>
                    <Dropdown
                        id="vehiculo"
                        value={orden.vehiculoId}
                        options={vehiculoOpciones}
                        optionLabel="etiqueta"
                        optionValue="id"
                        placeholder="Seleccione un vehículo"
                        filter
                        onChange={(e) => setOrden({ ...orden, vehiculoId: e.value })}
                        className={classNames({ "p-invalid": errorValidacion && !orden.vehiculoId })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="mecanico" className="font-bold block mb-2">Mecánico asignado (opcional)</label>
                    <Dropdown
                        id="mecanico"
                        value={orden.mecanicoId}
                        options={mecanicoOpciones}
                        optionLabel="etiqueta"
                        optionValue="id"
                        placeholder="Sin asignar"
                        filter
                        showClear
                        onChange={(e) => setOrden({ ...orden, mecanicoId: e.value })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="fecha" className="font-bold block mb-2">Fecha</label>
                    <Calendar
                        id="fecha"
                        value={orden.fecha}
                        onChange={(e) => setOrden({ ...orden, fecha: e.value })}
                        dateFormat="dd/mm/yy"
                        className={classNames({ "p-invalid": errorValidacion && !orden.fecha })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="observaciones" className="font-bold block mb-2">Observaciones (opcional)</label>
                    <InputTextarea
                        id="observaciones"
                        value={orden.observaciones}
                        onChange={(e) => setOrden({ ...orden, observaciones: e.target.value })}
                        rows={2}
                        maxLength={OBSERVACIONES_MAX}
                        autoResize
                    />
                </div>

                <div className="field">
                    <label className="font-bold block mb-2">Repuestos y Servicios</label>

                    <div className="flex flex-col sm:flex-row gap-2 mb-3">
                      <Dropdown
                          value={detalleSeleccionado}
                          options={repuestoServicioOpciones}
                          optionLabel="etiqueta"
                          optionValue="id"
                          placeholder="Seleccione un repuesto o servicio"
                          filter
                          onChange={(e) => setDetalleSeleccionado(e.value)}
                          className="flex-1"
                      />
                      <input
                          type="number"
                          min={1}
                          value={cantidadDetalle ?? ""}
                          onChange={(e) => setCantidadDetalle(e.target.value ? Number(e.target.value) : null)}
                          placeholder="Cant."
                          className="w-full sm:w-24 text-center border border-gray-300 rounded-md px-2 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
                      />
                      <Button
                          icon="pi pi-plus"
                          label="Agregar"
                          type="button"
                          size="small"
                          onClick={agregarDetalle}
                      />
                  </div>

                    {/* DT para el detalle de la orden */}
                    <DataTable
                        value={orden.detalleOrden}
                        emptyMessage="Todavía no se ha agregado ningún ítem."
                        className="p-datatable-sm mb-2"
                        scrollable
                        scrollHeight="220px"
                    >
                        <Column field="nombre" header="Repuesto / Servicio" />
                        <Column field="cantidad" header="Cantidad" style={{ width: "7rem", textAlign: "center" }} />
                        <Column header="Subtotal" body={(d) => formatearMoneda(d.subtotal)} style={{ width: "9rem" }} />
                        <Column
                            header=""
                            body={(d) => (
                                <Button
                                    icon="pi pi-times"
                                    rounded
                                    text
                                    severity="danger"
                                    type="button"
                                    onClick={() => quitarDetalle(d.repuestoServicioId)}
                                />
                            )}
                            style={{ width: "3rem" }}
                        />
                    </DataTable>

                    {orden.detalleOrden.length > 0 && (
                        <p className="text-right font-bold mt-2">Total estimado: {formatearMoneda(totalEstimado)}</p>
                    )}

                    <p className="text-xs text-gray-400 mt-1">
                        El total final lo calcula el servidor con el precio actual de cada ítem al momento de guardar.
                    </p>
                </div>

                {errorValidacion && <small className="p-error block mt-1">{errorValidacion}</small>}
            </Dialog>

            {/* Cambio de estado */}
            <Dialog
                visible={dialogEstado}
                style={{ width: "26rem" }}
                header={`Cambiar estado - Orden ${ordenParaEstado?.numero ?? ""}`}
                modal
                onHide={() => setDialogEstado(false)}
                footer={
                    <div className="flex justify-end gap-2">
                        <Button label="Cancelar" icon="pi pi-times" outlined onClick={() => setDialogEstado(false)} />
                        <Button label="Confirmar" icon="pi pi-check" onClick={confirmarCambioEstado} />
                    </div>
                }
            >
                <p className="mb-3 text-sm text-gray-600">
                    Estado actual: <Tag value={ordenParaEstado?.estado} severity={COLORES_ESTADO[ordenParaEstado?.estado]} />
                </p>
                <div className="field">
                    <label className="font-bold block mb-2">Nuevo estado</label>
                    <Dropdown
                        value={nuevoEstado}
                        options={opcionesEstadoDisponibles}
                        placeholder="Seleccione el nuevo estado"
                        onChange={(e) => setNuevoEstado(e.value)}
                    />
                </div>
            </Dialog>

            {/* Ver detalle - solo lectura, ningún dato editable aquí */}
            <Dialog
                visible={dialogVer}
                style={{ width: "40rem" }}
                breakpoints={{ "960px": "85vw", "641px": "95vw" }}
                header={`Orden de Trabajo ${ordenVista?.numero ?? ""}`}
                modal
                onHide={() => setDialogVer(false)}
                footer={
                    <div className="flex justify-end">
                        <Button label="Cerrar" icon="pi pi-times" outlined onClick={() => setDialogVer(false)} />
                    </div>
                }
            >
                {ordenVista && (
                    <div className="flex flex-col gap-4">
                        <div className="grid grid-cols-1 sm:grid-cols-2 gap-3 text-sm">
                            <div>
                                <span className="font-bold block text-gray-600">Fecha</span>
                                <span>{ordenVista.fecha}</span>
                            </div>
                            <div>
                                <span className="font-bold block text-gray-600">Estado</span>
                                <Tag value={ordenVista.estado} severity={COLORES_ESTADO[ordenVista.estado]} />
                            </div>
                            <div>
                                <span className="font-bold block text-gray-600">Vehículo</span>
                                <span>
                                    {ordenVista.vehiculo?.placa ?? "—"}
                                    {ordenVista.vehiculo?.modelo &&
                                        ` (${[ordenVista.vehiculo.modelo.marca?.nombre, ordenVista.vehiculo.modelo.nombre]
                                            .filter(Boolean)
                                            .join(" ")})`}
                                </span>
                            </div>
                            <div>
                                <span className="font-bold block text-gray-600">Cliente</span>
                                <span>{ordenVista.vehiculo?.cliente?.nombre ?? "—"}</span>
                            </div>
                            <div>
                                <span className="font-bold block text-gray-600">Mecánico</span>
                                <span>{ordenVista.mecanico?.nombre ?? "Sin asignar"}</span>
                            </div>
                            <div>
                                <span className="font-bold block text-gray-600">Total</span>
                                <span className="font-bold">{formatearMoneda(ordenVista.total)}</span>
                            </div>
                        </div>
 
                        {ordenVista.observaciones && (
                            <div>
                                <span className="font-bold block text-gray-600 text-sm">Observaciones</span>
                                <p className="text-sm">{ordenVista.observaciones}</p>
                            </div>
                        )}
 
                        <div>
                            <span className="font-bold block text-gray-600 text-sm mb-2">Repuestos y Servicios</span>
                            <DataTable
                                value={ordenVista.detalleOrden}
                                className="p-datatable-sm"
                                scrollable
                                scrollHeight="220px"
                            >
                                <Column field="repuestoServicio.nombre" header="Repuesto / Servicio" />
                                <Column field="cantidad" header="Cantidad" style={{ width: "6rem", textAlign: "center" }} />
                                <Column
                                    header="Precio unitario"
                                    body={(d) => formatearMoneda(d.precioUnitario)}
                                    style={{ width: "9rem" }}
                                />
                                <Column header="Subtotal" body={(d) => formatearMoneda(d.subtotal)} style={{ width: "9rem" }} />
                            </DataTable>
                        </div>
                    </div>
                )}
            </Dialog>

        </div>
    );
}
```

## 6.6 Decisiones de diseño que merecen explicación

**El precio y el total mostrados en el formulario de creación/edición son solo un estimado.** Al enviar los detalles, únicamente se manda `{ repuestoServicioId, cantidad }` — nunca `precioUnitario` ni `subtotal`. El servidor calcula esos valores con el precio **actual** del `RepuestoServicio`, nunca confía en un precio que mande el cliente. La vista de "Ver detalle" sí muestra el total real (`ordenVista.total`), porque ese dato viene confirmado directamente del backend, no calculado en el navegador.

**La cantidad usa un `<input type="number">` nativo, no `InputNumber` de PrimeReact.** El componente de PrimeReact, con o sin su widget de botones (`showButtons`), mostraba un ancho impredecible que llegaba a ocultar por completo el número. Un input nativo, estilizado a mano con Tailwind, da control total sobre su tamaño sin depender del layout interno de un componente de terceros.

**Solo `CANCELADA` pide confirmación con SweetAlert2 al cambiar de estado.** Las demás transiciones son avances normales del flujo diario de trabajo; exigir confirmación en cada una interrumpiría innecesariamente el uso cotidiano. `CANCELADA` revierte el stock reservado y, en la práctica, no tiene vuelta atrás.

**"Ver detalle" está disponible para cualquier usuario autenticado, sin restricción de rol** — a diferencia de "Editar", "Eliminar" y "Cambiar estado". Como es una acción de solo lectura, no hay ninguna razón de negocio para limitarla; incluso un `MECANICO` que no puede editar una orden `ENTREGADA` sigue pudiendo consultarla.

**Editar y eliminar se restringen por estado en la interfaz, aunque el backend sea más permisivo.** El backend permite actualizar (`PUT`) una orden en cualquier estado, y solo bloquea la eliminación cuando el estado no es `PENDIENTE`. En la interfaz se decidió ser más conservador: "Editar" se oculta para órdenes `ENTREGADA` o `CANCELADA`, y "Eliminar" solo se ofrece para `PENDIENTE`. Es una decisión de experiencia de usuario, no una regla que exista en el backend.

## 6.7 Conectar la ruta

```jsx
// src/app/Router.jsx
import OrdenesTrabajo from "../components/ordenes-trabajo/OrdenesTrabajo";
// ...
<Route path="/ordenes-trabajo" element={<OrdenesTrabajo />} />
```

## 6.8 Verificar que todo funciona

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Crear una orden con vehículo, un repuesto y un servicio | Se guarda, aparece al inicio de la tabla sin recargar, el total coincide con el estimado (si los precios no cambiaron) |
| 2 | Intentar guardar sin seleccionar vehículo, o sin ningún detalle | Error inline, sin petición al backend |
| 3 | Agregar un repuesto con cantidad mayor al stock mostrado | Advertencia (`warn`) instantánea, sin llamar al backend |
| 4 | Agregar el mismo repuesto dos veces con distinta cantidad | Se suman en una sola línea, no se duplica la fila |
| 5 | Agregar más ítems de los que caben en pantalla | La tabla de detalles gana su propio scroll vertical, sin empujar el resto del formulario |
| 6 | Filtrar por rango de fechas | La tabla se actualiza con las órdenes de ese rango únicamente |
| 7 | Abrir el `Dropdown` de "Mecánico asignado" | Solo aparecen empleados con rol `MECANICO`, no todos los empleados |
| 8 | Cambiar estado de `PENDIENTE` a `EN_PROCESO` | Se aplica de inmediato, sin SweetAlert2, solo el Toast de éxito |
| 9 | Cambiar estado a `CANCELADA` | Aparece la confirmación de SweetAlert2 antes de ejecutar el cambio |
| 10 | Hacer clic en el ícono de ojo de cualquier orden | Se abre el detalle de solo lectura, con la tabla de ítems y el total real |
| 11 | Ver el detalle con un usuario `MECANICO` en una orden `ENTREGADA` | El botón de ver detalle sigue disponible, aunque no haya ni editar ni cambiar estado |
| 12 | Ver una orden que no está `PENDIENTE` | No aparece el botón de eliminar |
