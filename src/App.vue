<template>
  <div class="converter">
    <h1 class="maintitle">日语汉字转平假名</h1>

    <h2 class="subtitle">请输入日语文本：</h2><br />
    <textarea
      id="inputText"
      v-model="inputText"
      placeholder="在这里输入日语汉字文本..."
      rows="10"
    ></textarea>
    <br /><br />

    <label for="inputFile" style="color: rgb(10, 148, 202);">
      上传支持的图片 (png, jpg, jpeg, webp) 或文档 (pdf, doc, docx)：
    </label>
    <br />
    <label for="inputFile" class="custom-file-btn">上传文件</label>
    <input
      class="button"
      id="inputFile"
      ref="fileInput"
      type="file"
      :accept="acceptedFileTypes"
      @change="onFileChange"
    />
    <br /><br />

    <div v-if="selectedFile">
      <strong>已选文件：</strong> {{ selectedFile.name }}
      <button @click="clearSelectedFile" :disabled="loading">移除文件</button>
    </div>

    <div v-if="previewUrl && isImageFile" class="preview">
      <h3>图片预览：</h3>
      <img :src="previewUrl" alt="图片预览" />
    </div>

    <button :disabled="loading" @click="convertToKana">
      {{ loading ? "处理中..." : "转换成平假名" }}
    </button>

    <h2 class="subtitle">转换结果：</h2>
    <pre class="result">{{ result }}</pre>
  </div>
</template>

<script lang="ts" setup>
import { ref, computed, watch, onUnmounted } from "vue";
import { SSE } from "sse.js";

interface ApiResponse {
  type: string;
  content: string;
}

const inputText = ref("");
const selectedFile = ref<File | null>(null);
const loading = ref(false);
const result = ref("");
const previewUrl = ref<string | null>(null);
const sseResult = ref("");
const acceptedFileTypes = [".png", ".jpg", ".jpeg", ".webp", ".pdf", ".doc", ".docx"].join(",");
const sseClient = ref<SSE | null>(null);

const isImageFile = computed(() => {
  if (!selectedFile.value) return false;
  const imgExts = ["png", "jpg", "jpeg", "webp"];
  const ext = selectedFile.value.name.split(".").pop()?.toLowerCase() ?? "";
  return imgExts.includes(ext);
});

const apiAddUrl = "http://34.94.102.46:3308/api/add";
const apiImageUrl = "http://34.94.102.46:3308/api/image";
const apiFileUrl = "http://34.94.102.46:3308/api/file";

function onFileChange(event: Event) {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    const file = target.files[0];
    if (!file) {
      alert("请选择有效的上传文件！");
      clearSelectedFile();
      return;
    }
    const fileExt = file.name.split(".").pop()?.toLowerCase() ?? "";
    const allowedExts = ["png", "jpg", "jpeg", "webp", "pdf", "doc", "docx"];

    if (!allowedExts.includes(fileExt)) {
      alert("不支持此文件类型！");
      clearSelectedFile();
      return;
    }

    selectedFile.value = file;

    if (isImageFile.value) {
      previewUrl.value = URL.createObjectURL(file);
    } else {
      previewUrl.value = null;
    }
  } else {
    clearSelectedFile();
  }
}

function clearSelectedFile() {
  selectedFile.value = null;
  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value);
    previewUrl.value = null;
  }
  const inputElem = fileInput.value;
  if (inputElem) inputElem.value = "";
}

const fileInput = ref<HTMLInputElement | null>(null);

function uploadText() {
  if (!inputText.value.trim()) {
    alert("输入文本不能为空！");
    return;
  }

  loading.value = true;
  startSSE(apiAddUrl, { text: inputText.value.trim() });
}

function convertToKana() {
  if (!inputText.value.trim() && !selectedFile.value) {
    alert("请至少输入文本或上传支持的文件！");
    return;
  }

  result.value = "";
  let uploadUrl = "";
  let payload: FormData | Record<string, string> = {};

  if (selectedFile.value) {
    const fileExt = selectedFile.value.name.split(".").pop()?.toLowerCase() ?? "";
    const allowedImageExts = ["png", "jpg", "jpeg", "webp"];
    const allowedDocExts = ["pdf", "doc", "docx"];

    if (allowedImageExts.includes(fileExt)) {
      uploadUrl = apiImageUrl;
    } else if (allowedDocExts.includes(fileExt)) {
      uploadUrl = apiFileUrl;
    } else {
      alert("不支持此文件类型。");
      clearSelectedFile();
      return;
    }

    const formData = new FormData();
    formData.append("file", selectedFile.value);
    payload = formData;

    startSSE(uploadUrl, payload);
    return;
  }

  if (inputText.value.trim()) {
    uploadUrl = apiAddUrl;
    payload = { text: inputText.value.trim() };

    startSSE(uploadUrl, payload);
  }
}

function startSSE(url: string, body: FormData | Record<string, string>) {
  const isFormData = body instanceof FormData;

  const headers = isFormData
    ? undefined
    : { "Content-Type": "application/x-www-form-urlencoded" };

  const payload = isFormData
    ? body
    : new URLSearchParams(body).toString();

  console.log("发送的请求 URL:", url);
  console.log("发送的请求 Headers:", headers);
  console.log("发送的请求 Body:", payload);

  sseClient.value = new SSE(url, {
    headers,
    payload,
  });

  sseClient.value.addEventListener("message", (event) => {
    console.log("输出:", event.data);
    const data: ApiResponse = JSON.parse(event.data);

    if (data.type === "message") {
      result.value += data.content;
      loading.value = false;
    } else if (data.type === "error") {
      result.value = `错误信息：${data.content}`;
      loading.value = false;
    }
  });

  sseClient.value.addEventListener("error", (event) => {
    console.error("发生错误:", event);
    result.value = "连接失败。";
    loading.value = false;
    stopSSE();
  });

  sseClient.value.stream();
}

function stopSSE() {
  if (sseClient.value) {
    sseClient.value.close();
    sseClient.value = null;
  }
}

onUnmounted(() => stopSSE());
</script>

<style scoped>
  .maintitle {
  font-size: 40px;
  color: #004080;
  text-align: center;
  font-weight: 700;
  margin-bottom: 16px;
}

.subtitle {
  color: rgb(10, 148, 202);
  margin-top: 24px;
  margin-bottom: 12px;
}

.converter {
  max-width: 600px;
  margin: 40px auto;
  font-family: Arial, sans-serif;
}

textarea {
  width: 100%;
  font-size: 1rem;
  padding: 8px;
}

button {
  cursor: pointer;
  padding: 6px 12px;
  font-size: 1rem;
  background-color: skyblue;
  border: 5px deepskyblue;
  border-radius: 10px;
  color: blue;
}
button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

#inputFile {
  display: none;
}

.custom-file-btn {
  display: inline-block;
  padding: 1px 1px;
  background-color: skyblue;
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1rem;
}

.custom-file-btn:hover {
  background-color: #005f73;
}

.result {
  white-space: pre-wrap;
  padding: 12px;
  margin-top: 12px;
  background-color: #fafafa;
  border: 1px solid #ccc;
  min-height: 100px;
}

.preview {
  margin-top: 20px;
}

.preview img {
  max-width: 100%;
  height: auto;
  border: 1px solid #ddd;
}
</style>