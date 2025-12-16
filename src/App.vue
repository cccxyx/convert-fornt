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
import { ref, computed, watch } from "vue";

interface ApiResponse {
  result: string;
}

const inputText = ref("");
const selectedFile = ref<File | null>(null);
const loading = ref(false);
const result = ref("");
const previewUrl = ref<string | null>(null);

const acceptedFileTypes = [
  ".png",
  ".jpg",
  ".jpeg",
  ".webp",
  ".pdf",
  ".doc",
  ".docx",
].join(",");

// 判断选中的文件是否是图片类型
const isImageFile = computed(() => {
  if (!selectedFile.value) return false;
  const imgExts = ["png", "jpg", "jpeg", "webp"];
  const ext = selectedFile.value.name.split(".").pop()?.toLowerCase() ?? "";
  return imgExts.includes(ext);
});

function onFileChange(event: Event) {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    const file = target.files[0];
    const allowedExts = ["png", "jpg", "jpeg", "webp", "pdf", "doc", "docx"];
    const fileExt = file.name.split(".").pop()?.toLowerCase() ?? "";

    if (!fileExt || !allowedExts.includes(fileExt)) {
      alert(
        "不支持此文件类型！只允许上传图片：png, jpg, jpeg, webp，和文档：pdf, doc, docx"
      );
      clearSelectedFile();
      return;
    }

    selectedFile.value = file;

    // 如果是图片，生成预览url
    if (["png", "jpg", "jpeg", "webp"].includes(fileExt)) {
      // 先释放旧的url，避免内存泄漏
      if (previewUrl.value) URL.revokeObjectURL(previewUrl.value);
      previewUrl.value = URL.createObjectURL(file);
    } else {
      // 非图片，清空预览
      if (previewUrl.value) URL.revokeObjectURL(previewUrl.value);
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

async function convertToKana() {
  result.value = "";

  if (!inputText.value && !selectedFile.value) {
    result.value = "请在输入框输入文本或上传支持的文件。";
    return;
  }

  loading.value = true;

  try {
    const formData = new FormData();
    if (selectedFile.value) {
      formData.append("file", selectedFile.value);
    }
    if (inputText.value.trim() !== "") {
      formData.append("text", inputText.value.trim());
    }

    // TODO: 替换成你后端的接口地址
    const response = await fetch("https://api.example.com/convert", {
      method: "POST",
      body: formData,
    });

    if (!response.ok) {
      throw new Error(`服务器错误，状态码：${response.status}`);
    }

    const data: ApiResponse = await response.json();

    if ("result" in data) {
      result.value = data.result;
    } else {
      result.value = "接口返回格式异常。";
    }
  } catch (error) {
    if (error instanceof Error) {
      result.value = `请求失败：${error.message}`;
    } else {
      result.value = "请求失败：未知错误";
    }
  } finally {
    loading.value = false;
  }
}
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