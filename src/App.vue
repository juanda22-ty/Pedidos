<template>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
<header>
  <img src="/img/logotipo-empresa-restaurantes-comida-rapida-o-marca-marketing-empresarial_686210-21-removebg-preview.png" alt="logo">
  <div class="header-buttons">
    <button class="cart-button" @click="toggleModal()">
      <i class="fas fa-shopping-cart"></i>
      <span class="cart-count">{{ totalProductos() }}</span>
    </button>
    <button style="padding: 10px;" @click="toggleModalAgregar()" title="Agregar producto"><i class="fa fa-plus" aria-hidden="true"></i></button>
  </div>
</header>

<div v-show="mostrarModal" class="ventana-modal">
  <div class="modal-contenido">
    <div class="modal-header">
      <h2>Carrito de Compras</h2>
      <button class="btn-cerrar" @click="toggleModal()">&times;</button>
    </div>

    <div class="contenido-scroll">
      <div v-if="mostrarExitoso" class="mensaje-exitoso">
        <i class="fas fa-check-circle"></i>
        <p>¡Compra realizada con éxito!</p>
      </div>
      <div v-else-if="procesando" class="spinner-container">
        <div class="spinner"></div>
        <p>Generando factura...</p>
      </div>
      <div v-else>
        <div v-if="carrito.length === 0" class="carrito-vacio">
          <p>El carrito está vacío</p>
        </div>
        <div v-else class="carrito-tabla">
          <div class="carrito-encabezado">
            <span class="col-prod">Producto</span>
            <span class="col-cant">Cant.</span>
            <span class="col-precio">Precio</span>
            <span class="col-eliminar"></span>
          </div>

          <div class="carrito-cuerpo">
            <div v-for="item in carrito" :key="item.id + (item.sabor || '')" class="item-carrito">
              <div class="item-info">
                <span class="col-prod item-nombre">{{ item.nombre }}</span>
                <span class="col-cant item-cantidad">x{{ item.cantidad }}</span>
              </div>
              <span class="col-precio item-subtotal">{{ formatearPrecio(item.precio * item.cantidad) }}</span>
              <div class="col-eliminar">
                <button class="btn-eliminar" @click="eliminarDelCarrito(item)" title="Eliminar del carrito">
                  <i class="fas fa-trash-alt"></i>
                </button>
              </div>
            </div>
          </div>

          <div class="total">
            <span>Total a pagar:</span>
            <strong>{{ formatearPrecio(calcularTotal()) }}</strong>
          </div>
        </div>
      </div>
    </div>

    <div v-if="!mostrarExitoso && !procesando && carrito.length > 0" class="factura">
      <button @click="exportToPDF()">Comprar</button>
    </div>
  </div>
</div>

<div v-show="mostrarModalAgregar" class="ventana-modal modal-agregar">
    <h2>Agregar Nuevo Producto</h2>
    <button class="btn-cerrar" @click="toggleModalAgregar()">&times;</button>
    <form novalidate @submit.prevent="guardarProducto" class="formulario-producto">
      <div class="form-group">
        <label>Nombre del Producto:</label>
        <input v-model="nuevoProducto.nombre" type="text" placeholder="Ej: Ensalada fresca">
      </div>

      <div class="form-group">
        <label>Descripción:</label>
        <textarea v-model="nuevoProducto.descripcion" placeholder="Describe el producto..."></textarea>
      </div>

      <div class="form-group">
        <label>Categoría:</label>
        <select v-model="nuevoProducto.categoria">
          <option value="">Selecciona una categoría</option>
          <option value="Hamburguesas">Hamburguesas</option>
          <option value="Sándwiches">Sándwiches</option>
          <option value="Perros">Perros</option>
          <option value="Pizzas">Pizzas</option>
          <option value="Pollo">Pollo</option>
          <option value="Mariscos">Mariscos</option>
          <option value="Mexicana">Mexicana</option>
          <option value="Asiática">Asiática</option>
          <option value="Italiana">Italiana</option>
          <option value="Ensaladas">Ensaladas</option>
          <option value="Platos Especiales">Platos Especiales</option>
          <option value="Entradas">Entradas</option>
          <option value="Acompañamientos">Acompañamientos</option>
          <option value="Postres">Postres</option>
          <option value="Bebidas">Bebidas</option>
          <option value="Otra">Otra</option>
        </select>
      </div>

      <div v-if="nuevoProducto.categoria === 'Otra'" class="form-group">
        <label>Nueva categoría:</label>
        <input v-model="nuevoProducto.categoriaNueva" type="text" placeholder="Ej: Sandwiches especiales">
      </div>

      <div class="form-group">
        <label>Precio ($):</label>
        <input v-model.number="nuevoProducto.precio" type="number" step="0.01" placeholder="0.00">
      </div>

      <div class="form-group">
        <label>URL de la Imagen:</label>
        <input v-model="nuevoProducto.imagen" type="url" placeholder="https://...">
      </div>

      <div class="form-group">
        <label>Unidades:</label>
        <input v-model.number="nuevoProducto.unidades" type="number" placeholder="15">
      </div>

      <div class="form-buttons">
        <button type="submit" class="btn-guardar">Guardar Producto</button>
        <button type="button" class="btn-cancelar" @click="toggleModalAgregar()">Cancelar</button>
      </div>
    </form>
</div>

