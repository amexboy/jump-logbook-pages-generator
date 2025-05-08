<template>
  <div class="app-container">
    <h1>Generate Your Epic Logbook (Cover + Pages)</h1>
    <div class="controls">
      <div>
        <input type="checkbox" id="includeCover" v-model="includeCoverPage"/>
        <label for="includeCover">Include Cover Page</label>
      </div>

      <fieldset class="layout-options">
        <legend>Logbook Page Layout</legend>
        <div v-for="(layout, key) in predefinedLogbookLayouts" :key="key">
          <input type="radio" :id="'layout_' + key" :value="key" v-model="selectedLogbookLayoutKey"
                 name="logbookLayoutSource"/>
          <label :for="'layout_' + key">{{ layout.name }}</label>
        </div>
      </fieldset>

      <div v-if="includeCoverPage" class="cover-options">
        <fieldset>
          <legend>Cover Background Image</legend>
          <div>
            <input type="radio" id="coverImagePredefined" value="predefined" v-model="coverImageSelectionMode"
                   name="coverImageSource"/>
            <label for="coverImagePredefined">Use Predefined</label>
            <select v-if="coverImageSelectionMode === 'predefined'" v-model="selectedPredefinedImageKey">
              <option v-for="(img, key) in predefinedCoverImages" :key="key" :value="key">
                {{ img.name }}
              </option>
            </select>
          </div>
          <div>
            <input type="radio" id="coverImageUpload" value="upload" v-model="coverImageSelectionMode"
                   name="coverImageSource"/>
            <label for="coverImageUpload">Upload Image</label>
            <input v-if="coverImageSelectionMode === 'upload'" type="file" @change="handleCoverImageUpload"
                   accept="image/*"/>
          </div>
        </fieldset>

        <fieldset>
          <legend>Cover SVG Design</legend>
          <div>
            <input type="radio" id="coverSvgPredefined" value="predefined" v-model="coverSvgSelectionMode"
                   name="coverSvgSource"/>
            <label for="coverSvgPredefined">Use Predefined</label>
            <select v-if="coverSvgSelectionMode === 'predefined'" v-model="selectedPredefinedSvgKey">
              <option v-for="(svg, key) in predefinedCoverSvgs" :key="key" :value="key">
                {{ svg.name }}
              </option>
            </select>
          </div>
          <div>
            <input type="radio" id="coverSvgUpload" value="upload" v-model="coverSvgSelectionMode"
                   name="coverSvgSource"/>
            <label for="coverSvgUpload">Upload SVG</label>
            <input v-if="coverSvgSelectionMode === 'upload'" type="file" @change="handleCoverSvgUpload"
                   accept=".svg, image/svg+xml"/>
          </div>
        </fieldset>
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
            (includeCoverPage && (!currentCoverSvgContent || !currentCoverImageSrc || !backgroundImageLoaded)) ||
            endPage < startPage ||
            !fontLoaded
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
    <div v-else class="status-message">
      <div v-if="generatingPdf">Generating PDF...</div>
      <div v-else>
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
        <p v-if="logbookSvgContent && (!includeCoverPage || (currentCoverSvgContent && currentCoverImageSrc && backgroundImageLoaded))">
          Ready to generate PDF.
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import {onBeforeUnmount, onMounted, ref, watch} from 'vue'; // Added watch
import jsPDF from 'jspdf';
// noinspection ES6UnusedImports
import {svg2pdf} from 'svg2pdf.js';
// Import your SVG files as raw text
import baseLogbookDesignSvg from '@/assets/base-design.svg?raw'; // Base logbook page design
import skydivingLogbookDesignSvg from '@/assets/skydiving-design.svg?raw'; // Skydiving logbook page design
// Cover design and background image will be handled dynamically
import '@/assets/Excalifont-normal.js';

// Predefined assets imports
import defaultCoverDesignSvg from '@/assets/cover-page/default-cover-design.svg?raw';
import defaultCoverImage from '@/assets/cover-page/default-cover-image.png';


