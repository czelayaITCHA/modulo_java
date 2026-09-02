# 7. Reporte de Órdenes de Trabajo en PDF (iText 5.5.x)

Esta guía construye el primer reporte del sistema: un PDF de las órdenes de trabajo de un rango de fechas, organizado por estado, con encabezado, detalle y resumen de totales. Se explica con detalle cada pieza de iText usada, para que sirva de base a cualquier reporte futuro del proyecto.

## 7.1 Dependencia

```xml
<!-- pom.xml, dentro de <dependencies> -->
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itextpdf</artifactId>
    <version>5.5.13.4</version>
</dependency>
```

> iText 5.5.x es la última versión bajo licencia AGPL/comercial de la línea "5" (antes del salto a iText 7, con una API completamente distinta). Para un proyecto académico o interno, la 5.5.x es la elección práctica más común.

## 7.2 Cómo funciona iText 5.5.x, en conceptos

Antes del código, cuatro ideas que explican todo lo demás:

1. **`Document`** representa el documento PDF en abstracto — tamaño de página, márgenes. No sabe nada de "a dónde" se está escribiendo.
2. **`PdfWriter`** conecta ese `Document` con un destino real (un archivo, o en este caso un arreglo de bytes en memoria). `Document` y `PdfWriter` siempre van de la mano.
3. **Todo se mide en puntos, no en centímetros ni píxeles** — 72 puntos = 1 pulgada. Por eso se define una constante de conversión (`CM = 28.3465f`) para poder pensar en centímetros y que el código haga la conversión.
4. **El contenido se agrega en flujo, de arriba hacia abajo, con `document.add(...)`** — párrafos, tablas, imágenes, todo se agrega en el orden en que debe aparecer, e iText decide automáticamente cuándo saltar de página si el contenido no cabe.

Hay una quinta pieza para lo que no sigue el flujo normal: encabezados/pies de página que deben repetirse **en todas las páginas**, sin importar cuánto contenido tenga cada una. Para eso existe `PdfPageEventHelper` (sección 7.6).

## 7.3 El `Service` completo