<div class="filters-container">
  <div class="search-box">
    <i class="fas fa-search"></i>
    <input 
      v-model="searchTerm" 
      type="text" 
      placeholder="Buscar productos..."
      class="search-input"
    >
    <button v-if="searchTerm" @click="searchTerm = ''" class="clear-btn" title="Limpiar búsqueda">
      <i class="fas fa-times"></i>
    </button>
  </div>
  
  <div class="category-filter">
    <label>Categoría:</label>
    <select v-model="selectedCategory" class="category-select">
      <option value="">Todas las categorías</option>
      <option v-for="categoria in obtenerCategorias()" :key="categoria" :value="categoria">
        {{ categoria }}
      </option>
    </select>
  </div>

  <div class="filter-info">
    <span v-if="productosFiltrados().length === 0" class="no-results">
      No se encontraron productos
    </span>
    <span v-else class="results-count">
      {{ productosFiltrados().length }} productos
    </span>
  </div>
</div>

<div id="menu">
  <div v-if="productosFiltrados().length === 0" class="no-results-menu">
    <p>No hay productos disponibles para este filtro.</p>
  </div>
  <template v-else>
    <div v-for="plato in productosFiltrados()" class="products" :class="{ agotado: plato.unidades === 0 }">
      <button class="btn-product-delete" @click="eliminarProducto(plato)" title="Eliminar producto">
        <i class="fas fa-times"></i>
      </button>
      <img :src="plato.imagen" :alt="plato.nombre">
      <h1>{{ plato.nombre }}</h1>
      <p>{{ plato.descripcion }}</p>
      <p class="precio">${{ plato.precio }}</p>
      <div class="product-footer">
        <select v-if="plato.sabores" v-model="saboresSeleccionados[plato.id]" class="select-sabor">
          <option v-for="sabor in plato.sabores" :value="sabor">{{ sabor }}</option>
        </select>
        <button
          :disabled="plato.unidades === 0"
          @click="agregarAlCarrito(plato, saboresSeleccionados[plato.id] || (plato.sabores ? plato.sabores[0] : null))">
          Agregar al carrito
        </button>
        <p class="unidad">Unidades: <span>{{ plato.unidades }}</span></p>
      </div>
    </div>
  </template>
</div>
</template>

<script setup>
import { ref } from 'vue';
import { jsPDF } from 'jspdf';
import Swal from 'sweetalert2';
const mostrarModal = ref(false);
const procesando = ref(false);
const mostrarExitoso = ref(false);
const mostrarModalAgregar = ref(false);
const searchTerm = ref('');
const selectedCategory = ref('');

const nuevoProducto = ref({
  nombre: '',
  descripcion: '',
  precio: 0,
  imagen: '',
  unidades: 15,
  categoria: '',
  categoriaNueva: ''
});

function cargarCarrito() {
  const carritoGuardado = localStorage.getItem('carrito');
  return carritoGuardado ? JSON.parse(carritoGuardado) : [];
}

function cargarUnidades() {
  const unidadesGuardadas = localStorage.getItem('unidades');
  return unidadesGuardadas ? JSON.parse(unidadesGuardadas) : null;
}

function cargarMenu() {
  const menuGuardado = localStorage.getItem('menu');
  return menuGuardado ? JSON.parse(menuGuardado) : null;
}

const unidadesGuardadas = cargarUnidades();

