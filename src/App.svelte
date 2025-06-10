<script>
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import monumentos from "/src/data/Monumentos - Hoja 1.csv";
  import { fade, fly } from 'svelte/transition';

  let svgMap = {};
  let activeColumn = null;
  let mostrarVisualizacion = false;
  let tooltip;

  // Lista de nombres de monumentos para flotar en el landing
  const monumentNames = [
    'Torre Eiffel', 'Coliseo', 'Machu Picchu', 'Taj Mahal', 'Cristo Redentor',
    'Estatua de la Libertad', 'Big Ben', 'Sagrada Familia', 'Petra', 'Chichen Itza',
    'Stonehenge', 'Alhambra', 'Neuschwanstein', 'Monte Rushmore', 'Angkor Wat',
    'Burj Khalifa', 'Sydney Opera', 'Golden Gate', 'Acropolis', 'Kremlin'
  ];
  
  let floatingElements = [];

  // Convierte los valores altura, anio y visitas a números
  monumentos.forEach(m => {
    m.altura = +m.altura;
    m.anio = +m.anio;
    m.visitas = +m.visitas;
  });

  // Asigna un color específico a cada calificación ("Malo", "Regular", "Bueno", "Excelente").
  const colorCalificacion = d3.scaleOrdinal()
    .domain(["Malo", "Regular", "Bueno", "Excelente"])
    .range(["#912F27", "#EA7B4D", "#A2D4D3", "#3B7B78"]);

  // Lista fija de continentes para estructurar la matriz visual (eje Y).
  const continentes = ["África", "América", "Asia", "Europa", "Oceanía"];

  // Genera los siglos del milenio (del 1000 al 1900, eje X) para posicionar los monumentos.
  const siglos = Array.from({ length: 10 }, (_, i) => 1000 + i * 100);
  
  // Función para convertir siglos a números romanos
  function sigloARomano(siglo) {
    const sigloNumero = siglo / 100;
    const romanos = ["XI", "XII", "XIII", "XIV", "XV", "XVI", "XVII", "XVIII", "XIX", "XX"];
    return romanos[sigloNumero - 10]; // Restamos 10 porque empezamos en el siglo XI (1000s)
  }

  // Devuelve el nombre del archivo SVG correspondiente según la cantidad de visitas anuales.
  function svgPorAltura(altura) {
    if (altura < 18) return "menosde18.svg";
    if (altura < 25) return "18a25.svg";
    if (altura < 30) return "25a30.svg";
    if (altura < 38) return "30a38.svg";
    if (altura < 47) return "38a47.svg";
    if (altura < 60) return "47a60.svg";
    if (altura < 80) return "60a80.svg";
    if (altura < 130) return "80a130.svg";
    return "masde130.svg";
  }

  async function fetchSVGs() {
    const nombres = monumentos.map(m => svgPorAltura(m.altura));
    const unicos = Array.from(new Set(nombres));
    // Modifica el contenido del SVG para permitir cambiar el color con CSS
    for (const nombre of unicos) {
      const res = await fetch(`/images/${nombre}`);
      let raw = await res.text();
      raw = raw.replace(/fill="(?!none).*?"/g, 'fill="currentColor"');
      svgMap[nombre] = raw;
    }
  }

  // Hace que los SVGs se carguen automáticamente cuando la visualización aparece en pantalla.
  onMount(() => {
    fetchSVGs();
    
    // Crear elementos flotantes con posiciones aleatorias
    floatingElements = monumentNames.map((name, index) => ({
      name,
      x: Math.random() * 80 + 10, // 10% a 90% del ancho
      y: Math.random() * 80 + 10, // 10% a 90% del alto
      delay: Math.random() * 5, // Delay aleatorio para animación
      duration: 15 + Math.random() * 10 // Duración de animación variable
    }));
  });

  function aplicarFiltroInteractivo(monumento) {
    filtroSigloActivo = Math.floor(monumento.anio / 100) * 100;
    filtroContinenteActivo = monumento.continente;
    filtroCalificacionActiva = monumento.calificacion;
    filtroAlturaActiva = svgPorAltura(monumento.altura);
  }

  function aplicarFiltros(monumento) {
    if (filtroSigloActivo && !(monumento.anio >= filtroSigloActivo && monumento.anio < filtroSigloActivo + 100)) return false;
    if (filtroContinenteActivo && monumento.continente !== filtroContinenteActivo) return false;
    if (filtroCalificacionActiva && monumento.calificacion !== filtroCalificacionActiva) return false;
    if (filtroAlturaActiva && svgPorAltura(monumento.altura) !== filtroAlturaActiva) return false;
    return true;
  }

  $: monumentosFiltrados = monumentos.filter(aplicarFiltros);

  function getMonumentosPorCelda(continente, siglo) {
    return monumentosFiltrados
      .filter(m => m.continente === continente && m.anio >= siglo && m.anio < siglo + 100)
      .slice(0, 9);
  }

  function showTooltip(event, m) {
    if (!tooltip) return;
    tooltip.innerHTML = `
      <div class="tooltip-header" style="background-color: ${colorCalificacion(m.calificacion)}">
        <strong>${m.nombre}</strong>
      </div>
      <div class="tooltip-content">
        <div class="tooltip-row"><span class="tooltip-label">Continente:</span> ${m.continente}</div>
        <div class="tooltip-row"><span class="tooltip-label">Altura:</span> ${m.altura.toLocaleString()} m</div>
        <div class="tooltip-row"><span class="tooltip-label">Año:</span> ${m.anio}</div>
        <div class="tooltip-row"><span class="tooltip-label">Visitas:</span> ${m.visitas} M</div>
        <div class="tooltip-row"><span class="tooltip-label">Calificación:</span> ${m.calificacion}</div>
      </div>
    `;
    tooltip.style.display = "block";
    tooltip.style.left = event.pageX + 15 + "px";
    tooltip.style.top = event.pageY + 15 + "px";
  }

  function hideTooltip() {
    if (!tooltip) return;
    tooltip.style.display = "none";
  }

  function showSigloTooltip(event, siglo) {
    if (!tooltip) return;
    tooltip.innerHTML = `
      <div class="tooltip-header tooltip-header-siglo">
        <strong>Siglo ${sigloARomano(siglo)}</strong>
      </div>
      <div class="tooltip-content">
        <div class="tooltip-row">Años ${siglo} - ${siglo + 99}</div>
      </div>
    `;
    tooltip.style.display = "block";
    tooltip.style.left = event.pageX + 15 + "px";
    tooltip.style.top = event.pageY + 15 + "px";
  }

  function setActiveColumn(siglo) {
    activeColumn = siglo;
  }

  function clearActiveColumn() {
    activeColumn = null;
  }

  // Filtros interactivos
  let filtroSigloActivo = null;
  let filtroContinenteActivo = null;
  let filtroCalificacionActiva = null;
  let filtroAlturaActiva = null;  
  $: monumentosFiltrados = monumentos.filter(m => {
    return (!filtroSigloActivo || (m.anio >= filtroSigloActivo && m.anio < filtroSigloActivo + 100)) &&
           (!filtroContinenteActivo || m.continente === filtroContinenteActivo) &&
           (!filtroCalificacionActiva || m.calificacion === filtroCalificacionActiva) &&
           (!filtroAlturaActiva || svgPorAltura(m.altura) === filtroAlturaActiva);
  });
  
  //Landing page functions
  function mostrarContenido() {
    mostrarVisualizacion = true;
  }

  function volverALanding() {
    mostrarVisualizacion = false;
    // Scroll to top when returning to landing
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
</script>

{#if !mostrarVisualizacion}
  <!-- Landing Page -->
  <div id="landing-page" in:fade={{ duration: 1000 }}>
    <div class="landing-content">
      <h1>Un viaje visual por los grandes monumentos de la humanidad</h1>
      <p>
        A lo largo del tiempo, la humanidad ha creado monumentos que reflejan su historia y cultura. Descubrí los más emblemáticos y las huellas que dejaron en el mundo.
      </p>
      <button class="comenzar-btn" on:click={mostrarContenido}>Comenzar el viaje</button>
    </div>
    
    <!-- Nombres de monumentos flotantes -->
    {#each floatingElements as element}
      <div 
        class="floating-monument"
        style="
          left: {element.x}%; 
          top: {element.y}%; 
          animation-delay: {element.delay}s;
          animation-duration: {element.duration}s;
        "
      >
        {element.name}
      </div>
    {/each}
  </div>
{:else}
  <!-- Contenido principal -->
  <div id="main-content" in:fly={{ y: 50, duration: 800, delay: 200 }}>
    <main>
      <div class="hero-section">
        <div class="hero-content">
          <h1>Monumentos icónicos del mundo</h1>
          <h2>Una mirada visual a los monumentos más representativos de la historia global contruidos entre el año 1000 y 1999.</h2>
        </div>
      </div>
      
      <div class="sistema">
        <div class="leyenda-container">
          <div class="altura">
            <h3>Altura (medida en metros)</h3>
            <div class="formas">
              <div class="forma">
                <img src="/images/menosde18.svg" alt="Menos de 18 metros de altura"/>
                <span>Menos de 18</span>
              </div>
              <div class="forma">
                <img src="/images/18a25.svg" alt="Entre 18 y 25 metros de altura"/>
                <span>18–25</span>
              </div>
              <div class="forma">
                <img src="/images/25a30.svg" alt="Entre 25 y 30 metros de altura"/>
                <span>25–30</span>
              </div>
              <div class="forma">
                <img src="/images/30a38.svg" alt="Entre 30 y 38 metros de altura"/>
                <span>30–38</span>
              </div>
              <div class="forma">
                <img src="/images/38a47.svg" alt="Entre 38 y 47 metros de altura"/>
                <span>38–47</span>
              </div>
              <div class="forma">
                <img src="/images/47a60.svg" alt="Entre 47 y 60 metros de altura"/>
                <span>47–60</span>
              </div>
              <div class="forma">
                <img src="/images/60a80.svg" alt="Entre 60 y 80 metros de altura"/>
                <span>60–80</span>
              </div>
              <div class="forma">
                <img src="/images/80a130.svg" alt="Entre 80 y 130 metros de altura"/>
                <span>80–130</span>
              </div>
              <div class="forma">
                <img src="/images/masde130.svg" alt="Más de 130 metros de altura"/>
                <span>Más de 130</span>
              </div>
            </div>
          </div>

          <div class="calificacion"> 
            <h3>Calificación (medido en cantidad de visitas)</h3>
            <div class="colores">
              <div class="color">
                <div class="color-circle" style="background-color: #912F27;"></div>
                <span class="color-name">Malo</span>
              </div>
              <div class="color">
                <div class="color-circle" style="background-color: #EA7B4D;"></div>
                <span class="color-name">Regular</span>
              </div>
              <div class="color">
                <div class="color-circle" style="background-color: #A2D4D3;"></div>
                <span class="color-name">Bueno</span>
              </div>
              <div class="color">
                <div class="color-circle" style="background-color: #3B7B78;"></div>
                <span class="color-name">Excelente</span>
              </div>
            </div>
          </div>
        </div>

      </div>

      <div class="matriz-container">

        <div class="intro"> 
          Cada columna representa un siglo. Cada fila, un continente. Cada figura, un monumento. Todo en una sola vista.  
          <div class="instrucciones">
            <p>Pase el cursor sobre un monumento para ver sus detalles. Pase el cursor sobre un siglo para ver su información</p>
        </div>       
        </div>
        

       
        <!-- Fila de siglos en números romanos (arriba) -->
        <div class="fila fila-siglos">
              <div class="label-y">Siglos</div>
          <div class="celdas-header">
            {#each siglos as siglo}
              <div 
                class="siglo-label" 
                class:active={activeColumn === siglo}
                on:mouseenter={(e) => {
                  setActiveColumn(siglo);
                  showSigloTooltip(e, siglo);
                }}
                on:mouseleave={() => {
                  clearActiveColumn();
                  hideTooltip();
                }}
                role="button"
                tabindex="0"
                aria-label={`Siglo ${sigloARomano(siglo)}, años ${siglo} a ${siglo + 99}`}
              >
                {sigloARomano(siglo)}
              </div>
            {/each}
          </div>
        </div>
      
        <div class="matriz">
          {#each continentes as cont}
            <div class="fila">
              <div class="label-y">{cont}</div>
              <div class="celdas">
                {#each siglos as siglo}
                  <div class="celda" class:active-celda={activeColumn === siglo}>
                    <div class="grid-3x3">
                      {#each Array(9).fill(null) as _, index}
                        <div class="celda-pequena">
                          {#if getMonumentosPorCelda(cont, siglo)[index]}
                            {@const m = getMonumentosPorCelda(cont, siglo)[index]}
                            <div
                              class="monumento"
                              style="color: {colorCalificacion(m.calificacion)}"
                              on:mousemove={(e) => showTooltip(e, m)}
                              on:mouseleave={hideTooltip}
                              role="img"
                              aria-label={`Monumento: ${m.nombre}, ${m.calificacion}, ${m.visitas}M visitas`}
                            >
                              {@html svgMap[svgPorAltura(m.altura)] || ''}
                            </div>
                          {/if}
                        </div>
                      {/each}
                    </div>
                  </div>
                {/each}
              </div>
            </div>
          {/each}
        </div>
      </div>

      <!-- Botón para volver -->
      <div class="volver-container">
        <button class="volver-btn" on:click={volverALanding}>
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M19 12H5M12 19l-7-7 7-7"/>
          </svg>
          Volver al inicio
        </button>
      </div>
      
      <footer>
        <div class="footer-content">
          <p>Dataviz creativa realizada por <b>Guadalupe Koruk</b>, <b>Paz Yunes</b> y <b>Melina Dborkin</b> | Tecnología Digital - Universidad Torcuato Di Tella
          <br> © {new Date().getFullYear()} - Visualización de monumentos icónicos del mundo
          <br>
          <a href="https://github.com/melidborkin/Monumentos" target="_blank">Github</a>  
          </p>      
        </div>
      </footer>

      <div class="tooltip" bind:this={tooltip}></div>
    </main>
  </div>
{/if}