const startPage = ref(1);
const endPage = ref(10);
const includeCoverPage = ref(true); // State for the cover page checkbox
const generatingPdf = ref(false);
const logbookSvgContent = ref(null); // Will be updated based on selection
const selectedLogbookLayoutKey = ref('base'); // Default layout key
// Removed coverSvgContent, will use currentCoverSvgContent
const fontLoaded = ref(false);
const pdfUrl = ref(null); // To store the URL of the generated PDF Blob
const backgroundImageLoaded = ref(false); // To track if the *current* cover image is loaded

// New refs for cover page customization
const coverImageSelectionMode = ref('predefined'); // 'predefined' or 'upload'
const selectedPredefinedImageKey = ref('default'); // Key for predefinedCoverImages
const uploadedImageFile = ref(null); // Stores the File object for uploaded image
const currentCoverImageSrc = ref(null); // Stores path or dataURL of the current image for the cover

const coverSvgSelectionMode = ref('predefined'); // 'predefined' or 'upload'
const selectedPredefinedSvgKey = ref('default'); // Key for predefinedCoverSvgs
const uploadedSvgFile = ref(null); // Stores the File object for uploaded SVG
const currentCoverSvgContent = ref(null); // Stores string content of the current cover SVG

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


// You MUST adjust this based on your LOGBOOK SVG's actual content aspect ratio
const logbookPageAspectRatio = 223.279 / 124.609 // 843.888469714838 / 470.963060858978;
const logbookPageHeightMM = logbookPageWidthMM / logbookPageAspectRatio;
const sheetHeightMM = logbookPageHeightMM + (padding * 2); // The sheet height is determined by the page height

console.log("Document size", sheetWidthMM, sheetHeightMM)

// Function to load the logbook SVG content (now handled by watcher)
const loadLogbookSvg = async () => {
  // logbookSvgContent is now set dynamically by the watcher
};

