# SIOAL — Sistema de organización de almacén y pronóstico de carga

Prototipo desarrollado para la sede de **Juliaca** de una empresa de transporte de carga (Transportes Ochoa).
Une dos partes:

| Parte | Carpeta | Qué hace |
|---|---|---|
| **Aplicación web** | [`sioal-web/`](sioal-web/) | Registro y ubicación de encomiendas por zonas A/B/C, clasificación ABC, optimización de ubicaciones, alertas, códigos QR y **pronóstico de la carga semanal** a partir de los Excel de TITANICSOFT. |
| **Modelos en Python** | [`modelo/`](modelo/) | Réplica independiente de los modelos de la app para **verificar** que calcula bien y **evaluar** el error con los datos reales. |

> Aplicación publicada: **https://gestion-sioal.netlify.app**

---

## 1. Ver la aplicación

La app es **estática**: HTML, CSS y JavaScript puro, sin servidor, sin base de datos y sin instalación.

**Opción A — En línea:** abrir https://gestion-sioal.netlify.app

**Opción B — En tu computadora:**

1. Descarga el repositorio (botón verde **Code → Download ZIP**) o clónalo:
   ```bash
   git clone <URL-del-repositorio>
   ```
2. Abre `sioal-web/index.html` con doble clic en Chrome o Edge. Listo.

   Para usar la cámara del lector QR conviene servirla en local:
   ```bash
   cd sioal-web
   python -m http.server 8080      # luego abrir http://localhost:8080
   ```

### Cómo se guardan los datos

- Todo se guarda en el **navegador** (`localStorage`), no en un servidor.
- Por eso, **cada computadora y cada navegador tienen sus propios datos**. Si abres la app en otra
  computadora verás los datos de demostración hasta que importes tus Excel.
- Para pasar los datos de una computadora a otra: **Configuración → exportar respaldo (JSON)** y luego
  importarlo en la otra.
- La primera vez la app muestra **datos de demostración** (sintéticos). Aparece el aviso “DATOS DEMO”.

### Cargar los datos reales (pronóstico)

1. Ir a **Entrenamiento IA → 01 Datos y Excel**.
2. **Elegir archivos** y seleccionar a la vez las exportaciones de TITANICSOFT (Ctrl + clic).
   Los manifiestos repetidos se cuentan una sola vez.
3. Pulsar **Ejecutar entrenamiento**. La app limpia los datos, arma las semanas, compara los métodos y
   pronostica las 4 semanas siguientes.

Los Excel se leen **dentro del navegador**: no se envían a internet.

**Avisos que pueden aparecer (no son errores):**

| Aviso | Significado | Qué hacer |
|---|---|---|
| “Faltan semanas recientes” | Los Excel terminan antes de la semana pasada. | Importar la exportación más reciente. |
| “Preliminar” | Hay menos de 24 semanas para validar. | Seguir cargando datos cada semana. |
| “Pesos atípicos en cuarentena” | Manifiestos con kg por bulto imposibles (ej. 9 740 kg/bulto). No entran al pronóstico. | Revisarlos con la empresa. |

---

## 2. Correr los modelos en Python

### Requisitos
- Python 3.10 o superior
- Los Excel de TITANICSOFT (**no están en el repositorio** porque contienen datos personales y
  comerciales; pedirlos a la empresa).

### Pasos

```bash
cd modelo
python -m venv .venv
# Windows:  .venv\Scripts\activate      |  Linux / Mac:  source .venv/bin/activate
pip install -r requirements.txt
```

Copia los Excel en una carpeta **`Datos/`** al lado de `modelo/` (ver [`Datos/LEEME.md`](Datos/LEEME.md)):

```
SIOAL/
├── Datos/            ← TITANICSOFT(12).xlsx, TITANICSOFT(13).xlsx, ...
├── modelo/
└── sioal-web/
```

Luego, desde `modelo/`:

