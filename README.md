# Banco de Semillas del Desierto Florido — Prototipo en contenedores

Guía para la Unidad 1 / Actividad 1 de Computación en la Nube: MariaDB +
WordPress + OwnCloud sobre tu VM Debian, con acceso desde el host, desde una
VM cliente y desde un equipo externo.

## 1. Qué cubre este prototipo (y qué se ignora a propósito)

La consigna pide levantar **solo** MariaDB, WordPress y OwnCloud, e ignorar
el resto de los requerimientos del caso CEAZA/INIA. Con esos tres
contenedores se cubren específicamente:

| Requerimiento del caso | Cómo se cubre |
|---|---|
| 3. Almacenamiento y gestión de datos | MariaDB (motor de base de datos) + OwnCloud (archivos: fotos de muestras, fichas de especies, informes) |
| 2. Soporte para aplicaciones web | WordPress como sitio institucional que explica CEAZA/INIA y el proyecto del Banco de Semillas |

Todo lo demás del enunciado (IoT, dashboards, búsqueda avanzada,
comunicación en tiempo real, alta disponibilidad, etc.) queda **fuera de
alcance** de este prototipo, tal como indica la actividad.

## 2. Arquitectura

```
                    VirtualBox NAT Network (10.0.3.0/24)
                    ┌─────────────────────────────────────┐
Equipo host  ──┐    │   VM servidor Debian (10.0.3.201)    │
Equipo externo─┼──► │  ┌────────┐ ┌───────────┐ ┌────────┐ │
VM cliente   ──┘    │  │WordPress│ │ OwnCloud  │ │ MariaDB│ │
                    │  │ :8081  │ │  :8082    │ │ :13306 │ │
                    │  └────────┘ └─────┬─────┘ └────────┘ │
                    │                    │  Redis (caché)  │
                    └─────────────────────────────────────┘
```

Un solo contenedor MariaDB aloja dos bases de datos: `wordpress` y
`owncloud`, cada una con su propio usuario. OwnCloud usa además un
contenedor Redis (recomendado oficialmente para el motor de caché/bloqueo
de archivos).

## 3. Estructura de archivos

```
desierto-florido-cloud/
├── docker-compose.yml
├── .env.example        → cópialo como .env y edítalo
└── db/
    └── init/
        └── 01-owncloud-db.sql   → crea la BD y el usuario de OwnCloud
```

## 4. Despliegue paso a paso

En tu VM Debian, con Docker y Docker Compose ya instalados:

```bash
# 1. Copia esta carpeta a la VM (scp, git, o pegando los archivos)
cd desierto-florido-cloud

# 2. Prepara las variables de entorno
cp .env.example .env
nano .env
#   - Cambia todas las contraseñas
#   - SERVER_IP: confirma que sea la IP real de tu VM (10.0.3.201 según
#     tu tabla de reenvío de puertos actual)
#   - HOST_MACHINE_IP: la IP de tu equipo host (o la IP/dominio con el
#     que accederás desde un equipo externo)

# 3. Si cambiaste OC_DB_PASSWORD, refleja la misma clave aquí:
nano db/init/01-owncloud-db.sql

# 4. Levanta los servicios
docker compose up -d

# 5. Verifica que todo esté sano (puede tardar 1-2 minutos)
docker compose ps
docker compose logs --follow owncloud    # espera "Starting apache daemon..."
```

## 5. Reenvío de puertos

### 5.1 Firewall de la VM (si usas ufw)

```bash
sudo ufw allow 8081/tcp   # WordPress
sudo ufw allow 8082/tcp   # OwnCloud
sudo ufw allow 13306/tcp  # MariaDB (ya deberías tenerlo si lo usas hoy)
```

### 5.2 VirtualBox — tabla "Reenvío de puertos" de tu NAT Network

Agrega dos filas nuevas junto a las que ya tienes (MariaDb, NodeRED, SSH),
siguiendo el mismo patrón que ya usaste (puerto de host = 10000 + puerto
invitado):

| Nombre | Protocolo | IP anfitrión | Puerto anfitrión | IP invitado | Puerto invitado |
|---|---|---|---|---|---|
| WordPress | TCP | *(vacío)* | 18081 | 10.0.3.201 | 8081 |
| OwnCloud | TCP | *(vacío)* | 18082 | 10.0.3.201 | 8082 |

## 6. Acceso desde los tres puntos que pide la actividad

| Desde dónde | WordPress | OwnCloud |
|---|---|---|
| La propia VM servidor | `http://localhost:8081` | `http://localhost:8082` |
| VM cliente (misma NAT Network) | `http://10.0.3.201:8081` | `http://10.0.3.201:8082` |
| Equipo host de VirtualBox | `http://localhost:18081` | `http://localhost:18082` |
| Equipo externo | `http://<IP_del_host>:18081` | `http://<IP_del_host>:18082` |

