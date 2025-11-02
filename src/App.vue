

<template>
  <div class="container">
    <h1>图像合成工具</h1>
    <div class="position-section">
      <div class="position-group">
        <h5>下标为偶数底图坐标</h5>
        <div>正面：小logo</div>
        <span>左:</span><input type="number" v-model.number="postionObj.min.left" placeholder="左"></input>
          <span>上:</span><input type="number" v-model.number="postionObj.min.top" placeholder="上"></input>
          <span>宽:</span><input type="number" v-model.number="postionObj.min.width" placeholder="宽"></input>
          <span>高:</span><input type="number" v-model.number="postionObj.min.height" placeholder="高"></input><br />
          <div>正面：大logo</div>
          <span>左:</span><input type="number" v-model.number="postionObj.max.left" placeholder="左"></input>
          <span>上:</span><input type="number" v-model.number="postionObj.max.top" placeholder="上"></input>
          <span>宽:</span><input type="number" v-model.number="postionObj.max.width" placeholder="宽"></input>
          <span>高:</span><input type="number" v-model.number="postionObj.max.height" placeholder="高"></input>
        </div>
         <div class="position-group">
          <h5>下标为奇数底图坐标</h5>
          <div>背面logo</div>
          <span>左:</span><input type="number" v-model.number="postionObj.last.left" placeholder="左"></input>
          <span>上:</span><input type="number" v-model.number="postionObj.last.top" placeholder="上"></input>
          <span>宽:</span><input type="number" v-model.number="postionObj.last.width" placeholder="宽"></input>
          <span>高:</span><input type="number" v-model.number="postionObj.last.height" placeholder="高"></input>
      </div>
    </div>
    <br />
    <br />
    
    <div class="upload-section">
      <div class="upload-group">
        <label>上传底图:</label>
        <input
          id="baseImageUpload"
          type="file"
          multiple
          accept="image/*"
          @change="handleBaseImageUpload"
        />
        <button @click="clearBaseImages" v-if="baseImages.length > 0">清空底图</button>
      </div>
      
      <div class="upload-group">
        <label>上传Logo:</label>
        <input
          id="logoUpload"
          type="file"
          multiple
          accept="image/*"
          @change="handleLogoUpload"
        />
        <button @click="clearLogos" v-if="logoFiles.length > 0">清空Logo</button>
      </div>
    </div>
    
    <div class="info-section" v-if="baseImages.length > 0 || logoFiles.length > 0">
      <p>已上传 {{ baseImages.length }} 个底图文件</p>
      <p>已上传 {{ logoFiles.length }} 个Logo文件</p>
      <p>已生成 {{ generatedImages.length }} 个合成图像</p>
    </div>
    
    <div class="loading" v-if="isProcessing">
      <p>正在生成图像，请稍候...</p>
      <div class="progress-bar">
        <div class="progress" :style="{ width: progress + '%' }"></div>
      </div>
      <p>{{ progress }}% ({{ completedTasks }}/{{ totalTasks }})</p>
    </div>
    
    <div class="error-message" v-if="errorMessage">
      <p>{{ errorMessage }}</p>
    </div>
    
    <div class="result-section" v-if="generatedImages.length > 0">
      <button class="batch-download" @click="downloadAll">批量下载(压缩包)</button>
      
      <div class="image-grid">
        <div class="image-item" v-for="(image, index) in generatedImages" :key="index">
          <img :src="image.dataUrl" :alt="image.name" class="preview-image" />
          <div class="image-actions">
            <span class="image-name">{{ image.name }}</span>
            <button @click="downloadImage(image)">下载</button>
          </div>
        </div>
      </div>
    </div>
    
    <div class="empty-state" v-if="baseImages.length === 0 && logoFiles.length === 0 && !isProcessing">
      <p>请上传底图和Logo文件来生成合成图像</p>
    </div>
  </div>
</template>
<script setup>
import { ref, onMounted, watch } from 'vue';
import JSZip from 'jszip'
import { saveAs } from 'file-saver'


// 默认位置信息
const defaultPositionObj = {
  min: {
    left: 270,
    top: 481,
    width: 119,
    height: 113
  },
  max: {
    left: 611,
    top: 264,
    width: 229,
    height: 218  },
  last: {
    left: 278,
    top:321,
    width: 242,
    height: 235
  }
};

// 位置信息对象 - 从localStorage读取或使用默认值
const postionObj = ref(defaultPositionObj);

