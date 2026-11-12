# bonuslife-backups

Repositorio **público** dedicado solo a los backups de bonus.life (esquema de
`work/backup-scripts/` y `supabase-backups-guia.docx`, igual que prestigiobb y phonecenterbb).

- Es público para tener minutos ilimitados de GitHub Actions.
- Lo único público es este workflow. **Los datos nunca se guardan en el repo**: van directo a un
  bucket **privado** de Backblaze B2.
- Las credenciales viven en *GitHub Secrets*. GitHub las enmascara en los logs y nunca las expone.
- El workflow no imprime datos en los logs: solo la cantidad de tablas y archivos.

## Hoy: keep-alive de staging (activo desde el 11/11/2026)

Repo: https://github.com/bonuslife-gh/bonuslife-backups. El workflow `.github/workflows/keep-alive.yml` (fuente
versionada acá, en `ops/bonuslife-backups/`, con su test de arquitectura) corre todos los días a las 03:00 UTC y se
puede disparar a mano:

1. Hace `SELECT 1` en `bonuslife-staging` con el rol `bonuslife_keepalive`, que no tiene permisos sobre ninguna
   tabla (`docs/plataforma/base-de-datos.md`). El plan Free de Supabase pausa el proyecto a los 7 días sin actividad.
2. Deja un commit de latido (`latido.txt`): GitHub desactiva los cron de un repo público tras 60 días sin commits.

Secret: `STAGING_KEEPALIVE_URL` = pooler en modo sesión (puerto 5432), usuario `bonuslife_keepalive.<ref>`,
`?sslmode=require`. La contraseña vive en `.env.supabase` del dev (`BONUSLIFE_KEEPALIVE_PASSWORD_STAGING`).

El backup diario a Backblaze de abajo se activa con la base productiva (P17-2); hasta entonces este repo sólo hace el
keep-alive.

## Backup productivo (P17): qué va a hacer todos los días a las 00:00 de Argentina

1. Exporta la **estructura** del schema `public` (`schema.sql`, con `pg_dump --schema-only`) y **cada
   tabla a CSV**.
2. Sube todo a `B2_BUCKET_NAME/AAAA-MM-DD/`.
3. Hace un `SELECT 1` para que Supabase no pause el proyecto por inactividad.

## Puesta en marcha

1. **Backblaze B2**: crear un bucket **Private** `bonuslife-backups`.
   - Activar *Default Encryption* (SSE-B2, gratis).
   - Crear una *Application Key* **restringida a ese bucket**, con permisos Read and Write y sin "List
     All Bucket Names". Guardar `keyID` y `applicationKey` en el gestor de contraseñas: Backblaze los
     muestra una sola vez. **Nunca guardarlos en archivos de texto.**
2. **Retención** con *Lifecycle Settings* del bucket ("Keep only the last version" + borrar archivos
   con más de N días), según el plan de mantenimiento vigente:

   | Plan | Retención |
   |---|---|
   | Sin plan | 7 días |
   | Esencial | 30 días |
   | Crecimiento | 90 días |
   | Premium | 365 días |

3. **GitHub**: crear el repo público `bonuslife-backups` en la cuenta dedicada del proyecto, copiar
   este README y `.github/workflows/daily-backup.yml`, y cargar los secrets:

   | Secret | Valor |
   |---|---|
   | `SUPABASE_DB_URL` | Connection string de Supabase (URI) |
   | `B2_KEY_ID` | keyID de la Application Key |
   | `B2_APP_KEY` | applicationKey |
   | `B2_ENDPOINT` | Endpoint del bucket (`s3.<región>.backblazeb2.com`) |
   | `B2_BUCKET_NAME` | `bonuslife-backups` |

4. Correrlo a mano una vez (*Actions → Run workflow*) y verificar la carpeta del día en el bucket.

## Restauración (probarla al menos una vez antes de producción)

1. Descargar la carpeta del día desde B2.
2. Crear una base vacía y aplicar `schema.sql`.
3. Importar los CSV respetando las claves foráneas: primero las tablas sin dependencias (users,
   brands, product_titles, product_types, flavors, weights, customers), después products,
   product_variants, sales, sale_items, sale_payments, orders, etc. Usar
   `\COPY tabla FROM 'tabla.csv' WITH CSV HEADER`.
4. Verificar conteos por tabla contra el día del backup y loguear en el sistema con un usuario admin.