```java
package com.devsv.autofix_api.services;

import com.devsv.autofix_api.dto.OrdenTrabajoDTO;
import com.devsv.autofix_api.enums.EstadoOrden;
import com.devsv.autofix_api.exceptions.BadRequestException;
import com.devsv.autofix_api.exceptions.ResourceNotFoundException;
import com.devsv.autofix_api.interfaces.IOrdenTrabajoService;
import com.itextpdf.text.*;
import com.itextpdf.text.pdf.ColumnText;
import com.itextpdf.text.pdf.PdfContentByte;
import com.itextpdf.text.pdf.PdfPCell;
import com.itextpdf.text.pdf.PdfPTable;
import com.itextpdf.text.pdf.PdfPageEventHelper;
import com.itextpdf.text.pdf.PdfWriter;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.io.ByteArrayOutputStream;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Objects;

@Service
@RequiredArgsConstructor
public class ReporteOrdenesTrabajoService {
    //inyectamos la interface
    private final IOrdenTrabajoService ordenTrabajoService;
    //definimos constantes al utilizar en el diseño del reporte
    private static final float CM = 28.3465f;
    private static final DateTimeFormatter FORMATO_FECHA = DateTimeFormatter.ofPattern("dd/MM/yyyy");
    private static final BaseColor COLOR_PRIMARIO = new BaseColor(37,99,235);
    private static final BaseColor COLOR_GRIS_CLARO = new BaseColor(243,244,246);

    public byte[] generarReporte(LocalDate fechaInicio, LocalDate fechaFin){
        if(fechaInicio == null || fechaFin == null){
            throw new BadRequestException("Seleccione un rango de fechas");
        }
        if(fechaInicio.isAfter(fechaFin)){
            throw new BadRequestException("La fecha de inicio no puede ser posterior a la final");
        }
        //obtenemos las ordenes
        List<OrdenTrabajoDTO> ordenes = ordenTrabajoService.findAll(null, fechaInicio, fechaFin);
        try{
            ByteArrayOutputStream salida = new ByteArrayOutputStream();
            //creamos la instancia del documento
            Document document = new Document(PageSize.LETTER, 2.5f * CM, 2.5f * CM, 2.5f * CM, 2.5f * CM);
            PdfWriter writer = PdfWriter.getInstance(document, salida);

            //obtenemos la fecha actual para la generación del reporte
            String fechaGeneracion = LocalDate.now().format(FORMATO_FECHA);
            writer.setPageEvent(new FooterEvent(fechaGeneracion));

            document.open();
            agregarEncabezado(document, fechaInicio, fechaFin);
            agregarDetalle(document, ordenes);
            agregarResumen(document, ordenes);

            document.close();
            return salida.toByteArray();
        }catch (DocumentException e){
            throw new RuntimeException("Error al generar el PDF", e);
        }
    }

    private void agregarEncabezado(Document document, LocalDate fechaInicio, LocalDate fechaFin) throws DocumentException {
        //se cambió para que el logo este a la par de los otros datos del encabezado
        PdfPTable tablaEncabezado = new PdfPTable(new float[]{1f, 4f});
        tablaEncabezado.setWidthPercentage(100);

        //logo de la empresa
        PdfPCell celdaLogo = new PdfPCell();
        celdaLogo.setBorder(Rectangle.NO_BORDER);
        celdaLogo.setVerticalAlignment(Element.ALIGN_MIDDLE);
        try{
            Image logo = Image.getInstance(getClass().getResource("/static/logo.png"));
            logo.scaleToFit(70,70);
            celdaLogo.addElement(logo);
        }catch (Exception e){
            throw new ResourceNotFoundException("No se encontró el archivo del logo");
        }
        tablaEncabezado.addCell(celdaLogo);

        //celda con nombre de la empresa, título y subtítulo, uno debajo del otro
        PdfPCell celdaTexto = new PdfPCell();
        celdaTexto.setBorder(Rectangle.NO_BORDER);
        celdaTexto.setVerticalAlignment(Element.ALIGN_MIDDLE);
        celdaTexto.setPaddingLeft(10);

        //nombre de la empresa
        Font fontEmpresa =
                new Font(Font.FontFamily.HELVETICA, 16, Font.BOLD, COLOR_PRIMARIO);
        celdaTexto.addElement(new Paragraph("Autofix - Taller Mecánico", fontEmpresa));

        //título del reporte
        Font fontTitulo = new Font(Font.FontFamily.HELVETICA,14, Font.BOLD, BaseColor.BLACK);
        Paragraph titulo = new Paragraph("Reporte de Órdenes de Trabajo", fontTitulo);
        titulo.setSpacingBefore(2);
        celdaTexto.addElement(titulo);

        //subtítulo del reporte
        Font fontSubTitulo = new Font(Font.FontFamily.HELVETICA,12, Font.NORMAL, BaseColor.GRAY);
        Paragraph rango = new Paragraph("Desde " + fechaInicio.format(FORMATO_FECHA) +
                " Hasta " + fechaFin.format(FORMATO_FECHA), fontSubTitulo);
        rango.setSpacingBefore(2);
        celdaTexto.addElement(rango);

        tablaEncabezado.addCell(celdaTexto);

        document.add(tablaEncabezado);

        //espacio antes de que empiece el detalle
        Paragraph espacio = new Paragraph(" ");
        espacio.setSpacingAfter(6);
        document.add(espacio);
    }

    private void agregarDetalle(Document document, List<OrdenTrabajoDTO> ordenes) throws DocumentException{
        Font fontSeccion = new Font(Font.FontFamily.HELVETICA, 12, Font.BOLD, BaseColor.BLACK);

        for (EstadoOrden estado : EstadoOrden.values()) {
            List<OrdenTrabajoDTO> ordenesDelEstado = ordenes.stream()
                    .filter(o -> o.getEstado() == estado)
                    .toList();

            if (ordenesDelEstado.isEmpty()) continue;

            Paragraph seccion = new Paragraph(traducirEstado(estado) + " (" + ordenesDelEstado.size() + ")", fontSeccion);
            seccion.setSpacingBefore(10);
            seccion.setSpacingAfter(6);
            document.add(seccion);

            document.add(construirTablaOrdenes(ordenesDelEstado));
        }

        if (ordenes.isEmpty()) {
            Font fontVacio = new Font(Font.FontFamily.HELVETICA, 10, Font.ITALIC, BaseColor.GRAY);
            document.add(new Paragraph("No se encontraron órdenes de trabajo en el rango seleccionado.", fontVacio));
        }
    }

    private void agregarResumen(Document document, List<OrdenTrabajoDTO> ordenes) throws DocumentException{
        if (ordenes.isEmpty()) return;

        Font fontSeccion = new Font(Font.FontFamily.HELVETICA, 12, Font.BOLD, BaseColor.BLACK);
        Paragraph titulo = new Paragraph("Resumen", fontSeccion);
        titulo.setSpacingBefore(16);
        titulo.setSpacingAfter(6);
        document.add(titulo);

        PdfPTable tabla = new PdfPTable(new float[]{3f, 1.5f, 2f});
        tabla.setWidthPercentage(60);
        tabla.setHorizontalAlignment(Element.ALIGN_LEFT);

        Font fontEncabezado = new Font(Font.FontFamily.HELVETICA, 9, Font.BOLD, BaseColor.WHITE);
        for (String columna : new String[]{"Estado", "Cantidad", "Total"}) {
            PdfPCell celda = new PdfPCell(new Phrase(columna, fontEncabezado));
            celda.setBackgroundColor(COLOR_PRIMARIO);
            celda.setPadding(5);
            tabla.addCell(celda);
        }

        Font fontFila = new Font(Font.FontFamily.HELVETICA, 9, Font.NORMAL, BaseColor.BLACK);
        BigDecimal totalGeneral = BigDecimal.ZERO;
        int cantidadGeneral = 0;

        for (EstadoOrden estado : EstadoOrden.values()) {
            List<OrdenTrabajoDTO> delEstado = ordenes.stream().filter(o -> o.getEstado() == estado).toList();
            if (delEstado.isEmpty()) continue;

            BigDecimal subtotal = delEstado.stream()
                    .map(OrdenTrabajoDTO::getTotal)
                    .filter(Objects::nonNull)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            tabla.addCell(celdaTexto(traducirEstado(estado), fontFila));
            tabla.addCell(celdaTexto(String.valueOf(delEstado.size()), fontFila));
            tabla.addCell(celdaMoneda(subtotal, fontFila));

            totalGeneral = totalGeneral.add(subtotal);
            cantidadGeneral += delEstado.size();
        }

        Font fontTotal = new Font(Font.FontFamily.HELVETICA, 9, Font.BOLD, BaseColor.BLACK);

        PdfPCell celdaEtiqueta = new PdfPCell(new Phrase("Total general", fontTotal));
        celdaEtiqueta.setPadding(5);
        celdaEtiqueta.setBackgroundColor(COLOR_GRIS_CLARO);
        tabla.addCell(celdaEtiqueta);

        PdfPCell celdaCantidad = new PdfPCell(new Phrase(String.valueOf(cantidadGeneral), fontTotal));
        celdaCantidad.setPadding(5);
        celdaCantidad.setBackgroundColor(COLOR_GRIS_CLARO);
        tabla.addCell(celdaCantidad);

        PdfPCell celdaTotal = new PdfPCell(new Phrase(formatearMoneda(totalGeneral), fontTotal));
        celdaTotal.setPadding(5);
        celdaTotal.setHorizontalAlignment(Element.ALIGN_RIGHT);
        celdaTotal.setBackgroundColor(COLOR_GRIS_CLARO);
        tabla.addCell(celdaTotal);

        document.add(tabla);
    }

    //método privado para constuir la tabla de ordenes
    private PdfPTable construirTablaOrdenes(List<OrdenTrabajoDTO> ordenes) throws DocumentException {
        PdfPTable tabla = new PdfPTable(new float[]{2f, 1.5f, 1.5f, 2f, 2f, 1.5f});
        tabla.setWidthPercentage(100);

        Font fontEncabezado = new Font(Font.FontFamily.HELVETICA, 9, Font.BOLD, BaseColor.WHITE);
        for (String columna : new String[]{"Número", "Fecha", "Placa", "Cliente", "Mecánico", "Total"}) {
            PdfPCell celda = new PdfPCell(new Phrase(columna, fontEncabezado));
            celda.setBackgroundColor(COLOR_PRIMARIO);
            celda.setPadding(5);
            tabla.addCell(celda);
        }

        Font fontFila = new Font(Font.FontFamily.HELVETICA, 9, Font.NORMAL, BaseColor.BLACK);
        for (OrdenTrabajoDTO orden : ordenes) {
            tabla.addCell(celdaTexto(orden.getNumero(), fontFila));
            tabla.addCell(celdaTexto(orden.getFecha().format(FORMATO_FECHA), fontFila));
            tabla.addCell(celdaTexto(orden.getVehiculo() != null ? orden.getVehiculo().getPlaca() : "-", fontFila));
            tabla.addCell(celdaTexto(
                    orden.getVehiculo() != null && orden.getVehiculo().getCliente() != null
                            ? orden.getVehiculo().getCliente().getNombre() : "-",
                    fontFila));
            tabla.addCell(celdaTexto(
                    orden.getMecanico() != null ? orden.getMecanico().getNombre() : "Sin asignar",
                    fontFila));
            tabla.addCell(celdaMoneda(orden.getTotal(), fontFila));
        }

        return tabla;
    }

    //métodos auxiliares
    private PdfPCell celdaTexto(String texto, Font font) {
        PdfPCell celda = new PdfPCell(new Phrase(texto != null ? texto : "-", font));
        celda.setPadding(4);
        return celda;
    }

    private PdfPCell celdaMoneda(BigDecimal valor, Font font) {
        PdfPCell celda = new PdfPCell(new Phrase(formatearMoneda(valor), font));
        celda.setPadding(4);
        celda.setHorizontalAlignment(Element.ALIGN_RIGHT);
        return celda;
    }

    private String formatearMoneda(BigDecimal valor) {
        BigDecimal v = (valor != null ? valor : BigDecimal.ZERO).setScale(2, RoundingMode.HALF_UP);
        return "$" + v;
    }

    private String traducirEstado(EstadoOrden estado) {
        return switch (estado) {
            case PENDIENTE -> "PENDIENTES";
            case EN_PROCESO -> "EN PROCESO";
            case COMPLETADA -> "COMPLETADAS";
            case ENTREGADA -> "ENTREGADAS";
            case CANCELADA -> "CANCELADAS";
        };
    }

    //clase privada Footervent
    private static class FooterEvent extends PdfPageEventHelper{
        private final String fechaGeneracion;

        //constructor de la clase privada
        FooterEvent(String fechaGeneracion){
            this.fechaGeneracion = fechaGeneracion;
        }

        @Override
        public void onEndPage(PdfWriter writer, Document document) {
            PdfContentByte cb = writer.getDirectContent();
            Font fontFooter = new Font(Font.FontFamily.HELVETICA, 8, Font.NORMAL, BaseColor.GRAY);

            Phrase footerIzquierda = new Phrase("Fecha Generación: " + fechaGeneracion, fontFooter);
            Phrase footerDerecha = new Phrase("Página: " + writer.getPageNumber(), fontFooter);

            ColumnText.showTextAligned(cb, Element.ALIGN_LEFT, footerIzquierda,
                    document.leftMargin(),document.bottom() - 20 , 0);
            ColumnText.showTextAligned(cb, Element.ALIGN_RIGHT, footerDerecha,
                    document.right(),document.bottom() - 20 , 0);
        }
    }
    //fin de la clase privada
}
```
## 7.4 Las 3 secciones, una por una

