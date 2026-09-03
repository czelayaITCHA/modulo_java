# 8. Panel General (Dashboard) en la ruta "/"

Esta guía reemplaza el marcador `<div>Inicio</div>` de la ruta raíz por un panel real: tarjetas de indicadores, dos gráficos (Chart.js), listas de apoyo, y restricción de contenido según el rol del usuario autenticado.

## 8.1 Instalar las dependencias de gráficos

```bash
npm install chart.js react-chartjs-2
```

Ya se instalaron desde la guía 1 ("chart.js / react-chartjs-2 para el Dashboard - los instalamos más adelante, cuando lleguemos a esa parte") — este es el momento. Sino revisa el archivo package.json

## 8.2 Decisión de diseño: cálculo en el frontend, sin endpoint nuevo de backend

Los indicadores (órdenes activas(con estado PENDIENTE + EN_PROCESO), ingresos, stock bajo, etc.) se calculan **en el navegador**, a partir de los servicios que ya existen y están probados (`ordenTrabajoService`, `vehiculoService`, `clienteService`, `repuestoServicioService`) — no se construyó un endpoint de backend que agregue estas estadísticas del lado del servidor.

**Por qué esta elección:** para el volumen de datos de un proyecto de este tamaño, traer las listas completas y calcular en JavaScript es simple, confiable, y reutiliza infraestructura ya construida y verificada. Si el negocio creciera mucho (miles de órdenes), traer listas completas al navegador solo para calcular unos totales dejaría de ser eficiente, y ahí sí valdría la pena mover estos cálculos a un endpoint de resumen en el backend (por ejemplo, `GET /api/dashboard/resumen`) que devuelva los números ya calculados. Se documenta aquí como una mejora futura conocida, no como algo pendiente de resolver ahora.

## 8.3 Registro de Chart.js

Chart.js 4 exige registrar explícitamente qué tipos de elementos se van a usar **antes** de renderizar cualquier gráfico:

```jsx
import {
    Chart as ChartJS,
    ArcElement,
    Tooltip,
    Legend,
    CategoryScale,
    LinearScale,
    PointElement,
    LineElement,
    Filler,
} from "chart.js";

ChartJS.register(ArcElement, Tooltip, Legend, CategoryScale, LinearScale, PointElement, LineElement, Filler);
```

Sin este registro, los gráficos simplemente no aparecen — **sin ningún error visible en consola**, lo cual lo hace fácil de pasar por alto la primera vez. `ArcElement` es necesario para el gráfico de dona (`Doughnut`) o pastel; `CategoryScale`, `LinearScale`, `PointElement`, `LineElement` y `Filler` son necesarios para el gráfico de línea (`Line`).

## 8.4 El componente completo

