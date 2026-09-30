# 2do-proyecto

## Generador de sigilos

Aplicación web de un solo archivo (`index.html`, sin dependencias) para crear sigilos a partir de una frase o intención.

### Cómo funciona

1. Escribe una **frase o intención** y pulsa **Generar sigilo**.
2. La app reduce la frase a sus **letras únicas** (elimina las repetidas).
3. Elige una **geometría** sobre la que dibujar:
   - **Rueda** de letras.
   - **Kamea de Mercurio**.

   Puedes cambiar de una a otra en cualquier momento.
4. Elige un **modo de trazo**:
   - **Guiado:** dibujas siguiendo las letras reducidas. El botón *Visualizar sigilo* muestra u oculta la guía.
   - **Libre:** dibujas sin guía.

   Cada modo conserva su propio historial de trazos, para poder compararlos.

### Controles

- **Deshacer** el último trazo, **Limpiar trazo** o **Reiniciar todo**.
- **Girar** el sigilo con un control deslizante (0–359°).
- **Invertir** horizontal o verticalmente, y **Restablecer giro**.
- El dibujo se hace con el puntero (ratón o táctil) y los trazos se mantienen al redimensionar la ventana.

### Uso

Abre `index.html` en un navegador. No requiere instalación ni servidor.
