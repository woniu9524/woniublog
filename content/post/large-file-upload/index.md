---
title: 大文件上传(视频压缩上传)
description: 实现大文件分片上传、断点续传和视频压缩的完整方案
date: 2024-08-15
categories:
    - coding-life
tags:
    - java
    - upload
    - video
image: cover.png
---


> 切片上传+断点续传+并发上传+ffmpeg后端压缩

### 需求背景

    我需要把实验视频上传到服务器上然后进行分析。分析视频不需要高清的视频，所以需要对视频进行压缩。开始尝试在前端进行压缩视频，测试发现不是很靠谱，遂放弃，转用前端上传，到后端压缩。

    因为网站使用cloudflare进行反向代理，可能cf在国内网络比较差的原因，导致网络十分不稳定，大文件没有办法上传。

### 处理流程

前端先计算视频的md5值；

接着带着md5请求后端，是否有视频碎片(为了能断点续传)；

接着对视频切片，跳过已经上传了的碎片，此处因为网络原因，我做了网络错误重试；

我设置三个一组，并发上传，使用Promise.all来控制；

在切片都完成上传后，发送合并请求；

请求后因为后端要合并和压缩视频，时间比较长，很容易造成网络中断，所以轮询请求获取当前上传状态。

视频合并和压缩，完成后，删除本地文件，把状态写入redis等待轮询；

### 过程中遇到的一些问题

网络问题：视频上传的过程中会因为网络问题照成中断，我对上传进行了错误重传。压缩过程比较长也有同样的问题，我使用轮询的方式获取状态。

视频合并OOM：起初把视频读取到内存中进行合并，这样非常危险，很容易照成OOM。后面改成流式处理，设置固定大小的缓冲区。

### 代码

> 前端