```jsx
// src/components/Dashboard.jsx
import { useState, useEffect } from "react";
import { Doughnut, Line } from "react-chartjs-2";
import {
    Chart as ChartJS,
    ArcElement,
    Tooltip,
    Legend,
    CategoryScale,
    LinearScale,
    PointElement,
    LineElement,
    Filler,
} from "chart.js";

import { ordenTrabajoService } from "../services/ordenTrabajoService";
import { vehiculoService } from "../services/vehiculoService";
import { clienteService } from "../services/clienteService";
import { repuestoServicioService } from "../services/repuestoServicioService";
import { useAuth } from "../auth/AuthContext";


ChartJS.register(ArcElement, Tooltip, Legend, CategoryScale, LinearScale, PointElement, LineElement, Filler);

const ETIQUETAS_ESTADO = {
    PENDIENTE: "Pendiente",
    EN_PROCESO: "En Proceso",
    COMPLETADA: "Completada",
    ENTREGADA: "Entregada",
    CANCELADA: "Cancelada",
};

const COLORES_ESTADO = {
    PENDIENTE: "#94a3b8",
    EN_PROCESO: "#3b82f6",
    COMPLETADA: "#22c55e",
    ENTREGADA: "#0ea5e9",
    CANCELADA: "#ef4444",
};

const STOCK_BAJO_UMBRAL = 5;

const formatearMoneda = (valor) =>
    (valor ?? 0).toLocaleString("es-SV", { style: "currency", currency: "USD" });

export default function Dashboard() {
    const [ordenes, setOrdenes] = useState([]);
    const [vehiculos, setVehiculos] = useState([]);
    const [clientes, setClientes] = useState([]);
    const [repuestosServicios, setRepuestosServicios] = useState([]);
    const [cargando, setCargando] = useState(true);

    const { usuario } = useAuth();
    // MECANICO solo ve lo operativo (órdenes activas, vehículos, y el gráfico de estado).
    const esMecanico = usuario?.rol === "MECANICO";

    useEffect(() => {
        cargarDatos();
        
    }, []);

    const cargarDatos = async () => {
        setCargando(true);
        try {
            const promesas = [ordenTrabajoService.getAll(), vehiculoService.getAll()];
            if (!esMecanico) {
                promesas.push(clienteService.getAll(), repuestoServicioService.getAll());
            }

            const resultados = await Promise.all(promesas);
            setOrdenes(resultados[0]);
            setVehiculos(resultados[1]);
            if (!esMecanico) {
                setClientes(resultados[2]);
                setRepuestosServicios(resultados[3]);
            }
        } catch {
            // Si alguna fuente falla, las tarjetas simplemente muestran 0.
        } finally {
            setCargando(false);
        }
    };

    // Cálculos derivados
    const ordenesPorEstado = Object.keys(ETIQUETAS_ESTADO).reduce((acumulador, estado) => {
        acumulador[estado] = ordenes.filter((o) => o.estado === estado).length;
        return acumulador;
    }, {});

    const ordenesActivas = (ordenesPorEstado.PENDIENTE ?? 0) + (ordenesPorEstado.EN_PROCESO ?? 0);

    const ingresosTotales = ordenes
        .filter((o) => o.estado !== "CANCELADA")
        .reduce((acumulador, o) => acumulador + (o.total ?? 0), 0);

    const repuestosStockBajo = repuestosServicios.filter(
        (r) => r.tipo === "REPUESTO" && r.stock !== null && r.stock <= STOCK_BAJO_UMBRAL
    );

    const ordenesRecientes = [...ordenes].sort((a, b) => new Date(b.fecha) - new Date(a.fecha)).slice(0, 5);

    const ingresosPorMes = (() => {
        const meses = [];
        const hoy = new Date();
        for (let i = 5; i >= 0; i--) {
            const fecha = new Date(hoy.getFullYear(), hoy.getMonth() - i, 1);
            meses.push({ anio: fecha.getFullYear(), mes: fecha.getMonth() });
        }

        return meses.map(({ anio, mes }) => {
            const total = ordenes
                .filter((o) => o.estado !== "CANCELADA")
                .filter((o) => {
                    const [ordenAnio, ordenMes] = o.fecha.split("-").map(Number);
                    return ordenAnio === anio && ordenMes - 1 === mes;
                })
                .reduce((acumulador, o) => acumulador + (o.total ?? 0), 0);

            const nombreMes = new Date(anio, mes, 1).toLocaleDateString("es-SV", { month: "short" });
            return { etiqueta: `${nombreMes} ${anio}`, total };
        });
    })();

    const datosDona = {
        labels: Object.keys(ETIQUETAS_ESTADO).map((e) => ETIQUETAS_ESTADO[e]),
        datasets: [
            {
                data: Object.keys(ETIQUETAS_ESTADO).map((e) => ordenesPorEstado[e] ?? 0),
                backgroundColor: Object.keys(ETIQUETAS_ESTADO).map((e) => COLORES_ESTADO[e]),
                borderWidth: 0,
            },
        ],
    };

    const datosLinea = {
        labels: ingresosPorMes.map((m) => m.etiqueta),
        datasets: [
            {
                label: "Ingresos",
                data: ingresosPorMes.map((m) => m.total),
                borderColor: "#2563eb",
                backgroundColor: "rgba(37, 99, 235, 0.1)",
                fill: true,
                tension: 0.35,
            },
        ],
    };

    const opcionesLinea = {
        responsive: true,
        maintainAspectRatio: false,
        plugins: { legend: { display: false } },
        scales: {
            y: { ticks: { callback: (valor) => `$${valor}` } },
        },
    };

    // Cuadrículas dinámicas: con menos tarjetas/paneles (caso MECANICO),
    // se usan menos columnas para que no queden espacios vacíos a la derecha.
    const claseCuadriculaKpi = esMecanico
        ? "grid grid-cols-1 sm:grid-cols-2 gap-4 mb-6"
        : "grid grid-cols-2 lg:grid-cols-4 gap-4 mb-6";
    const claseCuadriculaGraficos = esMecanico
        ? "grid grid-cols-1 gap-4 mb-6"
        : "grid grid-cols-1 lg:grid-cols-2 gap-4 mb-6";
    const claseCuadriculaInferior = "grid grid-cols-1 lg:grid-cols-2 gap-4";

    return (
        <div className="p-2 md:p-4">
            <h4 className="text-xl font-bold text-gray-700 mb-4">Panel General</h4>

            {/* Tarjetas de indicadores */}
            <div className={claseCuadriculaKpi}>
                <TarjetaKpi titulo="Órdenes Activas" valor={ordenesActivas} icono="pi pi-clipboard" color="bg-blue-500" cargando={cargando} />
                {!esMecanico && (
                    <TarjetaKpi titulo="Ingresos Totales" valor={formatearMoneda(ingresosTotales)} icono="pi pi-dollar" color="bg-green-500" cargando={cargando} />
                )}
                <TarjetaKpi titulo="Vehículos" valor={vehiculos.length} icono="pi pi-car" color="bg-indigo-500" cargando={cargando} />
                {!esMecanico && (
                    <TarjetaKpi titulo="Clientes" valor={clientes.length} icono="pi pi-users" color="bg-purple-500" cargando={cargando} />
                )}
            </div>

            {/* Gráficos */}
            <div className={claseCuadriculaGraficos}>
                <div className="bg-white shadow-md rounded-xl p-5">
                    <h5 className="font-bold text-gray-700 mb-4">Órdenes por Estado</h5>
                    {ordenes.length === 0 && !cargando ? (
                        <p className="text-sm text-gray-400 text-center py-8">Sin datos todavía</p>
                    ) : (
                        <div className="h-56">
                            <Doughnut
                                data={datosDona}
                                options={{
                                    maintainAspectRatio: false,
                                    plugins: { legend: { position: "bottom" } },
                                }}
                            />
                        </div>
                    )}
                </div>

                {!esMecanico && (
                    <div className="bg-white shadow-md rounded-xl p-5">
                        <h5 className="font-bold text-gray-700 mb-4">Ingresos - Últimos 6 meses</h5>
                        <div className="h-56">
                            <Line data={datosLinea} options={opcionesLinea} />
                        </div>
                    </div>
                )}
            </div>

            
            {!esMecanico && (
                <div className={claseCuadriculaInferior}>
                    <div className="bg-white shadow-md rounded-xl p-5">
                        <h5 className="font-bold text-gray-700 mb-4">Órdenes Recientes</h5>
                        {ordenesRecientes.length === 0 ? (
                            <p className="text-sm text-gray-400">Sin órdenes registradas todavía.</p>
                        ) : (
                            <div className="divide-y divide-gray-100">
                                {ordenesRecientes.map((orden) => (
                                    <div key={orden.id} className="flex items-center justify-between py-2 text-sm">
                                        <div>
                                            <span className="font-medium">{orden.numero}</span>
                                            <span className="text-gray-400 ml-2">{orden.vehiculo?.placa}</span>
                                        </div>
                                        <div className="flex items-center gap-3">
                                            <span
                                                className="text-xs font-medium px-2 py-1 rounded-full text-white"
                                                style={{ backgroundColor: COLORES_ESTADO[orden.estado] }}
                                            >
                                                {ETIQUETAS_ESTADO[orden.estado]}
                                            </span>
                                            <span className="font-semibold">{formatearMoneda(orden.total)}</span>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        )}
                    </div>

                    <div className="bg-white shadow-md rounded-xl p-5">
                        <h5 className="font-bold text-gray-700 mb-4 flex items-center gap-2">
                            Repuestos con Stock Bajo
                            {repuestosStockBajo.length > 0 && (
                                <span className="text-xs font-bold bg-red-100 text-red-600 px-2 py-0.5 rounded-full">
                                    {repuestosStockBajo.length}
                                </span>
                            )}
                        </h5>
                        {repuestosStockBajo.length === 0 ? (
                            <p className="text-sm text-gray-400">Todo el inventario está en niveles saludables.</p>
                        ) : (
                            <div className="divide-y divide-gray-100">
                                {repuestosStockBajo.map((repuesto) => (
                                    <div key={repuesto.id} className="flex items-center justify-between py-2 text-sm">
                                        <span>{repuesto.nombre}</span>
                                        <span className="text-red-600 font-bold">{repuesto.stock} unidades</span>
                                    </div>
                                ))}
                            </div>
                        )}
                    </div>
                </div>
            )}
        </div>
    );
}

function TarjetaKpi({ titulo, valor, icono, color, cargando }) {
    return (
        <div className="bg-white shadow-md rounded-xl p-5 flex items-center gap-4">
            <div className={`${color} text-white rounded-lg p-3`}>
                <i className={`${icono} text-xl`} />
            </div>
            <div>
                <p className="text-xs text-gray-500 uppercase tracking-wide">{titulo}</p>
                <p className="text-2xl font-bold text-gray-800">{cargando ? "…" : valor}</p>
            </div>
        </div>
    );
}
```