// 状态管理
const logoFiles = ref([]);
const baseImages = ref([]); // 修改为用户上传的底图
const generatedImages = ref([]);
const isProcessing = ref(false);
const progress = ref(0);
const totalTasks = ref(0);
const completedTasks = ref(0);
const errorMessage = ref('');

// 处理底图上传
const handleBaseImageUpload = (event) => {
  const files = Array.from(event.target.files);
  if (files.length) {
    // 将文件对象转换为可用于图片加载的URL
    const fileUrls = files.map(file => URL.createObjectURL(file));
    baseImages.value = [...baseImages.value, ...fileUrls];
  }
};

// 处理Logo上传
const handleLogoUpload = (event) => {
  const files = Array.from(event.target.files);
  if (files.length) {
    logoFiles.value = [...logoFiles.value, ...files];
    // 只有当底图和Logo都上传后才生成图片
    if (baseImages.value.length > 0) {
      generateCombinations();
    }
  }
};

// 生成组合图片 - 根据底图下标奇偶性选择不同的logo位置
const generateCombinations = async () => {
  isProcessing.value = true;
  generatedImages.value = [];
  progress.value = 0;
  completedTasks.value = 0;
  errorMessage.value = '';
  
  // 检查是否有底图和Logo
  if (baseImages.value.length === 0) {
    errorMessage.value = '请先上传底图文件';
    isProcessing.value = false;
    return;
  }
  
  if (logoFiles.value.length === 0) {
    errorMessage.value = '请先上传Logo文件';
    isProcessing.value = false;
    return;
  }
  
  // 计算总任务数 - 底图数量 * logo文件数量
  totalTasks.value = baseImages.value.length * logoFiles.value.length;
  
  try {
    // 预加载底图，验证路径是否正确
    const preloadPromises = baseImages.value.map(url => {
      return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => resolve(url);
        img.onerror = () => reject(new Error(`底图加载失败: ${url}`));
        img.src = url;
      });
    });
    
    await Promise.all(preloadPromises);
    
    // 创建所有任务 - 每个logo在每个底图上生成一张图片
    const tasks = [];
    baseImages.value.forEach((baseImageUrl, baseIndex) => {
      logoFiles.value.forEach(logoFile => {
        // 根据底图下标奇偶性选择不同的位置
        const positionsToUse = baseIndex % 2 === 0 
          ? { min: postionObj.value.min, max: postionObj.value.max } // 偶数下标：使用min和max
          : { last: postionObj.value.last }; // 奇数下标：使用last
          
        // 传递选定的位置和底图下标信息
        tasks.push(overlayLogoOnBase(baseImageUrl, logoFile, positionsToUse, baseIndex));
      });
    });
    
    // 限制并发数，避免浏览器资源耗尽
    const concurrencyLimit = 2; // 降低并发数以减少资源占用
    const chunks = [];
    for (let i = 0; i < tasks.length; i += concurrencyLimit) {
      chunks.push(tasks.slice(i, i + concurrencyLimit));
    }
    
    // 分批次执行，每批次之间添加短暂延迟以释放浏览器资源
    for (const chunk of chunks) {
      try {
        const results = await Promise.allSettled(chunk);
        results.forEach(result => {
          if (result.status === 'fulfilled') {
            generatedImages.value.push(result.value);
          } else {
            console.error('生成图片失败:', result.reason);
            errorMessage.value = '部分图片生成失败，请查看控制台';
          }
        });
        
        // 添加短暂延迟，让浏览器有时间处理UI更新和释放资源
        if (chunks.indexOf(chunk) < chunks.length - 1) {
          await new Promise(resolve => setTimeout(resolve, 200));
        }
      } catch (batchError) {
        console.error('批次处理失败:', batchError);
        errorMessage.value = `批次处理失败: ${batchError.message}`;
        // 继续处理下一批，而不是完全中断
      }
    }
    
    if (generatedImages.value.length === 0) {
      errorMessage.value = '没有成功生成任何图片，请检查底图和Logo文件';
    }
  } catch (error) {
    console.error('生成图片时出错:', error);
    errorMessage.value = error.message || '生成过程中发生错误';
  } finally {
    isProcessing.value = false;
    progress.value = 100;
  }
};

