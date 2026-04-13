## Paso a paso: captura en servidor Windows, análisis en local con Visual Studio

El truco es usar **standalone profiling tools** que Microsoft distribuye por separado, sin necesidad de instalar Visual Studio completo en el servidor.

---

### Lo que necesitas descargar

**En el servidor Windows** (sin VS):
Descarga los **"Standalone Profiler Tools"** que vienen incluidos en el paquete de **Remote Tools for Visual Studio**.

La URL oficial según tu versión de VS local:
- VS 2022 → `https://aka.ms/vs/17/release/RemoteTools.amd64ret.enu.exe`


La versión de los Remote Tools **debe coincidir** con tu Visual Studio local. Instala solo los componentes mínimos — no necesitas el Remote Debugger activo, solo los binarios del profiler.

**En tu máquina local**: tu Visual Studio instalado (2017/2019/2022) ya incluye todo para abrir los ficheros `.vsp` o `.diagsession`.

---

### Localizar los binarios del profiler tras la instalación

Tras instalar los Remote Tools en el servidor, los ejecutables del profiler estarán en:

```
C:\Program Files\Microsoft Visual Studio <version>\Team Tools\Performance Tools\
```

Los archivos clave son:
```
VSPerfCmd.exe      ← lanza y controla la sesión
VSPerfMon.exe      ← monitor auxiliar
VSPerfCLREnv.cmd   ← prepara variables de entorno para .NET
VSPerfReport.exe   ← opcional, genera informes desde línea de comandos
```

---

### Paso 1 — Preparar el entorno en el servidor

Abre una **cmd como Administrador** y ejecuta:

```cmd
REM Navegar a la carpeta del profiler
cd "C:\Program Files\Microsoft Visual Studio\Team Tools\Performance Tools"

REM Configurar variables de entorno para profiling de .NET
VSPerfCLREnv.cmd /sampleon
```

Si vas a hacer profiling de **memoria** (allocations), usa en su lugar:
```cmd
VSPerfCLREnv.cmd /traceon
```

---

### Paso 2 — Identificar el PID del proceso w3wp.exe

```cmd
REM Lista los worker processes de IIS con su PID y application pool
%windir%\system32\inetsrv\appcmd list wp

REM Alternativa con tasklist filtrando por nombre
tasklist /fi "imagename eq w3wp.exe" /fo list
```

Anota el **PID** del pool que te interesa. Si solo hay uno, fácil.

---

### Paso 3 — Iniciar la sesión de profiling (attach al proceso)

```cmd
REM CPU Sampling (bajo overhead, recomendado para producción)
VSPerfCmd.exe /start:sample /output:C:\profiling\resultado.vsp

REM Attach al proceso w3wp (sustituye 1234 por el PID real)
VSPerfCmd.exe /attach:1234
```

Si prefieres **tracing** (más detallado pero más overhead):
```cmd
VSPerfCmd.exe /start:trace /output:C:\profiling\resultado.vsp
VSPerfCmd.exe /attach:1234
```

Crea la carpeta de salida antes si no existe:
```cmd
mkdir C:\profiling
```

---

### Paso 4 — Reproducir la carga

Con la sesión activa, ejecuta las peticiones o escenarios que quieres analizar — navega la web, lanza el test de carga, reproduce el comportamiento lento. El tiempo recomendado es entre **30 y 120 segundos** de carga real.

---

### Paso 5 — Detener y cerrar la sesión

```cmd
REM Detach del proceso (sin matarlo)
VSPerfCmd.exe /detach

REM Cerrar la sesión y escribir el fichero final
VSPerfCmd.exe /shutdown
```

Esto genera el fichero `C:\profiling\resultado.vsp` con todos los datos capturados.

---

### Paso 6 — Limpiar las variables de entorno

```cmd
VSPerfCLREnv.cmd /off
```

Importante hacerlo para que el entorno vuelva a su estado normal.

---

### Paso 7 — Copiar el fichero al local

```cmd
REM Desde tu máquina local (PowerShell o cmd)
REM Puedes usar SCP, RDP con copia, o un share de red
copy \\servidor\C$\profiling\resultado.vsp C:\local\resultado.vsp
```

O simplemente lo descargas via RDP arrastrando el fichero.

---

### Paso 8 — Abrir en Visual Studio local

1. Abre Visual Studio
2. **File → Open → File** y selecciona el `.vsp`
3. Visual Studio abre el **Performance Report** automáticamente

Las vistas más útiles dentro del reporte:

| Vista | Para qué sirve |
|---|---|
| **Summary** | Visión general, top hot paths |
| **Call Tree** | Árbol de llamadas con tiempo inclusivo/exclusivo |
| **Functions** | Ranking de funciones por CPU consumido |
| **Caller/Callee** | Quién llama a qué función y con qué coste |
| **Marks** | Si añadiste marcas temporales durante la sesión |

---

### Truco: añadir PDBs para ver nombres de métodos correctamente

Si el `.vsp` muestra direcciones de memoria en vez de nombres de métodos, necesitas que Visual Studio encuentre los `.pdb` de tu aplicación. Ve a:

**Performance Report → Actions → Set Symbol Paths**

Y apunta a la carpeta donde tengas los `.pdb` de tu build. Si son los PDBs del runtime de .NET, configura el servidor de símbolos de Microsoft:

```
srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```