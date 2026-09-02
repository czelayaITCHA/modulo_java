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

