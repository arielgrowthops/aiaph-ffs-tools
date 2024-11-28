<script setup>
import { reactive, ref } from "vue";
import JSZip from "jszip";
import { saveAs } from "file-saver";
import { PDFDocument } from "pdf-lib";

// State Vars
const zip = new JSZip();
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
  sortText: false,
});

const processFile = async (file) => {
  return new Promise(async (resolve, reject) => {
    try {
      console.log(file);
      const actualFile = file.file;

      if (!(actualFile instanceof File)) {
        console.error("Invalid file:", actualFile);
        reject(new Error("The provided object is not a valid File"));
        return;
      }

      const reader = new FileReader();

      reader.onload = async (e) => {
        const pdfBytes = new Uint8Array(e.target.result);
        const pdfDoc = await PDFDocument.load(pdfBytes);

        // Edit PDF metadata
        pdfDoc.setTitle(file.metadata);
        pdfDoc.setAuthor("Your Author Name");

        const modifiedPdfBytes = await pdfDoc.save();
        const modifiedPdfBlob = new Blob([modifiedPdfBytes], { type: "application/pdf" });

        // Update the file status and content
        file.status = "Processed";
        // Store the modified content
        file.modifiedContent = modifiedPdfBlob;

        resolve();
      };

      reader.onerror = (error) => {
        reject(error);
      };

      // Read the actual file as an ArrayBuffer
      reader.readAsArrayBuffer(actualFile);
    } catch (error) {
      console.error("Error processing file:", error);
      reject(error);
    }
  });
};

// Handle file uploads
const handleFileUpload = (event) => {
  const uploadedFiles = Array.from(event.target.files);

  uploadedFiles.forEach((file) => {
    // Debugging log
    console.log(file);

    if (file instanceof File && file.type === "application/pdf") {
      files.push({
        file,
        name: file.name,
        metadata: "Default Title",
        originalName: file.name,
        originalMetadata: "Default Title",
        isModified: false,
        status: "Pending",
      });
    } else {
      alert(`${file.name} is not a valid PDF file!`);
    }
  });

  // Reset the file input
  event.target.value = "";
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

// Remove a file node
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
      const { search, replace, regex } = selectedToolOptions.replaceText;
      // Handle regex replacement
      if (regex) {
        const re = new RegExp(search, "gi");
        updatedName = updatedName.replace(re, replace);
      } else {
        const searchText = search.replace(/\s+/g, " ");
        const re = new RegExp(searchText, "gi");
        updatedName = updatedName.replace(re, replace);
      }
    }

    // Handle Remvoe HTML
    if (selectedToolOptions.stripHTML) {
      updatedName = updatedName.replace(/<\/?[^>]+(>|$)/g, "");
    }

    // Handle Remove spaces
    if (selectedToolOptions.removeSpaces) {
      updatedName = updatedName.replace(/\s+/g, "");
    }

    file.name = updatedName;
  });
};

// Update processPDFs to include PDF metadata modification
const processPDFs = async () => {
  if (processDisabled.value) {
    alert("Please save all changes before processing!");
    return;
  }

  isProcessing.value = true;

  // Process files one by one
  for (let i = 0; i < files.length; i++) {
    if (files[i].status === "Pending") {
      processingIndex.value = i;
      files[i].status = "Processing";
      await processFile(files[i]);
    }
  }

   // Reset after processing
  processingIndex.value = null;
  isProcessing.value = false;
  validateProcessButton();
};

// Process PDFs to Zip
const downloadZip = async () => {
  const zip = new JSZip();

  files.forEach((file) => {
    if (file.status === "Processed" && file.modifiedContent) {
      zip.file(file.name, file.modifiedContent);
    }
  });

  const blob = await zip.generateAsync({ type: "blob" });
  saveAs(blob, "processed_files.zip");
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

    <!-- Tool Tab Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Tool Options</h2>
      <div class="space-y-4">
        <!-- To LowerCase -->
        <label class="flex items-center space-x-2">
          <input type="checkbox" class="form-checkbox text-blue-600" v-model="selectedToolOptions.toLowerCase" />
          <span>To LowerCase</span>
        </label>

        <!-- To UpperCase -->
        <label class="flex items-center space-x-2">
          <input type="checkbox" class="form-checkbox text-blue-600" v-model="selectedToolOptions.toUpperCase" />
          <span>To UpperCase</span>
        </label>

        <!-- Replace Text -->
        <label class="flex items-center space-x-2">
          <input type="checkbox" class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.replaceText.enabled" />
          <span>Replace Text</span>
        </label>
        <div v-if="selectedToolOptions.replaceText.enabled" class="space-y-2 pl-4">
          <div class="flex items-center space-x-2">
            <input type="text" placeholder="Search" class="border rounded px-2 py-1 w-1/3"
              v-model="selectedToolOptions.replaceText.search" />
            <input type="text" placeholder="Replace with" class="border rounded px-2 py-1 w-1/3"
              v-model="selectedToolOptions.replaceText.replace" />
          </div>
        </div>

        <!-- Strip HTML -->
        <label class="flex items-center space-x-2">
          <input type="checkbox" class="form-checkbox text-blue-600" v-model="selectedToolOptions.stripHTML" />
          <span>Strip HTML Code</span>
        </label>

        <!-- Remove Spaces -->
        <label class="flex items-center space-x-2">
          <input type="checkbox" class="form-checkbox text-blue-600" v-model="selectedToolOptions.removeSpaces" />
          <span>Remove Spaces</span>
        </label>
      </div>
      <button class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600" @click="applyToolOptions">
        Apply Tools
      </button>
    </div>

    <!-- File List Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Uploaded Files</h2>
      <!-- File table content here -->

      <table class="w-full border border-gray-200 rounded-lg">
        <thead class="bg-gray-200 text-left text-sm">
          <tr>
            <th class="p-2 border-b">No.</th>
            <th class="p-2 border-b">Filename</th>
            <th class="p-2 border-b">Metadata Title</th>
            <th class="p-2 border-b">Status</th>
            <th class="p-2 border-b">Action</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(file, index) in files" :key="index" class="hover:bg-gray-100"
            :class="{ 'bg-yellow-100': index === processingIndex }">
            <td class="p-2 border-b w-10 text-center">{{ index + 1 }}</td>
            <td class="p-2 border-b">
              <input type="text" :value="file.name" class="w-full border rounded px-4 py-2"
                @input="handleInputChange(index, 'name', $event.target.value)" />
            </td>
            <td class="p-2 border-b">
              <input type="text" :value="file.metadata" class="w-full border rounded px-4 py-2"
                @input="handleInputChange(index, 'metadata', $event.target.value)" />
            </td>

            <td class="p-2 border-b">
              <span class="px-2 py-1 rounded text-sm" :class="{
                'bg-gray-200': file.status === 'Pending',
                'bg-blue-200': file.status === 'Processing',
                'bg-green-200': file.status === 'Processed',
              }">
                {{ file.status }}
              </span>
            </td>
            <td class="p-2 border-b w-10 text-center space-y-2">
              <button class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600" :disabled="!file.isModified"
                @click="saveChanges(index)">
                Save
              </button>
              <button class="bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600" @click="removeFile(index)">
                Remove
              </button>
            </td>
          </tr>
        </tbody>
      </table>

      <button class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600"
        :disabled="processDisabled || isProcessing" @click="processPDFs">
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