const menuPorDefecto = [
  { id: 1, nombre: "Hamburguesa", imagen: "https://png.pngtree.com/png-vector/20240829/ourmid/pngtree-delicious-and-testy-cheese-burger-png-image_13659847.png", descripcion: "Carne de res, cebolla salteada, lechuga, tomate y papas", precio: 12.50, unidades: unidadesGuardadas?.[0] ?? 15, categoria: "Hamburguesas" },
  { id: 2, nombre: "Hamburguesa Doble", imagen: "https://png.pngtree.com/png-clipart/20240321/original/pngtree-double-cheese-burger-png-image_14644513.png", descripcion: "Doble carne, doble queso y vegetales", precio: 15.00, unidades: unidadesGuardadas?.[1] ?? 15, categoria: "Hamburguesas" },
  { id: 3, nombre: "Hamburguesa Triple", imagen: "https://png.pngtree.com/png-vector/20250408/ourmid/pngtree-delicious-triple-layer-cheeseburger-with-lettuce-and-tomato-for-fast-food-png-image_15942712.png", descripcion: "Triple carne, queso y vegetales frescos", precio: 8.00, unidades: unidadesGuardadas?.[2] ?? 15, categoria: "Hamburguesas" },
  { id: 4, nombre: "Perro Caliente", imagen: "https://png.pngtree.com/png-vector/20241123/ourmid/pngtree-loaded-hot-dog-with-savory-toppings-png-image_14549006.png", descripcion: "Salchica americana, tocineta, queso, salsas a tu elección y carne", precio: 10.00, unidades: unidadesGuardadas?.[3] ?? 15, categoria: "Perros" },
  { id: 5, nombre: "Salchipapa", imagen: "https://static.vecteezy.com/system/resources/thumbnails/071/169/358/small/delicious-bacon-cheese-fries-isolated-on-transparent-background-free-png.png", descripcion: "Papas crujientes con salchicha y salsas", precio: 9.50, unidades: unidadesGuardadas?.[4] ?? 15, categoria: "Acompañamientos" },
  { id: 6, nombre: "Patacón relleno", imagen: "https://lalagourmetkc.com/wp-content/uploads/2024/02/patacon-pollo-con-lomito-324x324.png", descripcion: "Plátano frito relleno de carne y queso", precio: 11.00, unidades: unidadesGuardadas?.[5] ?? 15, categoria: "Platos Especiales" },
  { id: 7, nombre: "Tacos de Pollo", imagen: "https://png.pngtree.com/png-vector/20241110/ourmid/pngtree-authentic-chicken-tacos-with-cilantro-png-image_14338727.png", descripcion: "Tacos con pollo desmenuzado y vegetales frescos", precio: 9.00, unidades: unidadesGuardadas?.[6] ?? 15, categoria: "Mexicana" },
  { id: 8, nombre: "Quesadilla", imagen: "https://png.pngtree.com/png-clipart/20241003/original/pngtree-hearty-mexican-quesadilla-perfectly-grilled-with-meat-and-veggies-png-image_16190692.png", descripcion: "Tortilla con queso fundido y relleno a elegir", precio: 8.50, unidades: unidadesGuardadas?.[7] ?? 15, categoria: "Mexicana" },
  { id: 9, nombre: "Pizza Personal", imagen: "https://static.vecteezy.com/system/resources/thumbnails/027/141/485/small/pizza-created-with-generative-ai-png.png", descripcion: "Pizza pequeña con queso y pepperoni", precio: 13.00, unidades: unidadesGuardadas?.[8] ?? 15, categoria: "Pizzas" },
  { id: 10, nombre: "Sandwich Cubano", imagen: "https://static.vecteezy.com/system/resources/thumbnails/055/669/484/small/delicious-cuban-sandwich-with-layers-of-meats-and-melted-cheese-served-fresh-png.png", descripcion: "Pan con jamón, cerdo y piña caramelizada", precio: 10.50, unidades: unidadesGuardadas?.[9] ?? 15, categoria: "Sándwiches" },
  { id: 11, nombre: "Pollo Frito", imagen: "https://png.pngtree.com/png-clipart/20231017/original/pngtree-fried-chicken-american-food-png-image_13325963.png", descripcion: "Pollo crujiente con papas y ensalada", precio: 11.50, unidades: unidadesGuardadas?.[10] ?? 15, categoria: "Pollo" },
  { id: 12, nombre: "Nuggets de Pollo", imagen: "https://png.pngtree.com/png-clipart/20250104/original/pngtree-mouth-watering-fried-chicken-nuggets-and-fries-for-fast-food-lovers-png-image_18953065.png", descripcion: "Nuggets dorados con salsa y papas", precio: 7.50, unidades: unidadesGuardadas?.[11] ?? 15, categoria: "Pollo" },
  { id: 13, nombre: "Empanadas", imagen: "https://png.pngtree.com/png-clipart/20231016/original/pngtree-mexican-empanadas-whole-and-broken-with-stuffing-in-half-on-plate-png-image_13322073.png", descripcion: "Empanadas de carne o queso, 3 unidades", precio: 6.00, unidades: unidadesGuardadas?.[12] ?? 15, categoria: "Entradas" },
  { id: 14, nombre: "Alitas BBQ", imagen: "https://png.pngtree.com/png-vector/20240917/ourmid/pngtree-chicken-wings-with-bbq-sauce-png-image_13846550.png", descripcion: "Alitas bañadas en salsa BBQ picante", precio: 10.00, unidades: unidadesGuardadas?.[13] ?? 15, categoria: "Pollo" },
  { id: 15, nombre: "Nachos con Queso", imagen: "https://png.pngtree.com/png-clipart/20240404/original/pngtree-nachos-beaf-plate-png-image_14754118.png", descripcion: "Nachos crujientes con queso derretido y jalapeños", precio: 8.00, unidades: unidadesGuardadas?.[14] ?? 15, categoria: "Entradas" },
  { id: 16, nombre: "Camarones Fritos", imagen: "https://png.pngtree.com/png-clipart/20250107/original/pngtree-crispy-fried-shrimp-showcasing-its-golden-brown-color-png-image_18842298.png", descripcion: "Camarones crujientes con limón y salsas", precio: 14.00, unidades: unidadesGuardadas?.[15] ?? 15, categoria: "Mariscos" },
  { id: 17, nombre: "Ceviche", imagen: "https://png.pngtree.com/png-clipart/20250115/original/pngtree-delicious-bowl-of-ceviche-with-fresh-ingredients-for-culinary-marketing-png-image_20002417.png", descripcion: "Pescado fresco marinado en limón y especias", precio: 12.00, unidades: unidadesGuardadas?.[16] ?? 15, categoria: "Mariscos" },
  { id: 18, nombre: "Arepa Rellena", imagen: "https://png.pngtree.com/png-clipart/20250518/original/pngtree-delicious-reina-pepiada-arepa-png-image_21021771.png", descripcion: "Arepa con queso, jamón y guacamole", precio: 7.00, unidades: unidadesGuardadas?.[17] ?? 15, categoria: "Platos Especiales" },
  { id: 19, nombre: "Churros", imagen: "https://png.pngtree.com/png-clipart/20250626/original/pngtree-delicious-churros-and-chocolate-png-image_21237513.png", descripcion: "Churros crujientes con chocolate caliente", precio: 5.50, unidades: unidadesGuardadas?.[18] ?? 15, categoria: "Postres" },
  { id: 20, nombre: "Pastel de Tres Leches", imagen: "https://png.pngtree.com/png-clipart/20240804/original/pngtree-pastel-de-tres-leches-with-mint-a-traditional-mexican-cake-on-png-image_15702010.png", descripcion: "Pastel húmedo de tres leches", precio: 6.50, unidades: unidadesGuardadas?.[19] ?? 15, categoria: "Postres" },
  { id: 21, nombre: "Flan de Caramelo", imagen: "https://png.pngtree.com/png-clipart/20250123/original/pngtree-irresistible-spanish-flan-with-smooth-caramel-sauce-on-a-plate-png-image_19996287.png", descripcion: "Flan casero con caramelo tradicional", precio: 5.00, unidades: unidadesGuardadas?.[20] ?? 15, categoria: "Postres" },
  { id: 22, nombre: "Batido de Frutas", imagen: "https://png.pngtree.com/png-vector/20250206/ourmid/pngtree-healthy-fruit-smoothie-with-strawberries-bananas-and-almonds-png-image_15310493.png", descripcion: "Batidos frescos de frutas de temporada", precio: 4.50, unidades: unidadesGuardadas?.[21] ?? 15, categoria: "Bebidas" },
  { id: 23, nombre: "Jugo Natural", imagen: "https://png.pngtree.com/png-clipart/20250609/original/pngtree-refreshing-fruit-juices-in-glasses-with-splashes-png-image_21144383.png", descripcion: "Jugo natural recién exprimido", sabores: ["Naranja", "Manzana", "Piña", "Mango", "Limón", "Maracuyá"], precio: 3.50, unidades: unidadesGuardadas?.[22] ?? 15, categoria: "Bebidas" },
  { id: 24, nombre: "Agua Fresca", imagen: "https://png.pngtree.com/png-clipart/20250420/original/pngtree-water-bottle-png-image_20750613.png", descripcion: "Agua fresca con sabor a tu elección", precio: 2.00, unidades: unidadesGuardadas?.[23] ?? 15, categoria: "Bebidas" },
  { id: 25, nombre: "Ensalada César", imagen: "https://png.pngtree.com/png-vector/20241122/ourmid/pngtree-fresh-caesar-salad-with-juicy-grilled-chicken-strips-png-image_14407381.png", descripcion: "Ensalada fresca con pechuga de pollo", precio: 9.00, unidades: unidadesGuardadas?.[24] ?? 15, categoria: "Ensaladas" },
  { id: 26, nombre: "Sushi Roll", imagen: "https://png.pngtree.com/png-clipart/20241110/original/pngtree-vegeterian-sushi-roll-with-dill-and-vegetables-isolated-on-transparent-background-png-image_16867538.png", descripcion: "Roll de sushi fresco con pescado y vegetales", precio: 11.00, unidades: unidadesGuardadas?.[25] ?? 15, categoria: "Asiática" },
  { id: 27, nombre: "Fideos a la Carbonara", imagen: "https://png.pngtree.com/png-clipart/20231005/original/pngtree-pasta-carbonara-on-plate-png-image_13126473.png", descripcion: "Fideos en salsa cremosa con tocino", precio: 10.50, unidades: unidadesGuardadas?.[26] ?? 15, categoria: "Italiana" },
  { id: 28, nombre: "Poutine", imagen: "https://static.vecteezy.com/system/resources/previews/060/630/933/non_2x/a-bowl-of-poutine-with-crispy-fries-and-cheese-curds-isolated-on-transparent-background-free-png.png", descripcion: "Papas fritas con queso fundido y salsa", precio: 8.50, unidades: unidadesGuardadas?.[27] ?? 15, categoria: "Acompañamientos" },
  { id: 29, nombre: "Tarta de Queso", imagen: "https://png.pngtree.com/png-vector/20240130/ourmid/pngtree-cheesecake-png-with-ai-generated-png-image_11571900.png", descripcion: "Tarta de queso cremosa con frutos rojos", precio: 7.00, unidades: unidadesGuardadas?.[28] ?? 15, categoria: "Postres" },
  { id: 30, nombre: "Helado Artesanal", imagen: "https://png.pngtree.com/png-clipart/20250108/original/pngtree-flavorful-ice-cream-collection-with-fresh-berries-png-image_19755817.png", descripcion: "Helado artesanal en varios sabores", precio: 5.50, unidades: unidadesGuardadas?.[29] ?? 15, sabores: ["Vainilla", "Chocolate", "Fresa", "Mango", "Limón", "Coco"], categoria: "Postres" },
];

