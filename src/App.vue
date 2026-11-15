
<script setup>
import { ref, onMounted } from 'vue'

const API = 'http://localhost:8080/api'
const seccion = ref('productos')
const productos = ref([])
const ventas = ref([])
const detalles = ref([])
const ventaSeleccionada = ref(null)

const nombre = ref('')
const descripcion = ref('')
const precio = ref('')
const stock = ref('')
const idEditar = ref(null)
const productoEliminar = ref(null)

const productoVenta = ref('')
const cantidadVenta = ref(1)
const carrito = ref([])

const mensaje = ref('')
const errorMensaje = ref(false)
let temporizador

function aviso(texto, error = false) {
  mensaje.value = texto
  errorMensaje.value = error
  clearTimeout(temporizador)
  temporizador = setTimeout(() => mensaje.value = '', 3000)
}

const dinero = valor => Number(valor).toFixed(2)
const fecha = valor => valor ? new Date(valor).toLocaleString('es-MX') : 'Sin fecha'

// PRODUCTOS

async function cargarProductos() {
  try {
    const r = await fetch(`${API}/productos`)
    if (!r.ok) throw new Error()
    productos.value = await r.json()
  } catch {
    aviso('Error al cargar productos', true)
  }
}

function limpiar() {
  nombre.value = ''
  descripcion.value = ''
  precio.value = ''
  stock.value = ''
  idEditar.value = null
}

async function guardarProducto() {
  const editando = idEditar.value !== null
  const url = editando ? `${API}/productos/${idEditar.value}` : `${API}/productos`

  const producto = {
    nombre: nombre.value,
    descripcion: descripcion.value,
    precio: Number(precio.value),
    stock: Number(stock.value)
  }

  try {
    const r = await fetch(url, {
      method: editando ? 'PUT' : 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(producto)
    })
    if (!r.ok) throw new Error()

    limpiar()
    await cargarProductos()
    aviso(editando ? 'Producto actualizado' : 'Producto agregado')
  } catch {
    aviso('Error al guardar producto', true)
  }
}

// EDITAR Y ELIMINAR PRODUCTOS

function editarProducto(p) {
  idEditar.value = p.id
  nombre.value = p.nombre
  descripcion.value = p.descripcion || ''
  precio.value = p.precio
  stock.value = p.stock
  window.scrollTo(0, 0)
}

async function eliminarProducto() {
  if (!productoEliminar.value) return

  const id = productoEliminar.value.id
  productoEliminar.value = null

  try {
    const r = await fetch(`${API}/productos/${id}`, { method: 'DELETE' })
    if (!r.ok) throw new Error()

    if (idEditar.value === id) limpiar()
    await cargarProductos()
    aviso('Producto eliminado correctamente')
  } catch {
    aviso('No se pudo eliminar. Puede tener ventas registradas.', true)
  }
}

// VENTAS

function agregarCarrito() {
  const p = productos.value.find(x => x.id === Number(productoVenta.value))
  const cantidad = Number(cantidadVenta.value)

  if (!p || !Number.isInteger(cantidad) || cantidad < 1) {
    return aviso('Selecciona un producto y cantidad válida', true)
  }

  const existe = carrito.value.find(x => x.id === p.id)
  if (cantidad + (existe?.cantidad || 0) > p.stock) {
    return aviso('Stock insuficiente', true)
  }

  if (existe) existe.cantidad += cantidad
  else carrito.value.push({
    id: p.id, nombre: p.nombre,
    precio: Number(p.precio), cantidad
  })

  productoVenta.value = ''
  cantidadVenta.value = 1
  aviso('Producto agregado a la venta')
}

const total = () => carrito.value.reduce(
  (suma, p) => suma + p.precio * p.cantidad, 0
)

async function registrarVenta() {
  if (!carrito.value.length) return aviso('Agrega productos', true)

  const datos = {
    detalles: carrito.value.map(p => ({
      productoId: p.id, cantidad: p.cantidad
    }))
  }

  try {
    const r = await fetch(`${API}/ventas`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(datos)
    })
    if (!r.ok) throw new Error()

    carrito.value = []
    await cargarProductos()
    await cargarVentas()
    aviso('Venta registrada correctamente')
  } catch {
    aviso('Error al registrar venta. Revisa el stock.', true)
  }
}

