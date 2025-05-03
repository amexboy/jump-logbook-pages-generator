<template>
  <div>
    <h1>Generate Your Epic Logbook (Cover + Pages)</h1>
    <div class="controls">
      <div>
        <input type="checkbox" id="includeCover" v-model="includeCoverPage">
        <label for="includeCover">Include Cover Page</label>
      </div>
      <div>
        <label for="startPage">From Page:</label>
        <input type="number" id="startPage" v-model.number="startPage" min="1">
      </div>
      <div>
        <label for="endPage">To Page:</label>
        <input type="number" id="endPage" v-model.number="endPage" :min="startPage">
      </div>
      <button @click="generatePdf" :disabled="generatingPdf || !logbookSvgContent || (includeCoverPage && !coverSvgContent) || endPage < startPage || !fontLoaded">
        {{ generatingPdf ? 'Generating...' : 'Generate PDF' }}
      </button>
    </div>

    <div class="printable-sheets-container" ref="printableSheetsContainer" v-if="logbookSvgContent">
      <!-- Cover Page Preview (Optional) -->
      <div class="printable-sheet" v-if="includeCoverPage && coverSvgContent">
        <div class="cover-page-svg">
          <div class="svg-wrapper" v-html="coverSvgContent"></div>
        </div>
      </div>

      <!-- Logbook Pages Preview -->
      <div
          v-for="(sheet, sheetIndex) in printableSheets"
          :key="sheetIndex"
          class="printable-sheet"
      >
        <div class="logbook-page-svg left-page">
          <div class="svg-wrapper" v-html="getLogbookSvgWithNumber(sheet[0])"></div>
        </div>
        <div class="gutter"></div>
        <div class="logbook-page-svg right-page">
          <div class="svg-wrapper" v-html="getLogbookSvgWithNumber(sheet[1])"></div>
        </div>
      </div>
    </div>
    <div v-else>
      Loading SVGs...
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import jsPDF from 'jspdf';
// noinspection ES6UnusedImports
import {svg2pdf} from 'svg2pdf.js';
// Import your SVG files as raw text
import logbookDesignSvg from '@/design.svg?raw'; // Your logbook page design
import coverDesignSvg from '@/cover-design.svg?raw'; // Your cover page design (assuming this filename)
import '@/assets/Excalifont-normal.js';

const startPage = ref(1);
const endPage = ref(10);
const includeCoverPage = ref(true); // State for the cover page checkbox
const generatingPdf = ref(false);
const printableSheetsContainer = ref(null);
const logbookSvgContent = ref(null);
const coverSvgContent = ref(null); // To hold the cover page SVG content
const fontLoaded = ref(false);

// Define physical dimensions in mm
const sheetWidthMM = 210;
const gutterWidthMM = 7.5; // Example gutter width for binding
const logbookPageWidthMM = (sheetWidthMM - gutterWidthMM) / 2;

// You MUST adjust this based on your LOGBOOK SVG's actual content aspect ratio
const logbookPageAspectRatio = 843.888469714838 / 470.963060858978;
const logbookPageHeightMM = logbookPageWidthMM / logbookPageAspectRatio;
const sheetHeightMM = logbookPageHeightMM; // The sheet height is determined by the page height

// **You will also need to determine the aspect ratio of your COVER PAGE SVG**
// Assuming your cover page SVG has a viewBox or dimensions that match the sheet aspect ratio
const coverPageAspectRatio = sheetWidthMM / sheetHeightMM;


// Computed property to get the range of pages to generate
const pageRange = computed(() => {
  const start = Math.max(1, startPage.value);
  const end = Math.max(start, endPage.value);
  const range = [];
  for (let i = start; i <= end; i++) {
    range.push(i);
  }
  // Ensure an even number of pages for pairing
  if (range.length % 2 !== 0) {
    range.push(undefined); // Add a blank page representation
  }
  return range;
});

// Computed property to group pages into printable sheets (pairs)
const printableSheets = computed(() => {
  const sheets = [];
  for (let i = 0; i < pageRange.value.length; i += 2) {
    sheets.push([pageRange.value[i], pageRange.value[i + 1]]);
  }
  return sheets;
});