const menuGuardado = cargarMenu();
const menu = ref(menuGuardado || menuPorDefecto);

const saboresSeleccionados = ref({});

menu.value.forEach(plato => {
  if (plato.sabores) {
    saboresSeleccionados.value[plato.id] = plato.sabores[0];
  }
});

const carrito = ref(cargarCarrito());

function obtenerCategorias() {
  const categorias = [...new Set(menu.value.map(plato => plato.categoria))];
  return categorias.sort();
}

function productosFiltrados() {
  return menu.value.filter(plato => {
    const coincideTexto = plato.nombre.toLowerCase().includes(searchTerm.value.toLowerCase()) ||
                          plato.descripcion.toLowerCase().includes(searchTerm.value.toLowerCase());
    const coincideCategoria = selectedCategory.value === '' || plato.categoria === selectedCategory.value;
    return coincideTexto && coincideCategoria;
  });
}

function totalProductos() {
  return carrito.value.reduce((total, item) => total + item.cantidad, 0);
}

function toggleModal() {
  // Si el modal del carrito está abierto, ciérralo.
  // Si el modal de agregar está abierto, también ciérralo para que el botón "X" funcione.
  mostrarModal.value = !mostrarModal.value;
  mostrarModalAgregar.value = false;
}

function guardarCarrito() {
  localStorage.setItem('carrito', JSON.stringify(carrito.value));
}