### Encabezado (`agregarEncabezado`)

**Para que el logo aparezca de verdad**, coloque un archivo real en `src/main/resources/static/logo.png` 

### Detalle (`agregarDetalle` + `construirTablaOrdenes`)

Se recorre `EstadoOrden.values()` — el orden natural del enum (`PENDIENTE, EN_PROCESO, COMPLETADA, ENTREGADA, CANCELADA`), que es también el orden real del flujo de negocio — y por cada estado que tenga al menos una orden, se agrega un subtítulo con el conteo y una `PdfPTable` con las columnas: Número, Fecha, Placa, Cliente, Mecánico, Total.

**`PdfPTable`, la pieza central de cualquier reporte tabular en iText:**
```java
PdfPTable tabla = new PdfPTable(new float[]{2f, 1.5f, 1.5f, 2f, 2f, 1.5f});
```
El arreglo de `float` define el **ancho relativo** de cada columna, no un ancho en puntos — la primera columna es proporcionalmente igual de ancha que la cuarta (`2f` cada una), y ambas más anchas que la segunda (`1.5f`). Las celdas se agregan una por una, en orden, con `tabla.addCell(...)` — no hay un concepto de "fila" explícito, iText simplemente llena la tabla de izquierda a derecha y salta de fila automáticamente al completar el número de columnas declarado.

