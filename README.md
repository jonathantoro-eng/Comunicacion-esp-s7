<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Panel de Control ESP32 - Adafruit IO</title>
  <!-- Chart.js para el gráfico de temperatura en tiempo real -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    :root {
      --bg-color: #0f172a;
      --card-bg: #1e293b;
      --text-main: #f8fafc;
      --text-muted: #94a3b8;
      --primary: #3b82f6;
      --primary-hover: #2563eb;
      --green-on: #22c55e;
      --green-glow: rgba(34, 197, 94, 0.5);
      --red-on: #ef4444;
      --red-glow: rgba(239, 68, 68, 0.5);
      --off-gray: #334155;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Segoe UI', system-ui, sans-serif; }

    body {
      background-color: var(--bg-color);
      color: var(--text-main);
      min-height: 100vh;
      display: flex;
      flex-direction: column;
    }

    /* Barra superior */
    header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 1rem 2rem;
      background-color: var(--card-bg);
      border-bottom: 1px solid #334155;
    }

    .status-badge {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      font-size: 0.9rem;
      background: #0f172a;
      padding: 6px 14px;
      border-radius: 20px;
    }

    .dot-conn { width: 10px; height: 10px; border-radius: 50%; background: var(--red-on); }
    .dot-conn.online { background: var(--green-on); box-shadow: 0 0 8px var(--green-on); }

    .btn-config {
      background: #334155;
      color: white;
      border: none;
      padding: 10px 16px;
      border-radius: 8px;
      cursor: pointer;
      font-weight: 600;
      display: flex;
      align-items: center;
      gap: 6px;
      transition: 0.2s;
    }
    .btn-config:hover { background: #475569; }

    /* Contenedor Principal */
    main {
      flex: 1;
      max-width: 900px;
      width: 100%;
      margin: 2rem auto;
      padding: 0 1rem;
    }

    .screen { display: none; animation: fadeIn 0.3s ease; }
    .screen.active { display: block; }

    @keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }

    .card {
      background: var(--card-bg);
      border-radius: 16px;
      padding: 2rem;
      box-shadow: 0 10px 25px rgba(0,0,0,0.3);
      margin-bottom: 1.5rem;
    }

    h1, h2 { margin-bottom: 0.5rem; }
    p.subtitle { color: var(--text-muted); margin-bottom: 2rem; }

    /* Pantalla Principal - Botones de Modo */
    .mode-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
      gap: 1.5rem;
    }

    .mode-btn {
      background: #0f172a;
      border: 2px solid #334155;
      border-radius: 16px;
      padding: 2.5rem 1.5rem;
      color: white;
      cursor: pointer;
      text-align: center;
      transition: all 0.25s;
    }
    .mode-btn:hover { border-color: var(--primary); transform: translateY(-4px); }
    .mode-btn.selected { border-color: var(--green-on); background: rgba(34, 197, 94, 0.08); }
    .mode-btn .icon { font-size: 3rem; display: block; margin-bottom: 1rem; }
    .mode-btn h3 { font-size: 1.4rem; margin-bottom: 0.5rem; }
    .mode-btn span { color: var(--text-muted); font-size: 0.9rem; }

    /* Barra de navegación interna */
    .top-bar-nav {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1.5rem;
      flex-wrap: wrap;
      gap: 1rem;
    }

    .btn-back {
      background: transparent;
      border: 1px solid #475569;
      color: var(--text-main);
      padding: 8px 16px;
      border-radius: 8px;
      cursor: pointer;
    }
    .btn-back:hover { background: #334155; }

    /* Indicadores Circulares (Relés) */
    .indicators-box {
      display: flex;
      justify-content: space-around;
      align-items: center;
      background: #0f172a;
      padding: 1.5rem;
      border-radius: 12px;
      margin-top: 1.5rem;
    }

    .indicator-item { text-align: center; }

    .circle {
      width: 64px;
      height: 64px;
      border-radius: 50%;
      background-color: var(--off-gray);
      margin: 0 auto 10px;
      border: 3px solid #475569;
      transition: all 0.3s ease;
    }

    .circle.green-active {
      background-color: var(--green-on);
      border-color: #86efac;
      box-shadow: 0 0 25px var(--green-glow);
    }

    .circle.red-active {
      background-color: var(--red-on);
      border-color: #fca5a5;
      box-shadow: 0 0 25px var(--red-glow);
    }

    /* Slider Umbral */
    .slider-container {
      background: #0f172a;
      padding: 1.5rem;
      border-radius: 12px;
      margin-top: 1.5rem;
    }
    .slider-header { display: flex; justify-content: space-between; margin-bottom: 10px; font-weight: bold; }
    input[type="range"] { width: 100%; accent-color: var(--primary); cursor: pointer; height: 8px; }

    /* Selector pequeño superior (Modo Manual: Web vs Físico) */
    .small-switch-box {
      display: flex;
      align-items: center;
      gap: 10px;
      background: #0f172a;
      padding: 8px 14px;
      border-radius: 20px;
      border: 1px solid #334155;
      font-size: 0.85rem;
    }

    .switch { position: relative; display: inline-block; width: 44px; height: 24px; }
    .switch input { opacity: 0; width: 0; height: 0; }
    .slider-round {
      position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0;
      background-color: var(--primary); transition: .3s; border-radius: 24px;
    }
    .slider-round:before {
      position: absolute; content: ""; height: 18px; width: 18px; left: 3px; bottom: 3px;
      background-color: white; transition: .3s; border-radius: 50%;
    }
    input:checked + .slider-round { background-color: #f59e0b; }
    input:checked + .slider-round:before { transform: translateX(20px); }

    /* Pulsadores Manuales */
    .pulse-controls {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 1.5rem;
      margin-top: 1rem;
    }

    .btn-pulse {
      padding: 1.5rem;
      font-size: 1.2rem;
      font-weight: bold;
      border: none;
      border-radius: 12px;
      cursor: pointer;
      color: white;
      transition: transform 0.1s, opacity 0.2s;
    }
    .btn-pulse:active { transform: scale(0.96); }
    .btn-on { background: #16a34a; }
    .btn-on:hover { background: #15803d; }
    .btn-off { background: #dc2626; }
    .btn-off:hover { background: #b91c1c; }

    /* Bloqueo por Modo Físico */
    .physical-alert {
      background: rgba(245, 158, 11, 0.15);
      border: 2px solid #f59e0b;
      color: #fcd34d;
      padding: 2rem;
      border-radius: 12px;
      text-align: center;
      margin: 1.5rem 0;
      display: none;
    }

    .disabled-area.locked {
      opacity: 0.25;
      pointer-events: none;
      filter: grayscale(1);
    }

    /* Modal Configuración API */
    .modal {
      display: none;
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.75);
      justify-content: center;
      align-items: center;
      z-index: 100;
    }
    .modal.open { display: flex; }
    .modal-content {
      background: var(--card-bg);
      padding: 2rem;
      border-radius: 16px;
      width: 90%;
      max-width: 420px;
    }
    .form-group { margin-bottom: 1rem; }
    .form-group label { display: block; margin-bottom: 6px; color: var(--text-muted); font-size: 0.9rem; }
    .form-group input {
      width: 100%;
      padding: 10px;
      border-radius: 8px;
      border: 1px solid #475569;
      background: #0f172a;
      color: white;
    }
    .modal-actions { display: flex; justify-content: flex-end; gap: 10px; margin-top: 1.5rem; }
    .btn-save { background: var(--primary); color: white; border: none; padding: 10px 18px; border-radius: 8px; cursor: pointer; }
  </style>
</head>
<body>

  <!-- ENCABEZADO -->
  <header>
    <div class="status-badge">
      <div id="connDot" class="dot-conn"></div>
      <span id="connText">Conectando a Adafruit IO...</span>
    </div>
    <button class="btn-config" onclick="abrirConfig()">⚙️ Configurar API Key</button>
  </header>

  <main>
    <!-- ================= PANTALLA 1: PRINCIPAL ================= -->
    <section id="screenHome" class="screen active">
      <div class="card">
        <h1>Control Central ESP32</h1>
        <p class="subtitle">Selecciona el modo de operación (Controla el Pin 27 - Selector de Modos)</p>

        <div class="mode-grid">
          <!-- BOTÓN 1: MODO AUTOMÁTICO -->
          <button id="cardBtnAuto" class="mode-btn" onclick="seleccionarModo(true)">
            <span class="icon">🌡️</span>
            <h3>Modo Automático</h3>
            <span>Pin 27: ACTIVADO<br>Control por temperatura y umbral</span>
          </button>

          <!-- BOTÓN 2: MODO MANUAL -->
          <button id="cardBtnManual" class="mode-btn" onclick="seleccionarModo(false)">
            <span class="icon">🎛️</span>
            <h3>Modo Manual</h3>
            <span>Pin 27: DESACTIVADO<br>Control directo Web o Físico</span>
          </button>
        </div>
      </div>
    </section>

    <!-- ================= PANTALLA 2: MODO AUTOMÁTICO ================= -->
    <section id="screenAuto" class="screen">
      <div class="top-bar-nav">
        <button class="btn-back" onclick="irAInicio()">⬅ Volver al Menú Principal</button>
        <span style="color: var(--green-on); font-weight: bold;">● MODO AUTOMÁTICO (Pin 27 ON)</span>
      </div>

      <div class="card">
        <div style="display: flex; justify-content: space-between; align-items: baseline;">
          <h2>Monitoreo de Temperatura (Pin 4)</h2>
          <div style="font-size: 1.8rem; font-weight: bold; color: var(--primary);">
            <span id="tempActualText">--.--</span> °C
          </div>
        </div>
        <p class="subtitle">Promedio en tiempo real de los 2 sensores Dallas DS18B20</p>

        <!-- Gráfico de Temperatura -->
        <div style="height: 250px; width: 100%;">
          <canvas id="tempChart"></canvas>
        </div>

        <!-- Barra de Umbral -->
        <div class="slider-container">
          <div class="slider-header">
            <span>Umbral de Activación de Relés:</span>
            <span style="color: var(--primary); font-size: 1.2rem;"><span id="umbralValText">30.0</span> °C</span>
          </div>
          <input type="range" id="umbralSlider" min="0" max="100" step="0.5" value="30"
                 oninput="actualizarTextoUmbral(this.value)" onchange="enviarUmbral(this.value)">
        </div>

        <!-- Indicadores Circulares Pin 25 y Pin 26 -->
        <div class="indicators-box">
          <div class="indicator-item">
            <div id="circlePin25" class="circle"></div>
            <strong>PIN 25 (ON)</strong><br>
            <small style="color: var(--text-muted);">Supera el umbral</small>
          </div>

          <div class="indicator-item">
            <div id="circlePin26" class="circle"></div>
            <strong>PIN 26 (OFF)</strong><br>
            <small style="color: var(--text-muted);">Bajo el umbral</small>
          </div>
        </div>
      </div>
    </section>

    <!-- ================= PANTALLA 3: MODO MANUAL ================= -->
    <section id="screenManual" class="screen">
      <div class="top-bar-nav">
        <button class="btn-back" onclick="irAInicio()">⬅ Volver al Menú Principal</button>

        <!-- Selector pequeño superior para Pin 32 -->
        <div class="small-switch-box">
          <span>Control Web</span>
          <label class="switch">
            <input type="checkbox" id="switchPin32" onchange="cambiarTipoManual(this.checked)">
            <span class="slider-round"></span>
          </label>
          <span>Control Físico (Pin 32)</span>
        </div>
      </div>

      <div class="card">
        <h2>Control Manual de Salidas</h2>
        <p class="subtitle">Pin 27 DESACTIVADO | Accionamiento exclusivo entre Pin 33 y Pin 18</p>

        <!-- Mensaje de Alerta si Modo Manual Físico está activo -->
        <div id="physicalWarning" class="physical-alert">
          <h3>⚠️ MODO MANUAL FÍSICO ACTIVO (Pin 32 ON)</h3>
          <p style="margin-top: 8px;">El control y la visualización desde la web están deshabilitados mientras el selector físico esté activo.</p>
        </div>

        <!-- Área de Control Web (Se desactiva si Pin 32 está en Físico) -->
        <div id="webManualContainer" class="disabled-area">
          <div class="pulse-controls">
            <button class="btn-pulse btn-on" onclick="enviarComandoManual(true)">
              PULSADOR ON (Pin 33)
            </button>
            <button class="btn-pulse btn-off" onclick="enviarComandoManual(false)">
              PULSADOR OFF (Pin 18)
            </button>
          </div>

          <!-- Indicadores Circulares Pin 33 y Pin 18 -->
          <div class="indicators-box">
            <div class="indicator-item">
              <div id="circlePin33" class="circle"></div>
              <strong>PIN 33 (ON)</strong><br>
              <small style="color: var(--text-muted);">Activado</small>
            </div>

            <div class="indicator-item">
              <div id="circlePin18" class="circle"></div>
              <strong>PIN 18 (OFF)</strong><br>
              <small style="color: var(--text-muted);">Desactivado</small>
            </div>
          </div>
        </div>
      </div>
    </section>
  </main>

  <!-- ================= MODAL CONFIGURACIÓN API KEY ================= -->
  <div id="configModal" class="modal">
    <div class="modal-content">
      <h2>Conexión Adafruit IO</h2>
      <p class="subtitle" style="margin-bottom: 1rem;">Credenciales de enlace con el ESP32</p>
      <div class="form-group">
        <label>IO_USERNAME</label>
        <input type="text" id="inputUser">
      </div>
      <div class="form-group">
        <label>IO_KEY</label>
        <input type="text" id="inputKey">
      </div>
      <div class="modal-actions">
        <button class="btn-back" onclick="cerrarConfig()">Cancelar</button>
        <button class="btn-save" onclick="guardarConfig()">Guardar y Conectar</button>
      </div>
    </div>
  </div>

  <script>
    // ================= CREDENCIALES POR DEFECTO =================
    let IO_USERNAME = localStorage.getItem('aio_user') || "JONATHAN2020";
    let IO_KEY      = localStorage.getItem('aio_key')  || "aio_AWdH67u8hMd8yrltgFWfj7MAFqdO";

    // Variables de estado
    let tempActual = 0;
    let umbralActual = 30;
    let modoAutomatico = true;
    let manualFisico = false;
    let manualWebOn = false;

    // ================= CONFIGURACIÓN DEL GRÁFICO =================
    const ctx = document.getElementById('tempChart').getContext('2d');
    const tempChart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Temperatura Promedio (°C)',
          data: [],
          borderColor: '#3b82f6',
          backgroundColor: 'rgba(59, 130, 246, 0.15)',
          borderWidth: 3,
          fill: true,
          tension: 0.35
        }]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: { ticks: { color: '#94a3b8' }, grid: { color: '#334155' } },
          y: { ticks: { color: '#94a3b8' }, grid: { color: '#334155' } }
        },
        plugins: { legend: { labels: { color: '#f8fafc' } } }
      }
    });

    // ================= NAVEGACIÓN ENTRE PANTALLAS =================
    function mostrarPantalla(id) {
      document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
      document.getElementById(id).classList.add('active');
    }

    function irAInicio() {
      mostrarPantalla('screenHome');
    }

    async function seleccionarModo(esAuto) {
      modoAutomatico = esAuto;
      actualizarEstilosMenu();
      await publicarFeed('modo-sistema', esAuto ? 1 : 0);
      mostrarPantalla(esAuto ? 'screenAuto' : 'screenManual');
    }

    function actualizarEstilosMenu() {
      document.getElementById('cardBtnAuto').classList.toggle('selected', modoAutomatico);
      document.getElementById('cardBtnManual').classList.toggle('selected', !modoAutomatico);
    }

    // ================= ACCIONES DE CONTROL =================
    function actualizarTextoUmbral(val) {
      document.getElementById('umbralValText').innerText = parseFloat(val).toFixed(1);
    }

    async function enviarUmbral(val) {
      umbralActual = parseFloat(val);
      evaluarCirculosAutomatico();
      await publicarFeed('umbral', umbralActual);
    }

    async function cambiarTipoManual(esFisico) {
      manualFisico = esFisico;
      actualizarVistaManualFisico();
      await publicarFeed('modo-manual-tipo', esFisico ? 1 : 0);
    }

    async function enviarComandoManual(encender) {
      if (manualFisico) return; // Bloqueado si está en modo físico
      manualWebOn = encender;
      evaluarCirculosManual();
      await publicarFeed('manual-cmd', encender ? 1 : 0);
    }

    // ================= LÓGICA VISUAL DE CÍRCULOS (ENCLAVAMIENTO) =================
    function evaluarCirculosAutomatico() {
      const c25 = document.getElementById('circlePin25'); // Verde ON
      const c26 = document.getElementById('circlePin26'); // Rojo OFF

      if (tempActual > umbralActual) {
        c25.className = 'circle green-active';
        c26.className = 'circle';
      } else {
        c25.className = 'circle';
        c26.className = 'circle red-active';
      }
    }

    function evaluarCirculosManual() {
      const c33 = document.getElementById('circlePin33'); // Verde ON
      const c18 = document.getElementById('circlePin18'); // Rojo OFF

      if (manualWebOn) {
        c33.className = 'circle green-active';
        c18.className = 'circle';
      } else {
        c33.className = 'circle';
        c18.className = 'circle red-active';
      }
    }

    function actualizarVistaManualFisico() {
      const warning = document.getElementById('physicalWarning');
      const webArea = document.getElementById('webManualContainer');
      document.getElementById('switchPin32').checked = manualFisico;

      if (manualFisico) {
        warning.style.display = 'block';
        webArea.classList.add('locked');
      } else {
        warning.style.display = 'none';
        webArea.classList.remove('locked');
        evaluarCirculosManual();
      }
    }

    // ================= COMUNICACIÓN REST CON ADAFRUIT IO =================
    async function publicarFeed(feedKey, valor) {
      const url = `https://io.adafruit.com/api/v2/${IO_USERNAME}/feeds/${feedKey}/data`;
      try {
        const res = await fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-AIO-Key': IO_KEY
          },
          body: JSON.stringify({ value: valor })
        });
        setEstadoConexion(res.ok);
      } catch (e) {
        setEstadoConexion(false);
      }
    }

    async function leerFeed(feedKey) {
      const url = `https://io.adafruit.com/api/v2/${IO_USERNAME}/feeds/${feedKey}/data/last`;
      const res = await fetch(url, { headers: { 'X-AIO-Key': IO_KEY } });
      if (!res.ok) throw new Error("Error HTTP");
      return await res.json();
    }

    async function sincronizarDatos() {
      try {
        const [dTemp, dUmbral, dModo, dTipo, dCmd] = await Promise.all([
          leerFeed('temperatura'),
          leerFeed('umbral'),
          leerFeed('modo-sistema'),
          leerFeed('modo-manual-tipo'),
          leerFeed('manual-cmd')
        ]);

        setEstadoConexion(true);

        // 1. Temperatura y Gráfico
        tempActual = parseFloat(dTemp.value);
        document.getElementById('tempActualText').innerText = tempActual.toFixed(1);
        agregarPuntoGrafico(tempActual);

        // 2. Umbral (solo actualizar slider si el usuario no lo está arrastrando)
        umbralActual = parseFloat(dUmbral.value);
        if (document.activeElement.id !== 'umbralSlider') {
          document.getElementById('umbralSlider').value = umbralActual;
          actualizarTextoUmbral(umbralActual);
        }

        // 3. Modos y estados
        modoAutomatico = (parseInt(dModo.value) === 1);
        manualFisico   = (parseInt(dTipo.value) === 1);
        manualWebOn    = (parseInt(dCmd.value)  === 1);

        actualizarEstilosMenu();
        evaluarCirculosAutomatico();
        actualizarVistaManualFisico();

      } catch (error) {
        setEstadoConexion(false);
      }
    }

    function agregarPuntoGrafico(valor) {
      const hora = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
      const labels = tempChart.data.labels;
      const data = tempChart.data.datasets[0].data;

      if (labels.length > 20) {
        labels.shift();
        data.shift();
      }
      labels.push(hora);
      data.push(valor);
      tempChart.update();
    }

    function setEstadoConexion(online) {
      const dot = document.getElementById('connDot');
      const txt = document.getElementById('connText');
      if (online) {
        dot.classList.add('online');
        txt.innerText = `Conectado (${IO_USERNAME})`;
      } else {
        dot.classList.remove('online');
        txt.innerText = 'Sin conexión / Revisar Feeds o API Key';
      }
    }

    // ================= MODAL DE CONFIGURACIÓN =================
    function abrirConfig() {
      document.getElementById('inputUser').value = IO_USERNAME;
      document.getElementById('inputKey').value = IO_KEY;
      document.getElementById('configModal').classList.add('open');
    }

    function cerrarConfig() {
      document.getElementById('configModal').classList.remove('open');
    }

    function guardarConfig() {
      IO_USERNAME = document.getElementById('inputUser').value.trim();
      IO_KEY = document.getElementById('inputKey').value.trim();
      localStorage.setItem('aio_user', IO_USERNAME);
      localStorage.setItem('aio_key', IO_KEY);
      cerrarConfig();
      sincronizarDatos();
    }

    // Iniciar ciclo de lectura cada 3 segundos
    sincronizarDatos();
    setInterval(sincronizarDatos, 3000);
  </script>
</body>
</html>
