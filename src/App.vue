<template>
  <div class="min-h-screen bg-gray-100 p-4 sm:p-8 font-sans">
    <h1 class="text-3xl sm:text-4xl font-bold text-center text-gray-800 mb-8 sm:mb-12">
      Generate Your Epic Logbook (Cover + Pages)
    </h1>
    <div class="bg-white p-6 sm:p-8 rounded-xl shadow-lg space-y-6 mb-8 sm:mb-12 max-w-4xl mx-auto">
      <!-- Include Cover Page Checkbox -->
      <div class="flex items-center">
        <input type="checkbox" id="includeCover" v-model="includeCoverPage"
               class="h-4 w-4 text-indigo-600 border-gray-300 rounded focus:ring-indigo-500"/>
        <label for="includeCover" class="ml-2 block text-sm text-gray-900">Include Cover Page</label>
      </div>

      <!-- Logbook Page Layout -->
      <fieldset class="border border-gray-300 p-4 rounded-md">
        <legend class="text-lg font-semibold text-gray-700 px-1 mb-2">Logbook Page Layout</legend>
        <div class="space-y-2">
          <div v-for="(layout, key) in predefinedLogbookLayouts" :key="key" class="flex items-center">
            <input type="radio" :id="'layout_' + key" :value="key" v-model="selectedLogbookLayoutKey"
                   name="logbookLayoutSource"
                   class="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500"/>
            <label :for="'layout_' + key" class="ml-2 block text-sm text-gray-900">{{ layout.name }}</label>
          </div>
        </div>
      </fieldset>

      <!-- Cover Options (Conditional) -->
      <div v-if="includeCoverPage" class="grid grid-cols-1 md:grid-cols-2 gap-6">
        <!-- Cover Background Image Fieldset -->
        <fieldset class="border border-gray-300 p-4 rounded-md space-y-3">
          <legend class="text-lg font-semibold text-gray-700 px-1 mb-2">Cover Background Image</legend>
          <!-- Predefined Image -->
          <div class="space-y-1">
            <div class="flex items-center">
              <input type="radio" id="coverImagePredefined" value="predefined" v-model="coverImageSelectionMode"
                     name="coverImageSource"
                     class="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500"/>
              <label for="coverImagePredefined" class="ml-2 block text-sm text-gray-900">Use Predefined</label>
            </div>
            <select v-if="coverImageSelectionMode === 'predefined'" v-model="selectedPredefinedImageKey"
                    class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md">
              <option v-for="(img, key) in predefinedCoverImages" :key="key" :value="key">
                {{ img.name }}
              </option>
            </select>
          </div>
          <!-- Upload Image -->
          <div class="space-y-1">
            <div class="flex items-center">
              <input type="radio" id="coverImageUpload" value="upload" v-model="coverImageSelectionMode"
                     name="coverImageSource"
                     class="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500"/>
              <label for="coverImageUpload" class="ml-2 block text-sm text-gray-900">Upload Image</label>
            </div>
            <input v-if="coverImageSelectionMode === 'upload'" type="file" @change="handleCoverImageUpload"
                   accept="image/*"
                   class="mt-1 block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100"/>
          </div>
        </fieldset>

        <!-- Cover SVG Design Fieldset -->
        <fieldset class="border border-gray-300 p-4 rounded-md space-y-3">
          <legend class="text-lg font-semibold text-gray-700 px-1 mb-2">Cover SVG Design</legend>
          <!-- Predefined SVG -->
          <div class="space-y-1">
            <div class="flex items-center">
              <input type="radio" id="coverSvgPredefined" value="predefined" v-model="coverSvgSelectionMode"
                     name="coverSvgSource"
                     class="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500"/>
              <label for="coverSvgPredefined" class="ml-2 block text-sm text-gray-900">Use Predefined</label>
            </div>
            <select v-if="coverSvgSelectionMode === 'predefined'" v-model="selectedPredefinedSvgKey"
                    class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm rounded-md">
              <option v-for="(svg, key) in predefinedCoverSvgs" :key="key" :value="key">
                {{ svg.name }}
              </option>
            </select>
          </div>
          <!-- Upload SVG -->
          <div class="space-y-1">
            <div class="flex items-center">
              <input type="radio" id="coverSvgUpload" value="upload" v-model="coverSvgSelectionMode"
                     name="coverSvgSource"
                     class="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500"/>
              <label for="coverSvgUpload" class="ml-2 block text-sm text-gray-900">Upload SVG</label>
            </div>
            <input v-if="coverSvgSelectionMode === 'upload'" type="file" @change="handleCoverSvgUpload"
                   accept=".svg, image/svg+xml"
                   class="mt-1 block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700 hover:file:bg-indigo-100"/>
          </div>
        </fieldset>
      </div>

      <!-- Page Range Inputs -->
      <div class="grid grid-cols-1 sm:grid-cols-2 gap-6">
        <div>
          <label for="startPage" class="block text-sm font-medium text-gray-700">From Page:</label>
          <input type="number" id="startPage" v-model.number="startPage" min="1"
                 class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"/>
        </div>
        <div>
          <label for="endPage" class="block text-sm font-medium text-gray-700">To Page:</label>
          <input type="number" id="endPage" v-model.number="endPage" :min="startPage"
                 class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"/>
        </div>
      </div>

      <!-- Generate PDF Button -->
      <button
          @click="generatePdf"
          :disabled="
            generatingPdf ||
            !logbookSvgContent ||
            (includeCoverPage && (!currentCoverSvgContent || !currentCoverImageSrc || !backgroundImageLoaded)) ||
            endPage < startPage ||
            !fontLoaded
          "
          class="w-full sm:w-auto flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed"
      >
        {{ generatingPdf ? "Generating..." : "Generate PDF" }}
      </button>
    </div>

    <!-- Status Message Area (when no PDF is displayed) -->
    <div v-if="!pdfUrl" class="bg-white p-6 rounded-xl shadow-lg text-center text-gray-600 max-w-4xl mx-auto">
      <div v-if="generatingPdf" class="text-lg font-semibold">Generating PDF...</div>
      <div v-else class="space-y-2">
        <p v-if="!logbookSvgContent">Loading Logbook Design...</p>
        <p v-if="includeCoverPage && !currentCoverSvgContent">
          <span v-if="coverSvgSelectionMode === 'upload' && !uploadedSvgFile">Please upload a cover SVG design or select a predefined one.</span>
          <span v-else-if="coverSvgSelectionMode === 'predefined' && !predefinedCoverSvgs[selectedPredefinedSvgKey]">Select a predefined cover SVG.</span>
          <span v-else>Waiting for Cover SVG...</span>
        </p>
        <p v-if="includeCoverPage && !currentCoverImageSrc">
          <span v-if="coverImageSelectionMode === 'upload' && !uploadedImageFile">Please upload a cover image or select a predefined one.</span>
          <span
              v-else-if="coverImageSelectionMode === 'predefined' && !predefinedCoverImages[selectedPredefinedImageKey]">Select a predefined cover image.</span>
          <span v-else>Waiting for Cover Image...</span>
        </p>
        <p v-if="includeCoverPage && currentCoverImageSrc && !backgroundImageLoaded">Loading Cover Image...</p>
        <p v-if="logbookSvgContent && (!includeCoverPage || (currentCoverSvgContent && currentCoverImageSrc && backgroundImageLoaded))"
           class="text-green-600 font-semibold">
          Ready to generate PDF.
        </p>
      </div>
    </div>

    <!-- PDF Modal -->
    <div v-if="pdfUrl"
         class="fixed inset-0 bg-black bg-opacity-75 flex items-center justify-center z-50 p-4 transition-opacity duration-300 ease-in-out"
         @click.self="closePdfModal"> <!-- Close modal on backdrop click -->
      <div class="bg-white rounded-lg shadow-xl w-full h-full max-w-6xl max-h-[95vh] flex flex-col overflow-hidden">
        <!-- Modal Header -->
        <div class="flex justify-between items-center p-4 border-b border-gray-200">
          <h2 class="text-xl font-semibold text-gray-800">Generated PDF</h2>
          <button @click="closePdfModal"
                  class="text-gray-500 hover:text-gray-700 focus:outline-none text-2xl leading-none">&times;
          </button>
        </div>
        <!-- Modal Body / Iframe Container -->
        <div class="flex-grow p-1 bg-gray-200">
          <iframe :src="pdfUrl" class="w-full h-full border-none"></iframe>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import {onBeforeUnmount, onMounted, ref, watch} from 'vue';