// Function to load an image from a given src (path or dataURL)
const loadImageFromSrc = (src) => {
  return new Promise((resolve, reject) => {
    if (!src) {
      backgroundImageLoaded.value = false; // Ensure flag is false if no src
      reject(new Error("Image source is not provided."));
      return;
    }
    const img = new Image();
    img.onload = () => {
      backgroundImageLoaded.value = true; // Set flag upon successful load
      resolve(img);
    };
    img.onerror = (err) => {
      backgroundImageLoaded.value = false; // Ensure flag is false on error
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
    uploadedImageFile.value = file; // Store the file object
    const reader = new FileReader();
    reader.onload = (e) => {
      currentCoverImageSrc.value = e.target.result; // This will trigger the watcher to load the image
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
    uploadedSvgFile.value = file; // Store the file object
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
  const textElement = svgDoc.getElementById('pageNumberText'); // Replace 'pageNumberText' with the actual ID

  if (textElement) {
    textElement.textContent = pageNumber !== undefined ? `# ${pageNumber}` : '';
    // Ensure the font-family is set on the text element in your SVG
    // For example, add style="font-family: 'Excalifont-Regular';" to the text element in design.svg
  }

  const serializer = new XMLSerializer();
  return serializer.serializeToString(svgDoc);
};

// Watch for changes in logbook layout selection
watch(selectedLogbookLayoutKey, (newKey) => {
  logbookSvgContent.value = predefinedLogbookLayouts[newKey]?.content || null;
  // Recalculate dimensions if aspect ratio changes - Placeholder for future enhancement if needed
  // For now, assuming both designs have the same aspect ratio or the difference is acceptable.
  // If aspect ratios differ significantly, logbookPageAspectRatio and related calculations
  // would need to become dynamic based on the selected layout.
}, {immediate: true});


// Watchers to update current cover assets based on selections and load them
watch([coverImageSelectionMode, selectedPredefinedImageKey], ([mode, key]) => {
  if (mode === 'predefined') {
    const newSrc = predefinedCoverImages[key]?.src || null;
    // Only update if the source is actually different to avoid re-triggering image load unnecessarily
    if (currentCoverImageSrc.value !== newSrc) {
      currentCoverImageSrc.value = newSrc;
    }
  } else { // mode === 'upload'
    if (uploadedImageFile.value) {
      // If a file was previously uploaded and user switches back to 'upload' mode
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
      // No file uploaded yet for 'upload' mode, or clear if switching to upload without a file
      if (currentCoverImageSrc.value !== null) { // Avoid redundant sets if already null
        currentCoverImageSrc.value = null;
      }
    }
  }
}, {immediate: true});

// Watch currentCoverImageSrc to load the image and update backgroundImageLoaded
watch(currentCoverImageSrc, async (newSrc, oldSrc) => {
  if (newSrc) {
    if (newSrc !== oldSrc || !backgroundImageLoaded.value) { // Load if new src or if previous attempt failed/not loaded
      backgroundImageLoaded.value = false; // Reset loading state
      try {
        await loadImageFromSrc(newSrc); // loadImageFromSrc sets backgroundImageLoaded to true on success
      } catch (error) {
        console.error("Watcher: Error loading image:", error);
        // backgroundImageLoaded is set to false by loadImageFromSrc on error
      }
    }
  } else {
    backgroundImageLoaded.value = false; // No source, so not loaded
  }
}, {immediate: true}); // immediate: true to attempt loading the initial image source

// Watch for changes in cover SVG selection
watch([coverSvgSelectionMode, selectedPredefinedSvgKey], ([mode, key]) => {
  if (mode === 'predefined') {
    currentCoverSvgContent.value = predefinedCoverSvgs[key]?.content || null;
  } else { // mode === 'upload'
    if (uploadedSvgFile.value) {
      // If an SVG file was previously uploaded
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
      currentCoverSvgContent.value = null; // No SVG uploaded yet for 'upload' mode
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

  // Add Cover Page (Optional)
  if (includeCoverPage.value && currentCoverSvgContent.value && currentCoverImageSrc.value) {
    if (!backgroundImageLoaded.value) {
      // This check is important. If image isn't loaded, we shouldn't proceed or should show an error.
      // For robustness, generatePdf should ideally only be callable if backgroundImageLoaded is true (if cover is included).
      // The button's :disabled state should enforce this.
      console.error("Cover image not loaded. PDF generation might be incorrect.");
      // Optionally, try to load it again, though watcher should handle this.
      // await loadImageFromSrc(currentCoverImageSrc.value); // This might be redundant if button is properly disabled.
    }

    const img = await loadImageFromSrc(currentCoverImageSrc.value); // Load the current image

    const imgAspectRatio = img.width / img.height;
    const imgHeightPts = sheetHeightPts; // Make image height match page height
    const imgWidthPts = imgHeightPts * imgAspectRatio;
    // Align image to the right of the cover page
    const imgX = sheetWidthPts - imgWidthPts;


    const parser = new DOMParser();
    const svgDoc = parser.parseFromString(currentCoverSvgContent.value, 'image/svg+xml');
    const coverSvgElement = svgDoc.documentElement; // Get the root SVG element

    // Add background image to the cover page
    // Using 0 for width/height tells jsPDF to auto-calculate based on aspect ratio if one dimension is provided.
    // Here, we provide height and want it to fill that height, positioned right.
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

    // Add a new page for the logbook sheets if a cover page was added
    // or if it's not the first sheet and no cover page was added.
    if (includeCoverPage.value || i > 0) {
      pdf.addPage();
    }

    // Add background image to the first logbook page if no cover page was included
    // This uses the *cover's selected image* as the background for the first sheet.
    if (!includeCoverPage.value && i === 0 && currentCoverImageSrc.value) {
      try {
        const img = await loadImageFromSrc(currentCoverImageSrc.value); // Load the current cover image
        // Add image as full background to the first logbook sheet
        pdf.addImage(img, 'PNG', 0, 0, sheetWidthPts, sheetHeightPts);
      } catch (error) {
        console.error("Failed to load background image for first logbook page:", error);
      }
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
  // Load essential logbook SVG
  await loadLogbookSvg();
  fontLoaded.value = true; // Assuming font is loaded via the import

  // Initial cover assets (SVG content and image source) are set by immediate watchers.
  // The image itself is loaded asynchronously by the currentCoverImageSrc watcher.
  // The user will click "Generate PDF" when ready.
});

// Clean up the Blob URL when the component is unmounted
onBeforeUnmount(() => {
  if (pdfUrl.value) {
    URL.revokeObjectURL(pdfUrl.value);
  }
});
</script>

<style scoped>
/* General App Container Styles */
.app-container {
  margin: 0; /* Use full width, remove horizontal auto margins */
  padding: 20px; /* Retain padding for content spacing from viewport edges */
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
  color: #333;
  background-color: #f4f7f9;
}

/* Header */
h1 {
  color: #2c3e50;
  text-align: center;
  margin-bottom: 40px;
  font-size: 2rem;
}

/* Controls Section */
.controls {
  margin-bottom: 30px;
  display: flex;
  flex-direction: column; /* Stack main control groups vertically */
  gap: 25px; /* Space between control groups */
  background-color: #ffffff;
  padding: 25px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

/* Specific layout for label-on-top-of-input groups (e.g., "Start Page", "End Page") */
.controls > div:has(label + input[type="number"]) {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* Styling for the "Include Cover Page" checkbox and label line */
.controls > div:has(input[type="checkbox"] + label) {
  display: flex;
  align-items: center;
  gap: 8px;
}

/* Styling for fieldsets and their legends */
.controls fieldset {
  border: 1px solid #dce4e9;
  padding: 20px;
  border-radius: 8px;
}

.controls legend {
  font-weight: 600;
  color: #34495e;
  padding: 0 8px;
  font-size: 1.1em;
  margin-bottom: 10px;
}

/* Styling for groups within fieldsets (e.g., radio button lines) */
.controls fieldset > div {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  margin-bottom: 12px;
}

.controls fieldset > div > select,
.controls fieldset > div > input[type="file"] {
  flex-grow: 1; /* Allow select/file input to take up remaining space */
  min-width: 150px; /* Ensure a minimum width */
}

/* Input and Label Styling */
label {
  font-weight: 500;
  color: #4a5568;
  font-size: 0.95rem;
}

input[type="checkbox"], input[type="radio"] {
  margin-right: 5px;
  accent-color: #4299e1; /* Blue accent */
  width: 1.15em;
  height: 1.15em;
}

input[type="number"],
input[type="file"],
select {
  padding: 12px 15px;
  border: 1px solid #cbd5e0;
  border-radius: 6px;
  font-size: 1rem;
  width: 100%;
  box-sizing: border-box;
  background-color: #fff;
  transition: border-color 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
}

input[type="number"]:focus,
input[type="file"]:focus,
select:focus {
  border-color: #4299e1;
  box-shadow: 0 0 0 3px rgba(66, 153, 225, 0.3);
  outline: none;
}

input[type="file"] {
  cursor: pointer;
}

input[type="file"]::file-selector-button {
  padding: 8px 15px;
  margin-right: 10px;
  background-color: #edf2f7;
  color: #2d3748;
  border: 1px solid #cbd5e0;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

input[type="file"]::file-selector-button:hover {
  background-color: #e2e8f0;
}

button {
  padding: 12px 25px;
  background-color: #4299e1; /* Primary blue */
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 1.05rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease, transform 0.1s ease;
  align-self: flex-start; /* Align button to the start of the flex column (.controls) */
}

button:hover:not(:disabled) {
  background-color: #3182ce; /* Darker blue */
  transform: translateY(-1px);
}

button:disabled {
  background-color: #a0aec0;
  cursor: not-allowed;
  opacity: 0.7;
}

/* Cover Options: two fieldsets side-by-side */
.cover-options {
  display: flex;
  flex-direction: row;
  gap: 20px;
  flex-wrap: wrap; /* Allow fieldsets to stack on smaller screens */
}

.cover-options > fieldset {
  flex: 1; /* Distribute space between the two fieldsets */
  min-width: 280px; /* Minimum width before stacking */
}

/* PDF Display and Status Message */
.pdf-display, .status-message {
  margin-top: 40px;
  padding: 25px;
  background-color: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  text-align: center;
}

.pdf-display h2 {
  color: #2c3e50;
  margin-bottom: 20px;
  font-size: 1.5rem;
}

.status-message p {
  color: #718096;
  font-size: 1.1rem;
  line-height: 1.6;
}

iframe {
  border: 1px solid #dce4e9;
  border-radius: 8px;
  width: 100%; /* Responsive width */
  max-width: 1200px; /* Consistent max-width */
  display: block; /* Remove extra space below */
  margin: 0 auto; /* Center if max-width is hit */
  background-color: #fff; /* Ensure background for PDF iframe */
}
</style>
