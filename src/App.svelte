<script lang="ts">
  import { onDestroy, onMount } from "svelte";

  const bands = [
    { label: "31", frequency: 31 },
    { label: "63", frequency: 63 },
    { label: "125", frequency: 125 },
    { label: "250", frequency: 250 },
    { label: "500", frequency: 500 },
    { label: "1k", frequency: 1000 },
    { label: "2k", frequency: 2000 },
    { label: "4k", frequency: 4000 },
    { label: "8k", frequency: 8000 },
    { label: "16k", frequency: 16000 },
  ] as const;

  type Preset = {
    id: string;
    name: string;
    category: string;
    description: string;
    preamp: number;
    bass: number;
    clarity: number;
    mids: number;
    bands: number[];
    custom?: boolean;
  };

  const presetList: Preset[] = [
    {
      id: "fps",
      name: "FPS Step-Tracker",
      category: "Gaming",
      description: "Pasos al frente, graves bajo control.",
      preamp: -2,
      bass: -2,
      clarity: 2,
      mids: 1,
      bands: [-3, -2, -2, -1, 0, 1, 3, 4, 3, 1],
    },
    {
      id: "explosion",
      name: "Explosion / Action",
      category: "Gaming",
      description: "Impacto y presencia para la acción.",
      preamp: -4,
      bass: 4,
      clarity: 1,
      mids: 0,
      bands: [4, 4, 3, 1, 0, 0, 1, 2, 1, 0],
    },
    {
      id: "dialogue",
      name: "Diálogos Claros",
      category: "Cine",
      description: "Voces definidas sin perder el ambiente.",
      preamp: -2,
      bass: -1,
      clarity: 1,
      mids: 3,
      bands: [-2, -1, 0, 0, 1, 3, 3, 2, 1, 0],
    },
    {
      id: "action-film",
      name: "Cine de Acción",
      category: "Cine",
      description: "Escena amplia con graves cinematográficos.",
      preamp: -4,
      bass: 3,
      clarity: 2,
      mids: 0,
      bands: [3, 3, 2, 1, 0, 0, 1, 2, 2, 1],
    },
    {
      id: "bass-boost",
      name: "Bass Boost",
      category: "Música",
      description: "Graves profundos con margen de headroom.",
      preamp: -4,
      bass: 4,
      clarity: 0,
      mids: 0,
      bands: [4, 4, 3, 2, 0, 0, 0, 1, 1, 0],
    },
    {
      id: "acoustic",
      name: "Acústico",
      category: "Música",
      description: "Detalle natural para instrumentos acústicos.",
      preamp: -2,
      bass: 0,
      clarity: 2,
      mids: 1,
      bands: [1, 1, 0, 1, 1, 2, 2, 2, 1, 0],
    },
    {
      id: "vocal",
      name: "Vocal Clean",
      category: "Música",
      description: "Voces presentes, brillantes y equilibradas.",
      preamp: -2,
      bass: -1,
      clarity: 2,
      mids: 2,
      bands: [-1, -1, 0, 0, 1, 2, 2, 2, 1, 0],
    },
    {
      id: "flat",
      name: "Flat / Studio",
      category: "Música",
      description: "Respuesta plana para escuchar con neutralidad.",
      preamp: 0,
      bass: 0,
      clarity: 0,
      mids: 0,
      bands: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
    },
    {
      id: "de-esser",
      name: "De-Esser / Anti-Noise",
      category: "Streaming / Mic",
      description: "Atenúa sibilancias y reduce el retumbe.",
      preamp: -2,
      bass: -2,
      clarity: -1,
      mids: 1,
      bands: [-2, -2, -1, 0, 0, 1, 1, -1, -3, -2],
    },
  ];

  const categories = ["Gaming", "Cine", "Música", "Streaming / Mic"];
  const initialPreset = presetList.find((preset) => preset.id === "flat")!;
  const clamp = (value: number, min: number, max: number) =>
    Math.min(max, Math.max(min, value));
  const dbToGain = (db: number) => Math.pow(10, db / 20);
  const formatTime = (seconds: number) => {
    if (!Number.isFinite(seconds)) return "0:00";
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${String(secs).padStart(2, "0")}`;
  };

  let preamp = initialPreset.preamp;
  let bass = initialPreset.bass;
  let clarity = initialPreset.clarity;
  let mids = initialPreset.mids;
  let bandValues = [...initialPreset.bands];
  let selectedPreset = initialPreset.id;
  let activeCategory = "Gaming";
  let expandedEq = true;
  let miniMode = false;
  let bypass = false;
  let monitorMic = false;
  let isPlaying = false;
  let micActive = false;
  let audioUrl = "";
  let audioName = "";
  let audioPosition = 0;
  let audioDuration = 0;
  let statusMessage = "Carga una pista o conecta un micrófono para empezar.";
  let customPresets: Preset[] = [];
  let presetName = "";
  let showSavePreset = false;
  let audioElement: HTMLAudioElement;
  let canvas: HTMLCanvasElement;
  let audioContext: AudioContext | null = null;
  let analyser: AnalyserNode | null = null;
  let inputAnalyser: AnalyserNode | null = null;
  let inputSource: AudioNode | null = null;
  let mediaElementSource: MediaElementAudioSourceNode | null = null;
  let micSource: MediaStreamAudioSourceNode | null = null;
  let micStream: MediaStream | null = null;
  let preampNode: GainNode | null = null;
  let bassNode: BiquadFilterNode | null = null;
  let midsNode: BiquadFilterNode | null = null;
  let clarityNode: BiquadFilterNode | null = null;
  let bandNodes: BiquadFilterNode[] = [];
  let processGain: GainNode | null = null;
  let bypassGain: GainNode | null = null;
  let outputBus: GainNode | null = null;
  let animationFrame = 0;
  let resizeObserver: ResizeObserver | null = null;
  let currentObjectUrl = "";

  $: visiblePresets = [
    ...presetList.filter((preset) => preset.category === activeCategory),
    ...customPresets.filter((preset) => preset.category === activeCategory),
  ];

  onMount(() => {
    try {
      const saved = localStorage.getItem("auraeq.custom-presets");
      if (saved) {
        const parsed: unknown = JSON.parse(saved);
        if (Array.isArray(parsed)) {
          customPresets = parsed.filter(isValidPreset).map((preset) => ({
            ...preset,
            custom: true,
          }));
        }
      }
    } catch (error) {
      console.error("No se pudieron cargar los presets guardados.", error);
      statusMessage = "No se pudieron cargar los presets guardados.";
    }

    if (canvas) {
      resizeObserver = new ResizeObserver(resizeCanvas);
      resizeObserver.observe(canvas);
      resizeCanvas();
      drawSpectrum();
    }
  });

  onDestroy(() => {
    cancelAnimationFrame(animationFrame);
    resizeObserver?.disconnect();
    stopMicrophone();
    if (currentObjectUrl) URL.revokeObjectURL(currentObjectUrl);
    void audioContext?.close();
  });

  function isValidPreset(value: unknown): value is Preset {
    if (!value || typeof value !== "object") return false;
    const preset = value as Partial<Preset>;
    return (
      typeof preset.id === "string" &&
      typeof preset.name === "string" &&
      typeof preset.category === "string" &&
      typeof preset.description === "string" &&
      [preset.preamp, preset.bass, preset.clarity, preset.mids].every(
        (number) => typeof number === "number" && Number.isFinite(number),
      ) &&
      Array.isArray(preset.bands) &&
      preset.bands.length === bands.length &&
      preset.bands.every(
        (number) => typeof number === "number" && Number.isFinite(number),
      )
    );
  }

  function createFilter(type: BiquadFilterType, frequency: number, q: number) {
    const filter = audioContext!.createBiquadFilter();
    filter.type = type;
    filter.frequency.value = Math.min(
      frequency,
      audioContext!.sampleRate / 2 - 1,
    );
    filter.Q.value = q;
    return filter;
  }

  function initializeAudioGraph() {
    if (audioContext) return;
    const AudioContextConstructor =
      window.AudioContext ||
      (window as typeof window & { webkitAudioContext?: typeof AudioContext })
        .webkitAudioContext;
    if (!AudioContextConstructor) {
      throw new Error("Este sistema no ofrece soporte para Web Audio API.");
    }

    audioContext = new AudioContextConstructor();
    preampNode = audioContext.createGain();
    bassNode = createFilter("lowshelf", 120, 0.7);
    midsNode = createFilter("peaking", 1000, 0.8);
    clarityNode = createFilter("highshelf", 5000, 0.7);
    bandNodes = bands.map((band) =>
      createFilter("peaking", band.frequency, 1.15),
    );
    processGain = audioContext.createGain();
    bypassGain = audioContext.createGain();
    outputBus = audioContext.createGain();
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 512;
    analyser.smoothingTimeConstant = 0.82;
    inputAnalyser = audioContext.createAnalyser();
    inputAnalyser.fftSize = 512;
    inputAnalyser.smoothingTimeConstant = 0.82;

    const filters = [
      preampNode,
      bassNode,
      midsNode,
      clarityNode,
      ...bandNodes,
      processGain,
    ];
    filters.forEach((filter, index) => {
      if (index < filters.length - 1) {
        filter.connect(filters[index + 1]);
      }
    });
    processGain.connect(outputBus);
    bypassGain.connect(outputBus);
    outputBus.connect(analyser);
    analyser.connect(audioContext.destination);

    applyControls();
    applyBypass();
    resizeCanvas();
  }

  async function ensureAudioContext() {
    initializeAudioGraph();
    if (audioContext?.state === "suspended") await audioContext.resume();
  }

  function connectSource(source: AudioNode) {
    if (!audioContext || !preampNode || !bypassGain || !inputAnalyser) return;
    inputSource?.disconnect();
    inputSource = source;
    source.connect(inputAnalyser);
    source.connect(preampNode);
    source.connect(bypassGain);
    drawSpectrum();
  }

  function setAudioParam(param: AudioParam, value: number) {
    if (!audioContext) return;
    const now = audioContext.currentTime;
    param.cancelScheduledValues(now);
    param.setTargetAtTime(value, now, 0.025);
  }

  function applyControls() {
    if (!audioContext || !preampNode || !bassNode || !midsNode || !clarityNode) {
      return;
    }
    setAudioParam(preampNode.gain, dbToGain(clamp(preamp, -12, 12)));
    setAudioParam(bassNode.gain, clamp(bass, -12, 12));
    setAudioParam(midsNode.gain, clamp(mids, -12, 12));
    setAudioParam(clarityNode.gain, clamp(clarity, -12, 12));
    bandNodes.forEach((node, index) => {
      setAudioParam(node.gain, clamp(bandValues[index] ?? 0, -12, 12));
    });
  }

  function applyBypass() {
    if (!processGain || !bypassGain || !audioContext) return;
    const shouldMonitor = !micActive || monitorMic;
    const processed = !bypass && shouldMonitor ? 1 : 0.0001;
    const direct = bypass && shouldMonitor ? 1 : 0.0001;
    setAudioParam(processGain.gain, processed);
    setAudioParam(bypassGain.gain, direct);
  }

  function updateControl() {
    applyControls();
  }

  function updateBand(index: number, value: number) {
    bandValues[index] = Number(value);
    bandValues = [...bandValues];
    applyControls();
    selectedPreset = "";
  }

  function applyPreset(preset: Preset) {
    preamp = preset.preamp;
    bass = preset.bass;
    clarity = preset.clarity;
    mids = preset.mids;
    bandValues = [...preset.bands];
    selectedPreset = preset.id;
    applyControls();
    statusMessage = `Preset aplicado: ${preset.name}`;
  }

  function resetToFlat() {
    const flat = presetList.find((preset) => preset.id === "flat");
    if (flat) applyPreset(flat);
  }

  async function loadAudioFile(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;
    try {
      if (micActive) stopMicrophone();
      if (currentObjectUrl) URL.revokeObjectURL(currentObjectUrl);
      currentObjectUrl = URL.createObjectURL(file);
      audioUrl = currentObjectUrl;
      audioName = file.name;
      audioPosition = 0;
      isPlaying = false;
      await ensureAudioContext();
      if (!audioElement) throw new Error("No se encontró el reproductor de audio.");
      mediaElementSource ??= audioContext!.createMediaElementSource(audioElement);
      connectSource(mediaElementSource);
      statusMessage = `Pista lista: ${file.name}`;
      await audioElement.play();
      isPlaying = true;
    } catch (error) {
      console.error("No se pudo abrir la pista de audio.", error);
      statusMessage =
        error instanceof Error ? error.message : "No se pudo abrir la pista.";
    } finally {
      input.value = "";
    }
  }

  async function togglePlayback() {
    if (!audioElement || !audioUrl) {
      statusMessage = "Primero selecciona un archivo de audio.";
      return;
    }
    try {
      await ensureAudioContext();
      if (!mediaElementSource) {
        mediaElementSource = audioContext!.createMediaElementSource(audioElement);
      }
      if (inputSource !== mediaElementSource) {
        connectSource(mediaElementSource);
      }
      if (audioElement.paused) {
        await audioElement.play();
        isPlaying = true;
      } else {
        audioElement.pause();
        isPlaying = false;
      }
    } catch (error) {
      console.error("No se pudo controlar la reproducción.", error);
      statusMessage =
        error instanceof Error ? error.message : "Error de reproducción.";
    }
  }

  function seekAudio(event: Event) {
    if (!audioElement) return;
    const input = event.currentTarget as HTMLInputElement;
    audioElement.currentTime = Number(input.value);
    audioPosition = audioElement.currentTime;
  }

  function onMetadata() {
    audioDuration = audioElement?.duration ?? 0;
  }

  function onTimeUpdate() {
    if (audioElement) audioPosition = audioElement.currentTime;
  }

  function onEnded() {
    isPlaying = false;
    audioPosition = 0;
  }

  async function toggleMicrophone() {
    if (micActive) {
      stopMicrophone();
      statusMessage = "Micrófono desconectado.";
      return;
    }
    if (!navigator.mediaDevices?.getUserMedia) {
      statusMessage = "La captura de micrófono no está disponible en este sistema.";
      return;
    }
    try {
      if (audioElement && !audioElement.paused) {
        audioElement.pause();
        isPlaying = false;
      }
      await ensureAudioContext();
      micStream = await navigator.mediaDevices.getUserMedia({
        audio: {
          echoCancellation: true,
          noiseSuppression: true,
          autoGainControl: false,
        },
      });
      micSource = audioContext!.createMediaStreamSource(micStream);
      connectSource(micSource);
      micActive = true;
      applyBypass();
      statusMessage = monitorMic
        ? "Micrófono activo. Usa auriculares para evitar acoples."
        : "Micrófono activo; monitorización apagada para evitar acoples.";
    } catch (error) {
      console.error("No se pudo iniciar el micrófono.", error);
      stopMicrophone();
      statusMessage =
        error instanceof Error
          ? error.message
          : "No se pudo iniciar el micrófono.";
    }
  }

  function stopMicrophone() {
    micSource?.disconnect();
    micSource = null;
    micStream?.getTracks().forEach((track) => track.stop());
    micStream = null;
    micActive = false;
    monitorMic = false;
    applyBypass();
  }

  function toggleMonitor() {
    monitorMic = !monitorMic;
    applyBypass();
    statusMessage = monitorMic
      ? "Monitorización activa. Se recomienda usar auriculares."
      : "Monitorización del micrófono apagada.";
  }

  function toggleBypass() {
    bypass = !bypass;
    applyBypass();
  }

  function saveCustomPreset() {
    const name = presetName.trim();
    if (!name) {
      statusMessage = "Escribe un nombre para el preset.";
      return;
    }
    const preset: Preset = {
      id: `custom-${Date.now()}`,
      name,
      category: activeCategory,
      description: "Preset personalizado.",
      preamp,
      bass,
      clarity,
      mids,
      bands: [...bandValues],
      custom: true,
    };
    customPresets = [...customPresets, preset];
    try {
      localStorage.setItem(
        "auraeq.custom-presets",
        JSON.stringify(customPresets),
      );
      selectedPreset = preset.id;
      presetName = "";
      showSavePreset = false;
      statusMessage = `Preset guardado: ${name}`;
    } catch (error) {
      console.error("No se pudo guardar el preset.", error);
      statusMessage = "No se pudo guardar el preset en el almacenamiento local.";
    }
  }

  function exportPresets() {
    const presets = [...presetList, ...customPresets];
    const blob = new Blob([JSON.stringify(presets, null, 2)], {
      type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = "auraeq-presets.json";
    anchor.click();
    URL.revokeObjectURL(url);
    statusMessage = "Presets exportados a auraeq-presets.json.";
  }

  async function importPresets(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;
    try {
      const parsed: unknown = JSON.parse(await file.text());
      if (!Array.isArray(parsed) || !parsed.every(isValidPreset)) {
        throw new Error(
          "El archivo debe ser un array de presets AuraEQ con 10 bandas.",
        );
      }
      const imported = (parsed as Preset[]).map((preset) => ({
        ...preset,
        id: `custom-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`,
        custom: true,
      }));
      customPresets = [...customPresets, ...imported];
      localStorage.setItem(
        "auraeq.custom-presets",
        JSON.stringify(customPresets),
      );
      statusMessage = `${imported.length} preset(s) importado(s).`;
    } catch (error) {
      console.error("No se pudieron importar los presets.", error);
      statusMessage =
        error instanceof Error
          ? error.message
          : "No se pudieron importar los presets.";
    } finally {
      input.value = "";
    }
  }

  function removePreset(preset: Preset) {
    customPresets = customPresets.filter((item) => item.id !== preset.id);
    try {
      localStorage.setItem(
        "auraeq.custom-presets",
        JSON.stringify(customPresets),
      );
      if (selectedPreset === preset.id) selectedPreset = "";
      statusMessage = `Preset eliminado: ${preset.name}`;
    } catch (error) {
      console.error("No se pudo actualizar la lista de presets.", error);
      statusMessage = "No se pudo actualizar la lista de presets guardados.";
    }
  }

  function resizeCanvas() {
    if (!canvas) return;
    const rect = canvas.getBoundingClientRect();
    if (!rect.width || !rect.height) return;
    const ratio = Math.min(window.devicePixelRatio || 1, 2);
    canvas.width = Math.round(rect.width * ratio);
    canvas.height = Math.round(rect.height * ratio);
    const context = canvas.getContext("2d");
    context?.setTransform(ratio, 0, 0, ratio, 0, 0);
  }

  function drawSpectrum() {
    if (!canvas) return;
    const context = canvas.getContext("2d");
    if (!context) return;
    const width = canvas.clientWidth;
    const height = canvas.clientHeight;
    if (!width || !height) {
      animationFrame = requestAnimationFrame(drawSpectrum);
      return;
    }

    context.clearRect(0, 0, width, height);
    const gradient = context.createLinearGradient(0, height, 0, 0);
    gradient.addColorStop(0, "#7158f5");
    gradient.addColorStop(0.52, "#2fe1db");
    gradient.addColorStop(1, "#f7a7ff");

    if (!analyser || (!inputSource && !isPlaying)) {
      for (let index = 0; index < 48; index += 1) {
        const barWidth = width / 72;
        const x = (index / 48) * width;
        const wave = Math.sin(index * 0.48) * 0.06 + Math.cos(index * 0.19) * 0.04;
        const barHeight = Math.max(3, height * (0.08 + wave + (index % 7) * 0.008));
        context.fillStyle = gradient;
        context.globalAlpha = 0.32;
        context.fillRect(x, height - barHeight, Math.max(2, barWidth * 0.55), barHeight);
      }
      context.globalAlpha = 1;
      animationFrame = requestAnimationFrame(drawSpectrum);
      return;
    }

    const activeAnalyser = micActive && inputAnalyser ? inputAnalyser : analyser;
    const data = new Uint8Array(activeAnalyser.frequencyBinCount);
    activeAnalyser.getByteFrequencyData(data);
    const bars = Math.min(72, Math.floor(width / 7));
    const gap = 3;
    const barWidth = Math.max(2, (width - gap * (bars - 1)) / bars);
    for (let index = 0; index < bars; index += 1) {
      const dataIndex = Math.floor(
        Math.pow(index / bars, 1.8) * (data.length * 0.85),
      );
      const magnitude = data[dataIndex] / 255;
      const barHeight = Math.max(2, magnitude * height * 0.9);
      const x = index * (barWidth + gap);
      context.shadowBlur = 12;
      context.shadowColor = index % 4 === 0 ? "#47e9e1" : "#8a70ff";
      context.fillStyle = gradient;
      context.globalAlpha = 0.48 + magnitude * 0.52;
      context.fillRect(x, height - barHeight, barWidth, barHeight);
    }
    context.shadowBlur = 0;
    context.globalAlpha = 1;
    animationFrame = requestAnimationFrame(drawSpectrum);
  }
</script>

<svelte:head>
  <title>AuraEQ — Audio Lab</title>
  <meta
    name="description"
    content="Ecualizador paramétrico, presets y espectro en tiempo real."
  />
</svelte:head>

<div class:mini={miniMode} class="app-shell">
  <header class="topbar">
    <a class="brand" href="#inicio" aria-label="AuraEQ inicio">
      <span class="brand-mark"><span></span><span></span><span></span></span>
      <span class="brand-name">AURA<span>EQ</span></span>
    </a>
    <div class="topbar-center"><span class="live-dot"></span> AUDIO LAB</div>
    <button class="icon-button mini-toggle" onclick={() => (miniMode = !miniMode)}>
      {miniMode ? "↗" : "↙"} <span>{miniMode ? "Expandir" : "Mini modo"}</span>
    </button>
  </header>

  <main id="inicio">
    <section class="hero-row">
      <div>
        <p class="eyebrow">TU SONIDO, A TU MANERA</p>
        <h1>Encuentra tu <span>frecuencia.</span></h1>
      </div>
      <button class:enabled={bypass} class="bypass-button" onclick={toggleBypass}>
        <span class="bypass-indicator"></span>
        <span><small>PROCESAMIENTO</small><strong>{bypass ? "Bypass activo" : "Activo"}</strong></span>
        <span class="switch" aria-hidden="true"><i></i></span>
      </button>
    </section>

    <section class="workspace">
      <div class="main-column">
        <section class="panel visualizer-panel">
          <div class="panel-heading">
            <div>
              <p class="eyebrow">ANÁLISIS EN TIEMPO REAL</p>
              <h2>Espectro <span class="heading-muted">/ 20 Hz — 20 kHz</span></h2>
            </div>
            <span class:recording={micActive || isPlaying} class="signal-state">
              <i></i>{micActive ? "MIC INPUT" : isPlaying ? "SIGNAL LIVE" : "STANDBY"}
            </span>
          </div>
          <canvas bind:this={canvas} class="spectrum" aria-label="Espectro de audio en tiempo real"></canvas>
          <div class="frequency-labels"><span>20 Hz</span><span>100 Hz</span><span>1 kHz</span><span>10 kHz</span><span>20 kHz</span></div>
        </section>

        <section class="panel controls-panel">
          <div class="panel-heading control-heading">
            <div>
              <p class="eyebrow">MOLDEA TU AUDIO</p>
              <h2>Controles rápidos</h2>
            </div>
            <button class="text-button" onclick={resetToFlat}>Restablecer</button>
          </div>
          <div class="quick-controls">
            {#each [
              { key: "preamp", label: "Preamp", hint: "Nivel de entrada", value: preamp, min: -12, max: 12, unit: "dB" },
              { key: "bass", label: "Bass Boost", hint: "Graves profundos", value: bass, min: -12, max: 12, unit: "dB" },
              { key: "mids", label: "Mids", hint: "Cuerpo y presencia", value: mids, min: -12, max: 12, unit: "dB" },
              { key: "clarity", label: "Clarity", hint: "Brillo y detalle", value: clarity, min: -12, max: 12, unit: "dB" },
            ] as control}
              <label class="quick-control">
                <span class="control-label"><strong>{control.label}</strong><b>{control.value > 0 ? "+" : ""}{control.value}<small> {control.unit}</small></b></span>
                <span class="control-hint">{control.hint}</span>
                <input
                  type="range"
                  min={control.min}
                  max={control.max}
                  step="1"
                  value={control.value}
                  style={`--range-progress: ${((control.value - control.min) / (control.max - control.min)) * 100}%`}
                  oninput={(event) => {
                    const value = Number((event.currentTarget as HTMLInputElement).value);
                    if (control.key === "preamp") preamp = value;
                    if (control.key === "bass") bass = value;
                    if (control.key === "mids") mids = value;
                    if (control.key === "clarity") clarity = value;
                    selectedPreset = "";
                    updateControl();
                  }}
                />
                <span class="range-scale"><small>{control.min} dB</small><small>0</small><small>+{control.max} dB</small></span>
              </label>
            {/each}
          </div>
        </section>

        <section class="panel eq-panel">
          <button class="eq-heading" onclick={() => (expandedEq = !expandedEq)} aria-expanded={expandedEq}>
            <span><span class="eq-icon">≋</span><span><small class="eyebrow">AJUSTE FINO</small><strong>Ecualizador paramétrico <em>10 bandas</em></strong></span></span>
            <span class="chevron">{expandedEq ? "−" : "+"}</span>
          </button>
          {#if expandedEq}
            <div class="eq-sliders">
              {#each bands as band, index}
                <label class="band-control">
                  <span class="band-value">{bandValues[index] > 0 ? "+" : ""}{bandValues[index]}</span>
                  <input
                    class="vertical-range"
                    type="range"
                    min="-12"
                    max="12"
                    step="1"
                    value={bandValues[index]}
                    aria-label={`${band.label} Hz, ${bandValues[index]} dB`}
                    style={`--range-progress: ${((bandValues[index] + 12) / 24) * 100}%`}
                    oninput={(event) =>
                      updateBand(index, Number((event.currentTarget as HTMLInputElement).value))}
                  />
                  <span class="band-label">{band.label}<small>Hz</small></span>
                </label>
              {/each}
            </div>
            <div class="eq-scale"><span>+12 dB</span><span>0 dB</span><span>−12 dB</span></div>
          {/if}
        </section>
      </div>

      <aside class="side-column">
        <section class="panel presets-panel">
          <div class="panel-heading">
            <div><p class="eyebrow">UN CLIC, OTRO SONIDO</p><h2>Presets</h2></div>
            <span class="preset-count">{presetList.length + customPresets.length}</span>
          </div>
          <div class="category-tabs" role="tablist" aria-label="Categorías de presets">
            {#each categories as category}
              <button
                class:active={activeCategory === category}
                role="tab"
                aria-selected={activeCategory === category}
                onclick={() => (activeCategory = category)}
              >
                {category === "Streaming / Mic" ? "Mic" : category}
              </button>
            {/each}
          </div>
          <div class="preset-list">
            {#each visiblePresets as preset}
              <div class="preset-row">
                <button class:chosen={selectedPreset === preset.id} class="preset-card" onclick={() => applyPreset(preset)}>
                  <span class="preset-symbol">{preset.category === "Gaming" ? "⌁" : preset.category === "Cine" ? "▹" : preset.category === "Música" ? "♫" : "◉"}</span>
                  <span class="preset-copy"><strong>{preset.name}</strong><small>{preset.description}</small></span>
                  {#if !preset.custom}<span class="preset-arrow">↗</span>{/if}
                </button>
                {#if preset.custom}
                  <button class="delete-preset" aria-label={`Eliminar ${preset.name}`} onclick={() => removePreset(preset)}>×</button>
                {/if}
              </div>
            {:else}
              <p class="empty-presets">Todavía no hay presets en esta categoría.</p>
            {/each}
          </div>
          <div class="preset-actions">
            <button class="outline-button" onclick={() => (showSavePreset = !showSavePreset)}>＋ Guardar preset</button>
            <button class="icon-button export-button" aria-label="Exportar presets JSON" title="Exportar presets JSON" onclick={exportPresets}>↥</button>
            <label class="icon-button import-button" aria-label="Importar presets JSON" title="Importar presets JSON">↧<input type="file" accept="application/json,.json" onchange={importPresets} /></label>
          </div>
          {#if showSavePreset}
            <form class="save-form" onsubmit={(event) => { event.preventDefault(); saveCustomPreset(); }}>
              <input bind:value={presetName} maxlength="40" placeholder="Nombre del preset" aria-label="Nombre del preset" />
              <button class="save-button" type="submit">Guardar</button>
            </form>
          {/if}
        </section>

        <section class="panel player-panel">
          <div class="panel-heading">
            <div><p class="eyebrow">PRUEBA TU ECUALIZACIÓN</p><h2>Fuente de audio</h2></div>
            <span class="source-icon">♫</span>
          </div>
          <label class="file-drop">
            <input type="file" accept="audio/*" onchange={loadAudioFile} />
            <span class="upload-icon">↑</span>
            <strong>{audioName || "Selecciona una pista"}</strong>
            <small>WAV, MP3, FLAC, OGG y más</small>
          </label>
          <audio
            bind:this={audioElement}
            src={audioUrl || undefined}
            onloadedmetadata={onMetadata}
            ontimeupdate={onTimeUpdate}
            onended={onEnded}
          ></audio>
          <div class="player-controls">
            <button class:playing={isPlaying} class="play-button" aria-label={isPlaying ? "Pausar" : "Reproducir"} onclick={togglePlayback}>
              {isPlaying ? "Ⅱ" : "▶"}
            </button>
            <div class="seek-area">
              <input type="range" min="0" max={audioDuration || 1} step="0.1" value={audioPosition} oninput={seekAudio} aria-label="Posición de reproducción" />
              <div><span>{formatTime(audioPosition)}</span><span>{formatTime(audioDuration)}</span></div>
            </div>
          </div>
          <div class="mic-row">
            <button class:mic-on={micActive} class="mic-button" onclick={toggleMicrophone}>
              <span>{micActive ? "●" : "◉"}</span>{micActive ? "Desconectar micrófono" : "Probar micrófono"}
            </button>
            {#if micActive}
              <button class:monitor-on={monitorMic} class="monitor-toggle" onclick={toggleMonitor} title="Activa solo con auriculares">
                Monitor {monitorMic ? "ON" : "OFF"}
              </button>
            {/if}
          </div>
          <p class="status-message" aria-live="polite">{statusMessage}</p>
        </section>

        <div class="note-card"><span>✦</span><p><strong>Hecho para escuchar mejor.</strong><br />Los cambios se aplican al instante y se quedan en tu dispositivo.</p></div>
      </aside>
    </section>
  </main>
  <footer><span>AURA<span class="footer-accent">EQ</span> <i>·</i> AUDIO LAB</span><span>El sonido se procesa localmente en tu dispositivo.</span></footer>
</div>

<style>
  :global(body) { background: #090a12; }
  .app-shell { --bg: #090a12; --panel: #11121d; --panel-soft: #151623; --border: rgba(165, 157, 211, .13); --muted: #8d8ba1; --dim: #555468; --purple: #a48bff; --cyan: #51e4db; max-width: 1500px; margin: 0 auto; padding: 0 52px; overflow: hidden; }
  .topbar { height: 78px; display: flex; align-items: center; justify-content: space-between; border-bottom: 1px solid var(--border); }
  .brand { display: flex; align-items: center; gap: 12px; color: inherit; text-decoration: none; }
  .brand-mark { width: 27px; height: 27px; display: flex; align-items: center; justify-content: center; gap: 3px; border: 1px solid rgba(164,139,255,.55); border-radius: 8px; background: rgba(164,139,255,.08); }
  .brand-mark span { width: 2px; border-radius: 3px; background: var(--cyan); box-shadow: 0 0 7px rgba(81,228,219,.6); }
  .brand-mark span:nth-child(1) { height: 8px; }.brand-mark span:nth-child(2) { height: 15px; background: var(--purple); }.brand-mark span:nth-child(3) { height: 11px; }
  .brand-name { font-size: 15px; font-weight: 800; letter-spacing: .13em; }.brand-name span,.footer-accent { color: var(--purple); }
  .topbar-center { display: flex; gap: 9px; align-items: center; color: var(--muted); font-size: 10px; font-weight: 700; letter-spacing: .18em; }
  .live-dot,.signal-state i { width: 6px; height: 6px; border-radius: 50%; background: var(--cyan); box-shadow: 0 0 10px var(--cyan); }
  .icon-button { display: inline-flex; align-items: center; justify-content: center; gap: 7px; border: 1px solid var(--border); color: #c6c3d6; background: #141521; border-radius: 9px; cursor: pointer; transition: .2s ease; }
  .icon-button:hover,.outline-button:hover { border-color: rgba(164,139,255,.6); color: white; }
  .mini-toggle { padding: 9px 12px; font-size: 11px; }.mini-toggle span { color: #aaa7bb; }
  main { padding: 42px 0 34px; }
  .hero-row { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 26px; }
  .eyebrow { margin: 0 0 9px; color: var(--purple); font-size: 9px; font-weight: 800; letter-spacing: .18em; }
  h1 { margin: 0; font-size: clamp(29px, 3.5vw, 43px); letter-spacing: -.055em; font-weight: 650; } h1 span { color: var(--purple); }
  .bypass-button { display: flex; align-items: center; gap: 11px; min-width: 168px; padding: 11px 13px; border: 1px solid rgba(81,228,219,.2); border-radius: 12px; background: rgba(81,228,219,.045); text-align: left; cursor: pointer; }
  .bypass-button.enabled { border-color: rgba(164,139,255,.4); background: rgba(164,139,255,.07); }
  .bypass-indicator { width: 7px; height: 7px; border-radius: 50%; background: var(--cyan); box-shadow: 0 0 10px var(--cyan); }.enabled .bypass-indicator { background: var(--purple); box-shadow: 0 0 10px var(--purple); }
  .bypass-button small,.bypass-button strong { display: block; }.bypass-button small { margin-bottom: 3px; color: var(--muted); font-size: 8px; letter-spacing: .11em; }.bypass-button strong { font-size: 11px; font-weight: 650; }
  .switch { width: 29px; height: 16px; margin-left: auto; padding: 2px; border-radius: 10px; background: #30303e; transition: background .2s ease; }.switch i { display: block; width: 12px; height: 12px; border-radius: 50%; background: #9997a7; transition: transform .2s ease, background .2s ease; }.bypass-button:not(.enabled) .switch { background: rgba(81,228,219,.3); }.bypass-button:not(.enabled) .switch i { transform: translateX(13px); background: var(--cyan); }
  .workspace { display: grid; grid-template-columns: minmax(0, 1.72fr) minmax(290px, .88fr); gap: 18px; align-items: start; }
  .main-column,.side-column { display: grid; gap: 16px; min-width: 0; }
  .panel { border: 1px solid var(--border); border-radius: 14px; background: linear-gradient(145deg, rgba(20,21,34,.97), rgba(15,16,27,.97)); box-shadow: 0 10px 35px rgba(0,0,0,.13); }
  .visualizer-panel { padding: 20px 21px 15px; background: radial-gradient(ellipse at 50% 120%, rgba(80,61,163,.16), transparent 58%), linear-gradient(145deg, #141522, #10111b); }
  .panel-heading { display: flex; align-items: center; justify-content: space-between; }.panel-heading .eyebrow { margin-bottom: 5px; font-size: 8px; }
  h2 { margin: 0; font-size: 16px; letter-spacing: -.025em; }.heading-muted { color: var(--dim); font-size: 10px; font-weight: 450; letter-spacing: 0; }
  .signal-state { display: flex; align-items: center; gap: 7px; color: var(--dim); font-size: 8px; font-weight: 750; letter-spacing: .1em; }.signal-state i { width: 5px; height: 5px; background: #5a596d; box-shadow: none; }.signal-state.recording { color: var(--cyan); }.signal-state.recording i { background: var(--cyan); box-shadow: 0 0 8px var(--cyan); }
  .spectrum { display: block; width: 100%; height: 135px; margin-top: 16px; }
  .frequency-labels { display: flex; justify-content: space-between; color: #676579; font-size: 8px; font-variant-numeric: tabular-nums; }
  .controls-panel { padding: 19px 21px 20px; }.control-heading { margin-bottom: 22px; }.text-button { padding: 5px 0; border: 0; color: var(--muted); background: none; font-size: 10px; cursor: pointer; }.text-button:hover { color: var(--purple); }
  .quick-controls { display: grid; grid-template-columns: repeat(4, minmax(0,1fr)); gap: 18px; }.quick-control { min-width: 0; }
  .control-label { display: flex; justify-content: space-between; align-items: center; gap: 4px; }.control-label strong { font-size: 11px; font-weight: 600; }.control-label b { color: var(--cyan); font-size: 10px; font-variant-numeric: tabular-nums; }.control-label b small { color: var(--muted); font-size: 8px; font-weight: 500; }
  .control-hint { display: block; margin: 5px 0 14px; color: var(--dim); font-size: 9px; }
  input[type="range"] { --range-progress: 50%; width: 100%; height: 4px; margin: 0; border-radius: 6px; outline: none; appearance: none; background: linear-gradient(to right, #967aff 0%, #54dfd9 var(--range-progress), #333342 var(--range-progress), #333342 100%); cursor: pointer; }
  input[type="range"]::-webkit-slider-thumb { width: 11px; height: 11px; border: 2px solid #e6e2ff; border-radius: 50%; appearance: none; background: #8b73f6; box-shadow: 0 0 8px rgba(150,122,255,.65); }
  input[type="range"]::-moz-range-thumb { width: 8px; height: 8px; border: 2px solid #e6e2ff; border-radius: 50%; background: #8b73f6; box-shadow: 0 0 8px rgba(150,122,255,.65); }
  .range-scale { display: flex; justify-content: space-between; margin-top: 8px; color: var(--dim); font-size: 8px; }
  .eq-panel { overflow: hidden; }.eq-heading { width: 100%; display: flex; justify-content: space-between; align-items: center; padding: 16px 20px; border: 0; background: transparent; text-align: left; cursor: pointer; }.eq-heading > span:first-child { display: flex; align-items: center; gap: 12px; }.eq-icon { width: 31px; height: 31px; display: grid; place-items: center; border: 1px solid rgba(164,139,255,.2); border-radius: 9px; color: var(--purple); font-size: 20px; }.eq-heading .eyebrow { margin-bottom: 4px; font-size: 8px; }.eq-heading strong { display: block; font-size: 12px; font-weight: 650; }.eq-heading em { margin-left: 6px; color: var(--muted); font-size: 9px; font-style: normal; font-weight: 450; }.chevron { color: var(--muted); font-size: 20px; }
  .eq-sliders { display: grid; grid-template-columns: repeat(10, 1fr); gap: 6px; padding: 13px 15px 3px; border-top: 1px solid var(--border); }.band-control { display: flex; flex-direction: column; align-items: center; gap: 8px; min-width: 0; }.band-value { min-height: 12px; color: #b6a8ff; font-size: 9px; font-variant-numeric: tabular-nums; }
  .vertical-range { width: 100%; max-width: 92px; height: 88px !important; background: linear-gradient(to right, #323140, #323140) !important; writing-mode: vertical-lr; direction: rtl; appearance: slider-vertical !important; accent-color: #9b83ff; }
  .vertical-range::-webkit-slider-runnable-track { width: 3px; border-radius: 5px; background: #393847; }.vertical-range::-webkit-slider-thumb { width: 10px !important; height: 10px !important; }
  .vertical-range::-moz-range-track { width: 3px; border-radius: 5px; background: #393847; }
  .band-label { display: flex; align-items: baseline; gap: 2px; color: #d3d0df; font-size: 9px; font-weight: 600; }.band-label small { color: var(--dim); font-size: 7px; font-weight: 450; }
  .eq-scale { display: flex; justify-content: space-between; padding: 4px 19px 13px; color: var(--dim); font-size: 8px; }
  .presets-panel { padding: 19px 17px 16px; }.preset-count { display: grid; place-items: center; width: 23px; height: 23px; border: 1px solid var(--border); border-radius: 7px; color: var(--muted); font-size: 9px; }
  .category-tabs { display: flex; gap: 4px; margin: 18px 0 11px; padding: 3px; border: 1px solid rgba(255,255,255,.04); border-radius: 9px; background: rgba(0,0,0,.17); }.category-tabs button { flex: 1; padding: 7px 4px; border: 0; border-radius: 6px; color: #7d7a90; background: transparent; font-size: 9px; cursor: pointer; transition: .18s ease; }.category-tabs button.active { color: #e7e1ff; background: #28243d; box-shadow: inset 0 0 0 1px rgba(164,139,255,.18); }
  .preset-list { display: grid; gap: 7px; min-height: 90px; }.preset-row { position: relative; }.preset-card { display: flex; align-items: center; gap: 10px; width: 100%; min-width: 0; padding: 10px; border: 1px solid rgba(255,255,255,.045); border-radius: 9px; background: rgba(255,255,255,.018); text-align: left; cursor: pointer; transition: border-color .2s ease, background .2s ease, transform .2s ease; }.preset-card:hover { transform: translateY(-1px); border-color: rgba(164,139,255,.35); background: rgba(164,139,255,.055); }.preset-card.chosen { border-color: rgba(81,228,219,.35); background: rgba(81,228,219,.045); }
  .preset-symbol { flex: 0 0 29px; display: grid; place-items: center; width: 29px; height: 29px; border-radius: 8px; color: var(--purple); background: rgba(164,139,255,.1); font-size: 15px; }.preset-row:nth-child(even) .preset-symbol { color: var(--cyan); background: rgba(81,228,219,.08); }.preset-copy { display: grid; gap: 3px; min-width: 0; }.preset-copy strong { overflow: hidden; color: #e1dfeb; font-size: 10px; font-weight: 600; text-overflow: ellipsis; white-space: nowrap; }.preset-copy small { overflow: hidden; color: var(--dim); font-size: 8px; text-overflow: ellipsis; white-space: nowrap; }.preset-arrow { margin-left: auto; color: #6b687c; font-size: 12px; }.delete-preset { position: absolute; top: 50%; right: 8px; transform: translateY(-50%); padding: 2px 5px; border: 0; color: #827e91; background: #171722; font-size: 16px; cursor: pointer; }.delete-preset:hover { color: #ff829b; }.empty-presets { margin: 0; padding: 18px 8px; color: var(--dim); font-size: 10px; text-align: center; }
  .preset-actions { display: flex; gap: 7px; margin-top: 13px; }.outline-button { flex: 1; min-height: 34px; border: 1px dashed rgba(164,139,255,.27); border-radius: 8px; color: #b9aaff; background: rgba(164,139,255,.045); font-size: 9px; cursor: pointer; transition: .2s ease; }.export-button,.import-button { width: 35px; min-height: 34px; font-size: 15px; }.import-button { position: relative; }.import-button input,.file-drop input { position: absolute; width: 1px; height: 1px; overflow: hidden; opacity: 0; }
  .save-form { display: flex; gap: 6px; margin-top: 9px; }.save-form input { flex: 1; min-width: 0; padding: 8px 9px; border: 1px solid var(--border); border-radius: 7px; outline: none; color: white; background: #0c0d16; font-size: 10px; }.save-form input:focus { border-color: rgba(164,139,255,.55); }.save-button { padding: 0 11px; border: 0; border-radius: 7px; color: #10101a; background: var(--purple); font-size: 9px; font-weight: 700; cursor: pointer; }
  .player-panel { padding: 18px 17px 15px; }.source-icon { display: grid; place-items: center; width: 29px; height: 29px; border-radius: 9px; color: var(--cyan); background: rgba(81,228,219,.08); font-size: 16px; }.file-drop { position: relative; display: grid; place-items: center; min-height: 87px; margin-top: 15px; padding: 12px; border: 1px dashed rgba(164,139,255,.25); border-radius: 10px; background: rgba(164,139,255,.025); cursor: pointer; transition: .2s ease; }.file-drop:hover { border-color: rgba(81,228,219,.4); background: rgba(81,228,219,.035); }.upload-icon { display: grid; place-items: center; width: 24px; height: 24px; margin-bottom: 6px; border-radius: 7px; color: var(--purple); background: rgba(164,139,255,.12); font-size: 14px; }.file-drop strong { max-width: 100%; overflow: hidden; color: #d6d3e2; font-size: 9px; font-weight: 600; text-overflow: ellipsis; white-space: nowrap; }.file-drop small { margin-top: 4px; color: var(--dim); font-size: 8px; }
  .player-controls { display: flex; align-items: center; gap: 11px; margin-top: 14px; }.play-button { flex: 0 0 31px; width: 31px; height: 31px; border: 1px solid rgba(164,139,255,.35); border-radius: 50%; color: #d8d0ff; background: rgba(164,139,255,.12); font-size: 11px; cursor: pointer; }.play-button.playing { color: var(--cyan); border-color: rgba(81,228,219,.4); }.seek-area { flex: 1; }.seek-area input { display: block; }.seek-area > div { display: flex; justify-content: space-between; margin-top: 6px; color: var(--dim); font-size: 8px; font-variant-numeric: tabular-nums; }
  .mic-row { display: flex; gap: 7px; margin-top: 13px; }.mic-button,.monitor-toggle { display: flex; align-items: center; justify-content: center; gap: 7px; min-height: 32px; border: 1px solid var(--border); border-radius: 8px; color: #aaa7bc; background: rgba(255,255,255,.018); font-size: 9px; cursor: pointer; }.mic-button { flex: 1; }.mic-button span { color: var(--purple); }.mic-button.mic-on { color: #f0c2cf; border-color: rgba(255,112,150,.25); }.mic-button.mic-on span { color: #ff7898; }.monitor-toggle { padding: 0 9px; color: var(--dim); font-size: 8px; }.monitor-toggle.monitor-on { border-color: rgba(255,112,150,.3); color: #ff9cb2; }
  .status-message { min-height: 12px; margin: 10px 0 0; color: #767388; font-size: 8px; line-height: 1.5; }.note-card { display: flex; align-items: center; gap: 10px; padding: 11px 13px; border: 1px solid rgba(164,139,255,.12); border-radius: 10px; background: rgba(164,139,255,.035); }.note-card > span { color: #c0adff; font-size: 15px; }.note-card p { margin: 0; color: var(--muted); font-size: 8px; line-height: 1.7; }.note-card strong { color: #c7c1da; font-size: 9px; font-weight: 600; }
  footer { display: flex; justify-content: space-between; gap: 15px; padding: 15px 0 19px; border-top: 1px solid var(--border); color: #5d5a70; font-size: 8px; letter-spacing: .05em; }footer > span:first-child { color: #9b97ac; font-size: 8px; font-weight: 750; letter-spacing: .12em; }footer i { padding: 0 4px; color: #696579; font-style: normal; }
  .mini main { max-width: 840px; margin: 0 auto; }.mini .hero-row { margin-bottom: 16px; }.mini .workspace { grid-template-columns: 1fr; }.mini .side-column { grid-template-columns: 1fr 1fr; align-items: start; }.mini .presets-panel { grid-row: span 2; }.mini .note-card { align-self: start; }
  @media (max-width: 960px) { .app-shell { padding: 0 25px; }.workspace { grid-template-columns: minmax(0,1.45fr) minmax(270px,.9fr); gap: 12px; }.quick-controls { gap: 12px; }.category-tabs button { font-size: 8px; } }
  @media (max-width: 760px) { .app-shell { padding: 0 17px; }.topbar { height: 66px; }.topbar-center { display: none; }main { padding-top: 30px; }.hero-row { align-items: center; gap: 12px; }.bypass-button { min-width: 147px; padding: 9px; }h1 { font-size: 30px; }.workspace { grid-template-columns: 1fr; }.side-column { grid-template-columns: 1fr 1fr; align-items: start; }.presets-panel { grid-row: span 2; }.note-card { align-self: start; }.quick-controls { gap: 18px; }.eq-sliders { gap: 3px; padding-right: 10px; padding-left: 10px; }.band-label { font-size: 8px; } }
  @media (max-width: 520px) { .app-shell { padding: 0 12px; }.brand-name { font-size: 13px; }.mini-toggle { padding: 8px; }.mini-toggle span { display: none; }main { padding-top: 24px; }.hero-row { align-items: flex-start; flex-direction: column; }.bypass-button { align-self: flex-end; margin-top: -6px; }.quick-controls { grid-template-columns: repeat(2, minmax(0,1fr)); row-gap: 22px; }.side-column,.mini .side-column { grid-template-columns: 1fr; }.presets-panel { grid-row: auto; }.visualizer-panel { padding: 17px 14px 13px; }.spectrum { height: 105px; }.controls-panel { padding: 17px 14px; }.eq-heading { padding: 14px; }.eq-sliders { grid-template-columns: repeat(5,1fr); row-gap: 13px; }.vertical-range { height: 72px !important; }.eq-scale { padding-right: 14px; padding-left: 14px; }footer { flex-direction: column; } }
</style>