### Resumen (`agregarResumen`)

Reutiliza exactamente el mismo patrón de tabla que el detalle, pero con subtotales: para cada estado, cuenta cuántas órdenes tiene y suma sus totales (`BigDecimal::add`, nunca sumando `double` a mano, para no arrastrar errores de redondeo con dinero). Al final, una fila de "Total general" con fondo gris claro para distinguirla visualmente del resto.

## 7.5 Márgenes, tamaño de papel, y la conversión de unidades

```java
private static final float CM = 28.3465f;
Document document = new Document(PageSize.LETTER, 2.5f * CM, 2.5f * CM, 2.5f * CM, 2.5f * CM);
```

`PageSize.LETTER` es una constante ya definida por iText (carta, 8.5 × 11 pulgadas). Los 4 argumentos de márgenes van en el orden: **izquierdo, derecho, superior, inferior** — como en este reporte los 4 son iguales (2.5 cm), no importa el orden, pero vale la pena tenerlo presente para reportes futuros con márgenes distintos entre sí.

## 7.6 Pie de página en todas las páginas: `PdfPageEventHelper`

Este es el mecanismo que resuelve "se necesita esto en cada página, sin saber de antemano cuántas páginas tendrá el documento":

```java
writer.setPageEvent(new FooterEvent(fechaGeneracion));
```

