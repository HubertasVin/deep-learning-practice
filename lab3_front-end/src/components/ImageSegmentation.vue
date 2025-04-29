<script setup lang="ts">
import { ref, computed } from "vue";
import axios from "axios";
import JSZip from "jszip";

const imageUrl = ref("");
const serverUrl = ref("http://localhost:8000");
const selectedFile = ref<File | null>(null);
const loading = ref(false);
const error = ref("");
const originalImage = ref("");
const masks = ref<{ [key: string]: string }>({});
const toggles = ref<Record<string, boolean>>({});

const handleFileSelect = (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (input.files && input.files[0]) {
    selectedFile.value = input.files[0];
    imageUrl.value = "";
    originalImage.value = URL.createObjectURL(input.files[0]);
  }
};

const handleSubmit = async () => {
  if (!serverUrl.value) {
    error.value = "Please enter the server URL";
    return;
  }

  try {
    loading.value = true;
    error.value = "";
    masks.value = {};

    const predictUrl = `${serverUrl.value}/predict`;
    let response;

    try {
      if (selectedFile.value) {
        const formData = new FormData();
        formData.append("file", selectedFile.value);
        response = await axios.post(predictUrl, formData, {
          responseType: "arraybuffer",
          headers: {
            "Content-Type": "multipart/form-data",
          },
        });
      } else if (imageUrl.value) {
        response = await axios.post(
          predictUrl,
          {
            url: imageUrl.value,
          },
          {
            responseType: "arraybuffer",
            headers: {
              "Content-Type": "application/json",
            },
          },
        );
      } else {
        throw new Error("Please provide an image URL or file");
      }
    } catch (axiosError: any) {
      if (axiosError.response) {
        throw new Error(
          `Server error: ${axiosError.response.status} - ${axiosError.response.statusText}`,
        );
      } else if (axiosError.request) {
        throw new Error(
          "No response from server. Please check if the server is running and the URL is correct.",
        );
      } else {
        throw new Error(`Error setting up request: ${axiosError.message}`);
      }
    }

    try {
      const zip = await JSZip.loadAsync(response.data);
      const files = zip.files;

      // First, get the original image
      const originalImageFile = files["original_image.png"];
      if (originalImageFile) {
        const blob = await originalImageFile.async("blob");
        originalImage.value = URL.createObjectURL(blob);
      }

      masks.value = {};
      toggles.value = {};

      // Then process mask files
      for (const filename in files) {
        if (
          filename === "original_image.png" ||
          filename.startsWith("class_0_")
        )
          continue;

        const key = filename.replace(".png", "");
        const blob = await files[filename].async("blob");
        masks.value[key] = URL.createObjectURL(blob);
        toggles.value[key] = true;
      }
    } catch (zipError) {
      throw new Error(
        "Failed to process server response. Please check if the server returned the expected ZIP file.",
      );
    }
  } catch (e) {
    error.value =
      e instanceof Error ? e.message : "An unexpected error occurred";
  } finally {
    loading.value = false;
  }
};

const clearForm = () => {
  imageUrl.value = "";
  selectedFile.value = null;
  originalImage.value = "";
  masks.value = {};
  error.value = "";
};

const visibleMasks = computed(() =>
  Object.entries(masks.value)
    .filter(([key]) => toggles.value[key])
    .map(([_, url]) => url),
);
</script>

<template>
  <div class="container">
    <div class="input-section">
      <div class="server-input">
        <input
          type="text"
          v-model="serverUrl"
          placeholder="Enter server URL (e.g., http://localhost:8000)"
          :disabled="loading"
        />
      </div>
      <div class="input-group">
        <input
          type="text"
          v-model="imageUrl"
          placeholder="Enter image URL"
          :disabled="loading || !!selectedFile"
        />
        <span>OR</span>
        <input
          type="file"
          accept="image/*"
          @change="handleFileSelect"
          :disabled="loading || !!imageUrl"
        />
      </div>
      <div class="button-group">
        <button
          @click="handleSubmit"
          :disabled="loading || (!imageUrl && !selectedFile)"
        >
          {{ loading ? "Processing..." : "Submit" }}
        </button>
        <button @click="clearForm" :disabled="loading">Clear</button>
      </div>
      <div v-if="error" class="error">{{ error }}</div>
    </div>

    <div v-if="originalImage" class="result-section">
      <div class="image-container">
        <img :src="originalImage" alt="Original image" class="original-image" />
        <img
          v-for="maskUrl in visibleMasks"
          :key="maskUrl"
          :src="maskUrl"
          class="mask-overlay"
          alt="Segmentation mask"
        />
      </div>

      <div class="toggles">
        <label v-for="(value, key) in toggles" :key="key">
          <input type="checkbox" v-model="toggles[key]" />
          {{ key.split("_").slice(2, -1).join(" ") }}
        </label>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.input-section {
  margin-bottom: 20px;
}

.server-input {
  margin-bottom: 15px;
}

.server-input input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.input-group {
  display: flex;
  gap: 10px;
  align-items: center;
  margin-bottom: 10px;
}

.input-group input[type="text"] {
  flex: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.button-group {
  display: flex;
  gap: 10px;
}

button {
  padding: 8px 16px;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.error {
  color: red;
  margin-top: 10px;
}

.result-section {
  margin-top: 20px;
}

.image-container {
  position: relative;
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
}

.original-image,
.mask-overlay {
  width: 100%;
  height: auto;
}

.mask-overlay {
  position: absolute;
  top: 0;
  left: 0;
  opacity: 0.5;
}

.toggles {
  margin-top: 20px;
  display: flex;
  gap: 20px;
  justify-content: center;
}

.toggles label {
  display: flex;
  align-items: center;
  gap: 5px;
  cursor: pointer;
}
</style>