## 8.5 Restricción por rol: qué ve cada uno

| | `ADMIN` / `RECEPCIONISTA` | `MECANICO` |
|---|---|---|
| Tarjetas | Órdenes Activas, Ingresos Totales, Vehículos, Clientes (4) | Órdenes Activas, Vehículos (2) |
| Gráficos | Órdenes por Estado + Ingresos - Últimos 6 meses | Solo Órdenes por Estado |
| Panel inferior | Órdenes Recientes + Repuestos con Stock Bajo | - |


## 8.6 Controlando el tamaño de los gráficos

Por defecto, Chart.js intenta llenar todo el ancho disponible manteniendo una proporción fija (cuadrada, en el caso de un gráfico de dona) — en una tarjeta ancha, eso lo hace ver enorme. La solución tiene dos partes:

```jsx
<div className="h-56">
    <Doughnut
        data={dataDoughnut}
        options={{
            maintainAspectRatio: false,
            plugins: { legend: { position: "bottom" } },
        }}
    />
</div>
```

- **`maintainAspectRatio: false`** en las opciones del gráfico — le dice a Chart.js que no intente mantener una proporción fija, sino llenar exactamente el contenedor que se le dé.
- **Un contenedor con altura fija** (`h-56`, o cualquier clase de Tailwind equivalente) — como ya no mantiene una proporción, el gráfico toma el tamaño exacto de ese `<div>`. Cambiar el número (`h-48`, `h-64`, etc.) ajusta el tamaño directamente.

Ambas partes son necesarias juntas — solo con `maintainAspectRatio: false` y sin un contenedor de altura definida, el gráfico podría colapsar a una altura mínima o comportarse de forma impredecible.

## 8.7 Conectar la ruta

```jsx
// src/app/Router.jsx
import Dashboard from "../components/Dashboard";
// ...
<Route path="/" element={
    <RutaProtegida rolesPermisos={["ADMIN","RECEPCIONISTA","MECANICO"]}>
        <Dashboard />
    </RutaProtegida>
} />
```

`"/"` ya quedó restringida a personal (no `CLIENTE`)

## 8.8 Verificar que todo funciona

