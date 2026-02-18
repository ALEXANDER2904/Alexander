# Funcionamiento del Modal

## Estructura HTML

El componente tiene dos modos según la prop `Tipo`. Si es "Eye" renderiza un botón que abrirá el modal, si es "NewPage" renderiza un enlace que abre el proyecto en otra pestaña. El atributo `data-abrir` es el puente entre el botón y su modal — usa `modal-${name}` como identificador único para que cada botón sepa exactamente qué modal le corresponde.

```astro
{ isModal && (
  <button data-abrir={`modal-${name}`} class="button">
    <Icono />
  </button>
)}
{ isLink && (
  <a href={url} target="_blank" title={`Ir al proyecto ${name}`} class="button">
    <Icono />
  </a> 
)}
```

El `<dialog>` es un elemento HTML nativo que maneja comportamientos accesibles por defecto como el foco y la tecla Escape. Su id coincide con el valor de `data-abrir` del botón. El botón de cerrar usa `data-cerrar` con el mismo identificador para saber qué modal debe cerrar. El `?.` en `project?.description` es optional chaining — si project es undefined no lanza un error sino que devuelve undefined silenciosamente.

```astro
<dialog class="dialog-content" id={`modal-${name}`}>
  <div class="modal-content">
    <button data-cerrar={`modal-${name}`}>Cerrar</button>
    <p>Nombre corto: {name}</p>
    <p>{project?.description}</p>
  </div>
</dialog>
```

## Función calcularOrigen

Recibe el botón y el dialog como argumentos y devuelve un string como "23% 48%" que GSAP entiende como `transformOrigin`. Primero obtiene el centro absoluto del botón en el viewport con `getBoundingClientRect()` sumando la mitad del ancho y alto. Luego obtiene la posición del modal y calcula qué tan lejos está ese punto del borde del modal, expresado como porcentaje de su tamaño. Se llama siempre en el momento justo de la animación para que las coordenadas sean frescas y no haya desfase por scroll.

```js
const calcularOrigen = (boton: HTMLElement, dialog: HTMLDialogElement) => {
  const rect = boton.getBoundingClientRect();
  const origenX = rect.left + rect.width / 2;
  const origenY = rect.top + rect.height / 2;

  const dialogRect = dialog.getBoundingClientRect();
  const ox = ((origenX - dialogRect.left) / dialogRect.width) * 100;
  const oy = ((origenY - dialogRect.top) / dialogRect.height) * 100;

  return `${ox}% ${oy}%`;
};
```

## Función cerrarDialog

Recibe el dialog, busca su botón correspondiente en el DOM usando el id del modal como selector, y recalcula el origen en ese momento. Usa `gsap.to()` porque parte del estado visual actual hacia `scale: 0` y `opacity: 0`. La `ease: "power2.in"` acelera hacia el final, dando sensación de que el modal "se va". En el `onComplete` primero cierra el dialog nativamente y luego limpia los estilos inline que GSAP dejó con `clearProps: "all"`, dejando el elemento listo para la próxima apertura.

```js
const cerrarDialog = (dialog: HTMLDialogElement) => {
  const modalId = dialog.id;
  const boton = document.querySelector(`[data-abrir="${modalId}"]`) as HTMLElement;
  const origin = boton ? calcularOrigen(boton, dialog) : "50% 50%";
  gsap.to(dialog, {
    scale: 0,
    opacity: 0,
    duration: 0.25,
    ease: "power2.in",
    transformOrigin: origin,
    onComplete: () => {
      dialog.close();
      gsap.set(dialog, { clearProps: "all" });
    },
  });
};
```

## Apertura

Selecciona todos los botones con `data-abrir` y les añade un listener. Al hacer click, primero llama a `showModal()` para que el elemento sea visible en el DOM — esto es necesario antes de llamar a `calcularOrigen` porque el dialog necesita estar renderizado para que `getBoundingClientRect()` devuelva valores reales. Luego `gsap.fromTo()` anima desde el estado "encogido" hasta el normal, usando `back.out(1.7)` que produce ese pequeño rebote elástico al final. El `transformOrigin` se pasa en ambos estados del fromTo para que GSAP no lo interpole sino que lo mantenga fijo.

```js
document.querySelectorAll("[data-abrir]").forEach((btn) => {
  btn.addEventListener("click", (event) => {
    const boton = event.currentTarget as HTMLElement;
    const modalId = boton.getAttribute("data-abrir");
    const dialog = document.getElementById(modalId) as HTMLDialogElement;
    dialog.showModal();
    
    const origin = calcularOrigen(boton, dialog);
    gsap.fromTo(
      dialog,
      { scale: 0, opacity: 0, transformOrigin: origin },
      { scale: 1, opacity: 1, duration: 0.4, ease: "back.out(1.7)", transformOrigin: origin }
    );

    dialog.dataset.botonOrigen = boton.getAttribute("data-abrir") ?? "";
  });
});
```

## Cierre con botón y cierre al clicar fuera

Dos formas de cerrar, ambas delegan en `cerrarDialog`. El cierre por click fuera funciona comparando `e.target === dialog` — cuando el usuario clica en el backdrop, el target del evento es el propio elemento `<dialog>` y no ninguno de sus hijos, por lo que esa condición es true solo cuando el click fue fuera del contenido.

```js
document.querySelectorAll("[data-cerrar]").forEach((btn) => {
  btn.addEventListener("click", (event) => {
    const modalId = (event.currentTarget as HTMLElement).getAttribute("data-cerrar");
    const dialog = document.getElementById(modalId) as HTMLDialogElement;
    if (dialog) cerrarDialog(dialog);
  });
});

document.querySelectorAll("dialog").forEach((dialog) => {
  dialog.addEventListener("click", (e) => {
    if (e.target === dialog) cerrarDialog(dialog as HTMLDialogElement);
  });
});
```

## Flujo completo

Click en botón → `showModal()` hace visible el dialog → `calcularOrigen()` calcula el punto de expansión → `gsap.fromTo()` anima con rebote desde ese punto → al cerrar, `calcularOrigen()` recalcula con coordenadas frescas → `gsap.to()` encoge el modal de vuelta al mismo punto → `dialog.close()` lo cierra nativamente → `clearProps` limpia estilos residuales.