// Function to load the SVG content
const loadSvgs = async () => {
  logbookSvgContent.value = logbookDesignSvg;
  coverSvgContent.value = coverDesignSvg; // Load the cover page SVG
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

  // Convert dimensions from mm to points
  const sheetWidthPts = sheetWidthMM * 2.83465;
  const sheetHeightPts = sheetHeightMM * 2.83465;
  const logbookPageWidthPts = logbookPageWidthMM * 2.83465;
  const gutterWidthPts = gutterWidthMM * 2.83465;

  // Set PDF size and orientation
  const pdf = new jsPDF('l', 'pt', [sheetWidthPts, sheetHeightPts]);


  await nextTick(); // Ensure the DOM is updated for logbook pages
  // Add Cover Page (Optional)
  if (includeCoverPage.value && coverSvgContent.value) {
    const parser = new DOMParser();
    const svgDoc = parser.parseFromString(coverSvgContent.value, 'image/svg+xml');
    const coverSvgElement = svgDoc.documentElement; // Get the root SVG element

    await pdf.svg(coverSvgElement, {
      x: 0,
      y: 0,
      width: sheetWidthPts, // Cover page fills the entire sheet width
      height: sheetHeightPts, // Cover page fills the entire sheet height
    });
  }

  const sheets = printableSheetsContainer.value.querySelectorAll('.printable-sheet');

  for (let i = 0; i < sheets.length; i++) {
    const sheet = sheets[i];
    const leftPageSvgWrapper = sheet.querySelector('.left-page .svg-wrapper');
    const rightPageSvgWrapper = sheet.querySelector('.right-page .svg-wrapper');

    // Add a new page for the logbook sheets if a cover page was added
    // or if it's not the first sheet and no cover page was added.
    if (includeCoverPage.value || i > 0) {
      pdf.addPage();
    }


    // Add left page SVG
    if (leftPageSvgWrapper && leftPageSvgWrapper.firstElementChild) {
      const leftSvgElement = leftPageSvgWrapper.firstElementChild;
      await pdf.svg(leftSvgElement, {
        x: 0,
        y: 0,
        width: logbookPageWidthPts,
        height: sheetHeightPts,
      });
    }

    // Add right page SVG
    if (rightPageSvgWrapper && rightPageSvgWrapper.firstElementChild) {
      const rightSvgElement = rightPageSvgWrapper.firstElementChild;
      await pdf.svg(rightSvgElement, {
        x: logbookPageWidthPts + gutterWidthPts,
        y: 0,
        width: logbookPageWidthPts,
        height: sheetHeightPts,
      });
    }
  }

  pdf.save('epic_logbook_with_cover.pdf');
  generatingPdf.value = false;
};

onMounted(async () => {
  await loadSvgs(); // Load both SVGs
  // You might need to load the font data here or ensure it's available
  fontLoaded.value = true;
});
</script>

<style scoped>
.printable-sheets-container {
  display: flex;
  flex-direction: column;
  gap: 20px; /* Space between sheets */
}

.printable-sheet {
  border: 1px solid #ccc;
  display: flex;
  justify-content: space-between;
  align-items: stretch;
  margin: 0 auto;
  background-color: white;
  overflow: hidden;

  width: calc(v-bind(sheetWidthMM) * 1mm);
  height: calc(v-bind(sheetHeightMM) * 1mm);
}

.cover-page-svg {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}


.logbook-page-svg {
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;

  width: calc(v-bind(logbookPageWidthMM) * 1mm);
  height: 100%;
}

.gutter {
  width: calc(v-bind(gutterWidthMM) * 1mm);
  height: 100%;
  background-color: #eee;
}


.svg-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.svg-wrapper svg {
  display: block;
  width: 100%;
  height: 100%;
  /* Adjust preserveAspectRatio based on your SVG's viewBox */
}

.controls {
  margin-bottom: 20px;
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap; /* Allow controls to wrap on smaller screens */
}
</style>
