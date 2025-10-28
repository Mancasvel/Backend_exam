# Guía Práctica para Resolver Exámenes DeliverUS Backend

Esta guía está diseñada para resolver exámenes que requieren añadir nuevas entidades y funcionalidades al sistema DeliverUS siguiendo un enfoque ordenado y sistemático.

---
[Pull Request Examen Resuelto](https://github.com/IISSI2-IS-2025/ExLab-Backend-Curso-Schedules/pull/10/files#diff-c9dca6352c49a52b7a3b966229023a71689065b389d43a584d791b04a2cb40dc)
## 📋 ÍNDICE DE REFERENCIA RÁPIDA

1. [Análisis de Requisitos](#1-análisis-de-requisitos-5-min)
2. [Migraciones de Base de Datos](#2-migraciones-de-base-de-datos-10-min)
3. [Modelos Sequelize](#3-modelos-sequelize-10-min)
4. [Rutas (Routes)](#4-rutas-routes-10-min)
5. [Validaciones](#5-validaciones-15-min)
6. [Controladores](#6-controladores-20-min)
7. [Funcionalidades Adicionales](#7-funcionalidades-adicionales-15-min)
8. [Testing y Verificación](#8-testing-y-verificación-5-min)

**Tiempo estimado total: 90 minutos**

---

## 🎯 SOLUCIÓN IMPLEMENTADA: Sistema de Horarios (Schedules)

### Contexto del Problema
Se implementó un sistema de **horarios (Schedules)** que permite a los restaurantes definir intervalos de tiempo en los que ciertos productos están disponibles. Esto habilita menús específicos por franjas horarias (desayunos, almuerzos, cenas).

### Modelo de Datos Implementado
```
Restaurant 1 ──── N Schedule 1 ──── N Product
```

- **Schedule**: Tabla intermedia con `startTime`, `endTime`, vinculada a un `Restaurant`
- **Product**: Añadió campo opcional `scheduleId` (FK a Schedule)

### Funcionalidades Implementadas

#### RF1-RF4: CRUD de Schedules
- **GET** `/restaurants/:restaurantId/schedules` - Listar horarios
- **POST** `/restaurants/:restaurantId/schedules` - Crear horario (owner)
- **PUT** `/restaurants/:restaurantId/schedules/:scheduleId` - Editar horario (owner)
- **DELETE** `/restaurants/:restaurantId/schedules/:scheduleId` - Eliminar horario (owner)

#### RF5: Productos Activos
- **GET** `/restaurants/:restaurantId/showWithActiveProducts` - Muestra solo productos cuyo horario incluye la hora actual del servidor

#### RF6-RF7: Gestión de scheduleId en Products
- Validación al crear/editar productos: el `scheduleId` debe pertenecer al restaurante del producto

---

## 📝 METODOLOGÍA PASO A PASO

## 1. ANÁLISIS DE REQUISITOS (5 min)

### Checklist Inicial
```
✅ Leer enunciado completo 2 veces
✅ Identificar nueva(s) entidad(es) a crear
✅ Dibujar diagrama de relaciones mentalmente
✅ Anotar las rutas requeridas con sus verbos HTTP
✅ Identificar permisos (public/authenticated/owner)
✅ Identificar validaciones especiales mencionadas
```

### Preguntas Clave
- ¿Qué tabla(s) nueva(s) necesito crear?
- ¿Qué tabla(s) existente(s) necesito modificar?
- ¿Qué tipo de relaciones hay? (1:N, N:M, opcional/obligatoria)
- ¿Qué rutas son públicas y cuáles requieren autenticación?
- ¿Hay validaciones de negocio complejas?

---

## 2. MIGRACIONES DE BASE DE DATOS (10 min)

### Paso 2.1: Crear Nueva Tabla
**Archivo**: `/src/database/migrations/YYYYMMDDHHMMSS-create-nueva-entidad.js`

#### Plantilla Base
```javascript
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('NombreTabla', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      campo1: {
        allowNull: false,
        type: Sequelize.STRING // o el tipo que corresponda
      },
      campo2: {
        allowNull: true,
        type: Sequelize.INTEGER
      },
      foreignKeyId: {
        type: Sequelize.INTEGER,
        allowNull: false,
        references: {
          model: { tableName: 'TablaPadre' },
          key: 'id'
        },
        onDelete: 'CASCADE' // o 'SET NULL' según necesidad
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE,
        defaultValue: new Date()
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE,
        defaultValue: new Date()
      }
    })
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('NombreTabla')
  }
}
```

#### Ejemplo Real (Schedule)
```javascript
// 20210718065004-create-schedules.js
await queryInterface.createTable('Schedules', {
  id: { /* ... */ },
  startTime: {
    allowNull: false,
    type: Sequelize.TIME
  },
  endTime: {
    allowNull: false,
    type: Sequelize.TIME
  },
  restaurantId: {
    type: Sequelize.INTEGER,
    allowNull: false,
    references: {
      model: { tableName: 'Restaurants' },
      key: 'id'
    },
    onDelete: 'CASCADE'
  },
  createdAt: { /* ... */ },
  updatedAt: { /* ... */ }
})
```

**⚠️ Puntos Clave**:
- `onDelete: 'CASCADE'` → Si se borra el padre, se borran los hijos
- `onDelete: 'SET NULL'` → Si se borra el padre, se pone null en el hijo (requiere `allowNull: true`)

### Paso 2.2: Modificar Tabla Existente
**Contexto**: Añadir campo `scheduleId` a la tabla `Products`

#### Modificación Directa en Migración Existente
```javascript
// En 20210718065005-create-product.js (ya existente)
// Añadir campo scheduleId:
scheduleId: {
  type: Sequelize.INTEGER,
  allowNull: true, // Opcional
  references: {
    model: { tableName: 'Schedules' },
    key: 'id'
  },
  onDelete: 'SET NULL'
}
```

**⚠️ Importante**: La migración de Products debe ejecutarse **DESPUÉS** de la de Schedules (por orden de timestamp).

---

## 3. MODELOS SEQUELIZE (10 min)

### Paso 3.1: Crear Nuevo Modelo
**Archivo**: `/src/models/NuevaEntidad.js`

#### Plantilla Base
```javascript
import { Model } from 'sequelize'

const loadModel = (sequelize, DataTypes) => {
  class NuevaEntidad extends Model {
    static associate (models) {
      // Pertenece a...
      NuevaEntidad.belongsTo(models.TablaPadre, { 
        foreignKey: 'tablaPadreId', 
        as: 'tablaPadre' 
      })

      // Tiene muchos...
      NuevaEntidad.hasMany(models.TablaHija, { 
        foreignKey: 'nuevaEntidadId', 
        as: 'tablasHijas' 
      })
    }
  }

  NuevaEntidad.init({
    campo1: DataTypes.STRING,
    campo2: DataTypes.INTEGER,
    foreignKeyId: DataTypes.INTEGER,
    createdAt: DataTypes.DATE,
    updatedAt: DataTypes.DATE
  }, {
    sequelize,
    modelName: 'NuevaEntidad'
  })

  return NuevaEntidad
}

export default loadModel
```

#### Ejemplo Real (Schedule)
```javascript
// /src/models/Schedule.js
import { Model } from 'sequelize'

const loadModel = (sequelize, DataTypes) => {
  class Schedule extends Model {
    static associate (models) {
      // Un horario pertenece a un restaurante
      Schedule.belongsTo(models.Restaurant, { 
        foreignKey: 'restaurantId', 
        as: 'restaurant' 
      })

      // Un horario puede tener varios productos asociados
      Schedule.hasMany(models.Product, { 
        foreignKey: 'scheduleId', 
        as: 'products' 
      })
    }
  }

  Schedule.init({
    startTime: {
      allowNull: false,
      type: DataTypes.TIME
    },
    endTime: {
      allowNull: false,
      type: DataTypes.TIME
    },
    restaurantId: {
      allowNull: false,
      type: DataTypes.INTEGER
    },
    createdAt: DataTypes.DATE,
    updatedAt: DataTypes.DATE
  }, {
    sequelize,
    modelName: 'Schedule'
  })

  return Schedule
}

export default loadModel
```

### Paso 3.2: Modificar Modelo Existente
**Archivo**: `/src/models/Product.js`

```javascript
// Añadir en static associate:
Product.belongsTo(models.Schedule, { 
  foreignKey: 'scheduleId', 
  as: 'schedule', 
  onDelete: 'set null' 
})

// Añadir en init:
scheduleId: DataTypes.INTEGER
```

### Paso 3.3: Registrar en models.js
**Archivo**: `/src/models/models.js`

```javascript
import loadScheduleModel from './Schedule.js'

const Schedule = loadScheduleModel(sequelizeSession, Sequelize.DataTypes)

const db = { ..., Schedule }

export { ..., Schedule }
```

---

## 4. RUTAS (Routes) (10 min)

### Paso 4.1: Crear Archivo de Rutas
**Archivo**: `/src/routes/ScheduleRoutes.js`

#### Plantilla Base CRUD Completo
```javascript
import * as EntidadValidation from '../controllers/validation/EntidadValidation.js'
import EntidadController from '../controllers/EntidadController.js'
import { isLoggedIn, hasRole } from '../middlewares/AuthMiddleware.js'
import { handleValidation } from '../middlewares/ValidationHandlingMiddleware.js'
import { checkEntityExists } from '../middlewares/EntityMiddleware.js'
import * as ParentMiddleware from '../middlewares/ParentMiddleware.js'
import { Entidad, Parent } from '../models/models.js'

const loadEntidadRoutes = function (app) {
  // Listar y crear entidades de un padre
  app.route('/parents/:parentId/entidades')
    .get(
      checkEntityExists(Parent, 'parentId'),
      EntidadController.indexParent
    )
    .post(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Parent, 'parentId'),
      ParentMiddleware.checkParentOwnership,
      EntidadValidation.create,
      handleValidation,
      EntidadController.create
    )

  // Actualizar y eliminar una entidad específica
  app.route('/parents/:parentId/entidades/:entidadId')
    .put(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Parent, 'parentId'),
      checkEntityExists(Entidad, 'entidadId'),
      ParentMiddleware.checkParentOwnership,
      EntidadValidation.update,
      handleValidation,
      EntidadController.update
    )
    .delete(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Parent, 'parentId'),
      checkEntityExists(Entidad, 'entidadId'),
      ParentMiddleware.checkParentOwnership,
      EntidadController.destroy
    )
}

export default loadEntidadRoutes
```

#### Ejemplo Real (Schedule)
```javascript
// /src/routes/ScheduleRoutes.js
const loadScheduleRoutes = function (app) {
  // Listar y crear horarios de un restaurante
  app.route('/restaurants/:restaurantId/schedules')
    .get(
      checkEntityExists(Restaurant, 'restaurantId'),
      ScheduleController.indexRestaurant
    )
    .post(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      ScheduleValidation.create,
      handleValidation,
      ScheduleController.create
    )

  // Obtener, actualizar y eliminar horario específico
  app.route('/restaurants/:restaurantId/schedules/:scheduleId')
    .put(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      checkEntityExists(Schedule, 'scheduleId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      ScheduleValidation.update,
      handleValidation,
      ScheduleController.update
    )
    .delete(
      isLoggedIn,
      hasRole('owner'),
      checkEntityExists(Restaurant, 'restaurantId'),
      checkEntityExists(Schedule, 'scheduleId'),
      RestaurantMiddleware.checkRestaurantOwnership,
      ScheduleController.destroy
    )
}

export default loadScheduleRoutes
```

### Paso 4.2: Cadena de Middlewares Explicada

```javascript
// Orden típico de middlewares:
.post(
  isLoggedIn,                    // 1. ¿Usuario autenticado?
  hasRole('owner'),              // 2. ¿Tiene rol correcto?
  checkEntityExists(Model, 'id'), // 3. ¿Existe la entidad padre/relacionada?
  checkOwnership,                // 4. ¿Es dueño del recurso?
  ValidationRules,               // 5. Validar datos del body
  handleValidation,              // 6. Procesar errores de validación
  Controller.action              // 7. Ejecutar lógica de negocio
)
```

**Reglas de Oro**:
- GET públicos: Solo `checkEntityExists`
- POST/PUT/DELETE owner: Autenticación + Role + Exists + Ownership + Validation
- El orden importa: primero auth, luego validaciones, luego controlador

### Paso 4.3: Añadir Ruta a Entidad Existente
**Contexto**: Añadir `showWithActiveProducts` a RestaurantRoutes

```javascript
// /src/routes/RestaurantRoutes.js
app.route('/restaurants/:restaurantId/showWithActiveProducts')
  .get(
    checkEntityExists(Restaurant, 'restaurantId'),
    RestaurantController.showWithActiveProducts
  )
```

**⚠️ Importante**: Colocar rutas específicas (con nombres completos) **ANTES** de rutas genéricas con parámetros:
```javascript
// ✅ CORRECTO
app.route('/restaurants/:restaurantId/showWithActiveProducts')
app.route('/restaurants/:restaurantId')

// ❌ INCORRECTO (el segundo atrapa todas las peticiones)
app.route('/restaurants/:restaurantId')
app.route('/restaurants/:restaurantId/showWithActiveProducts') // Nunca se alcanza
```

---

## 5. VALIDACIONES (15 min)

### Paso 5.1: Validaciones de Nueva Entidad
**Archivo**: `/src/controllers/validation/ScheduleValidation.js`

#### Funciones Auxiliares
```javascript
import { check } from 'express-validator'

// Validador de formato personalizado
const validateTimeFormat = (value) => {
  const timeRegex = /^([01]\d|2[0-3]):([0-5]\d):([0-5]\d)$/ // HH:mm:ss
  if (!timeRegex.test(value)) {
    throw new Error('Time must be in HH:mm:ss format')
  }
  return true
}

// Validador con acceso a req
const validateEndTimeAfterStartTime = (endTime, { req }) => {
  if (!endTime || !req.body.startTime) return true
  if (endTime <= req.body.startTime) {
    throw new Error('End time must be after start time')
  }
  return true
}
```

#### Reglas de Validación
```javascript
const create = [
  check('startTime')
    .exists().withMessage('Start time is required')
    .custom(validateTimeFormat),

  check('endTime')
    .exists().withMessage('End time is required')
    .custom(validateTimeFormat)
    .custom(validateEndTimeAfterStartTime)
]

const update = [
  check('startTime')
    .exists().withMessage('Start time is required')
    .custom(validateTimeFormat),

  check('endTime')
    .exists().withMessage('End time is required')
    .custom(validateTimeFormat)
    .custom(validateEndTimeAfterStartTime)
]

export { create, update }
```

### Paso 5.2: Validaciones de Relaciones en Entidad Existente
**Archivo**: `/src/controllers/validation/ProductValidation.js`

#### Validación en Creación
```javascript
import { Schedule } from '../../models/models.js'

const checkScheduleBelongsToRestaurantOnCreate = async (value, { req }) => {
  if (!value) return Promise.resolve() // scheduleId opcional
  
  const schedule = await Schedule.findByPk(value)
  if (!schedule) {
    return Promise.reject(new Error('The scheduleId does not exist.'))
  }
  if (schedule.restaurantId !== req.body.restaurantId) {
    return Promise.reject(new Error('The scheduleId does not belong to the given restaurantId.'))
  }
  return Promise.resolve()
}

// Añadir al array create:
const create = [
  // ... validaciones existentes
  check('scheduleId')
    .optional({ nullable: true, checkFalsy: true })
    .isInt({ min: 1 })
    .toInt()
    .custom(checkScheduleBelongsToRestaurantOnCreate)
]
```

#### Validación en Actualización
```javascript
import { Product } from '../../models/models.js'

const checkScheduleBelongsToRestaurantOnUpdate = async (value, { req }) => {
  if (!value) return Promise.resolve() // scheduleId opcional
  
  const schedule = await Schedule.findByPk(value)
  if (!schedule) {
    return Promise.reject(new Error('The scheduleId does not exist.'))
  }
  
  // En update, el producto ya existe y no recibimos restaurantId en body
  const product = await Product.findByPk(req.params.productId)
  if (product.restaurantId !== schedule.restaurantId) {
    return Promise.reject(new Error('The scheduleId does not belong to the restaurant of this product.'))
  }
  return Promise.resolve()
}

// Añadir al array update:
const update = [
  // ... validaciones existentes
  check('scheduleId')
    .optional({ nullable: true, checkFalsy: true })
    .isInt({ min: 1 })
    .toInt()
    .custom(checkScheduleBelongsToRestaurantOnUpdate)
]
```

### Paso 5.3: Cheatsheet de Validaciones
```javascript
// Básicas
check('campo').exists().withMessage('Campo requerido')
check('campo').optional({ nullable: true, checkFalsy: true })

// Tipos
.isString()
.isInt({ min: 1 }).toInt()
.isFloat({ min: 0 }).toFloat()
.isBoolean().toBoolean()
.isEmail()

// Strings
.isLength({ min: 1, max: 255 })
.trim()

// Personalizadas
.custom(async (value, { req }) => {
  // Lógica de validación
  if (/* condición falla */) {
    return Promise.reject(new Error('Mensaje de error'))
  }
  return Promise.resolve()
})
```

---

## 6. CONTROLADORES (20 min)

### Paso 6.1: CRUD Básico de Nueva Entidad
**Archivo**: `/src/controllers/ScheduleController.js`

#### Index (Listar)
```javascript
import { Schedule } from '../models/models.js'

const indexRestaurant = async function (req, res) {
  try {
    const schedules = await Schedule.findAll({
      where: { restaurantId: req.params.restaurantId }
    })
    res.json(schedules)
  } catch (err) {
    res.status(500).send(err)
  }
}
```

#### Create (Crear)
```javascript
const create = async function (req, res) {
  try {
    const newSchedule = Schedule.build(req.body)
    newSchedule.restaurantId = req.params.restaurantId // Asignar FK desde URL
    const schedule = await newSchedule.save()
    res.json(schedule)
  } catch (err) {
    res.status(500).send(err)
  }
}
```

#### Update (Actualizar)
```javascript
const update = async function (req, res) {
  try {
    await Schedule.update(req.body, { 
      where: { id: req.params.scheduleId } 
    })
    const updatedSchedule = await Schedule.findByPk(req.params.scheduleId)
    res.json(updatedSchedule)
  } catch (err) {
    res.status(500).send(err)
  }
}
```

#### Destroy (Eliminar)
```javascript
const destroy = async function (req, res) {
  try {
    const result = await Schedule.destroy({ 
      where: { id: req.params.scheduleId } 
    })
    let message = ''
    if (result === 1) {
      message = 'Successfully deleted schedule id: ' + req.params.scheduleId
    } else {
      message = 'Could not delete schedule.'
    }
    res.json(message)
  } catch (err) {
    res.status(500).send(err)
  }
}
```

#### Exportar
```javascript
const ScheduleController = {
  indexRestaurant,
  create,
  update,
  destroy
}

export default ScheduleController
```

---

## 7. FUNCIONALIDADES ADICIONALES (15 min)

### Caso: Mostrar Productos Activos por Horario
**Contexto**: Endpoint que devuelve restaurante con solo productos cuyo horario incluye la hora actual.

#### Paso 7.1: Añadir Método al Controlador
**Archivo**: `/src/controllers/RestaurantController.js`

```javascript
import { Restaurant, Product, RestaurantCategory, ProductCategory, Schedule } from '../models/models.js'
import { Op } from 'sequelize'

const showWithActiveProducts = async function (req, res) {
  try {
    const now = new Date().toTimeString().split(' ')[0] // 'HH:mm:ss'

    const restaurant = await Restaurant.findByPk(req.params.restaurantId, {
      attributes: { exclude: ['userId'] },
      include: [
        {
          model: Product,
          as: 'products',
          required: false, // LEFT JOIN (incluir restaurante aunque no tenga productos activos)
          include: [
            { model: ProductCategory, as: 'productCategory' },
            {
              model: Schedule,
              as: 'schedule',
              required: true, // INNER JOIN (solo productos CON schedule)
              where: {
                startTime: { [Op.lte]: now }, // startTime <= now
                endTime: { [Op.gt]: now }     // endTime > now
              }
            }
          ]
        },
        {
          model: RestaurantCategory,
          as: 'restaurantCategory'
        }
      ],
      order: [[{ model: Product, as: 'products' }, 'order', 'ASC']]
    })

    res.json(restaurant)
  } catch (err) {
    res.status(500).send(err)
  }
}

// Añadir al export
export default {
  // ... otros métodos
  showWithActiveProducts
}
```

#### Paso 7.2: Explicación de la Query

**Include Anidado**:
```javascript
include: [
  {
    model: Product,
    as: 'products',
    required: false, // LEFT JOIN
    include: [
      {
        model: Schedule,
        as: 'schedule',
        required: true, // INNER JOIN
        where: { /* condiciones */ }
      }
    ]
  }
]
```

**Comportamiento**:
1. `required: false` en Product → Incluye el restaurante aunque no tenga productos activos
2. `required: true` en Schedule → Solo incluye productos que tengan un schedule
3. `where` en Schedule → Filtra por horario actual

**Operadores de Comparación**:
```javascript
import { Op } from 'sequelize'

where: {
  campo: { [Op.eq]: value },    // igual (=)
  campo: { [Op.ne]: value },    // distinto (!=)
  campo: { [Op.gt]: value },    // mayor que (>)
  campo: { [Op.gte]: value },   // mayor o igual (>=)
  campo: { [Op.lt]: value },    // menor que (<)
  campo: { [Op.lte]: value },   // menor o igual (<=)
  campo: { [Op.between]: [a, b] }, // entre
  campo: { [Op.in]: [1, 2, 3] }, // en lista
  campo: { [Op.like]: '%text%' } // contiene
}
```

---

## 8. TESTING Y VERIFICACIÓN (5 min)

### Paso 8.1: Preparar Entorno
```bash
# Instalar dependencias (primera vez)
npm run install:all:win   # Windows
npm run install:all:bash  # Linux/Mac

# Recrear base de datos
npm run migrate:backend

# Iniciar servidor (desarrollo)
npm run start:backend

# Iniciar servidor (depuración)
# VSCode: Run and Debug → Debug Backend
```

### Paso 8.2: Ejecutar Tests
```bash
# Todos los tests
npm run test:backend

# Ver resultados en terminal
# ✅ PASS indica éxito
# ❌ FAIL indica error con detalles
```

### Paso 8.3: Interpretar Errores Comunes

#### Error 404 - Not Found
```
Expected: 200, Received: 404
```
**Causas**:
- Ruta mal definida (typo en URL)
- Middleware `checkEntityExists` fallando
- Orden incorrecto de rutas (ruta específica después de genérica)

#### Error 422 - Unprocessable Entity
```
Expected: 200, Received: 422
Validation errors: [...]
```
**Causas**:
- Faltan validaciones requeridas
- Validación personalizada fallando
- Formato de datos incorrecto

#### Error 401/403 - Unauthorized/Forbidden
```
Expected: 200, Received: 401
```
**Causas**:
- Falta middleware `isLoggedIn`
- Falta middleware `hasRole('owner')`
- Falta middleware de ownership

#### Error 500 - Internal Server Error
```
Expected: 200, Received: 500
```
**Causas**:
- Error en lógica del controlador
- Query SQL mal formada
- Modelo o migración con errores
- Relación no definida en `associate()`

### Paso 8.4: Debugging Estratégico

#### 1. Revisar Logs del Servidor
```bash
# Terminal donde corre npm run start:backend
# Buscar stack traces con detalles del error
```

#### 2. Usar Console.log Estratégicos
```javascript
// En controlador
console.log('req.params:', req.params)
console.log('req.body:', req.body)
console.log('req.user:', req.user)
console.log('Query result:', result)
```

#### 3. Verificar SQL Generado
```javascript
// En modelos/controllers
const result = await Model.findAll({...})
console.log(result.toJSON()) // Ver datos completos
```

#### 4. Probar Endpoints con Thunder Client / Postman
- GET: Sin autenticación para públicos
- POST/PUT/DELETE: Añadir header `Authorization: Bearer <token>`

---

## 📚 CHECKLIST FINAL PRE-ENTREGA

### Archivos Modificados/Creados
```
✅ Migraciones:
   - /src/database/migrations/*-create-nueva-entidad.js
   - /src/database/migrations/*-create-product.js (modificado)

✅ Modelos:
   - /src/models/NuevaEntidad.js
   - /src/models/Product.js (modificado)
   - /src/models/models.js (import y export)

✅ Rutas:
   - /src/routes/NuevaEntidadRoutes.js
   - /src/routes/RestaurantRoutes.js (modificado si aplica)

✅ Validaciones:
   - /src/controllers/validation/NuevaEntidadValidation.js
   - /src/controllers/validation/ProductValidation.js (modificado)

✅ Controladores:
   - /src/controllers/NuevaEntidadController.js
   - /src/controllers/RestaurantController.js (modificado si aplica)

✅ Tests:
   - npm run test:backend → TODOS PASAN
```

### Pre-Entrega
```
✅ Todos los tests pasan
✅ No hay console.log de debugging en código final
✅ Código formateado correctamente
✅ Borrar node_modules antes de comprimir
✅ Crear ZIP del proyecto completo
✅ Verificar que el ZIP contiene tu solución
✅ Avisar al profesor antes de subir
✅ Esperar confirmación del profesor
✅ Subir a plataforma y esperar enlace del ZIP
✅ Descargar ZIP de la plataforma y verificar
✅ Enviar examen
```

---

## 🔍 PATRONES Y ANTI-PATRONES

### ✅ HACER

#### En Migraciones
```javascript
// Respetar onDelete según lógica de negocio
onDelete: 'CASCADE'   // Hijo depende del padre (Schedule → Restaurant)
onDelete: 'SET NULL'  // Relación opcional (Product → Schedule)
```

#### En Modelos
```javascript
// Definir relaciones en AMBOS modelos
// En Schedule.js
Schedule.hasMany(models.Product, { foreignKey: 'scheduleId', as: 'products' })

// En Product.js
Product.belongsTo(models.Schedule, { foreignKey: 'scheduleId', as: 'schedule' })
```

#### En Rutas
```javascript
// Orden correcto de middlewares
.post(
  isLoggedIn,        // Auth primero
  hasRole('owner'),  // Luego permisos
  checkEntityExists, // Luego existencia
  checkOwnership,    // Luego propiedad
  Validations,       // Luego validaciones
  handleValidation,  // Procesar errores
  Controller.action  // Por último acción
)
```

#### En Validaciones
```javascript
// Campos opcionales correctamente marcados
check('campoOpcional')
  .optional({ nullable: true, checkFalsy: true })
  .isInt()
  .toInt()
  .custom(validacionPersonalizada)
```

#### En Controladores
```javascript
// Asignar FK desde params, no desde body
const create = async (req, res) => {
  const entity = Model.build(req.body)
  entity.parentId = req.params.parentId // ✅ Desde URL
  await entity.save()
}
```

### ❌ NO HACER

#### En Migraciones
```javascript
// ❌ No olvidar referencias
foreignKeyId: {
  type: Sequelize.INTEGER
  // FALTA: references, onDelete
}

// ❌ No crear tabla hija antes que padre
// INCORRECTO: Products (20210718065005) antes que Schedules (20210718065006)
```

#### En Modelos
```javascript
// ❌ No olvidar exportar en models.js
import loadScheduleModel from './Schedule.js'
const Schedule = loadScheduleModel(...)
// FALTA: Añadir Schedule a 'export { ... }'

// ❌ No definir relación solo en un lado
// Si Schedule hasMany Product, Product DEBE tener belongsTo Schedule
```

#### En Rutas
```javascript
// ❌ No invertir orden de rutas
app.route('/restaurants/:restaurantId')        // Primero genérica
app.route('/restaurants/:restaurantId/active') // Nunca se alcanza

// ❌ No omitir middlewares
.post(
  ScheduleValidation.create,
  handleValidation,
  ScheduleController.create
)
// FALTA: isLoggedIn, hasRole, checkEntityExists, checkOwnership
```

#### En Validaciones
```javascript
// ❌ No usar .exists() para campos opcionales
check('scheduleId')
  .exists() // ← MAL: marca como requerido
  .optional() // ← Contradictorio

// ✅ CORRECTO:
check('scheduleId')
  .optional({ nullable: true, checkFalsy: true })
```

#### En Controladores
```javascript
// ❌ No confiar en FK del body (seguridad)
const create = async (req, res) => {
  const schedule = await Schedule.create(req.body) // ← Vulnerable
  // Usuario podría enviar restaurantId diferente
}

// ✅ CORRECTO:
const create = async (req, res) => {
  const schedule = Schedule.build(req.body)
  schedule.restaurantId = req.params.restaurantId // ← De la URL verificada
  await schedule.save()
}
```

---

## 🧠 TIPS PARA EL EXAMEN

### Gestión del Tiempo
1. **Minuto 0-5**: Leer enunciado y anotar requisitos clave
2. **Minuto 5-15**: Migraciones y modelos
3. **Minuto 15-25**: Rutas básicas
4. **Minuto 25-40**: Validaciones
5. **Minuto 40-60**: Controladores CRUD
6. **Minuto 60-75**: Funcionalidades adicionales complejas
7. **Minuto 75-85**: Testing y corrección de errores
8. **Minuto 85-90**: Limpieza y preparación de entrega

### Estrategia de Resolución
1. **Implementar en orden**: Migraciones → Modelos → Rutas → Validaciones → Controladores
2. **Probar progresivamente**: Ejecutar `npm run test:backend` después de cada sección
3. **No bloquear**: Si algo no funciona, comentar y continuar, volver después
4. **Reutilizar código**: Copiar estructura de entidades similares (Restaurant, Product)

### Errores Comunes a Evitar
- Olvidar añadir nuevo modelo a `/src/models/models.js`
- No respetar orden de timestamps en migraciones
- Definir rutas específicas después de genéricas
- Olvidar middlewares de autenticación/autorización
- No validar que FK pertenezcan a la entidad correcta

### Recursos Durante el Examen
- **Este README**: Consultar plantillas y ejemplos
- **Código existente**: `RestaurantController`, `ProductController` como referencia
- **Tests**: Leer `/tests/e2e/*.test.js` para entender estructura esperada
- **Logs**: Revisar terminal del backend para errores SQL

---

## 📖 REFERENCIA RÁPIDA SEQUELIZE

### Queries Básicas
```javascript
// Buscar todos con filtro
Model.findAll({ where: { campo: valor } })

// Buscar por clave primaria
Model.findByPk(id)

// Crear
const entity = Model.build(data)
entity.campo = valor
await entity.save()

// O directo:
await Model.create(data)

// Actualizar
await Model.update(data, { where: { id: entityId } })

// Eliminar
await Model.destroy({ where: { id: entityId } })
```

### Relaciones en Queries
```javascript
// Include simple
Model.findByPk(id, {
  include: { model: Related, as: 'related' }
})

// Include múltiple
Model.findAll({
  include: [
    { model: Related1, as: 'related1' },
    { model: Related2, as: 'related2' }
  ]
})

// Include anidado
Model.findByPk(id, {
  include: {
    model: Parent,
    as: 'parent',
    include: { model: GrandParent, as: 'grandparent' }
  }
})
```

### Filtros Complejos
```javascript
import { Op } from 'sequelize'

Model.findAll({
  where: {
    [Op.and]: [
      { campo1: valor1 },
      { campo2: { [Op.gt]: valor2 } }
    ]
  }
})
```

---

## 🎓 RESUMEN EJECUTIVO

### Flujo de Trabajo
```
1. ANALIZAR requisitos y diagrama
2. CREAR migraciones (nueva tabla + modificar existentes)
3. CREAR/MODIFICAR modelos (definir relaciones)
4. REGISTRAR modelo en models.js
5. CREAR rutas con middlewares correctos
6. IMPLEMENTAR validaciones (formato + negocio)
7. IMPLEMENTAR controladores (CRUD + custom)
8. EJECUTAR tests y corregir errores
9. VERIFICAR y entregar
```

### Comandos Esenciales
```bash
npm run migrate:backend   # Recrear BD
npm run start:backend     # Iniciar servidor
npm run test:backend      # Ejecutar tests
```

### Archivos Clave por Ejercicio
| Ejercicio | Archivos a Modificar/Crear |
|-----------|----------------------------|
| 1. Migraciones | `/src/database/migrations/*.js` |
| 2. Modelos | `/src/models/*.js` + `models.js` |
| 3. Rutas | `/src/routes/*Routes.js` |
| 4. Validaciones | `/src/controllers/validation/*Validation.js` |
| 5. Controladores | `/src/controllers/*Controller.js` |
| 6-7. Validaciones FK | `/src/controllers/validation/ProductValidation.js` |
| 8. Funcionalidad Custom | `/src/controllers/RestaurantController.js` + Routes |

---

**¡Buena suerte en el examen! 🚀**

*Recuerda: Mantén la calma, sigue el orden establecido, y confía en tu preparación.*
