# Sistema de Control de Nómina (`nomina-app`)

**Tema Asignado:** Sistema de Control de Nómina (`nomina-app`)

**Autor:** Mateo Nicolay Rogeron Maila ([KaZuhiraXOF](https://github.com/KaZuhiraXOF))

**Institución:** Universidad Tecnológica UTE — Escuela de Tecnologías (Desarrollo de Software)

## Características Principales

* **Modelo de Datos Relacional:** Cuatro entidades principales (`Departamento`, `Empleado`, `ConceptoSalarial` y `RolPago`) conectadas mediante relaciones `ForeignKey` (con protección `PROTECT` contra borrado accidental) y `ManyToManyField` para la asignación múltiple de rubros salariales.

* **Cálculo Contable Automatizado:** Determinación dinámica de ingresos adicionales, descuentos y neto a pagar a partir del salario base del empleado y los conceptos vinculados al rol.

* **Autenticación y Seguridad:** Vistas protegidas mediante `@login_required`, registro público de nuevos usuarios y control estricto de permisos por grupo (`administradores` vs. `consultores`) y por propiedad del registro.

* **Panel Administrativo Configurado:** Registro de los cuatro modelos en `/admin/` con búsqueda (`search_fields`), filtros laterales (`list_filter`) y columnas personalizadas (`list_display`).

* **Poblado Automatizado (`poblar_datos`):** Comando personalizado de Django que configura grupos, permisos, superusuario, usuarios de prueba y registros iniciales idempotentes en un solo paso.

## Modelo de Datos (Tablas y Relaciones)

El sistema se estructura sobre cuatro tablas principales en la base de datos relacional, vinculadas además con la tabla nativa de usuarios (`auth_user`) para la auditoría de creación (`creado_por`):

1. **`Departamento` (`nomina_departamento`):** Almacena las áreas funcionales o centros de costo de la empresa (`codigo`, `nombre`, `ubicacion`).

2. **`Empleado` (`nomina_empleado`):** Registra la ficha laboral del personal (`cedula`, `nombres`, `apellidos`, `correo`, `fecha_ingreso`, `salario_base`, `estado`).

3. **`ConceptoSalarial` (`nomina_conceptosalarial`):** Define los rubros económicos aplicables a la nómina (`nombre`, `tipo` \[`INGRESO` o `DESCUENTO`\], `monto_fijo`, `es_recurrente`).

4. **`RolPago` (`nomina_rolpago`):** Representa la liquidación salarial periódica emitida a un trabajador (`periodo_inicio`, `periodo_fin`, `fecha_emision`, `total_ingresos`, `total_descuentos`, `neto_a_pagar`, `estado_pago`).

### Relaciones entre entidades

* **`Departamento` (1)** $\rightarrow$ **(N) `Empleado` (`ForeignKey`, `on_delete=PROTECT`):** Un departamento agrupa varios empleados, pero cada empleado pertenece a un único departamento. La restricción `PROTECT` impide eliminar departamentos que tengan personal asignado.

* **`Empleado` (1)** $\rightarrow$ **(N) `RolPago` (`ForeignKey`, `on_delete=PROTECT`):** Un empleado acumula múltiples roles de pago a lo largo del tiempo, mientras que cada rol corresponde a un solo trabajador.

* **`RolPago` (N)** $\leftrightarrow$ **(M) `ConceptoSalarial` (`ManyToManyField`):** Un rol de pago puede incluir múltiples ingresos y descuentos simultáneamente, y un mismo concepto salarial puede aplicarse a distintos roles de pago (generando la tabla intermedia `nomina_rolpago_conceptos`).

* **`auth_user` (1)** $\rightarrow$ **(N) Todas las entidades (`ForeignKey` `creado_por`):** Registra el autor de cada registro para validar las reglas de propiedad del grupo `consultores`.

## Arquitectura del Proyecto

```
nomina-app/
├── config/                      # Configuración principal de Django (settings, urls, wsgi)
├── nomina/
│   ├── forms/                   # Formularios ModelForm y registro de usuarios
│   ├── management/
│   │   └── commands/
│   │       └── poblar_datos.py  # Comando de inicialización de BD y permisos
│   ├── models/                  # Modelos independientes (Departamento, Empleado, ConceptoSalarial, RolPago)
│   ├── static/nomina/css/       # Hoja de estilos propia (estilos.css)
│   ├── templates/nomina/        # Plantillas HTML modulares (base, auth, conceptos, departamentos, empleados, roles)
│   ├── urls/                    # Enrutamiento desacoplado por módulo
│   ├── views/                   # Vistas CRUD con control de permisos y mensajes
│   └── admin.py                 # Configuración del panel administrativo
├── manage.py
├── requirements.txt
└── README.md
```

## Requisitos Previos

* Python 3.12 o superior

* Git

* Entorno virtual (`venv`)

## Instalación y Ejecución Local (Desde Cero)

1. **Clonar el repositorio:**

   ```bash
   git clone https://github.com/KaZuhiraXOF/nomina-app.git
   cd nomina-app
   ```

2. **Crear y activar el entorno virtual:**

   * En Linux / macOS:

     ```bash
     python3 -m venv venv
     source venv/bin/activate
     ```

   * En Windows:

     ```bash
     python -m venv venv
     venv\Scripts\activate
     ```

3. **Instalar las dependencias:**

   ```bash
   pip install -r requirements.txt
   ```

4. **Aplicar las migraciones de la base de datos:**

   ```bash
   python manage.py migrate
   ```

5. **Cargar los datos de prueba, grupos y usuarios:**

   ```bash
   python manage.py poblar_datos
   ```

6. **Iniciar el servidor de desarrollo:**

   ```bash
   python manage.py runserver
   ```

7. **Acceder a la aplicación:**

   * Sistema principal (Login): `http://127.0.0.1:8000/`

   * Panel administrativo: `http://127.0.0.1:8000/admin/`

## Credenciales de Prueba y Permisos (RBAC)

El comando `python manage.py poblar_datos` genera automáticamente los siguientes usuarios, grupos y reglas de negocio:

| Usuario | Contraseña | Rol / Grupo | Permisos en el Sistema |
| :--- | :--- | :--- | :--- |
| **`admin`** | `Admin1234!` | Superusuario (`is_staff`, `is_superuser`) | Acceso total al panel `/admin/` y a todas las operaciones del sistema. |
| **`jefe_nomina`** | `Nomina1234!` | `administradores` | Ver, crear, editar y eliminar **Departamentos**, **Conceptos Salariales**, **Empleados** y **Roles de Pago** (propios y de terceros). |
| **`asistente_rrhh`** | `Consulta1234!` | `consultores` | **Solo lectura** en Departamentos y Conceptos Salariales. Puede crear, editar y eliminar **únicamente sus propios** Empleados y Roles de Pago (`403 Forbidden` sobre registros ajenos). |

*(Nota: Los usuarios creados desde el formulario público `/cuenta/registro/` ingresan sin grupo asignado, contando únicamente con permisos de visualización hasta ser categorizados por un administrador).*

## Rutas Principales

| Módulo | Listado | Creación | Edición / Eliminación |
| :--- | :--- | :--- | :--- |
| **Roles de Pago** | `/` | `/crear/` | `/<pk>/editar/` · `/<pk>/eliminar/` |
| **Empleados** | `/empleados/` | `/empleados/crear/` | `/empleados/<pk>/editar/` · `/empleados/<pk>/eliminar/` |
| **Departamentos** | `/departamentos/` | `/departamentos/crear/` | `/departamentos/<pk>/editar/` · `/departamentos/<pk>/eliminar/` |
| **Conceptos Salariales** | `/conceptos/` | `/conceptos/crear/` | `/conceptos/<pk>/editar/` · `/conceptos/<pk>/eliminar/` |
| **Autenticación** | `/cuenta/login/` | `/cuenta/registro/` | `/cuenta/logout/` |