Para el acceso externo, además del reenvío en VirtualBox necesitas que el
firewall del equipo host (y, si corresponde, el router de tu red) permitan
tráfico entrante hacia esos mismos puertos (18081/18082) apuntando a la IP
del equipo host.

**Nota sobre WordPress:** la primera vez que completes el asistente de
instalación (`/wp-admin/install.php`), hazlo desde la URL que vayas a usar
principalmente para la demostración/defensa (por ejemplo, la del equipo
host). WordPress guarda esa URL como "dirección del sitio"; si luego
necesitas acceder desde otra IP y ves problemas de redirección, puedes
corregirlo en **Ajustes → Generales**.

## 7. Primer arranque de WordPress (gestor de contenidos)

1. Entra a la URL de WordPress y completa el asistente: título del sitio,
   usuario y clave de administrador.
2. Estructura sugerida de páginas para cumplir "explicar a qué se dedica
   la institución y el proyecto":
   - **Inicio**: quiénes somos y qué es el Banco de Semillas del Desierto
     Florido.
   - **El proyecto**: resumen adaptado del caso — debe incluir estos
     datos concretos, que son los que suele pedir la pauta:
     - Banco gestionado por el Instituto de Investigaciones
       Agropecuarias (INIA), sede Intihuasi (Vicuña, región de
       Coquimbo).
     - Conserva más de 1.300 especies de flora nativa de Chile.
     - Condiciones de conservación: 15% de humedad relativa (HRe) y
       -20°C.
     - El "Banco de semillas para la conservación del desierto
       florido" es la continuación de ese proyecto, enfocada en
       especies endémicas del desierto florido (regiones de Atacama y
       Coquimbo).
   - **Almacenamiento**: enlace/explicación de que los archivos e
     informes de campo se gestionan en OwnCloud.
   - **Contacto**.
3. Copy sugerido para "Inicio" (ajústalo con tu nombre/curso y el
   logo/colores que definas):

   > **Banco de Semillas del Desierto Florido**
   > Proyecto desarrollado en colaboración con INIA Intihuasi (sede
   > Intihuasi, Vicuña) para la conservación y reproducción de especies
   > endémicas del desierto florido de las regiones de Atacama y
   > Coquimbo, dando continuidad al Banco Base de Semillas de INIA, que
   > ya conserva más de 1.300 especies de la flora nativa chilena bajo
   > condiciones controladas de 15% de humedad relativa y -20°C.
   > Coordinamos a investigadores y técnicos en la identificación de
   > especies, el registro de su ubicación geográfica y los ensayos de
   > germinación y reproducción, centralizando la información en una
   > plataforma segura de base de datos y almacenamiento en la nube.

4. Como esta es una actividad académica que usa el nombre de tu propio
   instituto (IP Santo Tomás) para representar a la organización que
   opera el sistema, la consigna pide que **tú definas** el logo y los
   colores — no existe una paleta "oficial" que debas replicar. Una
   opción coherente con el tema (desierto florido: tonos tierra + flores
   moradas/rosadas) es:
   - Terracota `#C97B4A`
   - Morado desierto florido `#8B5FBF`
   - Verde salvia `#7A8B6F`
   - Arena `#EDE0D0` (fondo)

## 8. Primer arranque de OwnCloud (almacenamiento tipo Drive)

1. Entra a la URL de OwnCloud e inicia sesión con `admin` y la clave que
   definiste en `OC_ADMIN_PASSWORD`.
2. Crea una estructura de carpetas que refleje los tres datos que el
   caso pide registrar explícitamente para cada muestra (especie,
   ubicación y prueba de reproducción, además de quién la tomó y su
   institución):
   - `Identificación de Especies/`
   - `Ubicaciones Geográficas/`
   - `Pruebas de Reproducción y Germinación/`
3. Crea al menos un usuario adicional (por ejemplo, un "investigador de
   campo") desde **Configuración → Usuarios**, para poder mostrar
   colaboración entre cuentas en tu defensa.

## 9. Notas de seguridad y troubleshooting

- Las claves de `OWNCLOUD_ADMIN_USERNAME`/`PASSWORD` solo se usan para el
  alta inicial; una vez que inicies sesión por primera vez puedes
  quitarlas del `docker-compose.yml` (ownCloud ya no vuelve a leerlas
  después del primer arranque).
- Si `docker compose ps` muestra `owncloud` reiniciándose, revisa
  `docker compose logs owncloud`: casi siempre es porque
  `OC_DB_PASSWORD` en `.env` no coincide con la clave del
  `01-owncloud-db.sql`.
- Si necesitas reiniciar todo desde cero (borra datos):
  `docker compose down -v`.
