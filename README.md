# Monumentos del Mundo - Visualización Interactiva 🌍

Este proyecto es una visualización de datos interactiva que representa monumentos históricos alrededor del mundo entre los años 1000 y 1999. La visualización está organizada como una matriz, donde el eje Y agrupa los monumentos por continente y el eje X por siglo de construcción. Cada monumento está representado por una figura SVG cuyo:

- **Color** indica su calificación cultural/turística.
- **Forma** representa la cantidad de visitas anuales (en millones).
- **Tamaño** relativo según la altura del monumento.

## 🖥 Ver en vivo

👉 [monumentos-blue.vercel.app](https://monumentos-blue.vercel.app)

## 📊 Datos

Los datos incluyen más de 100 monumentos reales (y algunos ficticios coherentes) y están estructurados con los siguientes campos:

- `nombre`: Nombre del monumento
- `continente`: Continente en el que se encuentra
- `altura`: Altura en metros (solo para referencia, no se utiliza en escala visual)
- `anio`: Año aproximado de construcción
- `visitas`: Visitas anuales en millones
- `calificacion`: Evaluación simbólica (Malo, Regular, Bueno, Excelente)

## ⚙️ Tecnologías

- [Svelte](https://svelte.dev/) – Framework principal para la interfaz
- [D3.js](https://d3js.org/) – Escalado de color y control de datos
- SVGs personalizados – Representan las visitas anuales por forma
- CSS Grid – Para estructurar la matriz de forma responsiva y clara

## 🧭 Cómo navegar

- Pasá el mouse por encima de cada figura para ver un tooltip con detalles del monumento.
- Las leyendas arriba explican el color (calificación) y la forma (visitas anuales).
- Los siglos están distribuidos horizontalmente y los continentes verticalmente.

## ✨ Créditos

Visualización creada por Melina Dborkin, Guadalupe Koruk y Maria Paz Yunes como parte del proyecto académico de **Visualización de Datos** en la Universidad Torcuato Di Tella.

## 📝 Licencia

Este proyecto es de uso académico y educativo. Los datos e imágenes se usan con fines ilustrativos.