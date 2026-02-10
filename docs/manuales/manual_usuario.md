# Manual de Usuario - Extagram

Bienvenido a Extagram Pro. Esta guía describe el uso de la plataforma de gestión y visualización de activos multimedia.

## 1. Acceso al Sistema

La plataforma es accesible vía navegador web.

* **URL de Acceso:** `https://52.21.109.232`
* **Requisitos:** Navegador (Chrome, Firefox, Safari, Edge).
* **Nota de Seguridad:** Si el navegador muestra una advertencia de "Sitio no seguro", es debido al uso de certificados autofirmados internos. Puede proceder haciendo clic en "Avanzado" → "Continuar".

---

## 2. Funcionalidades Principales

### 2.1. Dashboard (Panel Principal)

Al ingresar, verás el **Muro de Activos**.

* **Vistas:** Utiliza los botones en la barra superior derecha para alternar entre:
    * ▦ **Vista Cuadrícula (Grid):** Ideal para visualización rápida de imágenes.
    * ≡ **Vista Lista:** Muestra detalles extendidos y metadatos.
* **Indicadores de Origen:** Cada imagen tiene una etiqueta que indica su almacenamiento:
    * `DISK (S5)`: Imagen servida desde el sistema de archivos de alto rendimiento.
    * `BLOB (S7)`: Imagen servida desde la base de datos (resguardo).
* **Información:** Cada tarjeta muestra el ID único del activo, la descripción y la fecha de subida.

### 2.2. Subir Nuevo Contenido

1. Haz clic en el botón **"Nueva Foto"** en la barra de navegación superior.
2. **Descripción:** Escribe un texto descriptivo para la imagen.
3. **Selección de Archivo:**
    * Haz clic en el área punteada o arrastra una imagen.
    * Formatos permitidos: JPG, PNG, GIF, WEBP.
    * Tamaño máximo: 50 MB.
4. Verás una vista previa de la imagen.
5. Pulsa **"Publicar ahora"**.
6. Serás redirigido automáticamente al Dashboard tras la carga exitosa.

---

## 3. Preguntas Frecuentes (FAQ)

**P: ¿Por qué mi foto no se sube?**  
R: Verifique que el archivo no supere los 50MB y que sea un formato de imagen válido. Si el problema persiste, contacte al administrador para revisar la conexión con el servicio de almacenamiento (S4).

**P: ¿Puedo borrar o editar una foto?**  
R: Actualmente la interfaz de usuario es de "Solo Agregado" (Append Only). Para eliminar contenido erróneo, solicite la baja del ID de la imagen al equipo de administración de sistemas.

**P: La página carga pero las imágenes aparecen rotas.**  
R: Esto puede indicar una desincronización entre la base de datos y el almacenamiento físico. Reporte el ID de la imagen afectada a soporte.

**P: ¿Qué significa "Served by Node: [Nombre]" al pie de página?**  
R: Extagram utiliza múltiples servidores para garantizar la velocidad. Ese código indica qué servidor específico procesó su solicitud en ese momento.

---

**Soporte Técnico:** Para asistencia adicional, contacte al equipo de administración de sistemas.
