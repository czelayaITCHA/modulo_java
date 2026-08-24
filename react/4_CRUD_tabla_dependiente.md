# 4. CRUD de Vehículo (entidad dependiente de Modelo y Cliente)

En esta guía se construye el CRUD de `Vehiculo`, esta entidad del frontend depende de otras dos (`Modelo` y `Cliente`) al mismo tiempo. Por esa razón, el formulario necesita dos listas desplegables cargadas previamente desde el backend, y se aprovecha para introducir dos mejoras que aplican a partir de aquí en todos los CRUD del proyecto: alertas diferenciadas según el código de estado HTTP, y control de qué acciones se muestran según el rol del usuario autenticado.

## 4.1 Pieza faltante en el backend: el endpoint de Cliente

El formulario de `Vehiculo` necesita listar los clientes existentes para llenar su lista desplegable, pero todavía no existía ningún endpoint público para eso (`ClienteDTO`, `ClienteMapper` y `ClienteRepository` ya existían como piezas de apoyo internas, pero sin un `Controller` que las expusiera). Se completa con el mismo patrón ya usado en `Marca`:

```java
package com.devsv.autofix_api.interfaces;

import com.devsv.autofix_api.dto.ClienteDTO;

import java.util.List;

public interface IClienteService {
    List<ClienteDTO> findAll();
    ClienteDTO findById(Integer id);
    ClienteDTO saveOrUpdate(ClienteDTO dto);
    void delete(Integer id);
}
```

```java
// services/ClienteService.java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.ClienteDTO;
import com.devsv.autofix_api.entities.Cliente;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ConflictException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IClienteService;
import com.devsv.autofix_api.mappers.ClienteMapper;
import com.devsv.autofix_api.repository.ClienteRepository;
import com.devsv.autofix_api.repository.VehiculoRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ClienteService implements IClienteService {

    private final ClienteRepository repository;
    private final VehiculoRepository vehiculoRepository;
    private final ClienteMapper mapper;

    @Override
    @Transactional(readOnly = true)
    public List<ClienteDTO> findAll() {
        return mapper.toDtoList(repository.findAll());
    }

    @Override
    @Transactional(readOnly = true)
    public ClienteDTO findById(Integer id) {
        return mapper.toDTO(buscarEntidad(id));
    }

    @Override
    @Transactional
    public ClienteDTO saveOrUpdate(ClienteDTO dto) {
        if(dto.getNombre() == null || dto.getNombre().isBlank()){
            throw new BadRequestException("El nombre es obligatorio");
        }
        if(dto.getTelefono() == null || dto.getTelefono().isBlank()){
            throw new BadRequestException("El teléfono es obligatorio");
        }
        //creamos instancia si es nuevo u obtenemos el cliente ya registrado
        Cliente entity = dto.getId() == null ? new Cliente() : buscarEntidad(dto.getId());
        //seteamos los nuevos datos
        entity.setNombre(dto.getNombre());
        entity.setTelefono(dto.getTelefono());
        entity.setEmail(dto.getEmail());
        return mapper.toDTO(repository.save(entity));
    }

    @Override
    @Transactional
    public void delete(Integer id) {
        Cliente entity = buscarEntidad(id);
        if(vehiculoRepository.existsByCliente(id)){
            throw new ConflictException("No se puede eliminar el cliente: " +
                    entity.getNombre() + ", porque ya tiene vehículos registrados");
        }
        repository.delete(entity);
    }

    private Cliente buscarEntidad(Integer id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No existe el cliente con ID: " + id));
    }
}
```

```java
// controllers/ClienteController.java
package com.devsv.autofix_api.controllers;


import com.devsv.autofix_api.dto.ClienteDTO;
import com.devsv.autofix_api.interfaces.IClienteService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@CrossOrigin
@RequestMapping("/api")
@RequiredArgsConstructor
public class ClienteController {
    private final IClienteService clienteService;

    @GetMapping("/clientes")
    public ResponseEntity<List<ClienteDTO>> getAll(){
        return ResponseEntity.ok(clienteService.findAll());
    }

    @GetMapping("/clientes/{id}")
    public ResponseEntity<ClienteDTO> getById(@PathVariable Integer id){
        return ResponseEntity.ok(clienteService.findById(id));
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'RECEPCIONISTA', 'CLIENTE')")
    @PostMapping("/clientes")
    public ResponseEntity<?> create(@RequestBody ClienteDTO dto) {
        Map<String, Object> response = new HashMap<>();

        ClienteDTO guardado = clienteService.saveOrUpdate(dto);

        response.put("message", "Cliente registrado correctamente...!");
        response.put("cliente", guardado);

        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'RECEPCIONISTA', 'CLIENTE')")
    @PutMapping("/clientes/{id}")
    public ResponseEntity<?> update(@PathVariable Integer id, @RequestBody ClienteDTO dto) {
        Map<String, Object> response = new HashMap<>();

        dto.setId(id);
        ClienteDTO actualizado = clienteService.saveOrUpdate(dto);

        response.put("message", "Cliente actualizado correctamente");
        response.put("cliente", actualizado);

        return new ResponseEntity<>(response, HttpStatus.OK);
    }

    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/clientes/{id}")
    public ResponseEntity<?> delete(@PathVariable Integer id) {
        clienteService.delete(id);

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Cliente eliminado con éxito");

        return new ResponseEntity<>(response, HttpStatus.OK);
    }
}
```

