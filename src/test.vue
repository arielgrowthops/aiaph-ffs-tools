<script setup>
import { reactive, ref } from "vue";

// Reactive state for file management
const files = reactive([]);
const isProcessing = ref(false); // Indicator for ongoing processing
const processDisabled = ref(true); // Disable Process button until conditions are met
const processingIndex = ref(null); // Tracks which file is being processed
const selectedToolOptions = reactive({
  toLowerCase: false,
  toUpperCase: false,
  replaceText: { enabled: false, search: "", replace: "", caseSensitive: false, regex: false },
  stripHTML: false,
  removeSpaces: false,
  removeLines: false,
  sortText: false,
});


// Simulate file processing (sequential processing)
const processFile = (file) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      file.status = "Processed"; // Update the file's status after processing
      resolve();
    }, 2000); // Simulate a delay for file processing
  });
};

// Handle file uploads (only PDF files)
const handleFileUpload = (event) => {
  const uploadedFiles = Array.from(event.target.files);

  // Validate files and add them to the queue
  uploadedFiles.forEach((file) => {
    if (file.type === "application/pdf") {
      files.push({
        name: file.name,
        metadata: "Default Title",
        originalName: file.name,
        originalMetadata: "Default Title",
        isModified: false,
        status: "Pending", // File status: Pending, Processing, Processed
      });
    } else {
      alert(`${file.name} is not a PDF!`);
    }
  });

  event.target.value = ""; // Reset the file input
  validateProcessButton();
};

// Update metadata or filename and track changes
const handleInputChange = (index, field, value) => {
  const file = files[index];
  file[field] = value;
  file.isModified =
    file.name !== file.originalName || file.metadata !== file.originalMetadata;
  validateProcessButton();
};

// Save changes for a file
const saveChanges = (index) => {
  const file = files[index];
  file.originalName = file.name;
  file.originalMetadata = file.metadata;
  file.isModified = false;
  validateProcessButton();
};

// Remove a file from the list
const removeFile = (index) => {
  files.splice(index, 1);
  validateProcessButton();
};

// Check if any unsaved changes exist
const validateProcessButton = () => {
  processDisabled.value =
    files.some((file) => file.isModified) || files.length === 0;
};

// Sequentially process the PDF files when "Process PDFs" is clicked
const processPDFs = async () => {
  if (processDisabled.value) {
    alert("Please save all changes before processing!");
    return;
  }

  isProcessing.value = true;

  // Process files one by one
  for (let i = 0; i < files.length; i++) {
    if (files[i].status === "Pending") {
      processingIndex.value = i; // Highlight the current file being processed
      files[i].status = "Processing";
      await processFile(files[i]);
    }
  }

  processingIndex.value = null; // Reset after processing
  isProcessing.value = false;
  validateProcessButton();
};

// Apply the selected tools to filenames
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
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.toLowerCase"
          />
          <span>To LowerCase</span>
        </label>

        <!-- To UpperCase -->
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.toUpperCase"
          />
          <span>To UpperCase</span>
        </label>

        <!-- Replace Text -->
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.replaceText.enabled"
          />
          <span>Replace Text</span>
        </label>
        <div v-if="selectedToolOptions.replaceText.enabled" class="space-y-2 pl-4">
          <div class="flex items-center space-x-2">
            <input
              type="text"
              placeholder="Search"
              class="border rounded px-2 py-1 w-1/3"
              v-model="selectedToolOptions.replaceText.search"
            />
            <input
              type="text"
              placeholder="Replace with"
              class="border rounded px-2 py-1 w-1/3"
              v-model="selectedToolOptions.replaceText.replace"
            />
          </div>
          <label class="flex items-center space-x-2">
            <input
              type="checkbox"
              class="form-checkbox text-blue-600"
              v-model="selectedToolOptions.replaceText.caseSensitive"
            />
            <span>Case sensitive</span>
          </label>
          <label class="flex items-center space-x-2">
            <input
              type="checkbox"
              class="form-checkbox text-blue-600"
              v-model="selectedToolOptions.replaceText.regex"
            />
            <span>Regular expressions</span>
          </label>
        </div>

        <!-- Strip HTML -->
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.stripHTML"
          />
          <span>Strip HTML Code</span>
        </label>

        <!-- Remove Spaces -->
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.removeSpaces"
          />
          <span>Remove Spaces</span>
        </label>

        <!-- Remove Lines -->
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            class="form-checkbox text-blue-600"
            v-model="selectedToolOptions.removeLines"
          />
          <span>Remove Lines</span>
        </label>
      </div>
      <button
        class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600"
        @click="applyToolOptions"
      >
        Apply Tools
      </button>
    </div>

    <!-- Table Display Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Uploaded Files</h2>
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
          <tr
            v-for="(file, index) in files"
            :key="index"
            class="hover:bg-gray-100"
            :class="{ 'bg-yellow-100': index === processingIndex }"
          >
            <td class="p-2 border-b w-10 text-center">{{ index + 1 }}</td>
            <td class="p-2 border-b">
              <input
                type="text"
                :value="file.name"
                class="w-full border rounded px-4 py-2"
                @input="handleInputChange(index, 'name', $event.target.value)"
              />
            </td>
            <td class="p-2 border-b">
              <input
                type="text"
                :value="file.metadata"
                class="w-full border rounded px-4 py-2"
                @input="handleInputChange(index, 'metadata', $event.target.value)"
              />
            </td>
            <td class="p-2 border-b">
              <span
                class="px-2 py-1 rounded text-sm"
                :class="{
                  'bg-gray-200': file.status === 'Pending',
                  'bg-blue-200': file.status === 'Processing',
                  'bg-green-200': file.status === 'Processed',
                }"
              >
                {{ file.status }}
              </span>
            </td>
            <td class="p-2 border-b w-10 text-center space-y-2">
              <button
                class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600"
                :disabled="!file.isModified"
                @click="saveChanges(index)"
              >
                Save
              </button>
              <button
                class="bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600"
                @click="removeFile(index)"
              >
                Remove
              </button>
            </td>
          </tr>
        </tbody>
      </table>

      <div>
        <button
          class="mt-5 bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600"
          :class="{ 'bg-gray-400 cursor-not-allowed': processDisabled || isProcessing }"
          :disabled="processDisabled || isProcessing"
          @click="processPDFs"
        >
          {{ isProcessing ? "Processing PDFs..." : "Process PDFs" }}
        </button>
      </div>
    </div>
  </main>
</template>