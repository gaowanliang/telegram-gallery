<script setup>
import { computed, ref, onMounted, onUnmounted, watch } from 'vue';
import PhotoSwipeLightbox from 'photoswipe/lightbox';
import 'photoswipe/style.css';

const props = defineProps({
  entries: Array,
  currentIndex: Number,
  onClose: Function,
  onPrev: Function,
  onNext: Function,
  onRefresh: Function
});

const copySuccess = ref('');
let lightbox = null;

const currentEntry = computed(() => props.entries[props.currentIndex]);

const metaLines = computed(() => {
  const m = currentEntry.value?.metadata || {};
  const lines = [];
  if (m.model) lines.push({ label: '模型', value: m.model });
  if (m.seed) lines.push({ label: '种子', value: m.seed });
  if (m.steps) lines.push({ label: '步数', value: m.steps });
  if (m.cfg_scale) lines.push({ label: 'CFG Scale', value: m.cfg_scale });
  if (m.sampler_name) lines.push({ label: '采样器', value: m.sampler_name });
  if (m.width && m.height) lines.push({ label: '尺寸', value: `${m.width} × ${m.height}` });
  return lines;
});

const negativePrompt = computed(() => {
  return currentEntry.value?.metadata?.negative_prompt || null;
});

async function copyToClipboard(text, label) {
  try {
    await navigator.clipboard.writeText(text);
    copySuccess.value = label;
    setTimeout(() => {
      copySuccess.value = '';
    }, 2000);
  } catch (err) {
    console.error('Failed to copy:', err);
  }
}

function copyAllInfo() {
  const allInfo = [];
  allInfo.push(`正向提示词: ${currentEntry.value.prompt}`);
  if (negativePrompt.value) {
    allInfo.push(`负向提示词: ${negativePrompt.value}`);
  }
  metaLines.value.forEach(item => {
    allInfo.push(`${item.label}: ${item.value}`);
  });
  copyToClipboard(allInfo.join('\n'), '全部信息');
}

function openPhotoSwipe() {
  if (lightbox) {
    lightbox.destroy();
    lightbox = null;
  }

  lightbox = new PhotoSwipeLightbox({
    dataSource: props.entries.map((entry, idx) => ({
      src: entry.src,
      width: entry.metadata?.width || 1200,
      height: entry.metadata?.height || 1600,
      alt: entry.prompt
    })),
    pswpModule: () => import('photoswipe'),
    index: props.currentIndex,
    bgOpacity: 0.95,
    spacing: 0.1,
    showHideAnimationType: 'fade',
  });

  lightbox.on('change', () => {
    const newIndex = lightbox.pswp.currIndex;
    if (newIndex !== props.currentIndex) {
      if (newIndex > props.currentIndex) {
        props.onNext();
      } else {
        props.onPrev();
      }
    }
  });

  lightbox.on('close', () => {
    if (lightbox) {
      lightbox.destroy();
      lightbox = null;
    }
  });

  lightbox.init();
  lightbox.loadAndOpen(props.currentIndex);
}

function handleKeydown(e) {
  if (lightbox && lightbox.pswp) return;

  if (e.key === 'ArrowLeft' && props.onPrev) {
    props.onPrev();
  } else if (e.key === 'ArrowRight' && props.onNext) {
    props.onNext();
  } else if (e.key === 'Escape' && props.onClose) {
    props.onClose();
  }
}

onMounted(() => {
  document.body.style.overflow = 'hidden';
  document.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  if (lightbox) {
    lightbox.destroy();
    lightbox = null;
  }
  document.body.style.overflow = '';
  document.removeEventListener('keydown', handleKeydown);
});
</script>