```vue
<template>
  <div class="container">
    <div v-if="videoUrl" class="video-container">
      <video :src="videoUrl" class="video-player" controls></video>
    </div>
    <div class="upload-container">
      <el-upload
        :auto-upload="false"
        :before-upload="beforeUpload"
        :file-list="fileList"
        :limit="1"
        :on-change="handleChange"
        :on-exceed="handleExceed"
        :on-preview="handlePreview"
        :on-remove="handleRemove"
        accept="video/*"
        class="upload-demo"
        drag
      >
        <el-icon class="el-icon--upload">
          <UploadFilled />
        </el-icon>
        <div class="el-upload__text">拖拽视频到此处，或<em>点击上传</em></div>
      </el-upload>
    </div>
    <el-dialog v-model="dialogVisible" :close-on-click-modal="false" title="处理进度" width="30%">
      <el-progress :percentage="progressPercentage" :status="progressStatus"></el-progress>
      <div>{{ progressText }}</div>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue';
import { ElMessage } from 'element-plus';
import { UploadFilled } from '@element-plus/icons-vue';
import { getMergeChunksStatus, getUploadedChunks, getVideoUrl, mergeChunksForExperimentVideo, uploadChunk } from '@/api/lab/content/labContent';
import { useDrawingStore } from '@/store/modules/drawing';
import SparkMD5 from 'spark-md5';
import axios from 'axios';

// 常量定义
const CHUNK_SIZE = 5 * 1024 * 1024; // 每个分片5MB
const MAX_RETRIES = 3; // 最大重试次数
const RETRY_DELAY = 2000; // 重试延迟时间（毫秒）
const MAX_FILE_SIZE = 2 * 1024 * 1024 * 1024; // 最大文件大小（2GB）
const CONCURRENT_UPLOADS = 3; // 并发上传数量

// 状态管理
const drawingStore = useDrawingStore();

// 响应式变量
const fileList = ref([]);
const videoUrl = ref('');
const dialogVisible = ref(false);
const progressPercentage = ref(0);
const progressStatus = ref('');
const progressText = ref('');

// 更新进度信息
const updateProgress = (percentage: number, text: string, status: string = '') => {
  progressPercentage.value = Math.min(100, Math.max(0, Number(percentage.toFixed(2))));
  progressText.value = text;
  progressStatus.value = status;
};

// 获取文件分片
const getFileChunks = (fileSize: number) => {
  const chunks = [];
  let start = 0;
  while (start < fileSize) {
    const end = Math.min(start + CHUNK_SIZE, fileSize);
    chunks.push({ start, end });
    start = end;
  }
  return chunks;
};

// 计算文件MD5
const computeMD5 = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const spark = new SparkMD5.ArrayBuffer();
    const reader = new FileReader();
    const chunks = getFileChunks(file.size);
    let currentChunk = 0;

    reader.onload = (e: any) => {
      spark.append(e.target.result);
      currentChunk++;
      if (currentChunk < chunks.length) {
        loadNext();
      } else {
        resolve(spark.end());
      }
    };

    reader.onerror = (error) => {
      console.error(error);
      reject('MD5计算失败');
    };

    const loadNext = () => {
      const { start, end } = chunks[currentChunk];
      reader.readAsArrayBuffer(file.slice(start, end));
    };

    loadNext();
  });
};

// 清理上传状态
const clearUpload = () => {
  fileList.value = [];
  videoUrl.value = '';
};

// 延迟执行
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

// 带重试的操作执行
const retryOperation = async (operation: () => Promise<any>, retries = MAX_RETRIES) => {
  try {
    return await operation();
  } catch (error) {
    if (retries > 0 && axios.isAxiosError(error)) {
      console.log(`操作重试中，剩余尝试次数：${retries}`);
      await sleep(RETRY_DELAY);
      return retryOperation(operation, retries - 1);
    }
    throw error;
  }
};

// 轮询合并状态
const pollMergeStatus = async (md5: string) => {
  while (true) {
    try {
      const response = await getMergeChunksStatus(md5);
      const status = response.msg;

      if (status.includes('error')) {
        throw new Error('合并失败: ' + status);
      } else if (status.includes('finish')) {
        return response.msg;
      }

      // 等待一段时间后再次轮询
      await sleep(3000);
    } catch (error) {
      console.error('轮询合并状态时出错:', error);
      throw error;
    }
  }
};

// 文件上传超出限制处理
const handleExceed = (files: File[], fileList: File[]) => {
  ElMessage.warning('只能上传一个视频文件。');
};

// 文件预览处理
const handlePreview = (file: File) => {
  console.log('预览文件', file);
};

// 文件移除处理
const handleRemove = (file: File, fileList: File[]) => {
  console.log('移除文件', file, fileList);
};

// 上传前的文件检查
const beforeUpload = (file: File) => {
  const isVideo = file.type.startsWith('video/');
  const isLt2G = file.size <= MAX_FILE_SIZE;

  if (!isVideo) {
    ElMessage.error('只能上传视频文件！');
    return false;
  }
  if (!isLt2G) {
    ElMessage.error('视频大小不能超过 2GB!');
    return false;
  }
  return true;
};

// 文件状态改变处理
const handleChange = async (file: any, fileList: any) => {
  if (file.status === 'ready') {
    dialogVisible.value = true;
    updateProgress(0, '准备上传视频...');

    try {
      const md5 = await computeMD5(file.raw);
      const chunks = getFileChunks(file.raw.size);
      const uploadedChunksResponse = await retryOperation(() => getUploadedChunks(md5));

      if (!uploadedChunksResponse.data) {
        throw new Error('服务器响应无效');
      }

      const uploadedChunks = uploadedChunksResponse.data;

      if (typeof uploadedChunks === 'boolean' && uploadedChunks) {
        updateProgress(100, '文件已存在，跳过上传', 'success');
        ElMessage.success('文件已存在，上传成功');
        setTimeout(() => {
          dialogVisible.value = false;
          window.location.reload();
        }, 1500);
        return;
      }

      if (!Array.isArray(uploadedChunks)) {
        throw new Error('服务器返回的数据格式不正确');
      }

      const chunksToUpload = chunks.filter((chunk, index) => !uploadedChunks.includes(index));
      const totalChunks = chunksToUpload.length;
      let uploadedCount = 0;

      // 并发上传函数
      const uploadChunkConcurrently = async (chunk: { start: number; end: number }, index: number) => {
        const { start, end } = chunk;
        const formData = new FormData();
        const blob = file.raw.slice(start, end);

        if (blob.size === 0) {
          console.warn('遇到空分片，跳过...');
          return;
        }

        formData.append('file', blob, `${file.raw.name}.part${index}`);
        formData.append('md5', md5);
        formData.append('chunkIndex', index.toString());

        await retryOperation(() => uploadChunk(formData));
        uploadedCount++;
        updateProgress((uploadedCount / totalChunks) * 90, `正在上传 ${uploadedCount}/${totalChunks}`);
      };

      // 使用 Promise.all 和 Array.slice 来控制并发量
      for (let i = 0; i < chunksToUpload.length; i += CONCURRENT_UPLOADS) {
        const uploadPromises = chunksToUpload.slice(i, i + CONCURRENT_UPLOADS).map((chunk, index) =>
          uploadChunkConcurrently(chunk, i + index)
        );
        await Promise.all(uploadPromises);
      }

      updateProgress(95, '正在合并压缩视频...');
      try {
        await mergeChunksForExperimentVideo(md5, file.raw.name, drawingStore.currentExperimentId);
      } catch (e) {
        console.error(e);
      }

      const result = await pollMergeStatus(md5);

      if (result) {
        updateProgress(100, '上传成功', 'success');
        ElMessage.success('上传成功');
        drawingStore.getStepStatus(drawingStore.currentExperimentId);
        setTimeout(() => {
          dialogVisible.value = false;
          window.location.reload();
        }, 1500);
      } else {
        throw new Error('上传失败');
      }
    } catch (error) {
      console.error('上传错误', error);
      updateProgress(100, '上传失败', 'exception');
      if (axios.isAxiosError(error) && error.response?.status === 502) {
        ElMessage.error(`上传失败: 服务器暂时无法响应，请稍后重试`);
      } else {
        ElMessage.error(`上传失败: ${error.msg || '未知错误'}`);
      }
      clearUpload();
    }
  }
};

// 生命周期钩子
onMounted(() => {
  getVideoUrl(drawingStore.currentExperimentId).then((response) => {
    videoUrl.value = response.data?.url;
  });
});
</script>

<style scoped>
.container {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 20px;
  padding: 20px;
  background-color: #f0f2f5;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.video-container {
  flex: 1;
  max-width: 60%;
}

.video-player {
  width: 100%;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.upload-container {
  flex: 1;
  max-width: 35%;
}

.upload-demo {
  border: 1px dashed #d9d9d9;
  border-radius: 6px;
  background-color: #ffffff;
  text-align: center;
  cursor: pointer;
  overflow: hidden;
  position: relative;
  padding: 20px;
  transition: border-color 0.3s;
}

.upload-demo:hover {
  border-color: #409eff;
}

.el-upload__text {
  color: #606266;
  font-size: 14px;
  margin-top: 16px;
}

.el-upload__tip {
  color: #909399;
  font-size: 12px;
  margin-top: 6px;
}
</style>

```