function agregarAlCarrito(plato, sabor = null) {
  // Validación de integridad: el producto debe existir en el menú
  const productoMenu = menu.value.find(p => p.id === plato?.id);
  if (!productoMenu) {
    Swal.fire({
      icon: 'error',
      title: 'Producto no disponible',
      text: 'El producto ya no existe en el menú. Recarga la página e inténtalo de nuevo.',
      background: '#ffffff'
    });
    return;
  }

  // Validación de stock
  if (productoMenu.unidades == 0) {
    Swal.fire({
      icon: 'warning',
      title: 'Sin unidades disponibles',
      text: 'Este producto está agotado.',
      background: '#ffffff'
    });
    return;
  }

  // Validación de sabor (si aplica)
  if (productoMenu.sabores && productoMenu.sabores.length > 0) {
    const saborValido = sabor && productoMenu.sabores.includes(sabor);
    if (!saborValido) {
      Swal.fire({
        icon: 'error',
        title: 'Sabor inválido',
        text: 'Selecciona un sabor válido para este producto.',
        background: '#ffffff'
      });
      return;
    }
  }

  const nombreCompleto = sabor ? `${productoMenu.nombre} (${sabor})` : productoMenu.nombre;
  const itemExistente = carrito.value.find(item => item.id === productoMenu.id && item.sabor === sabor);

  if (itemExistente) {
    itemExistente.cantidad++;
  } else {
    carrito.value.push({
      id: productoMenu.id,
      nombre: nombreCompleto,
      precio: productoMenu.precio,
      cantidad: 1,
      sabor: sabor
    });
  }

  productoMenu.unidades--;
  guardarCarrito();
  guardarUnidades();
}

function guardarUnidades() {
  const unidades = menu.value.map(plato => plato.unidades);
  localStorage.setItem('unidades', JSON.stringify(unidades));
}

function calcularTotal() {
  return carrito.value.reduce((total, item) => total + (item.precio * item.cantidad), 0);
}

