<template>
  <div>
    <h1>Generate Your Epic Logbook (Cover + Pages)</h1>
    <div class="controls">
      <div>
        <input type="checkbox" id="includeCover" v-model="includeCoverPage"/>
        <label for="includeCover">Include Cover Page</label>
      </div>
      <div>
        <label for="startPage">From Page:</label>
        <input type="number" id="startPage" v-model.number="startPage" min="1"/>
      </div>
      <div>
        <label for="endPage">To Page:</label>
        <input type="number" id="endPage" v-model.number="endPage" :min="startPage"/>
      </div>
      <button
          @click="generatePdf"
          :disabled="
          generatingPdf ||
          !logbookSvgContent ||
          (includeCoverPage && !coverSvgContent) ||
          endPage < startPage ||
          !fontLoaded ||
          !backgroundImageLoaded
        "
      >
        {{ generatingPdf ? "Generating..." : "Generate PDF" }}
      </button>
    </div>

    <!-- PDF Display Area -->
    <div v-if="pdfUrl" class="pdf-display">
      <h2>Generated PDF</h2>
      <iframe :src="pdfUrl" height="600px" width="1200"></iframe>
    </div>
    <div v-else>
      {{ logbookSvgContent && backgroundImageLoaded ? 'Generating PDF...' : 'Loading Resources...' }}
    </div>
  </div>
</template>

<script setup>
import {onBeforeUnmount, onMounted, ref} from 'vue';
import jsPDF from 'jspdf';
// noinspection ES6UnusedImports
import {svg2pdf} from 'svg2pdf.js';
// Import your SVG files as raw text
import logbookDesignSvg from '@/design.svg?raw'; // Your logbook page design
import coverDesignSvg from '@/assets/cover-page/default-cover-design.svg?raw'; // Your cover page design (assuming this filename)
import '@/assets/Excalifont-normal.js';
import backgroundImage from '@/assets/cover-page/default-cover-image.png'; // Import your background image

const startPage = ref(1);
const endPage = ref(10);
const includeCoverPage = ref(true); // State for the cover page checkbox
const generatingPdf = ref(false);
const logbookSvgContent = ref(null);
const coverSvgContent = ref(null); // To hold the cover page SVG content
const fontLoaded = ref(false);
const pdfUrl = ref(null); // To store the URL of the generated PDF Blob
const backgroundImageLoaded = ref(false); // To track if the background image is loaded

// Define physical dimensions in mm
let padding = 3;
const sheetWidthMM = 420;
const gutterWidthMM = 10; // Example gutter width for binding
const logbookPageWidthMM = (sheetWidthMM - gutterWidthMM) / 2;


// You MUST adjust this based on your LOGBOOK SVG's actual content aspect ratio
const logbookPageAspectRatio = 223.279 / 124.609 // 843.888469714838 / 470.963060858978;
const logbookPageHeightMM = logbookPageWidthMM / logbookPageAspectRatio;
const sheetHeightMM = logbookPageHeightMM + (padding * 2); // The sheet height is determined by the page height

console.log("Document size", sheetWidthMM, sheetHeightMM)

// Function to load the SVG content
const loadSvgs = async () => {
  logbookSvgContent.value = logbookDesignSvg;
  coverSvgContent.value = coverDesignSvg; // Load the cover page SVG
};

// Function to load the background image
const loadImage = () => {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => {
      backgroundImageLoaded.value = true;
      resolve(img);
    };
    img.onerror = reject;
    img.src = backgroundImage; // Use the imported image URL
  });
};

// Function to get the Logbook SVG content with the updated page number
const getLogbookSvgWithNumber = (pageNumber) => {
  if (!logbookSvgContent.value || pageNumber === undefined) return '';

  const parser = new DOMParser();
  const svgDoc = parser.parseFromString(logbookSvgContent.value, 'image/svg+xml');
  const textElement = svgDoc.getElementById('pageNumberText'); // Replace 'pageNumberText' with the actual ID

  if (textElement) {
    textElement.textContent = pageNumber !== undefined ? `# ${pageNumber}` : '';
    // Ensure the font-family is set on the text element in your SVG
    // For example, add style="font-family: 'Excalifont-Regular';" to the text element in design.svg
  }

  const serializer = new XMLSerializer();
  return serializer.serializeToString(svgDoc);
};

