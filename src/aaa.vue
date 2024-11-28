<script setup>
import { reactive, ref } from "vue";

// File upload and processing state
const files = reactive([]); // List of uploaded files
const processingInProgress = ref(false); // Indicates if the Process PDFs button is running

// Tool options state
const selectedToolOptions = reactive({
  toLowerCase: false,
  toUpperCase: false,
  replaceText: { enabled: false, search: "", replace: "", caseSensitive: false, regex: false },
  stripHTML: false,
  removeSpaces: false,
  removeLines: false,
});

// Handle file upload
const handleFileUpload = (event) => {
  const uploadedFiles = Array.from(event.target.files);
  uploadedFiles.forEach((file) => {
    if (file.type === "application/pdf") {
      files.push({
        name: file.name,
        originalName: file.name,
        metadata: "",
        isModified: false,
        status: "Pending",
      });
    } else {
      alert(`${file.name} is not a PDF!`);
    }
  });
  event.target.value = ""; // Reset file input
};

// Apply tool options to all filenames
const applyToolOptions = () => {
  files.forEach((file) => {
    let updatedName = file.originalName;

    if (selectedToolOptions.toLowerCase) {
      updatedName = updatedName.toLowerCase();
    }
    if (selectedToolOptions.toUpperCase) {
      updatedName = updatedName.toUpperCase();
    }
    if (selectedToolOptions.replaceText.enabled) {
      const { search, replace, caseSensitive, regex } = selectedToolOptions.replaceText;
      const flags = caseSensitive ? "" : "i";
      const pattern = regex ? new RegExp(search, flags) : new RegExp(search.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"), flags);
      updatedName = updatedName.replace(pattern, replace);
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

    // Update filename and mark as modified
    file.name = updatedName;
    file.isModified = file.name !== file.originalName;
  });
};

// Save changes for a single file
const saveChanges = (index) => {
  files[index].originalName = files[index].name;
  files[index].isModified = false;
};

// Remove a file
const removeFile = (index) => {
  files.splice(index, 1);
};

// Process PDFs one by one
const processPDFs = async () => {
  if (files.some((file) => file.isModified)) {
    alert("Please save all changes before processing.");
    return;
  }

  processingInProgress.value = true;
  for (const file of files) {
    file.status = "Processing";
    await new Promise((resolve) => setTimeout(resolve, 1000)); // Simulate processing delay
    file.status = "Processed";
  }
  processingInProgress.value = false;
};
</script>

<template>
  <main class="bg-gray-100 p-4 space-y-6">
    <!-- Upload Files Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Upload Files</h2>
      <div class="flex items-center space-x-4">
        <label
          for="file-upload"
          class="cursor-pointer bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
        >
          Choose Files
        </label>
        <input
          id="file-upload"
          type="file"
          multiple
          class="hidden"
          @change="handleFileUpload"
        />
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

    <!-- File Table Section -->
    <div class="bg-white shadow rounded-lg p-6">
      <h2 class="text-lg font-semibold mb-4">Uploaded Files</h2>
      <table class="w-full border rounded">
        <thead class="bg-gray-200">
          <tr>
            <th class="p-2 border">No.</th>
            <th class="p-2 border">Filename</th>
            <th class="p-2 border">Status</th>
            <th class="p-2 border">Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(file, index) in files" :key="index">
            <td class="p-2 border text-center">{{ index + 1 }}</td>
            <td class="p-2 border">
              <input
                type="text"
                v-model="file.name"
                class="border rounded px-2 py-1 w-full"
              />
            </td>
            <td class="p-2 border text-center">
              <span
                :class="{
                  'bg-gray-300': file.status === 'Pending',
                  'bg-blue-300': file.status === 'Processing',
                  'bg-green-300': file.status === 'Processed'
                }"
                class="px-2 py-1 rounded"
              >
                {{ file.status }}
              </span>
            </td>
            <td class="p-2 border space-x-2">
              <button
                class="bg-green-500 text-white px-2 py-1 rounded"
                :disabled="!file.isModified"
                @click="saveChanges(index)"
              >
                Save
              </button>
              <button
                class="bg-red-500 text-white px-2 py-1 rounded"
                @click="removeFile(index)"
              >
                Remove
              </button>
            </td>
          </tr>
        </tbody>
      </table>
      <div class="mt-4">
        <button
          class="bg-blue-500 text-white px-4 py-2 rounded"
          :disabled="processingInProgress"
          @click="processPDFs"
        >
          {{ processingInProgress ? "Processing..." : "Process PDFs" }}
        </button>
      </div>
    </div>
  </main>
</template>