function formatearPrecio(valor) {
  const numero = Number(valor) || 0;
  return numero.toLocaleString('es-CO', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}

function eliminarDelCarrito(item) {
  Swal.fire({
    title: '¿Estás seguro?',
    text: `¿Deseas eliminar "${item.nombre}" del carrito?`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#000000',
    cancelButtonColor: '#ff4400',
    confirmButtonText: 'Sí, eliminar',
    cancelButtonText: 'Cancelar',
    background: '#ffffff'
  }).then((result) => {
    if (result.isConfirmed) {
      const indice = carrito.value.findIndex(carritoItem => carritoItem.id === item.id && carritoItem.sabor === item.sabor);
      
      if (indice !== -1) {
        const itemCarrito = carrito.value[indice];
        
        // Encontrar el producto en el menú para recuperar el stock
        const productoMenu = menu.value.find(plato => plato.id === item.id);
        
        if (productoMenu) {
          // Recuperar el stock según la cantidad en el carrito
          productoMenu.unidades += itemCarrito.cantidad;
        }
        
        // Eliminar el producto del carrito
        carrito.value.splice(indice, 1);
        
        // Guardar los cambios
        guardarCarrito();
        guardarUnidades();
        
        Swal.fire({
          icon: 'success',
          title: '¡Eliminado!',
          text: `"${item.nombre}" ha sido quitado del carrito.`,
          timer: 1500,
          showConfirmButton: false,
          background: '#ffffff'
        });
      }
    }
  });
}

function exportToPDF() {
  // Validaciones antes de generar la factura
  if (!Array.isArray(carrito.value) || carrito.value.length === 0) {
    Swal.fire({
      icon: 'warning',
      title: 'Carrito vacío',
      text: 'Agrega productos al carrito antes de comprar.',
      background: '#ffffff'
    });
    return;
  }

  const total = calcularTotal();
  if (!Number.isFinite(total) || total <= 0) {
    Swal.fire({
      icon: 'error',
      title: 'Error al calcular el total',
      text: 'El total del carrito no es válido. Revisa el contenido del carrito.',
      background: '#ffffff'
    });
    return;
  }

  procesando.value = true;
  
  setTimeout(() => {
    const doc = new jsPDF();
    const pageWidth = doc.internal.pageSize.getWidth();
    const margin = 20;
    const now = new Date();
    const invoiceNumber = `FAC-${now.getFullYear()}${(now.getMonth() + 1).toString().padStart(2, '0')}${now.getDate().toString().padStart(2, '0')}-${now.getHours().toString().padStart(2, '0')}${now.getMinutes().toString().padStart(2, '0')}`;

    doc.setFontSize(18);
    doc.setFont(undefined, 'bold');
    doc.text('Restaurante La Sazón', pageWidth / 2, 18, { align: 'center' });

    doc.setFontSize(10);
    doc.setFont(undefined, 'normal');
    doc.text('Comida rápida y delivery - Sabores auténticos', pageWidth / 2, 24, { align: 'center' });
    doc.text('Tel: +57 300 123 4567 | Carrera 15 #34-56, Bogotá', pageWidth / 2, 30, { align: 'center' });
    doc.text('www.restauranteslason.com', pageWidth / 2, 36, { align: 'center' });

    doc.setLineWidth(0.5);
    doc.line(margin, 40, pageWidth - margin, 40);

    doc.setFontSize(12);
    doc.setFont(undefined, 'bold');
    doc.text('Factura de Compra', margin, 50);

    doc.setFontSize(10);
    doc.setFont(undefined, 'normal');
    doc.text(`Fecha: ${now.toLocaleDateString()} ${now.toLocaleTimeString()}`, margin, 56);
    doc.text(`Factura #: ${invoiceNumber}`, margin, 62);
    doc.text(`Productos: ${carrito.value.length}`, margin, 68);
    doc.text(`Total: $${calcularTotal().toFixed(2)}`, pageWidth - margin, 62, { align: 'right' });

    doc.setFontSize(12);
    doc.setFont(undefined, 'bold');
    doc.text('Producto', margin, 80);
    doc.text('Cantidad', 100, 80);
    doc.text('Precio', 140, 80);
    doc.text('Subtotal', pageWidth - margin, 80, { align: 'right' });
    doc.line(margin, 82, pageWidth - margin, 82);

    doc.setFontSize(10);
    doc.setFont(undefined, 'normal');
    let yPosition = 92;

    carrito.value.forEach(item => {
      const subtotal = item.precio * item.cantidad;
      doc.text(item.nombre, margin, yPosition);
      doc.text(item.cantidad.toString(), 100, yPosition);
      doc.text(`$${item.precio.toFixed(2)}`, 140, yPosition);
      doc.text(`$${subtotal.toFixed(2)}`, pageWidth - margin, yPosition, { align: 'right' });
      yPosition += 8;

      if (yPosition > 270) {
        doc.addPage();
        yPosition = 20;
      }
    });

    doc.line(margin, yPosition + 4, pageWidth - margin, yPosition + 4);

    doc.setFontSize(12);
    doc.setFont(undefined, 'bold');
    const total = calcularTotal();
    doc.text(`TOTAL: $${total.toFixed(2)}`, pageWidth - margin, yPosition + 14, { align: 'right' });

    doc.setFontSize(10);
    doc.setFont(undefined, 'normal');
    doc.text('Gracias por preferirnos. ¡Vuelva pronto!', pageWidth / 2, yPosition + 26, { align: 'center' });
    doc.text('Este documento no es válido como factura fiscal.', pageWidth / 2, yPosition + 32, { align: 'center' });

    doc.save('factura.pdf');

    procesando.value = false;
    mostrarExitoso.value = true;

    carrito.value = [];
    localStorage.removeItem('carrito');

    setTimeout(() => {
      mostrarExitoso.value = false;
      mostrarModal.value = false;
    }, 2000);
  }, 1500);
}

function toggleModalAgregar() {
  mostrarModalAgregar.value = !mostrarModalAgregar.value;
  if (!mostrarModalAgregar.value) {
    nuevoProducto.value = {
      nombre: '',
      descripcion: '',
      precio: 0,
      imagen: '',
      unidades: 15,
      categoria: '',
      categoriaNueva: ''
    };
  }
}

function mostrarAlertaCampo(titulo, texto) {
  Swal.fire({
    icon: 'error',
    title: titulo,
    text: texto,
    confirmButtonColor: '#ff4400',
    background: '#ffffff'
  });
}

function validarProducto() {
  const producto = nuevoProducto.value;

  if (!producto.nombre || !producto.nombre.trim()) {
    mostrarAlertaCampo('Nombre requerido', 'Ingresa el nombre del producto.');
    return false;
  }

  if (!producto.descripcion || !producto.descripcion.trim()) {
    mostrarAlertaCampo('Descripción requerida', 'Agrega una descripción para el producto.');
    return false;
  }

  if (!producto.categoria) {
    mostrarAlertaCampo('Categoría requerida', 'Selecciona una categoría para el producto.');
    return false;
  }

  if (producto.categoria === 'Otra' && (!producto.categoriaNueva || !producto.categoriaNueva.trim())) {
    mostrarAlertaCampo('Nueva categoría requerida', 'Escribe el nombre de la nueva categoría.');
    return false;
  }

  if (producto.precio === '' || producto.precio === null || Number.isNaN(Number(producto.precio))) {
    mostrarAlertaCampo('Precio inválido', 'Ingresa un precio válido para el producto.');
    return false;
  }

  if (Number(producto.precio) <= 0) {
    mostrarAlertaCampo('Precio inválido', 'El precio debe ser mayor a cero.');
    return false;
  }

  if (!producto.imagen || !producto.imagen.trim()) {
    mostrarAlertaCampo('URL de imagen requerida', 'Proporciona una URL válida para la imagen del producto.');
    return false;
  }

  try {
    new URL(producto.imagen);
  } catch (error) {
    mostrarAlertaCampo('URL de imagen inválida', 'La dirección de la imagen no es válida. Usa un enlace completo, por ejemplo https://...');
    return false;
  }

  if (producto.unidades === '' || producto.unidades === null || Number.isNaN(Number(producto.unidades))) {
    mostrarAlertaCampo('Unidades inválidas', 'Ingresa un número válido de unidades.');
    return false;
  }

  const unidadesEnteras = parseInt(producto.unidades, 10);
  if (unidadesEnteras < 0) {
    mostrarAlertaCampo('Unidades inválidas', 'Las unidades no pueden ser negativas.');
    return false;
  }

  return true;
}

function guardarProducto() {
  if (!validarProducto()) {
    return;
  }

  const maxId = Math.max(...menu.value.map(p => p.id), 0);
  const categoriaFinal = nuevoProducto.value.categoria === 'Otra'
    ? nuevoProducto.value.categoriaNueva.trim()
    : nuevoProducto.value.categoria;

  const productoNuevo = {
    id: maxId + 1,
    nombre: nuevoProducto.value.nombre.trim(),
    descripcion: nuevoProducto.value.descripcion.trim(),
    precio: parseFloat(nuevoProducto.value.precio),
    imagen: nuevoProducto.value.imagen.trim(),
    unidades: parseInt(nuevoProducto.value.unidades, 10) || 0,
    categoria: categoriaFinal
  };

  menu.value.push(productoNuevo);
  guardarMenu();

  nuevoProducto.value = {
    nombre: '',
    descripcion: '',
    precio: 0,
    imagen: '',
    unidades: 15,
    categoria: '',
    categoriaNueva: ''
  };

  mostrarModalAgregar.value = false;

  Swal.fire({
    icon: 'success',
    title: '¡Excelente!',
    text: `El producto "${productoNuevo.nombre}" se ha agregado correctamente.`,
    toast: true,
    position: 'top-end',
    showConfirmButton: false,
    timer: 3000,
    timerProgressBar: true,
    background: '#ffffff',
    iconColor: '#27ae60'
  });
}

function guardarMenu() {
  localStorage.setItem('menu', JSON.stringify(menu.value));
}

function eliminarProducto(plato) {
  // 1. Lanzamos la alerta moderna de SweetAlert2
  Swal.fire({
    title: '¿Estás seguro?',
    text: `¿Deseas eliminar "${plato.nombre}"? Esta acción no se puede deshacer.`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#000000', // Color negro para confirmar
    cancelButtonColor: '#ff4400',  // Color rojo para cancelar
    confirmButtonText: 'Sí, eliminar',
    cancelButtonText: 'Cancelar',
    background: '#ffffff'
  }).then((result) => {
    
    // 2. Si el usuario presiona el botón "Sí, eliminar"
    if (result.isConfirmed) {
      
      // ============================================================
      // TU LÓGICA COMPLETA DE BORRADO
      // ============================================================
      const index = menu.value.findIndex(item => item.id === plato.id);
      if (index !== -1) {
        menu.value.splice(index, 1);
        guardarMenu();
        // Validación extra: si el producto estaba dentro del carrito, ya se eliminan esos items
        const antes = carrito.value.length;
        carrito.value = carrito.value.filter(item => item.id !== plato.id);
        const despues = carrito.value.length;

        guardarCarrito();
        guardarUnidades();

        if (antes === despues) {
          // Caso: el producto se borró del menú pero no estaba en el carrito
          Swal.fire({
            icon: 'info',
            title: 'Eliminado',
            text: `"${plato.nombre}" se eliminó del menú (no estaba en el carrito).`,
            timer: 1800,
            showConfirmButton: false,
            background: '#ffffff'
          });
        }
      }
      // ============================================================

      // 3. Mostramos el mensaje flotante de éxito tras la eliminación
      Swal.fire({
        icon: 'success',
        title: '¡Eliminado!',
        text: `"${plato.nombre}" ha sido quitado correctamente.`,
        timer: 2000,
        showConfirmButton: false
      });
    }
  });
}

</script>

<style>
#menu {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
  padding: 20px;
}