También se agrega el método que faltaba en `VehiculoRepository`, necesario para que `ClienteService` pueda bloquear la eliminación de un cliente con vehículos registrados:

```java
// VehiculoRepository.java - método nuevo
boolean existsByCliente(Integer clienteId);
```
Correr y probar los endpoints de clientes

## 4.2 Alertas diferenciadas según el código de estado HTTP

Hasta ahora, cualquier error de la API se mostraba siempre con severidad `"error"` en el Toast, sin distinguir su naturaleza. A partir de esta guía, se introduce un criterio único, reutilizable en cualquier componente:

```js
// src/utils/alertasApi.js

const obtenerSeveridadPorStatus = (status) =>{
   if(status === 400 || status === 403 || status === 404 || status === 409) {
       return "warn";            
   }
   return "error";
};

export const mostrarErrorApi = 
    (toastRef, error, msjPorDefecto="Ocurrió un error inesperado") => {
    
    const status = error.response?.status;
    const message = error.response?.data?.message || msjPorDefecto;
    const severity = obtenerSeveridadPorStatus(status);
    
    //construimos el mensaje
    toastRef.current.show(
        {
            severity,
            sumary: severity === "warn" ? "Atención" : "Error",
            detail: message,
            life: severity === "warn" ? 4000 : 5000,
        });

};

export const mostrarExitoApi = (toastRef, msj) => {
    toastRef.current.show({
        severity: "success", sumary:"Éxito", detail: msj, life: 3000
    });
}
```

Con este criterio, una placa duplicada (`409`) se muestra como una advertencia amarilla, no como un error rojo — el usuario puede corregirlo cambiando el dato, no es una falla del sistema. Solo un `500` (o cualquier código no contemplado) se sigue mostrando como error real.

> Nota: esta misma utilidad puede aplicarse también al CRUD de `Marca` construido en la guía anterior, para que todo el proyecto use el mismo criterio de manera consistente.

## 4.3 Servicios del frontend

```js
// src/services/vehiculoService.js
import createCatalogoService from "./catalogosService";

export const vehiculoService = createCatalogoService("/vehiculos");
```

```js
// src/services/modeloService.js
import createCatalogoService from "./catalogosService";

export const modeloService = createCatalogoService("/modelos");
```

```js
// src/services/clienteService.js
import createCatalogoService from "./catalogosService";

export const clienteService = createCatalogoService("/clientes");
```

Los tres reutilizan la misma factory (catalogoService.js) de la guía anterior — ninguno necesita lógica propia, porque los tres endpoints siguen exactamente el mismo contrato REST.

## 4.4 El componente `Vehiculos`

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
import { IconField } from "primereact/iconfield";
import { InputIcon } from "primereact/inputicon";
import { classNames } from "primereact/utils";
import { Toast } from "primereact/toast";
import Swal from "sweetalert2";

import { vehiculoService } from "../../services/vehiculoService";
import { modeloService } from "../../services/modeloService";
import { clienteService } from "../../services/clienteService";
import { mostrarErrorApi, mostrarExitoApi } from "../../utils/alertasApi";
import { useAuth } from "../../auth/AuthContext";

const emptyVehiculo = { id: null, placa: "", anio: null, caracteristicas: "", modeloId: null, clienteId: null };
const PLACA_MIN = 4;
const PLACA_MAX = 20;
const CARACTERISTICAS_MAX = 200;
const ANIO_MINIMO = 1980;

