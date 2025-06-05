<template>
  <section class="Upload">
    <input type="file" multiple @change="fileListChange" :accept="UploadConfig.AcceptTypes" />
    <div class="placeholder">
      
 
    </div>
  </section>
</template>
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useToast } from '@/components/ui/toast/use-toast';
const { toast } = useToast();
// 参数
const props = defineProps(['modelValue', 'UploadConfig', 'uploadAPI']);
const emits = defineEmits(['update:modelValue']);
const UploadConfig = ref<any>(props.UploadConfig);
// 文件上传列表变化事件
const fileListChange = async (v: Event, type: boolean = false) => {
  let targetFileListArr: any = [];
  if (!type) {
    if (!v.target) return;
    targetFileListArr = (v.target as HTMLInputElement).files || [];
  } else {
    targetFileListArr = v;
  }
  // 处理图片格式
  targetFileListArr = await imgTypeFormat(targetFileListArr);
  const FileListArr: Array<any> = [...props.modelValue, ...Array.from(targetFileListArr || [])];
  // 过滤不符合Size的文件
  let fileListFilter = FileListArr.filter((i: any) => UploadConfig.value.MaxSize && (i.size <= UploadConfig.value.MaxSize * 1024 * 1024 || i.upload_status == 'success'));
  if (fileListFilter.length != FileListArr.length) toast({ title: 'Tips', description: `已过滤Size超过 ${UploadConfig.value.MaxSize}MB 的文件` });
  // 过滤超过数量的文件
  if (UploadConfig.value.Max && fileListFilter.length > UploadConfig.value.Max) {
    toast({ title: 'Tips', description: `已过滤超过最大上传 ${UploadConfig.value.Max}个 的文件` });
    fileListFilter = Array.from(targetFileListArr || []).slice(0, UploadConfig.value.Max);
  }
  emits('update:modelValue', fileListFilter);
  fileUpload(fileListFilter);
};

// 图片格式webp 转换为png
const imgTypeFormat = async (files: File[]) => {
  const _fileList = Array.from(files || []);
  const convertWebPToPNG = async (file: File): Promise<File> => {
    if (!file.type.startsWith('image/webp')) return file;
    return new Promise((resolve) => {
      const img = new Image();
      img.src = URL.createObjectURL(file);
      img.onload = () => {
        const canvas = document.createElement('canvas');
        canvas.width = img.width;
        canvas.height = img.height;
        const ctx = canvas.getContext('2d')!;
        ctx.drawImage(img, 0, 0);
        canvas.toBlob((blob) => {
          const newFile = new File([blob!], file.name.replace(/\.webp$/i, '.png'), { type: 'image/png' });
          URL.revokeObjectURL(img.src);
          resolve(newFile);
        }, 'image/png');
      };
      img.onerror = () => {
        URL.revokeObjectURL(img.src);
        resolve(file);
      };
    });
  };
  return await Promise.all(_fileList.map(convertWebPToPNG));
};

// 上传
const fileUpload = async (FileListArr: Array<any>) => {
  FileListArr.forEach(async (i: any) => {
    if (i.upload_status) return;
    const formData = new FormData();
    formData.append('file', i);
    // 做图片预览blob======
    if (i.type.startsWith('image/')) {
      const blob = URL.createObjectURL(i);
      i.upload_blob = blob;
    }
    // 做图片预览blob======
    // 同步上传状态======
    i.upload_status = 'uploading';
    i.upload_progress = 96;
    emits('update:modelValue', [...FileListArr]);
    // 同步上传状态======
    try {
      // 发送请求
      const res = await fetch(props.uploadAPI, {
        method: 'POST',
        body: formData,
      });
      const result = await res.json();
      i.upload_result = result;
      i.upload_result._vh_filename = i.name;
      i.upload_status = 'success';
    } catch (error) {
      i.upload_status = 'error';
      i.upload_result = error;
    } finally {
      // 同步上传状态======
      i.upload_progress = 100;
      emits('update:modelValue', [...FileListArr]);
      // 同步上传状态======
    }
  });
};

// 粘贴上传
const pasteUpload = (v: any) => {
  v.preventDefault();
  const pasteData = v.clipboardData || (window as any).clipboardData;
  const files = pasteData.files;
  fileListChange(files, true);
};

onMounted(() => {
  document.addEventListener('paste', pasteUpload);
});
</script>

<style scoped lang="less">
@import 'Upload.less';
</style>