.products {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  text-align: center;
  background-color: #f9f9f9;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  cursor: pointer;
  width: 250px;
  flex-shrink: 0;
}

.products:hover {
  transform: scale(1.05);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  background-color: #fff;
}

.products h1 {
  font-size: 1.2em;
  margin: 10px 0;
}

.products p {
  font-size: 0.9em;
  color: #666;
}

.precio {
  font-size: 1.3em;
  color: #27ae60;
  font-weight: bold;
  margin: 10px 0;
}

.select-sabor {
  width: 100%;
  padding: 8px;
  margin: 10px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.9em;
}

.ventana-modal {
  position: fixed;
  top: 0;
  right: 20px;
  width: 380px;
  max-width: calc(100% - 40px);
  background-color: white;
  border: 2px solid #333;
  border-radius: 8px;
  padding: 20px;
  height: 100vh;
  max-height: 100vh;
  overflow-x: auto;
  z-index: 1000;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  animation: slideInFromRight 0.3s ease forwards;
  display: flex;
  flex-direction: column;
}

h2 {
  margin-bottom: 20px;
}

.btn-cerrar {
  position: absolute;
  top: 10px;
  right: 10px;
  background: none;
  border: none;
  font-size: 25px;
  cursor: pointer;
  color: #333;
  box-shadow: none;
  padding: 5px 12px;
}

.btn-cerrar:hover {
  color: #ffffff;
}

.contenido-scroll {
  flex-grow: 1;
  min-height: 0;
  overflow-y: auto;
  overflow-x: auto;
}

.factura {
  padding-top: 10px;
  border-top: 1px solid #eee;
}

@keyframes slideInFromRight {
  from {
    opacity: 0;
    transform: translateX(40px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.ventana-modal h2 {
  margin-top: 0;
  color: #333;
}

.carrito-vacio {
  text-align: center;
  padding: 20px;
  color: #999;
}

.spinner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  gap: 20px;
}

.spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #27ae60;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.spinner-container p {
  color: #333;
  font-size: 1.1em;
}

.mensaje-exitoso {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px;
  gap: 15px;
}

.mensaje-exitoso i {
  font-size: 3em;
  color: #27ae60;
}

.mensaje-exitoso p {
  font-size: 1.3em;
  color: #27ae60;
  font-weight: bold;
}

.filters-container {
  background-color: black;
  padding: 20px;
  display: flex;
  gap: 15px;
  flex-wrap: wrap;
  align-items: center;
  box-shadow: 0 4px 18px rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(10px);
}

.search-box {
  flex: 1;
  min-width: 250px;
  position: relative;
  display: flex;
  align-items: center;
  background: rgb(255, 255, 255);
  border-radius: 8px;
  padding: 0 12px;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.35);
  border: 1px solid rgba(255, 107, 53, 0.3);
  transition: box-shadow 0.3s ease, border-color 0.3s ease, background 0.3s ease;
}