import jsPDF from 'jspdf';
import {svg2pdf} from 'svg2pdf.js';
import baseLogbookDesignSvg from '@/assets/base-design.svg?raw'; // Base logbook page design
import skydivingLogbookDesignSvg from '@/assets/skydiving-design.svg?raw'; // Skydiving logbook page design
import '@/assets/Excalifont-normal.js';

// Predefined assets imports
import defaultCoverDesignSvg from '@/assets/cover-page/default-cover-design.svg?raw';
import defaultCoverImage from '@/assets/cover-page/default-cover-image.png';


const startPage = ref(1);
const endPage = ref(10);
const includeCoverPage = ref(true);
const generatingPdf = ref(false);
const logbookSvgContent = ref(null);
const selectedLogbookLayoutKey = ref('base');
const fontLoaded = ref(false);
const pdfUrl = ref(null);
const backgroundImageLoaded = ref(false);

// New refs for cover page customization
const coverImageSelectionMode = ref('predefined');
const selectedPredefinedImageKey = ref('default');
const uploadedImageFile = ref(null);
const currentCoverImageSrc = ref(null);

const coverSvgSelectionMode = ref('predefined');
const selectedPredefinedSvgKey = ref('default');
const uploadedSvgFile = ref(null);
const currentCoverSvgContent = ref(null);