`PdfPageEventHelper` es una clase base de iText con métodos vacíos para cada momento del ciclo de vida del documento (`onOpenDocument`, `onStartPage`, `onEndPage`, `onCloseDocument`, entre otros) — se sobrescribe solo el que se necesita. Aquí, `onEndPage` se dispara automáticamente **al terminar cada página**, sin importar si el documento termina teniendo 1 página o 50.

Dentro de `onEndPage`, en vez de `document.add(...)` (que agrega contenido al flujo normal) se usa `PdfContentByte` — el "lienzo" de bajo nivel de esa página específica, que permite dibujar en una coordenada `(x, y)` exacta:

```java
ColumnText.showTextAligned(cb, Element.ALIGN_LEFT, footerIzquierda,
        document.leftMargin(), document.bottom() - 20, 0);
```

`document.bottom()` es el borde inferior del área de contenido (ya dentro del margen); se resta un poco más (`-20`) para separar visualmente el pie de página del margen inferior. `writer.getPageNumber()` devuelve el número de la página **actual**, disponible automáticamente en este punto del ciclo de vida.

> Nota: este reporte muestra "Página N", no "Página N de M" — mostrar el total de páginas exige una técnica adicional de iText (`PdfTemplate`, un marcador de posición que se rellena después de conocer el total), fuera del alcance de esta guía. Puede agregarse más adelante si se necesita.

## 7.7 Crear el controlador

```java

package com.devsv.autofix_api.controllers;

import com.devsv.autofix_api.services.ReporteOrdenesTrabajoService;
import lombok.RequiredArgsConstructor;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ContentDisposition;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@RestController
@CrossOrigin
@RequestMapping("/api/reportes")
@RequiredArgsConstructor
public class ReporteController {
    private final ReporteOrdenesTrabajoService reporteOrdenesTrabajoService;

    @PreAuthorize("hasAnyRole('ADMIN', 'RECEPCIONISTA')")
    @GetMapping("ordenes-trabajo")
    public ResponseEntity<ByteArrayResource> generarPdfOrdenes(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fechaInicio,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fechaFin
            ){
        byte[] pdf = reporteOrdenesTrabajoService.generarReporte(fechaInicio, fechaFin);

        String nombreArchivo = "ordenes-trabajo_" +
                fechaInicio.format(DateTimeFormatter.ofPattern("yyyyMMdd")) +
                fechaFin.format(DateTimeFormatter.ofPattern("yyyyMMdd")) + ".pdf";
        ByteArrayResource resource = new ByteArrayResource(pdf);

        return ResponseEntity.ok()
                .contentType(MediaType.APPLICATION_PDF)
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        ContentDisposition.attachment().filename(nombreArchivo)
                                .build().toString())
                .contentLength(pdf.length)
                .body(resource);
    }
}
```