export default function Vehiculos() {
    const [vehiculos, setVehiculos] = useState([]);
    const [modelos, setModelos] = useState([]);
    const [clientes, setClientes] = useState([]);
    const [vehiculo, setVehiculo] = useState(emptyVehiculo);
    const [dialog, setDialog] = useState(false);
    const [submitted, setSubmitted] = useState(false);
    const [globalFilter, setGlobalFilter] = useState(null);
    const [loading, setLoading] = useState(false);
    const [guardando, setGuardando] = useState(false);

    const toast = useRef(null);
    const { tienePermiso } = useAuth();

    const puedeEditar = tienePermiso(["ADMIN", "RECEPCIONISTA"]);
    const puedeEliminar = tienePermiso(["ADMIN"]);

    useEffect(() => {
        fetchVehiculos();
        fetchModelos();
        fetchClientes();
    }, []);

    const fetchVehiculos = async () => {
        setLoading(true);
        try {
            const data = await vehiculoService.getAll();
            setVehiculos(data);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los vehículos");
        } finally {
            setLoading(false);
        }
    };

    const fetchModelos = async () => {
        try {
            const data = await modeloService.getAll();
            setModelos(data);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los modelos");
        }
    };

    const fetchClientes = async () => {
        try {
            const data = await clienteService.getAll();
            setClientes(data);
        } catch (error) {
            mostrarErrorApi(toast, error, "No se pudieron obtener los clientes");
        }
    };

    const modelosOpciones = modelos.map((m) => ({
        ...m,
        etiqueta: [m.marca?.nombre, m.nombre].filter(Boolean).join(" "),
    }));

    const clientesOpciones = clientes.map((c) => ({
        ...c,
        etiqueta: [c.nombre, c.telefono].filter(Boolean).join(" - "),
    }));

    const openNew = () => {
        setVehiculo(emptyVehiculo);
        setSubmitted(false);
        setDialog(true);
    };

    const openEdit = (rowData) => {
        setVehiculo({
            id: rowData.id,
            placa: rowData.placa,
            anio: rowData.anio,
            caracteristicas: rowData.caracteristicas ?? "",
            modeloId: rowData.modeloId,
            clienteId: rowData.clienteId,
        });
        setSubmitted(false);
        setDialog(true);
    };

    const validarFormulario = () => {
        const placa = vehiculo.placa?.trim() ?? "";
        if (!placa) return "La placa es obligatoria.";
        if (placa.length < PLACA_MIN) return `La placa debe tener al menos ${PLACA_MIN} caracteres.`;
        if (placa.length > PLACA_MAX) return `La placa no puede superar los ${PLACA_MAX} caracteres.`;

        const anioActual = new Date().getFullYear();
        if (!vehiculo.anio) return "El año es obligatorio.";
        if (vehiculo.anio < ANIO_MINIMO || vehiculo.anio > anioActual + 1) {
            return `El año debe estar entre ${ANIO_MINIMO} y ${anioActual + 1}.`;
        }

        if (!vehiculo.modeloId) return "Debe seleccionar el modelo.";
        if (!vehiculo.clienteId) return "Debe seleccionar el cliente.";

        if (vehiculo.caracteristicas && vehiculo.caracteristicas.length > CARACTERISTICAS_MAX) {
            return `Las características no pueden superar los ${CARACTERISTICAS_MAX} caracteres.`;
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
                ...vehiculo,
                placa: vehiculo.placa.trim().toUpperCase(),
                caracteristicas: vehiculo.caracteristicas?.trim() || null,
            };

            const respuesta = vehiculo.id
                ? await vehiculoService.update(vehiculo.id, datosLimpios)
                : await vehiculoService.create(datosLimpios);

            
            const vehiculoGuardado = respuesta.vehiculo;

            if (vehiculo.id) {
                setVehiculos((prev) => {
                    const indice = prev.findIndex((v) => v.id === vehiculoGuardado.id);
                    if (indice === -1) return prev; 
                    const copia = [...prev];
                    copia[indice] = vehiculoGuardado;
                    return copia;
                });
            } else {
                setVehiculos((prev) => [vehiculoGuardado, ...prev]);
            }

            mostrarExitoApi(toast, respuesta.message);
            setDialog(false);
        } catch (error) {
            mostrarErrorApi(toast, error, "Ocurrió un error al guardar el vehículo");
        } finally {
            setGuardando(false);
        }
    };

    const confirmDelete = (rowData) => {
        Swal.fire({
            title: "¿Eliminar vehículo?",
            html: `Esta acción no se puede deshacer.<br/>Se eliminará el vehículo con placa <b>${rowData.placa}</b>.`,
            icon: "question",
            showCancelButton: true,
            confirmButtonText: "Sí, eliminar",
            cancelButtonText: "Cancelar",
            confirmButtonColor: "#dc2626",
            cancelButtonColor: "#6b7280",
            reverseButtons: true,
        }).then((resultado) => {
            if (resultado.isConfirmed) {
                deleteVehiculo(rowData.id, rowData.placa);
            }
        });
    };

    const deleteVehiculo = async (id, placa) => {
        try {
            const respuesta = await vehiculoService.delete(id);
            setVehiculos((prev) => prev.filter((v) => v.id !== id));
            mostrarExitoApi(toast, respuesta.message);
        } catch (error) {
            mostrarErrorApi(toast, error, `No se pudo eliminar el vehículo con placa "${placa}"`);
        }
    };

    const templateModelo = (rowData) => {
        const marcaNombre = rowData.modelo?.marca?.nombre ?? "";
        const modeloNombre = rowData.modelo?.nombre ?? "";
        return <span>{[marcaNombre, modeloNombre].filter(Boolean).join(" ") || "—"}</span>;
    };

    const templateCliente = (rowData) => <span>{rowData.cliente?.nombre ?? "—"}</span>;

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

    const header = (
        <div className="flex flex-col md:flex-row md:items-center justify-between gap-4 p-1">
            <h4 className="m-0 text-xl font-bold text-gray-700">Catálogo de Vehículos Registrados</h4>

            <div className="w-full md:w-72">
                <IconField iconPosition="left">
                    <InputIcon className="pi pi-search" />
                    <InputText
                        type="search"
                        onInput={(e) => setGlobalFilter(e.target.value)}
                        placeholder="Buscar por placa..."
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
                            <Button label="Nuevo Vehículo" icon="pi pi-plus" severity="primary" onClick={openNew} />
                        )}
                    />
                )}

                {/* responsiveLayout="stack" quedó deprecado desde PrimeReact 9.2
                    y fue removido en la versión 10 - en su lugar, se envuelve
                    la tabla en un contenedor con scroll horizontal. */}
                <div className="overflow-x-auto">
                    <DataTable
                        value={vehiculos}
                        loading={loading}
                        paginator
                        rows={10}
                        rowsPerPageOptions={[5, 10, 25, 50]}
                        header={header}
                        globalFilter={globalFilter}
                        globalFilterFields={["placa"]}
                        className="p-datatable-sm"
                        emptyMessage="No se encontraron vehículos."
                    >
                        <Column field="placa" header="Placa" sortable className="font-semibold" />
                        <Column field="anio" header="Año" sortable />
                        <Column header="Modelo" body={templateModelo} />
                        <Column header="Cliente" body={templateCliente} />
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
                header={vehiculo.id ? "Actualizar Vehículo" : "Registrar Vehículo"}
                modal
                className="p-fluid"
                onHide={() => setDialog(false)}
                footer={
                    <div className="flex justify-end gap-2">
                        <Button label="Cancelar" icon="pi pi-times" outlined onClick={() => setDialog(false)} disabled={guardando} />
                        <Button
                            label={vehiculo.id ? "Actualizar" : "Guardar"}
                            icon="pi pi-save"
                            onClick={saveOrUpdate}
                            loading={guardando}
                        />
                    </div>
                }
            >
                <div className="field">
                    <label htmlFor="placa" className="font-bold block mb-2">Placa</label>
                    <InputText
                        id="placa"
                        value={vehiculo.placa}
                        onChange={(e) => setVehiculo({ ...vehiculo, placa: e.target.value })}
                        maxLength={PLACA_MAX}
                        autoFocus
                        className={classNames({ "p-invalid": errorValidacion && !vehiculo.placa?.trim() })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="anio" className="font-bold block mb-2">Año</label>
                    <InputNumber
                        id="anio"
                        value={vehiculo.anio}
                        onValueChange={(e) => setVehiculo({ ...vehiculo, anio: e.value })}
                        useGrouping={false}
                        className={classNames({ "p-invalid": errorValidacion && !vehiculo.anio })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="modelo" className="font-bold block mb-2">Modelo</label>
                    <Dropdown
                        id="modelo"
                        value={vehiculo.modeloId}
                        options={modelosOpciones}
                        optionLabel="etiqueta"
                        optionValue="id"
                        placeholder="Seleccione un modelo"
                        filter
                        onChange={(e) => setVehiculo({ ...vehiculo, modeloId: e.value })}
                        className={classNames({ "p-invalid": errorValidacion && !vehiculo.modeloId })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="cliente" className="font-bold block mb-2">Cliente</label>
                    <Dropdown
                        id="cliente"
                        value={vehiculo.clienteId}
                        options={clientesOpciones}
                        optionLabel="etiqueta"
                        optionValue="id"
                        placeholder="Seleccione un cliente"
                        filter
                        onChange={(e) => setVehiculo({ ...vehiculo, clienteId: e.value })}
                        className={classNames({ "p-invalid": errorValidacion && !vehiculo.clienteId })}
                    />
                </div>

                <div className="field">
                    <label htmlFor="caracteristicas" className="font-bold block mb-2">Características (opcional)</label>
                    <InputTextarea
                        id="caracteristicas"
                        value={vehiculo.caracteristicas}
                        onChange={(e) => setVehiculo({ ...vehiculo, caracteristicas: e.target.value })}
                        rows={3}
                        maxLength={CARACTERISTICAS_MAX}
                        autoResize
                    />
                </div>

                {errorValidacion && <small className="p-error block mt-1">{errorValidacion}</small>}
            </Dialog>
        </div>
    );
}
```

## 4.5 Validaciones aplicadas

| Campo | Regla | Origen |
|---|---|---|
| Placa | Obligatoria, entre 4 y 20 caracteres, se convierte a mayúsculas antes de enviarse | Solo frontend (el backend valida unicidad, no formato ni longitud) |
| Año | Obligatorio, entre 1950 y el año actual más uno | Solo frontend (columna del backend acepta cualquier entero) |
| Modelo | Obligatorio (selección de la lista) | Frontend y backend (`BadRequestException` si falta) |
| Cliente | Obligatorio (selección de la lista) | Frontend y backend (`BadRequestException` si falta) |
| Características | Opcional, máximo 200 caracteres | Coincide con la longitud real de la columna en la base de datos |
| Placa duplicada | — | Solo el backend la detecta (`ConflictException`, 409); el frontend no tiene forma de saberlo sin consultar, así que no se duplica esa validación aquí |

Debe tenerse presente que la validación de placa duplicada no se replica en el frontend como sí se hizo con el nombre de `Marca` — en ese caso, la lista completa de marcas ya estaba cargada en memoria; aquí, comparar contra cientos de vehículos cargados solo para esa validación no sería una mejora real de experiencia, y se prefiere dejar que el backend responda con el mensaje correspondiente.

## 4.6 Autorización en la interfaz, según el rol

```js
const puedeEditar = tienePermiso(["ADMIN", "RECEPCIONISTA"]);
const puedeEliminar = tienePermiso(["ADMIN"]);
```

Estas dos variables controlan qué se muestra en pantalla: el botón "Nuevo Vehículo", el botón de editar, y el botón de eliminar, cada uno visible solo si el usuario autenticado tiene el rol correspondiente — exactamente los mismos roles que ya exige `@PreAuthorize` en `VehiculoController`. Debe recordarse que esto es únicamente una mejora de experiencia (evita que alguien vea un botón que de todas formas no podría usar); la validación real sigue viviendo en el backend, y ningún cambio en el frontend puede saltársela.

## 4.7 Conectar la ruta y el menú

```jsx
// src/app/Router.jsx
import Vehiculos from "../components/vehiculos/Vehiculos";
// ...
<Route path="/vehiculos" element={<Vehiculos />} />
```

```js
// src/layout/Sidebar.jsx - dentro de opcionesMenu
{ etiqueta: "Vehículos", icono: "pi pi-car", ruta: "/vehiculos" },
```

Se ubica entre "Clientes" y "Gestión de Órdenes", siguiendo el orden natural del flujo de negocio: primero existe el cliente, luego su vehículo, y después la orden de trabajo sobre ese vehículo.

## 4.8 Verificar que todo funciona

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Cargar la pantalla con un usuario `ADMIN` o `RECEPCIONISTA` | Se ven los botones "Nuevo Vehículo", editar y eliminar |
| 2 | Cargar la pantalla con un usuario `MECANICO` | No se ve ningún botón de acción, solo la tabla de solo lectura |
| 3 | Crear un vehículo válido | El diálogo se cierra, aparece un Toast verde, la tabla se actualiza |
| 4 | Crear con placa vacía o muy corta | Error inline, sin petición al backend |
| 5 | Crear con año fuera de rango (por ejemplo, 1800) | Error inline, sin petición al backend |
| 6 | Crear con una placa que ya existe | Toast **amarillo** ("Atención"), no rojo, con el mensaje exacto del backend |
| 7 | Editar un vehículo | El diálogo se abre con los datos actuales, incluyendo el modelo y el cliente ya seleccionados |
| 8 | Eliminar un vehículo sin órdenes de trabajo asociadas | Confirmación de SweetAlert2, luego Toast verde |
| 9 | Eliminar un cliente que ya tiene vehículos registrados | Toast amarillo con el mensaje de conflicto, no rojo |
| 10 | Buscar por placa | Filtra la tabla en vivo |