// Predefined assets collections
const predefinedCoverSvgs = {
  'default': {name: 'Default SVG Design', content: defaultCoverDesignSvg},
};

const predefinedCoverImages = {
  'default': {name: 'Default Background Image', src: defaultCoverImage},
};

// Predefined Logbook Layouts
const predefinedLogbookLayouts = {
  'base': {name: 'Base Design', content: baseLogbookDesignSvg},
  'skydiving': {name: 'Skydiving Design', content: skydivingLogbookDesignSvg},
};


// Define physical dimensions in mm
let padding = 3;
const sheetWidthMM = 420;
const gutterWidthMM = 10; // Example gutter width for binding
const logbookPageWidthMM = (sheetWidthMM - gutterWidthMM) / 2;


const logbookPageAspectRatio = 223.279 / 124.609 // 843.888469714838 / 470.963060858978;
const logbookPageHeightMM = logbookPageWidthMM / logbookPageAspectRatio;
const sheetHeightMM = logbookPageHeightMM + (padding * 2);

console.log("Document size", sheetWidthMM, sheetHeightMM)

const loadLogbookSvg = async () => {
};

const loadImageFromSrc = (src) => {
  return new Promise((resolve, reject) => {
    if (!src) {
      backgroundImageLoaded.value = false;
      reject(new Error("Image source is not provided."));
      return;
    }
    const img = new Image();
    img.onload = () => {
      backgroundImageLoaded.value = true;
      resolve(img);
    };
    img.onerror = (err) => {
      backgroundImageLoaded.value = false;
      console.error("Error loading image from src:", src, err);
      reject(err);
    };
    img.src = src;
  });
};

// Handler for custom cover image upload
const handleCoverImageUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    uploadedImageFile.value = file;
    const reader = new FileReader();
    reader.onload = (e) => {
      currentCoverImageSrc.value = e.target.result;
    };
    reader.onerror = (err) => {
      console.error("Error reading uploaded image file:", err);
      currentCoverImageSrc.value = null;
      backgroundImageLoaded.value = false;
    };
    reader.readAsDataURL(file);
  }
};

// Handler for custom cover SVG upload
const handleCoverSvgUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    uploadedSvgFile.value = file;
    const reader = new FileReader();
    reader.onload = (e) => {
      currentCoverSvgContent.value = e.target.result;
    };
    reader.onerror = (err) => {
      console.error("Error reading uploaded SVG file:", err);
      currentCoverSvgContent.value = null;
    };
    reader.readAsText(file);
  }
};

// Function to get the Logbook SVG content with the updated page number
const getLogbookSvgWithNumber = (pageNumber) => {
  if (!logbookSvgContent.value || pageNumber === undefined) return '';

  const parser = new DOMParser();
  const svgDoc = parser.parseFromString(logbookSvgContent.value, 'image/svg+xml');
  const textElement = svgDoc.getElementById('pageNumberText');

  if (textElement) {
    textElement.textContent = pageNumber !== undefined ? `# ${pageNumber}` : '';
  }

  const serializer = new XMLSerializer();
  return serializer.serializeToString(svgDoc);
};

