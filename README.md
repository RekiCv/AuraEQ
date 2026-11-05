# AuraEQ

**Dale forma a tu sonido.**

AuraEQ es un ecualizador paramétrico de escritorio creado con Tauri 2, Rust, Svelte y Vite. Procesa localmente una pista de audio o una entrada de micrófono, aplica filtros Web Audio en tiempo real y muestra su espectro en una interfaz oscura de inspiración neón.

## Funcionalidades

- **Controles rápidos** — Preamp, Bass Boost, Mids y Clarity con cambios inmediatos.
- **Ecualizador de 10 bandas** — Filtros peaking en 31 Hz, 63 Hz, 125 Hz, 250 Hz, 500 Hz, 1 kHz, 2 kHz, 4 kHz, 8 kHz y 16 kHz.
- **Presets temáticos** — FPS Step-Tracker, Explosion / Action, Diálogos Claros, Cine de Acción, Bass Boost, Acústico, Vocal Clean, Flat / Studio y De-Esser / Anti-Noise.
- **Presets personalizados** — Guarda presets en el almacenamiento local, impórtalos y expórtalos como JSON.
- **Espectro en vivo** — Canvas 2D con `AnalyserNode`, FFT de 512 y resolución adaptada a la pantalla.
- **Audio de prueba** — Reproduce archivos locales compatibles con el WebView del sistema.
- **Entrada de micrófono** — Captura local mediante `getUserMedia`; la monitorización está desactivada por defecto para evitar acoples.
- **Bypass y mini modo** — Alterna suavemente entre la señal procesada y la directa, o usa el layout compacto.

## Requisitos

- Node.js 20 o posterior y npm.
- Rust estable y Cargo.
- Dependencias de compilación de Tauri 2 para tu sistema operativo.

Consulta la [guía de prerrequisitos de Tauri](https://v2.tauri.app/start/prerequisites/) para instalar las bibliotecas nativas necesarias en Windows, macOS o Linux.

## Crear el proyecto desde cero

El comando recomendado para generar el esqueleto oficial Svelte + TypeScript de Tauri 2 es:

```bash
npm create tauri-app@latest auraeq -- --template svelte-ts
```

En el asistente selecciona **TypeScript** si se solicita. Este repositorio ya contiene un esqueleto equivalente, preparado con la interfaz AuraEQ.

## Instalación y desarrollo

```bash
npm install
npm run tauri dev
```

Para trabajar solo en el navegador:

```bash
npm run dev
```

Vite sirve la interfaz en [http://127.0.0.1:1420](http://127.0.0.1:1420).

## Descargar versiones para Windows y Linux

Descarga la versión más reciente desde [GitHub Releases](https://github.com/RekiCv/AuraEQ/releases). Windows ofrece instaladores `.msi` y `.exe`; Linux ofrece paquetes `.deb` y `.AppImage`.

El workflow de GitHub Actions compila ambos sistemas cuando se publica una etiqueta `v*`. Para crear una nueva versión:

```bash
git tag v0.1.0
git push origin v0.1.0
```

## Apoya AuraEQ

Si disfrutas de AuraEQ y quieres apoyar su desarrollo, puedes invitarme a un café. ¡Cada aporte ayuda a que el proyecto siga creciendo!

<p align="center">
  <a href="https://donatr.ee/senkiu?utm_source=copy&amp;utm_medium=share">
    <img src="https://img.shields.io/badge/%E2%98%95_Apoya_AuraEQ-Donar-purple?style=for-the-badge" alt="Apoya AuraEQ con una donación"/>
  </a>
</p>

<p align="center">
  <sub>Hecho con curiosidad, código y un poco de vibecoding.</sub>
</p>

## Compilar

```bash
npm run tauri build
```

Los instaladores y ejecutables se generan dentro de `src-tauri/target/release/bundle/`. El tamaño final depende de la plataforma, el WebView disponible y la configuración de empaquetado; el objetivo de menos de 15 MB no se puede garantizar para todas las plataformas. Del mismo modo, el consumo real de memoria depende del sistema operativo y del WebView.

## Estructura

```text
AuraEQ/
├── index.html
├── package.json
├── package-lock.json
├── svelte.config.js
├── tsconfig.json
├── vite.config.ts
├── src/
│   ├── App.svelte       Interfaz, DSP, presets y espectro
│   ├── app.css          Estilos base
│   └── main.ts          Punto de entrada de Svelte
└── src-tauri/
    ├── Cargo.toml
    ├── build.rs
    ├── tauri.conf.json
    └── src/
        ├── lib.rs       Inicialización de Tauri
        └── main.rs      Entrada de escritorio
```

La interfaz y la cadena de filtros de audio están centralizadas en `src/App.svelte`. Rust mantiene la capa nativa mínima de Tauri; no hay servidor, cuenta ni servicio remoto.

## Alcance del audio

La Web Audio API procesa las fuentes que AuraEQ recibe explícitamente: archivos seleccionados en el reproductor y un micrófono autorizado por el usuario. **No captura ni ecualiza automáticamente el audio global de Windows, macOS o Linux.** El loopback del sistema requeriría una implementación nativa específica por plataforma y permisos adicionales.

La monitorización del micrófono permanece apagada al conectar la entrada. Actívala solo si la necesitas y utiliza auriculares para reducir el riesgo de realimentación acústica.

## Estado del proyecto

AuraEQ es un prototipo inicial. Los cambios de filtros y el análisis se ejecutan localmente con Web Audio API; los presets personalizados se guardan en el almacenamiento local del WebView.