const generatePdf = async () => {
  generatingPdf.value = true;
  pdfUrl.value = null; // Clear previous PDF

  // Convert dimensions from mm to points
  const sheetWidthPts = sheetWidthMM * 2.83465;
  const sheetHeightPts = sheetHeightMM * 2.83465;
  const logbookPageWidthPts = logbookPageWidthMM * 2.83465;
  const gutterWidthPts = gutterWidthMM * 2.83465;

  // Set PDF size and orientation
  const pdf = new jsPDF('l', 'pt', [sheetWidthPts, sheetHeightPts]);

  // Ensure background image is loaded before adding it
  const img = await loadImage();

  const imgAspectRatio = img.width / img.height;
  const imgHeightPts = sheetHeightPts; // Make image height match page height
  const imgWidthPts = imgHeightPts * imgAspectRatio;
  const imgX = sheetWidthPts - imgWidthPts; // Calculate x to align right


  // Add Cover Page (Optional)
  if (includeCoverPage.value && coverSvgContent.value) {
    const parser = new DOMParser();
    const svgDoc = parser.parseFromString(coverSvgContent.value, 'image/svg+xml');
    const coverSvgElement = svgDoc.documentElement; // Get the root SVG element

    // Add background image to the cover page
    pdf.addImage(img, 'PNG', imgX, 0, 0, imgHeightPts); // Add image before SVG

    await pdf.svg(coverSvgElement, {
      x: 0,
      y: 0,
      width: sheetWidthPts, // Cover page fills the entire sheet width
      height: sheetHeightPts, // Cover page fills the entire sheet height
    });
  }

  // Generate and add logbook pages
  let pagesToGenerate = endPage.value - startPage.value + 1;
  if (pagesToGenerate % 2 !== 0) {
    pagesToGenerate += 1;
    endPage.value += 1;
  }

  for (let i = startPage.value - 1; i < pagesToGenerate/2; i ++) {
    const rightPageNumber = startPage.value + i;
    const leftPageNumber = pagesToGenerate - (startPage.value - 1) - i;

    // Add a new page for the logbook sheets if a cover page was added
    // or if it's not the first sheet and no cover page was added.
    if (includeCoverPage.value || i > 0) {
      pdf.addPage();
    }

    // Add background image to the first logbook page if no cover page was included
    if (!includeCoverPage.value && i === 0) {
      pdf.addImage(img, 'PNG', 0, 0, sheetWidthPts, sheetHeightPts); // Add image as background to the first logbook page
    }

    // Create a temporary div to hold the left page SVG for rendering
    if (leftPageNumber !== undefined) {
      const tempDivLeft = document.createElement('div');
      tempDivLeft.innerHTML = getLogbookSvgWithNumber(leftPageNumber);
      const leftSvgElement = tempDivLeft.firstElementChild;
      if (leftSvgElement) {
        await pdf.svg(leftSvgElement, {
          x: padding,
          y: 0,
          width: logbookPageWidthPts,
          height: sheetHeightPts,
        });
      }
    }

    // Create a temporary div to hold the right page SVG for rendering
    if (rightPageNumber !== undefined) {
      const tempDivRight = document.createElement('div');
      tempDivRight.innerHTML = getLogbookSvgWithNumber(rightPageNumber);
      const rightSvgElement = tempDivRight.firstElementChild;
      if (rightSvgElement) {
        await pdf.svg(rightSvgElement, {
          x: logbookPageWidthPts + gutterWidthPts - padding,
          y: 0,
          width: logbookPageWidthPts,
          height: sheetHeightPts,
        });
      }
    }
  }

  // Get the PDF data as a Blob
  const pdfBlob = pdf.output('blob');

  // Create a URL for the Blob
  pdfUrl.value = URL.createObjectURL(pdfBlob);

  generatingPdf.value = false;
};

onMounted(async () => {
  await Promise.all([loadSvgs(), loadImage()]); // Load SVGs and image concurrently
  fontLoaded.value = true; // Assuming font is loaded via the import
  // Generate PDF on mount if all resources are loaded
  if (logbookSvgContent.value && (includeCoverPage.value ? coverSvgContent.value : true) && fontLoaded.value && backgroundImageLoaded.value) {
    generatePdf();
  }
});

// Clean up the Blob URL when the component is unmounted
onBeforeUnmount(() => {
  if (pdfUrl.value) {
    URL.revokeObjectURL(pdfUrl.value);
  }
});
</script>

<style scoped>
.controls {
  margin-bottom: 20px;
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap; /* Allow controls to wrap on smaller screens */
}

.pdf-display {
  margin-top: 20px;
}

.full-width-iframe {
  width: 100%;
}
</style>