watch(selectedLogbookLayoutKey, (newKey) => {
  logbookSvgContent.value = predefinedLogbookLayouts[newKey]?.content || null;
}, {immediate: true});


watch([coverImageSelectionMode, selectedPredefinedImageKey], ([mode, key]) => {
  if (mode === 'predefined') {
    const newSrc = predefinedCoverImages[key]?.src || null;
    if (currentCoverImageSrc.value !== newSrc) {
      currentCoverImageSrc.value = newSrc;
    }
  } else {
    if (uploadedImageFile.value) {
      const reader = new FileReader();
      reader.onload = (e) => {
        currentCoverImageSrc.value = e.target.result;
      };
      reader.onerror = (err) => {
        console.error("Error re-reading uploaded image:", err);
        currentCoverImageSrc.value = null;
        backgroundImageLoaded.value = false;
      };
      reader.readAsDataURL(uploadedImageFile.value);
    } else {
      if (currentCoverImageSrc.value !== null) {
        currentCoverImageSrc.value = null;
      }
    }
  }
}, {immediate: true});

watch(currentCoverImageSrc, async (newSrc, oldSrc) => {
  if (newSrc) {
    if (newSrc !== oldSrc || !backgroundImageLoaded.value) {
      backgroundImageLoaded.value = false;
      try {
        await loadImageFromSrc(newSrc);
      } catch (error) {
        console.error("Watcher: Error loading image:", error);
      }
    }
  } else {
    backgroundImageLoaded.value = false;
  }
}, {immediate: true});

watch([coverSvgSelectionMode, selectedPredefinedSvgKey], ([mode, key]) => {
  if (mode === 'predefined') {
    currentCoverSvgContent.value = predefinedCoverSvgs[key]?.content || null;
  } else {
    if (uploadedSvgFile.value) {
      const reader = new FileReader();
      reader.onload = (e) => {
        currentCoverSvgContent.value = e.target.result;
      };
      reader.onerror = (err) => {
        console.error("Error re-reading uploaded SVG:", err);
        currentCoverSvgContent.value = null;
      };
      reader.readAsText(uploadedSvgFile.value);
    } else {
      currentCoverSvgContent.value = null;
    }
  }
}, {immediate: true});

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

  if (includeCoverPage.value && currentCoverSvgContent.value && currentCoverImageSrc.value) {
    if (!backgroundImageLoaded.value) {
      console.error("Cover image not loaded. PDF generation might be incorrect.");
    }

    const img = await loadImageFromSrc(currentCoverImageSrc.value);

    const imgAspectRatio = img.width / img.height;
    const imgHeightPts = sheetHeightPts;
    const imgWidthPts = imgHeightPts * imgAspectRatio;
    const imgX = sheetWidthPts - imgWidthPts;


    const parser = new DOMParser();
    const svgDoc = parser.parseFromString(currentCoverSvgContent.value, 'image/svg+xml');
    const coverSvgElement = svgDoc.documentElement;

    pdf.addImage(img, 'PNG', imgX, 0, imgWidthPts, imgHeightPts);

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

  for (let i = startPage.value - 1; i < pagesToGenerate / 2; i++) {
    const mainPageNumber = startPage.value + i;
    const otherPageNumber = pagesToGenerate - (startPage.value - 1) - i;
    const rightPageNumber = i % 2 === 0 ? mainPageNumber : otherPageNumber;
    const leftPageNumber = i % 2 === 0 ? otherPageNumber : mainPageNumber;

    if (includeCoverPage.value || i > 0) {
      pdf.addPage();
    }

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

  const pdfBlob = pdf.output('blob');
  pdfUrl.value = URL.createObjectURL(pdfBlob);

  generatingPdf.value = false;
};

onMounted(async () => {
  await loadLogbookSvg();
  fontLoaded.value = true;
});

const closePdfModal = () => {
  if (pdfUrl.value) {
    URL.revokeObjectURL(pdfUrl.value);
    pdfUrl.value = null;
  }
};

onBeforeUnmount(() => {
  if (pdfUrl.value) {
    URL.revokeObjectURL(pdfUrl.value);
  }
});
</script>