> 后端

```java
public class UploadController {

    @Autowired
    private ISysOssService ossService;

    @Autowired
    private SysOssMapper ossMapper;

    // 使用系统临时目录作为基础路径
    private static final String BASE_DIR = System.getProperty("java.io.tmpdir");
    // 临时存储上传分片的目录
    private static final String CHUNK_FOLDER = "uploads" + File.separator + "chunks" + File.separator;
    // 合并后的视频临时存储目录
    private static final String MERGED_FOLDER = "uploads" + File.separator + "merged" + File.separator;

    /**
     * 获取已上传的分片信息
     */
    @GetMapping("/getUploadedChunks")
    public R<List<Integer>> getUploadedChunks(@RequestParam String md5) {
        Path chunkDir = Paths.get(BASE_DIR, CHUNK_FOLDER, md5);
        if (!Files.exists(chunkDir)) {
            return R.ok(new ArrayList<>());
        }

        List<Integer> uploadedChunks = new ArrayList<>();
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(chunkDir)) {
            for (Path path : stream) {
                uploadedChunks.add(Integer.parseInt(path.getFileName().toString()));
            }
        } catch (IOException e) {
            return R.fail("获取已上传分片失败: " + e.getMessage());
        }
        return R.ok(uploadedChunks);
    }

    /**
     * 上传单个分片
     */
    @Log(title = "上传视频分片", businessType = BusinessType.INSERT)
    @PostMapping("/uploadChunk")
    public R<Void> uploadChunk(@RequestParam("file") MultipartFile file,
                               @RequestParam("md5") String md5,
                               @RequestParam("chunkIndex") int chunkIndex) {
        if (file.isEmpty()) {
            return R.fail("上传文件不能为空");
        }

        Path chunkDir = Paths.get(BASE_DIR, CHUNK_FOLDER, md5);
        try {
            Files.createDirectories(chunkDir);
            Path chunkPath = chunkDir.resolve(String.valueOf(chunkIndex));
            Files.copy(file.getInputStream(), chunkPath, StandardCopyOption.REPLACE_EXISTING);
            return R.ok();
        } catch (IOException e) {
            return R.fail("分片上传失败: " + e.getMessage());
        }
    }

    /**
     * 合并所有分片
     */
    /**
     * 合并所有分片
     */
    @Log(title = "合并视频分片", businessType = BusinessType.INSERT)
    @PostMapping("/mergeChunks/experimentVideo")
    public R<SysOssUploadVo> mergeChunksExperimentVideo(@RequestParam("md5") String md5,
                                                        @RequestParam("filename") String filename,
                                                        @RequestParam("experimentId") Long experimentId) {
        // 获取分片目录路径
        Path chunkDir = Paths.get(BASE_DIR, CHUNK_FOLDER, md5);
        if (!Files.exists(chunkDir)) {
            return R.fail("没有找到上传的分片");
        }
        Path mergedFile = Paths.get(BASE_DIR, MERGED_FOLDER, md5, filename);
        Path compressedFile = Paths.get(BASE_DIR, MERGED_FOLDER, md5, "compressed_" + filename);

        try {
            Files.createDirectories(mergedFile.getParent());

            RedisUtils.setCacheObject(CacheConstants.VIDEO_COMPRESS_STATUS_KEY + md5, "start compress", Duration.ofMinutes(60));

            Integer testDuration = ossMapper.queryTestDuration(experimentId);
            FFmpegUtils.mergeAndCompressVideo(chunkDir, mergedFile, compressedFile, testDuration);

            // 创建 MultipartFile 对象
            MultipartFile multipartFile;
            try (InputStream inputStream = Files.newInputStream(compressedFile)) {
                multipartFile = new MockMultipartFile(filename, filename, Files.probeContentType(compressedFile), IOUtils.toByteArray(inputStream));
            }

            // 上传到 OSS
            SysOssVo oss = ossService.upload(multipartFile, experimentId);

            // 创建上传结果对象
            SysOssUploadVo uploadVo = new SysOssUploadVo();
            uploadVo.setUrl(oss.getUrl());
            uploadVo.setFileName(oss.getOriginalName());
            uploadVo.setOssId(oss.getOssId().toString());

            // 保存 ossId 到实验
            ossService.saveOssId(experimentId, oss.getOssId());

            // 清理临时文件
            deleteDirectory(chunkDir);
            deleteDirectory(mergedFile.getParent());
            // redis 设置状态: 完成上傳
            RedisUtils.setCacheObject(CacheConstants.VIDEO_COMPRESS_STATUS_KEY+md5,"finish upload", Duration.ofMinutes(60));

            return R.ok(uploadVo);
        } catch (IOException e) {
            // redis 设置状态: 壓縮失敗
            RedisUtils.setCacheObject(CacheConstants.VIDEO_COMPRESS_STATUS_KEY+md5,"compress video error", Duration.ofMinutes(60));
            // 合并失败时，清理临时文件和目录
            try {
                deleteDirectory(mergedFile.getParent());
                deleteDirectory(chunkDir);
            } catch (IOException cleanupException) {
                log.error("清理临时文件失败", cleanupException);
            }
            return R.fail("合并分片失败: " + e.getMessage());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    /*
    *  獲取當前視頻上傳狀態
    * */
    @GetMapping("/getMergeChunksStatus/{md5}")
    public R<String> getMergeChunksStatus(@PathVariable("md5") String md5) {
        String status = RedisUtils.getCacheObject(CacheConstants.VIDEO_COMPRESS_STATUS_KEY+md5);
        if (status == null) {
            return R.ok("no status");
        } else {
            return R.ok(status);
        }
    }
    

    /**
     * 递归删除目录
     */
    private void deleteDirectory(Path directory) throws IOException {
        Files.walkFileTree(directory, new SimpleFileVisitor<Path>() {
            @Override
            public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
                Files.delete(file);
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
                Files.delete(dir);
                return FileVisitResult.CONTINUE;
            }
        });
    }
}
```

