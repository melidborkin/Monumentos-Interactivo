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

  // CORREGIDO: Función para obtener el siglo base (1000, 1100, etc.) desde un año
  function obtenerSiglo(anio) {
    if (!anio) return null;
    return Math.floor(anio / 100) * 100;
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
    monumentos.forEach(m => {
      m.altura = +m.altura;
      m.anio = +m.anio;
      m.visitas = +m.visitas;
      m.calificacion = m.calificacion?.trim();
      m.continente = m.continente?.trim();
    });
    fetchSVGs();
    
    // Crear elementos flotantes con posiciones aleatorias
    floatingElements = monumentNames.map(name => ({
      name,
      x: Math.random() * 80 + 10,
      y: Math.random() * 80 + 10,
      delay: Math.random() * 5,
      duration: 15 + Math.random() * 10
    }));
  });

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

  // FILTROS CORREGIDOS
  let filtrosFormas = [];
  let filtrosColores = [];
  let filtrosSiglos = [];
  let filtrosContinentes = [];

  // Toggle de valores dentro de un filtro
  function toggleValor(array, valor) {
    console.log("Toggle valor:", valor, "en array:", array);
    const index = array.indexOf(valor);
    if (index === -1) {
      array.push(valor);
      console.log("Agregado. Array ahora:", array);
    } else {
      array.splice(index, 1);
      console.log("Removido. Array ahora:", array);
    }
    
    // Forzar reactividad de manera más explícita
    if (array === filtrosFormas) {
      filtrosFormas = [...filtrosFormas];
    } else if (array === filtrosColores) {
      filtrosColores = [...filtrosColores];
    } else if (array === filtrosSiglos) {
      filtrosSiglos = [...filtrosSiglos];
    } else if (array === filtrosContinentes) {
      filtrosContinentes = [...filtrosContinentes];
    }
  }

  $: monumentosFiltrados = monumentos.filter(m => {
    const forma = svgPorAltura(m.altura);
    const color = m.calificacion;
    const continente = m.continente;
    const siglo = obtenerSiglo(m.anio);

    return (!filtrosFormas.length || filtrosFormas.includes(forma)) &&
          (!filtrosColores.length || filtrosColores.includes(color)) &&
          (!filtrosContinentes.length || filtrosContinentes.includes(continente)) &&
          (!filtrosSiglos.length || filtrosSiglos.includes(siglo));
  });


  // CORREGIDO: Función para determinar si un monumento pasa todos los filtros
  function cumpleFiltros(m) {
    const forma = svgPorAltura(m.altura)?.trim();
    const color = m.calificacion?.trim();
    const continente = m.continente?.trim();
    const siglo = Math.floor(m.anio / 100) * 100;

    return (!filtrosFormas.length || filtrosFormas.includes(forma)) &&
          (!filtrosColores.length || filtrosColores.includes(color)) &&
          (!filtrosContinentes.length || filtrosContinentes.includes(continente)) &&
          (!filtrosSiglos.length || filtrosSiglos.includes(siglo));
  }

  // Función para obtener monumentos por celda (limitado a 9)
  function getMonumentosPorCelda(continente, siglo) {
    return monumentos
      .filter(m => 
        m.continente === continente &&
        m.anio >= siglo &&
        m.anio < siglo + 100 &&
        cumpleFiltros(m)
      )
      .slice(0, 9);
  }


  // Landing page functions
  function mostrarContenido() {
    mostrarVisualizacion = true;
  }

  function volverALanding() {
    mostrarVisualizacion = false;
    // Scroll to top when returning to landing
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  // Función para limpiar todos los filtros
  function limpiarFiltros() {
    filtrosFormas = [];
    filtrosColores = [];
    filtrosSiglos = [];
    filtrosContinentes = [];
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
          <h2>Una mirada visual a los monumentos más representativos de la historia global construidos entre el año 1000 y 1999.</h2>
        </div>
      </div>
      
      <div class="sistema">
        <div class="leyenda-container">
          <div class="altura">
            <h3>Altura (medida en metros)</h3>

            <div class="formas">
              <button class="forma" 
                class:selected={filtrosFormas.includes("menosde18.svg")}  
                on:click={() => toggleValor(filtrosFormas, "menosde18.svg")}>
                <img src="/images/menosde18.svg" alt="Menos de 18 metros de altura"/>
                <span>Menos de 18</span>
              </button>
              <button class="forma"
                class:selected={filtrosFormas.includes("18a25.svg")}  
                on:click={() => toggleValor(filtrosFormas, "18a25.svg")}>
                <img src="/images/18a25.svg" alt="18–25 metros de altura"/>
                <span>18–25</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("25a30.svg")}  
                on:click={() => toggleValor(filtrosFormas, "25a30.svg")}>
                <img src="/images/25a30.svg" alt="25–30 metros de altura"/>
                <span>25–30</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("30a38.svg")}  
                on:click={() => toggleValor(filtrosFormas, "30a38.svg")}>
                <img src="/images/30a38.svg" alt="Entre 30 y 38 metros de altura"/>
                <span>30–38</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("38a47.svg")}  
                on:click={() => toggleValor(filtrosFormas, "38a47.svg")}>
                <img src="/images/38a47.svg" alt="Entre 38 y 47 metros de altura"/>
                <span>38–47</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("47a60.svg")}  
                on:click={() => toggleValor(filtrosFormas, "47a60.svg")}>
                <img src="/images/47a60.svg" alt="Entre 47 y 60 metros de altura"/>
                <span>47–60</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("60a80.svg")}  
                on:click={() => toggleValor(filtrosFormas, "60a80.svg")}>
                <img src="/images/60a80.svg" alt="Entre 60 y 80 metros de altura"/>
                <span>60–80</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("80a130.svg")}  
                on:click={() => toggleValor(filtrosFormas, "80a130.svg")}>
                <img src="/images/80a130.svg" alt="Entre 80 y 130 metros de altura"/>
                <span>80–130</span>
              </button>
              <button class="forma" 
                class:selected={filtrosFormas.includes("masde130.svg")}  
                on:click={() => toggleValor(filtrosFormas, "masde130.svg")}>
                <img src="/images/masde130.svg" alt="Más de 130 metros de altura"/>
                <span>Más de 130</span>
              </button>
            </div>
          </div>

          <div class="calificacion"> 
            <h3>Calificación (medido en cantidad de visitas)</h3>
            <div class="colores">
              <button class="color"
                class:selected={filtrosColores.includes("Malo")}
                on:click={() => toggleValor(filtrosColores, 'Malo')}>
                <div class="color-circle" style="background-color: #912F27;"></div>
                <span class="color-name">Malo</span>
              </button>
              <button class="color"
                class:selected={filtrosColores.includes("Regular")}
                on:click={() => toggleValor(filtrosColores, 'Regular')}>
                <div class="color-circle" style="background-color: #EA7B4D;"></div>
                <span class="color-name">Regular</span>
              </button>
              <button class="color"
                class:selected={filtrosColores.includes("Bueno")}
                on:click={() => toggleValor(filtrosColores, 'Bueno')}>
                <div class="color-circle" style="background-color: #A2D4D3;"></div>
                <span class="color-name">Bueno</span>
              </button>
              <button class="color"
                class:selected={filtrosColores.includes("Excelente")}
                on:click={() => toggleValor(filtrosColores, 'Excelente')}>
                <div class="color-circle" style="background-color: #3B7B78;"></div>
                <span class="color-name">Excelente</span>
              </button>
            </div>
          </div>

          <!-- Botón para limpiar filtros -->
          {#if filtrosFormas.length > 0 || filtrosColores.length > 0 || filtrosSiglos.length > 0 || filtrosContinentes.length > 0}
            <div class="filtros-actions">
              <button class="limpiar-btn" on:click={limpiarFiltros}>
                Limpiar todos los filtros
              </button>
            </div>
          {/if}
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
              <button 
                type="button"
                class="siglo-label" 
                class:active={activeColumn === siglo}
                class:selected={filtrosSiglos.includes(siglo)}
                on:click={() => toggleValor(filtrosSiglos, siglo)}
                on:mouseenter={(e) => {
                  setActiveColumn(siglo);
                  showSigloTooltip(e, siglo);
                }}
                on:mouseleave={() => {
                  clearActiveColumn();
                  hideTooltip();
                }}
                aria-label={`Siglo ${sigloARomano(siglo)}, años ${siglo} a ${siglo + 99}`}
              >
                {sigloARomano(siglo)}
              </button>
            {/each}
          </div>
        </div>
      
        <div class="matriz">
          {#each continentes as cont}
            <div class="fila">
              <button type="button"
                      class="label-y"
                      class:selected={filtrosContinentes.includes(cont)}
                      on:click={() => toggleValor(filtrosContinentes, cont)}>
                {cont}
              </button>
              <div class="celdas">
                {#each siglos as siglo}
                  <div class="celda" class:active-celda={activeColumn === siglo}>
                    <div class="grid-3x3">
                      {#each monumentosFiltrados.filter(m => m.continente === cont && obtenerSiglo(m.anio) === siglo).slice(0, 9) as m}

                        <div class="monumento visible"
                            style="color: {colorCalificacion(m.calificacion)}"
                            role = "img"
                            on:mousemove={(e) => showTooltip(e, m)}
                            on:mouseleave={hideTooltip}>

                          {@html svgMap[svgPorAltura(m.altura)]}
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