// HISTORIAL

async function cargarVentas() {
  try {
    const r = await fetch(`${API}/ventas`)
    if (!r.ok) throw new Error()
    ventas.value = await r.json()
  } catch {
    aviso('Error al cargar ventas', true)
  }
}

async function verDetalles(v) {
  try {
    const r = await fetch(`${API}/ventas/${v.id}/detalles`)
    if (!r.ok) throw new Error()
    detalles.value = await r.json()
    ventaSeleccionada.value = v
  } catch {
    aviso('Error al cargar detalles', true)
  }
}

function cambiarSeccion(valor) {
  seccion.value = valor
  if (valor === 'ventas') cargarProductos()
  if (valor === 'historial') {
    ventaSeleccionada.value = null
    cargarVentas()
  }
}

onMounted(() => {
  cargarProductos()
  cargarVentas()
})
</script>

<template>
  <div class="app">
    <header>
      <h1>Sistema de Ventas</h1>
      <p>Administración de productos y ventas</p>
    </header>

    <nav>
      <button v-for="op in ['productos', 'ventas', 'historial']"
        :key="op" :class="{ activo: seccion === op }"
        @click="cambiarSeccion(op)">
        {{ op.charAt(0).toUpperCase() + op.slice(1) }}
      </button>
    </nav>

    <main>
      <!-- PRODUCTOS -->
      <section v-if="seccion === 'productos'">
        <h2>Productos</h2>
        <h3>{{ idEditar !== null ? 'Editar producto' : 'Agregar producto' }}</h3>

        <form @submit.prevent="guardarProducto">
          <input v-model="nombre" placeholder="Nombre" required>
          <input v-model="descripcion" placeholder="Descripción">
          <input v-model="precio" type="number" min="0" step="0.01"
            placeholder="Precio" required>
          <input v-model="stock" type="number" min="0" step="1"
            placeholder="Stock" required>

          <button type="submit">
            {{ idEditar !== null ? 'Actualizar' : 'Guardar' }}
          </button>
          <button v-if="idEditar !== null" type="button" @click="limpiar">
            Cancelar
          </button>
        </form>

        <hr>
        <h3>Lista de productos</h3>

        <table>
          <thead>
            <tr>
              <th>ID</th><th>Nombre</th><th>Descripción</th>
              <th>Precio</th><th>Stock</th><th>Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in productos" :key="p.id">
              <td>{{ p.id }}</td>
              <td>{{ p.nombre }}</td>
              <td>{{ p.descripcion }}</td>
              <td>${{ dinero(p.precio) }}</td>
              <td>{{ p.stock }}</td>
              <td>
                <!-- EDITAR Y ELIMINAR PRODUCTOS -->
                <button @click="editarProducto(p)">Editar</button>
                <button class="rojo" @click="productoEliminar = p">
                  Eliminar
                </button>
              </td>
            </tr>
            <tr v-if="!productos.length">
              <td colspan="6">No hay productos registrados</td>
            </tr>
          </tbody>
        </table>
      </section>

      <!-- VENTAS -->
      <section v-else-if="seccion === 'ventas'">
        <h2>Registrar venta</h2>

        <select v-model="productoVenta">
          <option value="">Selecciona un producto</option>
          <option v-for="p in productos.filter(x => x.stock > 0)"
            :key="p.id" :value="p.id">
            {{ p.nombre }} - ${{ dinero(p.precio) }} (Stock: {{ p.stock }})
          </option>
        </select>

        <input v-model.number="cantidadVenta" type="number"
          min="1" step="1" placeholder="Cantidad">
        <button @click="agregarCarrito">Agregar</button>

        <hr>
        <h3>Productos de la venta</h3>

        <table>
          <thead>
            <tr>
              <th>Producto</th><th>Precio</th><th>Cantidad</th>
              <th>Subtotal</th><th>Acción</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in carrito" :key="p.id">
              <td>{{ p.nombre }}</td>
              <td>${{ dinero(p.precio) }}</td>
              <td>{{ p.cantidad }}</td>
              <td>${{ dinero(p.precio * p.cantidad) }}</td>
              <td>
                <button class="rojo" @click="carrito = carrito.filter(x => x.id !== p.id)">
                  Quitar
                </button>
              </td>
            </tr>
            <tr v-if="!carrito.length">
              <td colspan="5">No hay productos agregados</td>
            </tr>
          </tbody>
        </table>

        <h3>Total: ${{ dinero(total()) }}</h3>
        <button class="azul" :disabled="!carrito.length"
          @click="registrarVenta">
          Registrar venta
        </button>
      </section>

      <!-- HISTORIAL -->
      <section v-else>
        <h2>Historial de ventas</h2>
        <button @click="cargarVentas">Actualizar</button>

        <table>
          <thead>
            <tr>
              <th>ID</th><th>Fecha</th><th>Total</th><th>Detalles</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="v in ventas" :key="v.id">
              <td>{{ v.id }}</td>
              <td>{{ fecha(v.fecha) }}</td>
              <td>${{ dinero(v.total) }}</td>
              <td><button @click="verDetalles(v)">Ver detalles</button></td>
            </tr>
            <tr v-if="!ventas.length">
              <td colspan="4">No hay ventas registradas</td>
            </tr>
          </tbody>
        </table>

        <div v-if="ventaSeleccionada" class="detalles">
          <h3>Detalle de venta #{{ ventaSeleccionada.id }}</h3>
          <table>
            <thead>
              <tr>
                <th>Producto</th><th>Cantidad</th>
                <th>Precio unitario</th><th>Subtotal</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="d in detalles" :key="d.id">
                <td>{{ d.producto?.nombre }}</td>
                <td>{{ d.cantidad }}</td>
                <td>${{ dinero(d.precioUnitario) }}</td>
                <td>${{ dinero(d.subtotal) }}</td>
              </tr>
            </tbody>
          </table>
          <h3>Total: ${{ dinero(ventaSeleccionada.total) }}</h3>
          <button @click="ventaSeleccionada = null">Cerrar detalles</button>
        </div>
      </section>
    </main>

    <!-- MENSAJES -->
    <div v-if="mensaje" class="mensaje" :class="{ rojo: errorMensaje }">
      {{ mensaje }}
    </div>

    <!-- CONFIRMACIÓN PARA ELIMINAR -->
    <div v-if="productoEliminar" class="fondo-modal">
      <div class="modal">
        <h3>Eliminar producto</h3>
        <p>¿Deseas eliminar "{{ productoEliminar.nombre }}"?</p>
        <button @click="productoEliminar = null">Cancelar</button>
        <button class="rojo" @click="eliminarProducto">Eliminar</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: #f3f4f6;
  color: #1f2937;
  font-family: Arial, sans-serif;
}
header {
  background: #1e293b;
  color: white;
  padding: 30px;
  text-align: center;
}
header h1 { margin: 0; }
header p { color: #cbd5e1; margin-bottom: 0; }

nav {
  display: flex;
  justify-content: center;
  gap: 12px;
  padding: 20px;
}
nav button {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  background: white;
  cursor: pointer;
}
nav .activo { background: #2563eb; color: white; }

main {
  max-width: 1000px;
  margin: auto;
  padding: 30px;
  background: white;
  border-radius: 12px;
}
main h2 { margin-top: 0; }

input, select { padding: 8px; margin: 5px; }
button { cursor: pointer; }
section button, .modal button { padding: 8px 12px; margin: 4px; }
button:disabled { opacity: 0.5; cursor: not-allowed; }

table { width: 100%; border-collapse: collapse; margin-top: 15px; }
th, td { border: 1px solid #ccc; padding: 10px; text-align: left; }
th { background: #e2e8f0; }

.rojo { background: #dc2626; color: white; border: none; border-radius: 4px; }
.azul { background: #2563eb; color: white; border: none; border-radius: 4px; }
.detalles { margin-top: 25px; padding: 15px; background: #f8fafc; }

.mensaje {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: #16a34a;
  color: white;
  padding: 15px 25px;
  border-radius: 8px;
  box-shadow: 0 3px 10px #0003;
}
.mensaje.rojo { background: #dc2626; }

.fondo-modal {
  position: fixed;
  inset: 0;
  background: #0007;
  display: flex;
  align-items: center;
  justify-content: center;
}
.modal {
  background: white;
  padding: 25px;
  border-radius: 8px;
  text-align: center;
  width: 320px;
  max-width: 90%;
}
</style>