<template>
  <!-- 全屏详情页面 -->
  <div class="detail-overlay">
    <!-- 复制成功提示 -->
    <transition name="bounce">
      <div v-if="copySuccess" class="toast">
        <span class="toast-icon">✓</span>
        <span>已复制{{ copySuccess }}</span>
      </div>
    </transition>

    <!-- 关闭按钮 -->
    <button @click="props.onClose" class="close-btn" title="关闭 (ESC)">
      ✕
    </button>

    <!-- 图片计数器 -->
    <div class="image-counter">
      {{ props.currentIndex + 1 }} / {{ props.entries.length }}
    </div>

    <!-- 主内容区域 -->
    <div class="detail-container">
      <!-- 导航按钮 - 移动端 -->
      <div class="nav-mobile">
        <button
          v-if="props.onPrev && props.currentIndex > 0"
          @click="props.onPrev"
          class="nav-btn nav-btn-prev"
          title="上一张 (←)"
        >
          ‹
        </button>
        <button
          v-if="props.onNext && props.currentIndex < props.entries.length - 1"
          @click="props.onNext"
          class="nav-btn nav-btn-next"
          title="下一张 (→)"
        >
          ›
        </button>
      </div>

      <!-- 左侧：图片区域 -->
      <div class="image-section">
        <!-- 导航按钮 - 桌面端 -->
        <button
          v-if="props.onPrev && props.currentIndex > 0"
          @click="props.onPrev"
          class="nav-btn nav-btn-prev nav-desktop"
          title="上一张 (←)"
        >
          ‹
        </button>

        <div class="image-wrapper" @click="openPhotoSwipe" title="点击查看大图">
          <img :src="currentEntry.src" :alt="currentEntry.prompt" class="detail-image" />
          <div class="zoom-hint">
            <span class="zoom-icon">🔍</span>
            <span>点击查看大图</span>
          </div>
        </div>

        <button
          v-if="props.onNext && props.currentIndex < props.entries.length - 1"
          @click="props.onNext"
          class="nav-btn nav-btn-next nav-desktop"
          title="下一张 (→)"
        >
          ›
        </button>
      </div>

      <!-- 右侧：信息面板 -->
      <div class="info-panel">
        <div class="info-scroll">
          <div class="info-header-title">
            <span class="info-icon">ℹ️</span>
            <h3>图片信息</h3>
          </div>

          <!-- 正向提示词 -->
          <div class="info-section prompt-section">
            <div class="section-header">
              <span class="header-icon">💬</span>
              <span>提示词</span>
              <button @click="copyToClipboard(currentEntry.prompt, '提示词')" class="copy-btn">
                📋 复制
              </button>
            </div>
            <div class="section-body">{{ currentEntry.prompt }}</div>
          </div>

          <!-- 负向提示词 -->
          <div class="info-section negative-section" v-if="negativePrompt">
            <div class="section-header">
              <span class="header-icon">🚫</span>
              <span>反向提示词</span>
              <button @click="copyToClipboard(negativePrompt, '反向提示词')" class="copy-btn">
                📋 复制
              </button>
            </div>
            <div class="section-body text-muted">{{ negativePrompt }}</div>
          </div>

          <!-- 生成参数 -->
          <div class="info-section params-section" v-if="metaLines.length">
            <div class="section-header">
              <span class="header-icon">⚙️</span>
              <span>参数</span>
            </div>
            <div class="params-grid">
              <div class="param-item" v-for="(item, i) in metaLines" :key="i">
                <small>{{ item.label }}</small>
                <strong>{{ item.value }}</strong>
              </div>
            </div>
          </div>

          <!-- 时间戳 -->
          <div class="timestamp" v-if="currentEntry.timestamp">
            <span class="time-icon">🕒</span>
            {{ new Date(currentEntry.timestamp).toLocaleString('zh-CN') }}
          </div>

          <!-- 操作按钮 -->
          <div class="action-buttons">
            <button @click="copyAllInfo" class="action-btn action-btn-primary">
              <span class="btn-icon">📋</span>
              <span>复制全部</span>
            </button>
            <button v-if="props.onRefresh" @click="props.onRefresh" class="action-btn action-btn-secondary">
              <span class="btn-icon">🔄</span>
              <span>刷新图片</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.detail-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: var(--bg-primary);
  z-index: 1000;
  display: flex;
  flex-direction: column;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: scale(0.98);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.close-btn {
  position: fixed;
  top: 1rem;
  right: 1rem;
  width: 3rem;
  height: 3rem;
  border-radius: 50%;
  border: 2px solid var(--border-color);
  background: linear-gradient(135deg, var(--bg-secondary) 0%, var(--bg-tertiary) 100%);
  color: var(--text-primary);
  font-size: 1.5rem;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1001;
  line-height: 1;
  padding: 0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.close-btn:hover {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
  color: white;
  border-color: var(--primary);
  transform: rotate(90deg) scale(1.1);
  box-shadow: 0 6px 20px rgba(var(--primary-rgb, 99, 102, 241), 0.4);
}

.image-counter {
  position: fixed;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
  color: white;
  padding: 0.5rem 1.25rem;
  border-radius: 2rem;
  font-weight: 700;
  font-size: 0.875rem;
  box-shadow: 0 4px 12px rgba(var(--primary-rgb, 99, 102, 241), 0.3);
  z-index: 1001;
  animation: slideDown 0.4s ease;
  letter-spacing: 0.05em;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.toast {
  position: fixed;
  top: 5rem;
  left: 50%;
  transform: translateX(-50%);
  background: linear-gradient(135deg, #10b981 0%, #059669 100%);
  color: white;
  padding: 0.875rem 1.5rem;
  border-radius: 2rem;
  font-weight: 600;
  box-shadow: 0 8px 24px rgba(16, 185, 129, 0.4);
  z-index: 1002;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.toast-icon {
  font-size: 1.25rem;
  animation: checkmark 0.6s ease;
}

@keyframes checkmark {
  0% { transform: scale(0) rotate(0deg); }
  50% { transform: scale(1.2) rotate(180deg); }
  100% { transform: scale(1) rotate(360deg); }
}

.bounce-enter-active {
  animation: bounceIn 0.5s cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

.bounce-leave-active {
  animation: bounceOut 0.3s ease;
}

@keyframes bounceIn {
  0% {
    opacity: 0;
    transform: translateX(-50%) scale(0.3);
  }
  50% {
    transform: translateX(-50%) scale(1.05);
  }
  100% {
    opacity: 1;
    transform: translateX(-50%) scale(1);
  }
}

@keyframes bounceOut {
  to {
    opacity: 0;
    transform: translateX(-50%) scale(0.8);
  }
}

.detail-container {
  flex: 1;
  display: flex;
  padding: 4.5rem 1rem 1rem;
  gap: 1rem;
  min-height: 0;
  overflow: hidden;
}

.image-section {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  min-width: 0;
  animation: slideInLeft 0.5s ease;
}

@keyframes slideInLeft {
  from {
    opacity: 0;
    transform: translateX(-30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.image-wrapper {
  position: relative;
  max-width: 100%;
  max-height: 100%;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  border-radius: 0.75rem;
  overflow: hidden;
}

.image-wrapper:hover {
  transform: scale(1.02);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
}

.image-wrapper:hover .zoom-hint {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
}

.image-wrapper:active {
  transform: scale(0.98);
}

.detail-image {
  max-width: 100%;
  max-height: calc(100vh - 6rem);
  width: auto;
  height: auto;
  object-fit: contain;
  display: block;
}

.zoom-hint {
  position: absolute;
  bottom: 1.5rem;
  left: 50%;
  transform: translateX(-50%) translateY(10px);
  background: linear-gradient(135deg, rgba(0, 0, 0, 0.9) 0%, rgba(0, 0, 0, 0.8) 100%);
  backdrop-filter: blur(10px);
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 2rem;
  font-size: 0.875rem;
  font-weight: 600;
  opacity: 0;
  transition: all 0.3s ease;
  pointer-events: none;
  white-space: nowrap;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
}

.zoom-icon {
  font-size: 1.125rem;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.nav-btn {
  width: 3.5rem;
  height: 3.5rem;
  border-radius: 50%;
  border: 2px solid var(--border-color);
  background: linear-gradient(135deg, var(--bg-secondary) 0%, var(--bg-tertiary) 100%);
  color: var(--text-primary);
  font-size: 2rem;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  line-height: 1;
  padding: 0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  flex-shrink: 0;
}

.nav-btn:hover {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
  color: white;
  border-color: var(--primary);
  transform: scale(1.15);
  box-shadow: 0 6px 20px rgba(var(--primary-rgb, 99, 102, 241), 0.4);
}

.nav-btn:active {
  transform: scale(1.05);
}

.nav-desktop {
  display: flex;
}

.nav-mobile {
  display: none;
}

.info-panel {
  width: 400px;
  background: var(--bg-secondary);
  border-radius: 1rem;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  animation: slideInRight 0.5s ease;
  border: 1px solid var(--border-color);
  flex-shrink: 0;
}

@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.info-scroll {
  overflow-y: auto;
  height: 100%;
  padding: 1.5rem;
}

.info-header-title {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 2px solid var(--border-color);
}

.info-icon {
  font-size: 1.5rem;
  animation: rotate360 3s linear infinite;
}

@keyframes rotate360 {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.info-header-title h3 {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 700;
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.info-section {
  margin-bottom: 1rem;
  padding: 1.25rem;
  background: var(--bg-tertiary);
  border-radius: 0.75rem;
  border: 1px solid var(--border-color);
  transition: all 0.3s ease;
  animation: fadeInUp 0.5s ease backwards;
}

.info-section:nth-child(2) { animation-delay: 0.1s; }
.info-section:nth-child(3) { animation-delay: 0.2s; }
.info-section:nth-child(4) { animation-delay: 0.3s; }

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.info-section:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  border-color: var(--primary);
}

.prompt-section {
  border-left: 4px solid #3b82f6;
}

.negative-section {
  border-left: 4px solid #ef4444;
}

.params-section {
  border-left: 4px solid #8b5cf6;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
  font-weight: 700;
  font-size: 0.813rem;
  text-transform: uppercase;
  color: var(--text-secondary);
  letter-spacing: 0.05em;
}

.header-icon {
  font-size: 1.125rem;
}

.copy-btn {
  margin-left: auto;
  padding: 0.375rem 0.875rem;
  font-size: 0.75rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  cursor: pointer;
  transition: all 0.2s ease;
  color: var(--text-primary);
  text-transform: none;
  letter-spacing: normal;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.copy-btn:hover {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
  color: white;
  border-color: var(--primary);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(var(--primary-rgb, 99, 102, 241), 0.3);
}

.copy-btn:active {
  transform: translateY(0);
}

.section-body {
  color: var(--text-primary);
  line-height: 1.7;
  font-size: 0.875rem;
  word-wrap: break-word;
}

.text-muted {
  color: var(--text-muted);
}

.params-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0.625rem;
}

.param-item {
  padding: 0.75rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
  transition: all 0.2s ease;
}

.param-item:hover {
  transform: translateY(-2px);
  border-color: var(--primary);
  box-shadow: 0 4px 12px rgba(var(--primary-rgb, 99, 102, 241), 0.15);
}

.param-item small {
  font-size: 0.7rem;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  font-weight: 600;
}

.param-item strong {
  font-size: 0.938rem;
  color: var(--text-primary);
  word-break: break-word;
  font-weight: 700;
}

.timestamp {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.875rem;
  background: linear-gradient(135deg, var(--bg-tertiary) 0%, var(--bg-secondary) 100%);
  border: 1px solid var(--border-color);
  border-radius: 0.75rem;
  color: var(--text-muted);
  font-size: 0.813rem;
  margin-bottom: 1rem;
  font-weight: 600;
}

.time-icon {
  font-size: 1rem;
  animation: spin 4s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.action-buttons {
  display: flex;
  gap: 0.625rem;
}

.action-btn {
  flex: 1;
  padding: 0.875rem 1rem;
  color: white;
  border: none;
  border-radius: 0.75rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  font-size: 0.875rem;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  letter-spacing: 0.02em;
}

.action-btn-primary {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-hover) 100%);
}

.action-btn-secondary {
  background: linear-gradient(135deg, #8b5cf6 0%, #7c3aed 100%);
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.25);
}

.action-btn:active {
  transform: translateY(0);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.btn-icon {
  font-size: 1.125rem;
}

@media (max-width: 1024px) {
  .detail-container {
    flex-direction: column;
    padding: 4rem 0.5rem 0.5rem;
    gap: 0.5rem;
  }

  .image-section {
    flex: 0 1 auto;
    max-height: 50vh;
  }

  .detail-image {
    max-height: 50vh;
  }

  .nav-desktop {
    display: none;
  }

  .nav-mobile {
    display: flex;
    justify-content: space-between;
    padding: 0 0.5rem;
    margin-bottom: 0.5rem;
  }

  .nav-mobile .nav-btn {
    width: 3rem;
    height: 3rem;
    font-size: 1.75rem;
  }

  .info-panel {
    width: 100%;
    flex: 1;
    border-radius: 1rem 1rem 0 0;
  }

  .info-scroll {
    padding: 1rem;
  }

  .params-grid {
    grid-template-columns: 1fr;
  }

  .action-buttons {
    flex-direction: column;
  }

  .image-counter {
    font-size: 0.813rem;
    padding: 0.375rem 1rem;
  }

  .close-btn {
    width: 2.5rem;
    height: 2.5rem;
    font-size: 1.25rem;
  }
}

.info-scroll::-webkit-scrollbar {
  width: 8px;
}

.info-scroll::-webkit-scrollbar-track {
  background: transparent;
}

.info-scroll::-webkit-scrollbar-thumb {
  background: linear-gradient(180deg, var(--primary) 0%, var(--primary-hover) 100%);
  border-radius: 4px;
}

.info-scroll::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(180deg, var(--primary-hover) 0%, var(--primary) 100%);
}
</style>