// 核心功能：根据坐标和宽高，把logo图贴在底图上 - 根据底图下标选择不同位置，使用真实底图尺寸
const overlayLogoOnBase = async (baseImageUrl, logoFile, positionsToUse, baseIndex) => {
  return new Promise((resolve, reject) => {
    try {
      // 添加超时处理
      const timeoutId = setTimeout(() => {
        reject(new Error('图片生成超时'));
      }, 15000); // 减少超时时间至15秒

      // 加载底图
      const baseImage = new Image();
      baseImage.crossOrigin = 'anonymous';
      
      baseImage.onload = () => {
        try {
          // 使用底图的真实宽高创建Canvas
          const canvas = document.createElement('canvas');
          canvas.width = baseImage.naturalWidth;
          canvas.height = baseImage.naturalHeight;
          const ctx = canvas.getContext('2d');
          
          // 绘制底图，保持原始尺寸
          ctx.drawImage(baseImage, 0, 0, baseImage.naturalWidth, baseImage.naturalHeight);
          
          // 加载logo
          const logoImage = new Image();
          logoImage.crossOrigin = 'anonymous';
          
          const logoReader = new FileReader();
          logoReader.onload = (e) => {
            logoImage.src = e.target.result;
            logoImage.onload = () => {
              try {
                // 根据底图下标选择的位置绘制logo
                for (const [positionKey, position] of Object.entries(positionsToUse)) {
                  ctx.drawImage(
                    logoImage, 
                    position.left, 
                    position.top, 
                    position.width, 
                    position.height
                  );
                }
                
                // 生成base64图片
                const dataUrl = canvas.toDataURL('image/jpeg', 0.8);
                
                // 清理资源
                clearTimeout(timeoutId);
                
                // 更新进度
                completedTasks.value++;
                progress.value = Math.floor((completedTasks.value / totalTasks.value) * 100);
                
                // 生成文件名，包含底图下标和使用的位置信息
                const baseName = logoFile.name.split('.')[0];
                const positionKeys = Object.keys(positionsToUse).join('_');
                resolve({
                  dataUrl,
                  name: `${baseName}_base${baseIndex}_${positionKeys}.jpg`
                });
              } catch (error) {
                clearTimeout(timeoutId);
                reject(new Error(`Logo处理失败: ${error.message}`));
              }
            };
            
            logoImage.onerror = () => {
              clearTimeout(timeoutId);
              reject(new Error('Logo加载失败'));
            };
          };
          
          logoReader.onerror = () => {
            clearTimeout(timeoutId);
            reject(new Error('Logo文件读取失败'));
          };
          
          logoReader.readAsDataURL(logoFile);
        } catch (error) {
          clearTimeout(timeoutId);
          reject(new Error(`底图处理失败: ${error.message}`));
        }
      };
      
      baseImage.onerror = () => {
        clearTimeout(timeoutId);
        reject(new Error(`底图加载失败: ${baseImageUrl}`));
      };
      
      baseImage.src = baseImageUrl;
    } catch (error) {
      reject(new Error(`初始化图片处理失败: ${error.message}`));
    }
  });
};

// 下载单个图片 - 使用saveAs优化
const downloadImage = async (image) => {
  try {
    // 将base64转换为Blob
    const byteString = atob(image.dataUrl.split(',')[1]);
    const mimeString = image.dataUrl.split(',')[0].split(':')[1].split(';')[0];
    const ab = new ArrayBuffer(byteString.length);
    const ia = new Uint8Array(ab);
    for (let i = 0; i < byteString.length; i++) {
      ia[i] = byteString.charCodeAt(i);
    }
    const blob = new Blob([ab], { type: mimeString });
    
    // 使用saveAs进行下载
    saveAs(blob, image.name);
  } catch (error) {
    console.error('下载图片失败:', error);
    errorMessage.value = `下载图片失败: ${error.message}`;
  }
};