| Comando | Qué hace | Resultado |
|---|---|---|
| `python preparar_datos_reales.py` | Lee y revisa los Excel: limpieza, observaciones y series semanales. | `datos/reales/` |
| `python entrenar_modelo.py` | Entrena igual que la app (partición, 6 métodos, elección) y pronostica 4 semanas. | `resultados/entrenamiento/` |
| `python caso_real_corto.py` | Error de cada método en las 15 semanas reales (origen móvil, 11 pronósticos). | `resultados/caso_real/wape_caso_real.csv` |
| `python proyeccion_intrasemana.py` | Corrección del total de la semana con lo ya despachado cada día. | `resultados/caso_real/proyeccion_intrasemana.csv` |

### Verificar que la app y Python calculan lo mismo (opcional)

Requiere Node.js 18+:

```bash
cd modelo
npm install jsdom@24
node scripts/verificar_con_app.js ../sioal-web datos/reales   # ejecuta el código real de la app sin navegador
python entrenar_modelo.py                                     # compara: debe decir "La app y Python entrenan exactamente lo mismo."
node scripts/medir_velocidad_app.js ../sioal-web '{"real_4":["../Datos/TITANICSOFT(12).xlsx","../Datos/TITANICSOFT(13).xlsx","../Datos/TITANICSOFT(14).xlsx","../Datos/TITANICSOFT(15).xlsx"]}'
```

---

## 3. Modelos que usa el sistema

| Problema | Modelo | Fórmula | Decisión |
|---|---|---|---|
| No se sabe a tiempo qué carga se recibió | Tiempos de recepción y ley de Little | `t = registro − llegada`, `L = λ·W` | Qué recepciones revisar; cuánto espacio libera despachar antes |
| Almacén sin zonas | Clasificación ABC + asignación entera | 20 % más pesado = A, 30 % = B, resto = C; `min Σ prioridad·distancia` sin exceder área ni peso | En qué zona va cada bulto |
| No se pronostica la carga | Media móvil, suavización exponencial (α = 0,35 y α óptimo) y su combinación | `ℓt = α·yt + (1−α)·ℓt−1` | Cuántos camiones propios reservar y de terceros contratar |
| Personal de descarga “a ojo” | Dimensionamiento por productividad | `N = ⌈bultos / (productividad × horas objetivo)⌉` | Cuántas personas enviar |

**Resultados con los datos reales (15 semanas, ago.–nov. 2026):** el pronóstico combinado tuvo un error
(WAPE) de **6,59 %** en el peso semanal, frente a **13,98 %** de repetir la semana anterior. En la pantalla
de la app, el método elegido (α óptimo) tiene 7,5 % de error en validación frente a 11 % del último valor.

---

## 4. Estructura del repositorio

```
├── README.md                  ← este archivo
├── Datos/LEEME.md             ← dónde poner los Excel (no se suben)
├── sioal-web/                 ← aplicación web (ver sioal-web/README.md y sioal-web/docs/)
│   ├── index.html
│   ├── assets/                   estilos e imagen del plano del almacén
│   ├── docs/                     arquitectura, pantallas y modelo de datos
│   └── src/                      código por capas: config, core, domain, services, modules
└── modelo/                    ← modelos en Python (ver modelo/README.md)
    ├── src/                      lectura de datos, modelos, métricas y evaluación
    ├── scripts/                  verificación y velocidad de la app con Node.js
    ├── preparar_datos_reales.py
    ├── entrenar_modelo.py
    ├── caso_real_corto.py
    ├── proyeccion_intrasemana.py
    └── resultados/               resultados ya calculados (entrenamiento, caso real, velocidad)
```

## 5. Publicar la app en Netlify

1. Entrar a https://app.netlify.com → **Add new site → Deploy manually**.
2. Arrastrar **solo la carpeta `sioal-web`**. No necesita comando de compilación.

## 6. Privacidad

- **No subir** los Excel de TITANICSOFT ni la carpeta `modelo/datos/`: contienen conductores, placas e importes.
  El archivo `.gitignore` ya los excluye.
- La app no envía datos a ningún servidor; todo se procesa en el navegador de cada usuario.
