<script setup>
import { reactive, ref } from "vue";
import JSZip from "jszip"; // Use JSZip for creating ZIP files
import { saveAs } from "file-saver"; // For downloading files

// Reactive state for file management
const files = reactive([]);
const isProcessing = ref(false);
const processDisabled = ref(true);
const processingIndex = ref(null);
const selectedToolOptions = reactive({
  toLowerCase: false,
  toUpperCase: false,
  replaceText: { enabled: false, search: "", replace: "", caseSensitive: false, regex: false },
  stripHTML: false,
  removeSpaces: false,
  removeLines: false,
  sortText: false,
});

const zip = new JSZip(); // Initialize a JSZip instance

// Simulate file processing
const processFile = (file) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      file.status = "Processed";
      resolve();
    }, 1000); // Simulate processing time
  });
};

// Handle file uploads
const handleFileUpload = (event) => {
  const uploadedFiles = Array.from(event.target.files);
  uploadedFiles.forEach((file) => {
    if (file.type === "application/pdf") {
      files.push({
        fileObject: file, // Store the actual file object
        name: file.name,
        metadata: "Default Title",
        originalName: file.name,
        originalMetadata: "Default Title",
        isModified: false,
        status: "Pending",
      });
    } else {
      alert(`${file.name} is not a PDF!`);
    }
  });
  event.target.value = ""; // Reset file input
  validateProcessButton();
};

// Handle input changes
const handleInputChange = (index, field, value) => {
  const file = files[index];
  file[field] = value;
  file.isModified = file.name !== file.originalName || file.metadata !== file.originalMetadata;
  validateProcessButton();
};

// Save changes
const saveChanges = (index) => {
  const file = files[index];
  file.originalName = file.name;
  file.originalMetadata = file.metadata;
  file.isModified = false;
  validateProcessButton();
};

// Remove a file
const removeFile = (index) => {
  files.splice(index, 1);
  validateProcessButton();
};

// Validate Process button
const validateProcessButton = () => {
  processDisabled.value = files.some((file) => file.isModified) || files.length === 0;
};

// Apply Tool Options
const applyToolOptions = () => {
  files.forEach((file) => {
    let updatedName = file.name;

    if (selectedToolOptions.toLowerCase) {
      updatedName = updatedName.toLowerCase();
    }

    if (selectedToolOptions.toUpperCase) {
      updatedName = updatedName.toUpperCase();
    }

    if (selectedToolOptions.replaceText.enabled) {
      const { search, replace, caseSensitive, regex } = selectedToolOptions.replaceText;
      if (regex) {
        const re = new RegExp(search, caseSensitive ? "" : "i");
        updatedName = updatedName.replace(re, replace);
      } else {
        const searchText = caseSensitive ? search : search.toLowerCase();
        updatedName = updatedName.replace(
          new RegExp(searchText, caseSensitive ? "" : "i"),
          replace
        );
      }
    }

    if (selectedToolOptions.stripHTML) {
      updatedName = updatedName.replace(/<\/?[^>]+(>|$)/g, "");
    }

    if (selectedToolOptions.removeSpaces) {
      updatedName = updatedName.replace(/\s+/g, "");
    }

    if (selectedToolOptions.removeLines) {
      updatedName = updatedName.replace(/[\r\n]+/g, "");
    }

    file.name = updatedName;
  });
};

// Process PDFs
const processPDFs = async () => {
  if (processDisabled.value) {
    alert("Please save all changes before processing!");
    return;
  }

  isProcessing.value = true;
  for (let i = 0; i < files.length; i++) {
    if (files[i].status === "Pending") {
      processingIndex.value = i;
      files[i].status = "Processing";
      await processFile(files[i]);
    }
  }
  processingIndex.value = null;
  isProcessing.value = false;

  validateProcessButton();
};

// Download ZIP of processed files
const downloadZip = () => {
  zip.folder("Processed PDFs");
  files.forEach((file) => {
    if (file.status === "Processed") {
      zip.file(file.name, file.fileObject);
    }
  });

  zip.generateAsync({ type: "blob" }).then((content) => {
    saveAs(content, "Processed_PDFs.zip");
  });
};
</script>


<template>
  <main class="bg-gray-100 p-4 space-y-6">
    <!-- Upload Files Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Upload Files</h2>
      <div class="flex items-center space-x-4">
        <label for="file-upload" class="cursor-pointer bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">
          Choose Files
        </label>
        <input id="file-upload" type="file" multiple class="hidden" @change="handleFileUpload" />
      </div>
    </div>

    <!-- Tool Options Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Tool Options</h2>
      <!-- Tool options content here -->
      <button class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600" @click="applyToolOptions">
        Apply Tools
      </button>
    </div>

    <!-- File List Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Uploaded Files</h2>
      <!-- File table content here -->
      <button
        class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600"
        :disabled="processDisabled || isProcessing"
        @click="processPDFs"
      >
        {{ isProcessing ? "Processing PDFs..." : "Process PDFs" }}
      </button>
    </div>

    <!-- Download ZIP Section -->
    <div v-if="!isProcessing && files.some(file => file.status === 'Processed')" class="bg-white shadow rounded-lg p-6">
      <button class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600" @click="downloadZip">
        Download Processed PDFs (ZIP)
      </button>
    </div>
  </main>
</template>