// 批量下载所有图片 - 使用JSZip压缩下载
const downloadAll = async () => {
  try {
    isProcessing.value = true;
    errorMessage.value = '';
    
    const zip = new JSZip();
    
    // 创建图片目录
    const imagesFolder = zip.folder('合成图片');
    
    // 处理所有图片，添加到压缩包
    const processPromises = generatedImages.value.map(async (image, index) => {
      // 更新进度
      progress.value = Math.floor((index + 1) / generatedImages.value.length * 100);
      
      // 将base64转换为Blob
      const byteString = atob(image.dataUrl.split(',')[1]);
      const mimeString = image.dataUrl.split(',')[0].split(':')[1].split(';')[0];
      const ab = new ArrayBuffer(byteString.length);
      const ia = new Uint8Array(ab);
      for (let i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }
      const blob = new Blob([ab], { type: mimeString });
      
      // 添加到压缩包
      imagesFolder.file(image.name, blob);
    });
    
    // 等待所有图片处理完成
    await Promise.all(processPromises);
    
    // 生成压缩包并下载
    const content = await zip.generateAsync({ type: 'blob' });
    saveAs(content, `合成图片_${new Date().toISOString().slice(0, 10)}.zip`);
    
    errorMessage.value = '批量下载完成！';
  } catch (error) {
    console.error('批量下载失败:', error);
    errorMessage.value = `批量下载失败: ${error.message}`;
  } finally {
    isProcessing.value = false;
    progress.value = 0;
  }
};

// 清空logo
const clearLogos = () => {
  logoFiles.value = [];
  generatedImages.value = [];
  document.getElementById('logoUpload').value = '';
};

// 清空底图
const clearBaseImages = () => {
  // 释放之前创建的Object URL
  baseImages.value.forEach(url => URL.revokeObjectURL(url));
  baseImages.value = [];
  generatedImages.value = [];
  document.getElementById('baseImageUpload').value = '';
};

  // 监听位置信息变化，自动保存到localStorage
  watch(
    () => postionObj.value,
    (newPosition) => {
      try {
        localStorage.setItem('podPositionObj', JSON.stringify(newPosition));
        console.log('坐标信息已保存到本地');
      } catch (error) {
        console.error('保存坐标信息失败:', error);
      }
    },
    { deep: true }
  );
  
  // 初始化应用 - 从localStorage加载坐标信息
  onMounted(() => {
    try {
      const savedPosition = localStorage.getItem('podPositionObj');
      if (savedPosition) {
        const parsedPosition = JSON.parse(savedPosition);
        // 验证解析后的数据结构是否完整
        if (parsedPosition.min && parsedPosition.max && parsedPosition.last) {
          postionObj.value = parsedPosition;
          console.log('已从本地缓存加载坐标信息');
        } else {
          console.warn('本地缓存的坐标信息结构不完整，使用默认值');
        }
      }
    } catch (error) {
      console.error('加载本地坐标信息失败:', error);
    }
    console.log('应用已加载，请上传底图和Logo文件');
  });
</script>
<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
}

.upload-section {
  margin-bottom: 20px;
  text-align: center;
}

.upload-group {
  margin-bottom: 15px;
}

.upload-group label {
  margin-right: 10px;
  font-weight: bold;
}

input[type="file"] {
  margin-right: 10px;
}

button {
  padding: 8px 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #45a049;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.info-section {
  background-color: #f0f0f0;
  padding: 15px;
  border-radius: 4px;
  margin-bottom: 20px;
}

.loading {
  text-align: center;
  padding: 20px;
  color: #666;
}

.progress-bar {
  width: 100%;
  height: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  overflow: hidden;
  margin: 10px 0;
}

.progress {
  height: 100%;
  background-color: #4CAF50;
  transition: width 0.3s ease;
}

.error-message {
  background-color: #ffebee;
  color: #c62828;
  padding: 15px;
  border-radius: 4px;
  margin: 15px 0;
  text-align: center;
}

.result-section {
  margin-top: 30px;
}

.batch-download {
  background-color: #4CAF50;
  padding: 10px 20px;
  font-size: 14px;
  margin-bottom: 20px;
  display: block;
  transition: background-color 0.3s;
}

.batch-download:hover {
  background-color: #45a049;
}

.batch-download:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.image-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.image-item {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  background-color: #f9f9f9;
}

.preview-image {
  max-width: 100%;
  height: auto;
  display: block;
  margin-bottom: 10px;
}

.image-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.image-name {
  font-size: 14px;
  color: #666;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
  margin-right: 10px;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: #999;
}

.position-section{
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  margin-bottom: 20px;
}

.position-group {
  background-color: #f5f5f5;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 15px;
  flex: 1;
  min-width: 300px;
  margin-right: 15px;
}

.position-group:last-child {
  margin-right: 0;
}

.position-group h5 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #333;
}

.position-group input {
  width: 70px;
  margin-right: 10px;
  margin-bottom: 10px;
  padding: 5px;
  border: 1px solid #ddd;
  border-radius: 4px;
}
</style>