`ContentDisposition.attachment().filename(...)` es lo que le indica al navegador "descarga esto con este nombre", en vez de intentar mostrarlo dentro de la misma pestaña.

## 7.8 Crear el servicio en el frontend

```js
import axiosClient from "./axiosClient";

export const reporteService = {
  descargarReporteOrdenes: async (fechaInicio, fechaFin) => {
    try {
      return await axiosClient.get("/reportes/ordenes-trabajo", {
        params: { fechaInicio, fechaFin },
        responseType: "blob",
      });
    } catch (error) {
      
      if (error.response?.data instanceof Blob) {
        const texto = await error.response.data.text();
        try {
          error.response.data = JSON.parse(texto);
        } catch {
          // el cuerpo no era JSON válido - se deja tal cual
        }
      }
      throw error;
    }
  },
};
```

**Por qué esto merece atención especial:** `responseType: "blob"` le dice a axios "trata la respuesta como datos binarios crudos" — necesario para poder descargar el PDF. Pero esa misma configuración aplica también a las respuestas de **error**: si el backend responde `400` con `{"message": "..."}`, axios igual la entrega como un `Blob`, no como el objeto JSON ya parseado que `mostrarErrorApi` espera leer (`error.response?.data?.message`). Sin esta conversión manual, cualquier error de este endpoint mostraría un mensaje genérico en vez del mensaje real del backend.

## 7.9 `ReporteOrdenes.jsx` — fechas en `Calendar` separados

```jsx
// src/reports/ReporteOrdenes.jsx
import { useState, useRef } from "react";
import { Dialog } from "primereact/dialog";
import { Calendar } from "primereact/calendar";
import { Button } from "primereact/button";
import { Toast } from "primereact/toast";

import { reporteService } from "../services/reporteService";
import { ordenTrabajoService } from "../services/ordenTrabajoService";
import { mostrarErrorApi } from "../utils/alertasApi";

const formatearFechaParaBackend = (fecha) => {
    if (!fecha) return null;
    const year = fecha.getFullYear();
    const month = String(fecha.getMonth() + 1).padStart(2, "0");
    const day = String(fecha.getDate()).padStart(2, "0");
    return `${year}-${month}-${day}`;
};

export default function ReporteOrdenes({ visible, onHide }) {
    const [fechaInicio, setFechaInicio] = useState(null);
    const [fechaFin, setFechaFin] = useState(null);
    const [generando, setGenerando] = useState(false);

    const toast = useRef(null);

    const generarReporte = async () => {
        if (!fechaInicio || !fechaFin) {
            toast.current.show({ severity: "warn", summary: "Atención", detail: "Seleccione ambas fechas" });
            return;
        }
        if (fechaInicio > fechaFin) {
            toast.current.show({
                severity: "warn",
                summary: "Atención",
                detail: "La fecha de inicio no puede ser posterior a la fecha fin",
            });
            return;
        }

        const fechaInicioStr = formatearFechaParaBackend(fechaInicio);
        const fechaFinStr = formatearFechaParaBackend(fechaFin);

        //abrir una nueva pestaña en el navegador
        const nuevaPestana = window.open("", "_blank");
        if (nuevaPestana) {
            nuevaPestana.document.write("Generando el reporte...");
        }

        setGenerando(true);
        try {
            const ordenesEnRango = await ordenTrabajoService.getAll({
                fechaInicio: fechaInicioStr,
                fechaFin: fechaFinStr,
            });

            if (ordenesEnRango.length === 0) {
                //no hay nada que mostrar se cierra la pestaña
                if (nuevaPestana) nuevaPestana.close();
                toast.current.show({
                    severity: "warn",
                    summary: "Atención",
                    detail: "No se encontraron órdenes de trabajo en el rango seleccionado.",
                    life: 4000,
                });
                return;
            }

            const respuesta = await reporteService.descargarReporteOrdenes(fechaInicioStr, fechaFinStr);

            const blobUrl = window.URL.createObjectURL(new Blob([respuesta.data], { type: "application/pdf" }));

            if (nuevaPestana) {
                nuevaPestana.location.href = blobUrl;
            } else {
                toast.current.show({
                    severity: "warn",
                    summary: "Atención",
                    detail: "El navegador bloqueó la ventana emergente. Habilite las ventanas emergentes para este sitio.",
                    life: 6000,
                });
            }

            onHide();
        } catch (error) {
            if (nuevaPestana) nuevaPestana.close();
            mostrarErrorApi(toast, error, "No se pudo generar el reporte");
        } finally {
            setGenerando(false);
        }
    };

    return (
        <Dialog
            visible={visible}
            style={{ width: "28rem" }}
            header="Reporte de Órdenes de Trabajo"
            modal
            onHide={onHide}
            footer={
                <div className="flex justify-end gap-2">
                    <Button label="Cancelar" icon="pi pi-times" outlined onClick={onHide} disabled={generando} />
                    <Button label="Generar Reporte" icon="pi pi-file-pdf" onClick={generarReporte} loading={generando} />
                </div>
            }
        >
            <Toast ref={toast} />

            <div className="field">
                <label className="font-bold block mb-2">Fecha inicio</label>
                <Calendar
                    value={fechaInicio}
                    onChange={(e) => setFechaInicio(e.value)}
                    dateFormat="dd/mm/yy"
                    className="w-full"
                    showIcon
                />
            </div>

            <div className="field">
                <label className="font-bold block mb-2">Fecha fin</label>
                <Calendar
                    value={fechaFin}
                    onChange={(e) => setFechaFin(e.value)}
                    dateFormat="dd/mm/yy"
                    className="w-full"
                    showIcon
                />
            </div>
        </Dialog>
    );
}
```