.search-box:hover {
  background: rgba(0, 0, 0, 0.66);
  border-color: rgba(255, 107, 53, 0.55);
}

.search-box i {
  color: #ff6b35;
  margin-right: 8px;
  font-size: 1.1em;
}

.search-input {
  flex: 1;
  border: none;
  padding: 12px 8px;
  font-size: 1em;
  outline: none;
  background: transparent;
  color: #ff6b35;
  font-weight: 500;
}

.search-input::placeholder {
  color: #ff6b35;
}

.clear-btn {
  background: none;
  border: none;
  color: #ff6b35;
  box-shadow: none;
  cursor: pointer;
  padding: 4px 4px;
  margin: 0;
  font-size: 1.2em;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.3s ease, transform 0.2s ease;
}

.clear-btn:hover {
  background-color: black;
  box-shadow: none;
  transform: scale(1.1);
}

.category-filter {
  display: flex;
  align-items: center;
  gap: 10px;
}

.category-filter label {
  color: #ffffff;
  font-weight: 700;
  white-space: nowrap;
}

.category-select {
  padding: 10px 12px;
  border: 1px solid rgba(255, 107, 53, 0.3);
  border-radius: 6px;
  font-size: 0.95em;
  background: rgb(255, 255, 255);
  cursor: pointer;
  min-width: 150px;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.18);
  transition: box-shadow 0.3s ease, border-color 0.3s ease, background 0.3s ease;
  color: #333;
  font-weight: 600;
  backdrop-filter: blur(6px);
}

.category-select:hover {
  background: rgba(0, 0, 0, 0.66);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.22);
  border-color: rgba(255, 107, 53, 0.5);
  color: #ffffff;
}

.category-select:focus {
  outline: none;
  border-color: #ff4400;
  box-shadow: 0 0 0 3px rgba(255, 107, 53, 0.15);
  background: rgba(0, 0, 0, 0.66);
  color: #fbfbfb;
}

.filter-info {
  color: #333;
  font-weight: 700;
  white-space: nowrap;
  margin-left: auto;
}

.results-count {
  background: #ff4400;
  color: white;
  padding: 6px 12px;
  border-radius: 20px;
  font-weight: 600;
}

.no-results {
  background: #ff6b35;
  color: white;
  padding: 6px 12px;
  border-radius: 20px;
  font-weight: 600;
}

#menu {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
  padding: 20px;
  min-height: 200px;
}

#menu:empty::after {
  content: "No hay productos disponibles";
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  color: #999;
  font-size: 1.2em;
}

/* ==========================================================================
   MEDIA QUERIES (RESPONSIVE)
   ========================================================================== */

/* Dispositivos Medianos (Tabletas - menos de 768px) */
@media (max-width: 768px) {
  .filters-container {
    padding: 15px;
    gap: 10px;
  }

  .search-box {
    min-width: 100%;
    order: 1;
  }

  .category-filter {
    width: 100%;
    justify-content: center;
    order: 2;
  }

  .category-select {
    flex-grow: 1;
    max-width: 70%;
  }

  .filter-info {
    margin-left: 0;
    width: 100%;
    text-align: center;
    order: 3;
    margin-top: 5px;
  }
  
  /* Unificamos el modal de agregar para que NO se centre y use la animación lateral */
  .modal-agregar {
  right: 20px !important;
  transform: none !important; 
  width: 380px !important;
  max-width: calc(100% - 40px);
  
  /* CAMBIADO: En vez de 100vh fijo, permitimos que use el alto necesario o máximo el de la pantalla */
  height: auto !important;
  max-height: 95vh !important; 
  top: 2.5% !important;
  
  border-radius: 8px !important;
  overflow-y: auto !important; /* Esto activa el scroll si el formulario es largo */
  overflow-x: auto !important;
  animation: slideInFromRight 0.3s ease forwards !important;
}
}

/* Dispositivos Móviles Pequeños (menos de 480px) */
@media (max-width: 480px) {
 /* Ambos modales (carrito y agregar) se verán iguales en móvil */
  .ventana-modal {
    right: 0 !important;
    width: 100% !important;
    max-width: 100% !important;
    border-radius: 0;
    border: none;
    height: 100vh;
  }

  .ventana-modal, .modal-agregar {
    right: 0 !important;
    top: 0 !important;
    width: 100% !important;
    max-width: 100% !important;
    height: 100vh !important;
    max-height: 100vh !important;
    border-radius: 0 !important;
    border: none !important;
    
    /* CAMBIADO: Forzamos que todo el contenedor scrollee en celulares */
    overflow-y: auto !important; 
    display: flex;
    flex-direction: column;
  }

  .products {
    width: 95% !important;
    max-width: 100% !important;
  }

  @keyframes slideInFromRight {
    from {
      transform: translateX(100%);
    }
    to {
      transform: translateX(0);
    }
  }
}
</style>

