<template>
  <div>
    <div class="upload">
      <div class="item">
        <input
          type="file"
          id="photoUpload"
          accept="image/*"
          @change="handleFileUpload"
        />
      </div>
      <p v-if="isRunning">圧縮中</p>
      <canvas
        ref="canvas"
        style="display: none;"
      ></canvas>
      <div class="item">
        <strong>toBlob type:</strong>
        <label
          v-for="item in blobTypeList"
          :key="item"
        >
          <input
            type="radio"
            v-model="toBlobType"
            :value="item"
          />{{ item }}
        </label>
      </div>
      <div class="item">
        <strong>toBlob quality:</strong> {{ toBlobQuality }}
      </div>
      <div class="item">
        <strong>blob size:</strong> <span v-if="blobSize">{{ formattedBlobSize }}</span>
      </div>
      <div class="item">
        <strong>Max Blob Size:</strong>
        <label
          v-for="size in 6"
          :key="size"
        >
          <input
            type="radio"
            v-model="maxBlobSizeMb"
            :value="size"
          /> {{ size }} MB
        </label>
      </div>
    </div>

    <div
      v-if="photoPreviewUrl"
      class="preview"
    >
      <div class="item">
        <strong>処理時間:</strong> {{ processingTime }} 秒
      </div>
      <img
        :src="photoPreviewUrl"
        class="preview__src"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';

type BlobType = 'webp' | 'jpeg';
const blobTypeList: BlobType[] = ['webp', 'jpeg'];
const initialQuality: number = 0.9;

const toBlobType = ref<BlobType>('webp');
const toBlobQuality = ref<number>(initialQuality); // 圧縮率
const maxBlobSizeMb = ref<number>(3); // 収めるデータサイズ
const blobSize = ref<number>(0);
const photoPreviewUrl = ref<string | null>(null);
const processingTime = ref<string>(''); // 経過時間
const trimImage = ref<boolean>(false); // 正方形にトリミングするか
const isRunning = ref<boolean>(false); // 処理中か

const canvas = ref<HTMLCanvasElement | null>(null);

const maxBlobSize = computed(() => maxBlobSizeMb.value * 1024 * 1024);
const formattedBlobSize = computed(() => getFormatSize(blobSize.value));

const getFormatSize = (size: number): string => {
  const units = ['B', 'KB', 'MB'];
  const index = Math.min(Math.floor(Math.log10(size) / 3), units.length - 1);
  const formattedSize = (size / Math.pow(1024, index)).toFixed(2);
  return `${formattedSize} ${units[index]}`;
};

const handleFileUpload = async (event: Event) => {
  console.log('handleFileUpload');
  if (isRunning.value) return;
  isRunning.value = true;

  const file = (event.target as HTMLInputElement).files?.[0];
  if (file) {
    const startTime = performance.now();

    const img = await loadImage(file);
    const ctx = canvas.value?.getContext('2d');
    if (!ctx || !canvas.value) return;

    drawImageOnCanvas(img, ctx);

    await compressImage();
    const endTime = performance.now();

    processingTime.value = ((endTime - startTime) / 1000).toFixed(2);
    isRunning.value = false;
  }
};

const loadImage = (file: File): Promise<HTMLImageElement> => {
  console.log('loadImage');
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => resolve(img);
      img.onerror = reject;
      img.src = reader.result as string;
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
};

const drawImageOnCanvas = (img: HTMLImageElement, ctx: CanvasRenderingContext2D) => {
  // trimImage.value が true の時は正方形にトリミング
  const size = trimImage.value
    ? { width: Math.min(img.width, img.height), height: Math.min(img.width, img.height) }
    : { width: img.width, height: img.height };

  // キャンバスをトリミング後のサイズに合わせる
  canvas.value!.width = size.width;
  canvas.value!.height = size.height;

  // 画像の中央部分をキャンバスに描画する
  const offsetX = (img.width - size.width) / 2;
  const offsetY = (img.height - size.height) / 2;

  ctx.drawImage(img, offsetX, offsetY, size.width, size.height, 0, 0, size.width, size.height);
};

const compressImage = async (): Promise<void> => {
  let blob: Blob | null;
  toBlobQuality.value = initialQuality;

  do {
    blob = await new Promise<Blob | null>((resolve) => {
      canvas.value?.toBlob(resolve, `image/${toBlobType.value}`, toBlobQuality.value);
    });
    if (blob) {
      blobSize.value = blob.size;
      // サイズが大きい場合、0.1ずつ品質を下げる
      toBlobQuality.value = ((toBlobQuality.value * 10) - 1) / 10;
    }
  } while (blob && blob.size > maxBlobSize.value && toBlobQuality.value > 0.01);

  if (!blob) return;
  photoPreviewUrl.value = URL.createObjectURL(blob!);
};
</script>

<style scoped lang="scss">
.item {
  margin-top: 0.5em;
  margin-bottom: 0.5em;
}

.preview {
  width: math.div(100vw, 3);
}

.preview__src {
  height: auto;
  width: 100%;
}
</style>