Dos `Calendar` de PrimeReact, cada uno con su propio estado (`fechaInicio`, `fechaFin`) — a diferencia del filtro de `OrdenesTrabajo.jsx`, que usa un solo `Calendar` en modo `selectionMode="range"`. Aquí se pidió explícitamente capturarlas por separado.



## 7.10 Crear `Reportes.jsx` — el contenedor

```jsx
// src/reports/Reportes.jsx
import { useState } from "react";
import ReporteOrdenes from "./ReporteOrdenes";

const reportesDisponibles = [
    {
        id: "ordenes-trabajo",
        titulo: "Órdenes de Trabajo",
        descripcion: "Reporte de órdenes agrupado por estado, para un rango de fechas.",
        icono: "pi pi-file-pdf",
    },
];

export default function Reportes() {
    const [reporteActivo, setReporteActivo] = useState(null);

    return (
        <div className="p-2 md:p-4">
            <h4 className="text-xl font-bold text-gray-700 mb-4">Reportes</h4>

            <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                {reportesDisponibles.map((reporte) => (
                    <button
                        key={reporte.id}
                        onClick={() => setReporteActivo(reporte.id)}
                        className="text-left bg-white shadow-md rounded-xl p-5 hover:shadow-lg transition-shadow border border-gray-100"
                    >
                        <i className={`${reporte.icono} text-3xl text-blue-600 mb-2 block`} />
                        <h5 className="font-bold text-gray-800">{reporte.titulo}</h5>
                        <p className="text-sm text-gray-500 mt-1">{reporte.descripcion}</p>
                    </button>
                ))}
            </div>

            {reporteActivo === "ordenes-trabajo" && (
                <ReporteOrdenes visible={true} onHide={() => setReporteActivo(null)} />
            )}
        </div>
    );
}
```

Es deliberadamente simple: un arreglo `reportesDisponibles` con una tarjeta por reporte, y un solo estado (`reporteActivo`) que decide cuál `Dialog` mostrar. Agregar un reporte futuro es: crear su propio componente (como `ReporteOrdenes.jsx`), agregar una entrada al arreglo, y una condición más al final del `return`.

## 7.11 Conectar la ruta

```jsx
// src/app/Router.jsx
import Reportes from "../reports/Reportes";
// ...
<Route path="/reportes" element={
    <RutaProtegida rolesPermisos={["ADMIN","RECEPCIONISTA"]}>
        <Reportes />
    </RutaProtegida>
} />
```
## 7.12 Resultado Final

<img width="1176" height="723" alt="image" src="https://github.com/user-attachments/assets/024d4f29-4948-4c6b-a7ac-fd966e2c89b6" />

<img width="1276" height="953" alt="image" src="https://github.com/user-attachments/assets/a0b73ed9-53fe-4d54-9c6f-a0229f03be80" />