> ffmpeg

```java
package org.dromara.common.core.utils.file;

import lombok.extern.slf4j.Slf4j;
import org.bytedeco.ffmpeg.global.avutil;
import org.springframework.stereotype.Component;
import org.bytedeco.javacv.*;
import org.bytedeco.ffmpeg.global.avcodec;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;


@Slf4j
@Component
public class FFmpegUtils {

    public static void mergeAndCompressVideo(Path chunkDir, Path mergedFile, Path compressedFile, Integer testDuration) throws IOException, FrameGrabber.Exception, FrameRecorder.Exception {
        log.info("开始合并和压缩视频: 输入目录 = {}, 合并文件 = {}, 压缩文件 = {}, 测试时长 = {} 秒", chunkDir, mergedFile, compressedFile, testDuration);

        // 合并分片
        mergeChunks(chunkDir, mergedFile);
        log.info("视频合并完成");

        // 压缩视频
        compressVideo(mergedFile.toString(), compressedFile.toString(), testDuration);

        log.info("视频合并和压缩成功完成");
    }

    private static void mergeChunks(Path chunkDir, Path mergedFile) throws IOException {
        List<Path> chunks;
        try (var pathStream = Files.list(chunkDir)) {
            chunks = pathStream
                .sorted((p1, p2) -> {
                    int i1 = Integer.parseInt(p1.getFileName().toString());
                    int i2 = Integer.parseInt(p2.getFileName().toString());
                    return Integer.compare(i1, i2);
                })
                .toList();
        }

        int bufferSize = 16 * 1024 * 1024; // 128MB 缓冲区

        try (OutputStream out = new BufferedOutputStream(Files.newOutputStream(mergedFile))) {
            byte[] buffer = new byte[bufferSize];
            int bytesRead;

            for (Path chunk : chunks) {
                try (InputStream in = new BufferedInputStream(Files.newInputStream(chunk))) {
                    while ((bytesRead = in.read(buffer)) != -1) {
                        out.write(buffer, 0, bytesRead);
                    }
                }
            }
        }
    }

    static {
        // 设置 FFmpeg 日志回调
        avutil.av_log_set_level(avutil.AV_LOG_ERROR);
        FFmpegLogCallback.set();
    }

    public static void compressVideo(String inputFilePath, String outputFilePath, Integer testDuration) throws IOException {
        log.info("开始压缩视频: 输入文件 = {}, 输出文件 = {}, 测试时长 = {} 秒", inputFilePath, outputFilePath, testDuration);

        try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(inputFilePath)) {
            grabber.start();
            log.info("输入视频信息: 宽度 = {}, 高度 = {}, 帧率 = {}, 像素格式 = {}",
                grabber.getImageWidth(), grabber.getImageHeight(), grabber.getFrameRate(), grabber.getPixelFormat());

            try (FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outputFilePath, grabber.getImageWidth(), grabber.getImageHeight())) {
                recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);
                recorder.setFormat("mp4");
                recorder.setFrameRate(30);
                recorder.setFrameRate(grabber.getFrameRate());
                recorder.setVideoBitrate(300000);
                recorder.setVideoOption("preset", "medium");  // 将预设改为 "medium" 以平衡压缩速度和质量
                recorder.setVideoOption("crf", "23");
                recorder.setVideoOption("threads", "auto");
                recorder.start();

                Frame frame;
                long startTime = System.currentTimeMillis();
                long duration = (testDuration != null && testDuration > 0) ? testDuration * 1000L : Long.MAX_VALUE;

                while ((frame = grabber.grab()) != null) {
                    if (System.currentTimeMillis() - startTime > duration) {
                        break;
                    }
                    recorder.record(frame);
                }
            }
        } catch (Exception e) {
            log.error("视频压缩过程中发生错误", e);
            throw e;
        }
        log.info("视频压缩完成: 输出文件 = {}", outputFilePath);
    }

}

```


