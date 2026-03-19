<script setup lang="ts">
import Pinyin from "../components/Pinyin.vue";
import Keyboard from "../components/Keyboard.vue";
import TypeSummary from "../components/TypeSummary.vue";
import MenuList from "../components/MenuList.vue";

import {
  ref,
  watch,
  watchPostEffect,
  onActivated,
  onDeactivated,
  onMounted,
  computed,
  nextTick,
} from "vue";
import { useStore } from "../store";
import { storeToRefs } from "pinia";

import rawArticles from "../utils/article.json";
import { getPinyinOf } from "../utils/hanzi";
import { matchSpToPinyin } from "../utils/keyboard";
import { TypingSummary } from "../utils/summary";

// ---------- 指标与分段配置接口 ----------
interface CriteriaConfig {
  open: boolean;
  speed: number;
  accuracy: number;
  pressPerHanzi: number;
  action: 'noop' | 'retry' | 'shuffle';
  paragraphSize: number;
}

const criteria = ref<CriteriaConfig>({
  open: false,
  speed: 200,
  accuracy: 95,
  pressPerHanzi: 3,
  action: 'noop',
  paragraphSize: 50,
});

// 加载配置
onMounted(() => {
  const saved = localStorage.getItem('paragraph-criteria');
  if (saved) {
    try {
      criteria.value = JSON.parse(saved);
    } catch (e) {
      console.error('Failed to load criteria config', e);
    }
  }
  console.log('Loaded criteria:', criteria.value);
});

// 保存配置
watch(criteria, (val) => {
  localStorage.setItem('paragraph-criteria', JSON.stringify(val));
  console.log('Criteria saved:', val);
}, { deep: true });

// ---------- Store 和状态 ----------
const store = useStore();
const articles = storeToRefs(store).articles;
const settings = storeToRefs(store).settings;

const summary = ref(new TypingSummary());

function onKeyPressed() {
  summary.value.onKeyPressed();
}

onActivated(() => {
  document.addEventListener("keypress", onKeyPressed);
});

onDeactivated(() => {
  document.removeEventListener("keypress", onKeyPressed);
});

// 初始化文章列表（补全 Progress 必需属性）
(function checkArticles() {
  const rawNames = new Set([...Object.keys(rawArticles)]);
  articles.value.forEach((v) => {
    rawNames.delete(v.type);
  });

  rawNames.forEach((v) => {
    const name = v as RawArticleName;
    const progress: Progress = {
      currentIndex: 0,
      total: rawArticles[name].length,
      history: [],
      correctSum: 0,
    };

    articles.value.push({ progress, type: name });
  });
})();

function loadArticleText(article: Article) {
  if (article.type === "CUSTOM") {
    const text = localStorage.getItem(article.name) ?? "";
    return {
      type: article.type,
      text,
      name: article.name,
      progress: article.progress,
    };
  }

  return {
    type: article.type,
    text: rawArticles[article.type] ?? "",
    name: article.type as string,
    progress: article.progress,
  };
}

// 跳到下一个有效汉字（非汉字字符跳过）
function jumpToNextValidHanzi(index: number, text: string) {
  while (index < text.length && getPinyinOf(text[index]).length === 0) {
    index += 1;
  }
  return index;
}

const index = storeToRefs(store).currentArticleIndex;

// 分段相关状态
const paragraphs = ref<string[]>([]);
const currentParagraphNo = ref(1);
const shuffledCurrentPara = ref<string | null>(null);

// 根据文章全文和段落大小重新计算段落
function splitIntoParagraphs(fullText: string): string[] {
  const result: string[] = [];
  const size = criteria.value.paragraphSize;
  for (let i = 0; i < fullText.length; i += size) {
    result.push(fullText.slice(i, i + size));
  }
  console.log(`Split into ${result.length} paragraphs, size=${size}`);
  return result;
}

// 切换文章时重新计算段落
function resetParagraphs(fullText: string) {
  paragraphs.value = splitIntoParagraphs(fullText);
  currentParagraphNo.value = 1;
  shuffledCurrentPara.value = null;
  // 重置段内进度
  const info = loadArticleText(articles.value[index.value % articles.value.length]);
  info.progress.currentIndex = 0;
  console.log('Reset paragraphs, total paragraphs:', paragraphs.value.length);
}

// 获取当前段实际显示的文本（若乱序则用乱序版本）
const currentParagraphText = computed(() => {
  if (paragraphs.value.length === 0) return '';
  const para = paragraphs.value[currentParagraphNo.value - 1] || '';
  const text = shuffledCurrentPara.value ?? para;
  return text;
});

// 构建当前段显示结构（[字符, 段内偏移]）
const currentDisplay = computed<Array<[string, number]>>(() => {
  return currentParagraphText.value.split('').map((char, idx) => [char, idx]);
});

const article = computed(() => {
  const articleIndex = index.value % articles.value.length;
  const info = loadArticleText(articles.value[articleIndex]);

  // 如果是第一次加载或全文变化，重新分段
  if (paragraphs.value.length === 0) {
    resetParagraphs(info.text);
  }

  const currentParaText = currentParagraphText.value;
  // 不再自动重置索引，让索引可以超过长度以便触发完成检测
  const currentChar = currentParaText[info.progress.currentIndex] ?? "";
  const pinyin = getPinyinOf(currentChar);

  return {
    type: info.type,
    currentChar,
    answer: [...new Set(pinyin)],
    spHints: (store.mode().py2sp.get(pinyin.at(0) ?? "") ?? "").split(""),
    progress: info.progress,
    name: info.name,
    originalFullText: info.text,
  };
});

const articleMenuItems = computed(() => {
  return articles.value
    .map((v) => {
      if (v.type === "CUSTOM") {
        return v.name;
      }
      return v.type;
    })
    .map((x) => getShortName(x))
    .concat("新建文章");
});

const isEditing = ref(false);
const editingTitle = ref("");
const editingContent = ref("");
const validInput = computed(() => {
  return editingTitle.value.length > 0 && editingContent.value.length > 0;
});

function onAriticleChange(i: number) {
  index.value = i;
  isEditing.value = i >= articles.value.length;
  const info = loadArticleText(articles.value[i % articles.value.length]);
  resetParagraphs(info.text);
  summary.value = new TypingSummary();
  console.log('Article changed to:', info.name);
}

const pinyin = ref<string[]>([]);
const isValidPinyin = ref(false);

function onSeq([lead, follow]: [string?, string?]) {
  console.log('onSeq called', lead, follow);
  for (const answer of article.value.answer) {
    const res = matchSpToPinyin(
      store.mode(),
      lead as Char,
      follow as Char,
      answer
    );
    pinyin.value = [res.lead, res.follow].filter((v) => !!v);

    if (!!lead && !!follow) {
      store.updateProgressOnValid(res.lead, res.follow, res.valid);
    }

    isValidPinyin.value ||= res.valid;

    if (isValidPinyin.value) break;
  }

  const fullInput = !!lead && !!follow;
  if (fullInput) {
    summary.value.onValid(isValidPinyin.value);
    console.log('Valid input, isValidPinyin:', isValidPinyin.value);
  }

  return isValidPinyin.value;
}

function scrollToFocus() {
  const cursor = document.getElementById("cursor");
  if (cursor) {
    cursor.scrollIntoView({
      inline: "nearest",
      block: "center",
      behavior: "smooth",
    });
    console.log('Scrolled to cursor');
  } else {
    console.warn('Cursor element not found');
  }
}

onActivated(() => scrollToFocus());

watchPostEffect(() => {
  scrollToFocus();

  if (isValidPinyin.value) {
    setTimeout(() => {
      pinyin.value = [];
      article.value.progress.currentIndex += 1;
      isValidPinyin.value = false;
    }, 30);
  }
});

// 监听当前段是否完成
watch(() => article.value.progress.currentIndex, (newVal, oldVal) => {
  const paraLength = currentParagraphText.value.length;
  console.log(`Index changed: ${oldVal} -> ${newVal}, paraLength=${paraLength}`);
  if (paraLength > 0 && newVal >= paraLength && oldVal < paraLength) {
    console.log('Paragraph finished!');
    handleParagraphFinish();
  }
});

// 监听段落号变化，确保光标可见
watch(currentParagraphNo, () => {
  nextTick(() => {
    scrollToFocus();
  });
});

function handleParagraphFinish() {
  // 防止重复调用
  if (article.value.progress.currentIndex < currentParagraphText.value.length) {
    console.log('handleParagraphFinish called but not finished?');
    return;
  }

  console.log('=== Paragraph Finish ===');
  console.log('Criteria open:', criteria.value.open);
  console.log('Speed:', summary.value.hanziPerMinutes, 'Threshold:', criteria.value.speed);
  console.log('Accuracy:', summary.value.totalAccuracy * 100, 'Threshold:', criteria.value.accuracy);
  console.log('Press per Hanzi:', summary.value.pressPerHanzi, 'Threshold:', criteria.value.pressPerHanzi);

  // 一段打完，检查指标
  if (!criteria.value.open) {
    console.log('Criteria not open, go to next paragraph');
    goToNextParagraph();
    return;
  }

  const speed = summary.value.hanziPerMinutes;
  const accuracy = summary.value.totalAccuracy * 100;
  const pressPerHanzi = summary.value.pressPerHanzi;

  let meet = true;
  if (criteria.value.speed > 0 && speed < criteria.value.speed) meet = false;
  if (criteria.value.accuracy > 0 && accuracy < criteria.value.accuracy) meet = false;
  if (criteria.value.pressPerHanzi > 0 && pressPerHanzi > criteria.value.pressPerHanzi) meet = false;

  console.log('Meet criteria?', meet);

  if (!meet) {
    console.log('Not meet, action:', criteria.value.action);
    // 未达标，根据动作处理
    switch (criteria.value.action) {
      case 'retry':
        // 重打本段：重置段内进度
        article.value.progress.currentIndex = 0;
        break;
      case 'shuffle':
        // 乱序本段
        shuffleCurrentParagraph();
        break;
      case 'noop':
        // 不处理，直接进入下一段
        goToNextParagraph();
        break;
    }
  } else {
    console.log('Meet criteria, go to next paragraph');
    goToNextParagraph();
  }

  // 重置统计（下一段重新累计）
  summary.value = new TypingSummary();
  console.log('Summary reset');
}

// 进入下一段
function goToNextParagraph() {
  if (currentParagraphNo.value < paragraphs.value.length) {
    currentParagraphNo.value++;
  } else {
    currentParagraphNo.value = 1;
  }
  article.value.progress.currentIndex = 0;
  shuffledCurrentPara.value = null;
  console.log('Go to next paragraph:', currentParagraphNo.value);

  // 确保光标可见
  nextTick(() => {
    scrollToFocus();
  });
}

// 乱序当前段（保持换行符位置不变）
function shuffleCurrentParagraph() {
  const original = paragraphs.value[currentParagraphNo.value - 1];
  // 按换行符分割
  const lines = original.split('\n');
  const lengths = lines.map(l => l.length);

  // 将所有字符打平并打乱
  const allChars = original.split('');
  for (let i = allChars.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [allChars[i], allChars[j]] = [allChars[j], allChars[i]];
  }

  // 按原行长度重新分配
  let newLines: string[] = [];
  let start = 0;
  for (let len of lengths) {
    const end = start + len;
    newLines.push(allChars.slice(start, end).join(''));
    start = end;
  }

  const shuffled = newLines.join('\n');
  shuffledCurrentPara.value = shuffled;
  article.value.progress.currentIndex = 0;
  console.log('Shuffled current paragraph');
}

function getShortName(s: string, n = 10) {
  let ret = s.slice(0, n);
  if (s.length > n) {
    ret = ret.slice(0, n - 2) + "...";
  }
  return ret;
}

function saveArticle() {
  if (!validInput.value) return;
  localStorage.setItem(editingTitle.value, editingContent.value);
  isEditing.value = false;

  articles.value.push({
    type: "CUSTOM",
    name: editingTitle.value,
    progress: {
      currentIndex: 0,
      total: editingContent.value.length,
      history: [],
      correctSum: 0,
    },
  });

  editingTitle.value = "";
  editingContent.value = "";
  index.value = articles.value.length - 1;
  resetParagraphs(editingContent.value);
  console.log('Custom article saved');
}

function deleteArticle() {
  articles.value.splice(index.value, 1);
  onAriticleChange(index.value);
  console.log('Article deleted');
}

function shortPinyin(pinyins: string[]) {
  let ret = [];
  let count = 0;
  for (const py of pinyins) {
    if (count + py.length <= 10) {
      count += py.length;
      ret.push(py.toUpperCase());
    }
  }
  return ret.join("/");
}
</script>

<template>
  <!-- 根元素绑定字体大小，以响应全局设置 -->
  <div class="p-mode" :style="{ fontSize: settings.fontSize }">
    <!-- 顶部区域：文章标题和菜单 -->
    <div class="display-area" :class="isEditing && 'editing'">
      <div class="p-title" :class="isEditing && 'editing'">
        <div class="pinyin">
          <Pinyin :chars="pinyin" />
        </div>

        <div class="title-info">
          <div v-if="settings.enablePinyinHint" class="answer">
            {{ shortPinyin(article.answer) }}
          </div>
          <div class="title-and-count">
            <div class="count">
              {{ article.progress.currentIndex }} / {{ currentParagraphText.length }} 字
            </div>
            <div class="title">
              {{ getShortName(article.name) }}
            </div>
          </div>
        </div>

        <div class="article-menu" :title="isEditing ? '' : article.name">
          <MenuList
            :items="articleMenuItems"
            :index="index"
            :on-menu-change="onAriticleChange"
          />

          <div
            v-if="article.type === 'CUSTOM'"
            class="delete-btn"
            @click="deleteArticle"
          >
            删除文章
          </div>
        </div>
      </div>

      <!-- 当前段文字显示区域 -->
      <div v-if="!isEditing" class="text-area">
        <div class="scroll-area">
          <p>
            <span
              v-for="([s, t], si) in currentDisplay"
              :key="si"
              class="bg-text"
              :class="t < article.progress.currentIndex ? 'done-text' : t === article.progress.currentIndex ? 'current-text' : ''"
              :id="t === article.progress.currentIndex ? 'cursor' : ''"
            >
              {{ s }}
            </span>
          </p>
        </div>
      </div>

      <!-- 编辑区域（新建文章时） -->
      <div v-else class="editing-text-area">
        <div class="editing-bar">
          <input
            v-model="editingTitle"
            class="editing-title"
            placeholder="键入标题"
          />
          <div
            class="save-btn"
            :class="!validInput && 'disable'"
            @click="saveArticle"
          >
            保存文章
          </div>
        </div>
        <textarea
          v-model="editingContent"
          class="editing-text"
          placeholder="键入范文……"
        />
      </div>
    </div>

    <!-- 底部控制区：指标栏 + 虚拟键盘 -->
    <div class="bottom-area">
      <!-- 指标栏（暗黑模式已通过 CSS 变量适配） -->
      <div class="criteria-bar" v-if="!isEditing">
        <div class="criteria-item">
          <span class="criteria-label">指标</span>
          <label class="switch">
            <input type="checkbox" v-model="criteria.open" />
            <span class="slider"></span>
          </label>
        </div>
        <template v-if="criteria.open">
          <div class="criteria-item">
            <span class="criteria-label">速度≥</span>
            <input type="number" v-model.number="criteria.speed" min="0" step="10" class="criteria-input" />
          </div>
          <div class="criteria-item">
            <span class="criteria-label">键准≥</span>
            <input type="number" v-model.number="criteria.accuracy" min="0" max="100" step="1" class="criteria-input" />
          </div>
          <div class="criteria-item">
            <span class="criteria-label">每字击键≤</span>
            <input type="number" v-model.number="criteria.pressPerHanzi" min="0" step="0.1" class="criteria-input" />
          </div>
          <div class="criteria-item">
            <span class="criteria-label">未达标时</span>
            <select v-model="criteria.action" class="criteria-select">
              <option value="noop">不处理</option>
              <option value="retry">重打</option>
              <option value="shuffle">乱序</option>
            </select>
          </div>
        </template>
        <!-- 分段设置：仅在指标开启时显示 -->
        <template v-if="criteria.open">
          <div class="criteria-item">
            <span class="criteria-label">每段字数</span>
            <input type="number" v-model.number="criteria.paragraphSize" min="1" step="1" class="criteria-input" @change="resetParagraphs(article.originalFullText)" />
          </div>
          <div class="criteria-item">
            <span class="criteria-label">段 {{ currentParagraphNo }}/{{ paragraphs.length }}</span>
          </div>
        </template>
      </div>

      <!-- 虚拟键盘（进一步缩小） -->
      <Keyboard v-if="!isEditing" :valid-seq="onSeq" :hints="article.spHints" class="small-keyboard" />
    </div>

    <!-- 统计摘要（固定在右下角） -->
    <div v-if="!isEditing" class="summary">
      <TypeSummary
        :speed="summary.hanziPerMinutes"
        :accuracy="summary.totalAccuracy"
        :avgpress="summary.pressPerHanzi"
      />
    </div>
  </div>
</template>

<style lang="less" scoped>
@import "../styles/color.less";
@import "../styles/var.less";

.p-mode {
  display: flex;
  flex-direction: column;
  height: 100vh; // 使用视口高度，让底部区域固定

  .display-area {
    flex: 1; // 占据剩余空间
    padding: 0 64px 32px 32px;
    display: flex;
    align-items: center;
    justify-content: center;

    @media (max-width: 576px) {
      flex-direction: column;
      padding: var(--app-padding);
    }

    &.editing {
      align-items: flex-start;
      @media (max-width: 576px) {
        align-items: center;
      }
    }

    .p-title {
      margin-right: 32px;
      width: 260px;
      display: flex;
      flex-direction: column;
      align-items: flex-end;

      @media (max-width: 576px) {
        width: 100vw;
        padding-right: calc(var(--app-padding) + 2px);
        margin-right: 0;
        box-sizing: border-box;
      }

      .pinyin {
        font-size: 12px;
      }

      .title-info {
        display: flex;
        align-items: center;
        margin-top: 8px;

        .answer {
          font-size: 20px;
          margin-right: 16px;
          font-weight: bold;
          border-top: 1px solid var(--black);
          border-bottom: 1px solid var(--black);
        }
      }

      .title-and-count {
        display: flex;
        flex-direction: column;
        align-items: flex-end;
        font-weight: bold;
        font-size: 12px;

        .title {
          max-width: 160px;
          text-align: right;

          @media (max-width: 576px) {
            max-width: 100vw;
          }
        }
      }

      .article-menu {
        display: none;
        height: 110px;
      }
    }

    .p-title:hover,
    .p-title.editing {
      flex-direction: column;

      @media (max-width: 576px) {
        align-items: center;
      }

      .pinyin,
      .title-info,
      .title-and-count {
        display: none;
      }

      .article-menu {
        display: flex;
        flex-direction: column;
        position: relative;

        .menu {
          overflow: visible;
        }

        .delete-btn {
          color: @primary-color;
          opacity: 0.5;
          font-size: 14px;
          cursor: pointer;
          font-weight: bold;
          transition: all ease 0.3s;
          margin-top: 16px;
          text-align: center;
          position: absolute;
          bottom: -10px;
          padding-left: 1.4em;

          &:hover {
            opacity: 1;
          }
        }
      }
    }

    .text-area {
      position: relative;
      width: 50vw;
      max-width: calc(0.6 * var(--page-max-width));

      @media (max-width: 576px) {
        width: 100vw;
        max-width: calc(100vw - var(--app-padding) * 2);
      }

      .scroll-area {
        overflow-y: scroll;
        height: 240px; // 增加高度，拉长打字区域
        position: relative;
        margin: 8px 0;

        @media (max-width: 576px) {
          height: 30vh;
        }

        .bg-text {
          opacity: 1;
        }

        .done-text {
          opacity: 0.2;
        }

        .current-text {
          text-decoration: underline;
          text-underline-offset: 2px;
          opacity: 0.8;
        }
      }
    }

    .editing-text-area {
      display: flex;
      flex-direction: column;
      margin-top: 40px;
      width: 50vw;
      max-width: calc(0.6 * var(--page-max-width));

      @media (max-width: 576px) {
        width: 100vw;
        max-width: calc(100vw - var(--app-padding) * 2);
      }

      .editing-bar {
        display: flex;
        align-items: center;
        margin-bottom: 16px;

        .editing-title {
          font-family: inherit;
          font-size: 14px;
          font-weight: bold;
          border: 0;
          outline: none;
          padding: 0 8px;
          color: @primary-color;
          border-left: 5px solid @primary-color;
          flex: 1;
          background-color: transparent;
        }

        .save-btn {
          color: @primary-color;
          font-size: 14px;
          cursor: pointer;

          &.disable {
            color: var(--gray-a);
          }
        }
      }

      .editing-text {
        font-family: inherit;
        font-size: 14px;
        font-weight: bold;
        border: 0;
        outline: none;
        padding: 8px;
        height: calc(var(--page-height) - 200px);
        resize: none;
        border: 3px double var(--gray-6);
        color: var(--black);
        background-color: transparent;
        padding-left: 10px;

        @media (max-width: 576px) {
          height: calc(var(--page-height) - 300px);
        }
      }
    }
  }

  // 底部区域（指标栏 + 键盘）
  .bottom-area {
    flex-shrink: 0; // 防止被压缩
    background: var(--gray-f8);
    border-top: 1px solid var(--gray-e0);
    padding: 8px 16px;

    .criteria-bar {
      display: flex;
      flex-wrap: wrap;
      align-items: center;
      gap: 1rem;
      font-size: 14px;
      margin-bottom: 8px; // 与键盘留出间距

      @media (max-width: 576px) {
        gap: 0.5rem;
      }

      .criteria-item {
        display: flex;
        align-items: center;
        gap: 0.3rem;

        .criteria-label {
          white-space: nowrap;
          color: var(--black);
        }

        .criteria-input,
        .criteria-select {
          width: 70px;
          padding: 4px;
          border: 1px solid var(--gray-c);
          border-radius: 4px;
          background: var(--white);
          color: var(--black);
          font-size: 14px;
          transition: all 0.2s;

          @media (max-width: 576px) {
            width: 60px;
          }

          // 暗黑模式适配（通过 CSS 变量自动切换）
          &:focus {
            border-color: @primary-color;
            outline: none;
            box-shadow: 0 0 0 2px fade(@primary-color, 20%);
          }
        }

        .criteria-select {
          width: auto;
          min-width: 80px;
        }

        .switch {
          position: relative;
          display: inline-block;
          width: 40px;
          height: 20px;
          margin-left: 4px;

          input {
            opacity: 0;
            width: 0;
            height: 0;
          }

          .slider {
            position: absolute;
            cursor: pointer;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background-color: var(--gray-c);
            transition: 0.3s;
            border-radius: 20px;

            &:before {
              position: absolute;
              content: "";
              height: 16px;
              width: 16px;
              left: 2px;
              bottom: 2px;
              background-color: white;
              transition: 0.3s;
              border-radius: 50%;
            }
          }

          input:checked + .slider {
            background-color: @primary-color;
          }

          input:checked + .slider:before {
            transform: translateX(20px);
          }
        }
      }
    }

    // 进一步缩小虚拟键盘
    .small-keyboard {
      transform: scale(0.7);
      transform-origin: bottom center;
      margin-bottom: -15px; // 补偿缩放造成的空白

      // 缩小键盘内部文字（通过深度选择器覆盖 Keyboard 组件的样式）
      :deep(.key-item) {
        .main-key {
          font-size: 1.2rem;
        }
        .follow-key,
        .lead-key {
          font-size: 0.8rem;
        }
      }
    }
  }

  .summary {
    position: absolute;
    right: var(--app-padding);
    bottom: 140px; // 调整到底部栏上方
    z-index: 1000; // 确保不被覆盖
    background: var(--white);
    padding: 8px 16px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    border: 1px solid var(--gray-e0);

    @media (max-width: 576px) {
      top: 36px;
      bottom: auto;
      right: auto;
      left: 50%;
      transform: translateX(-50%);
      width: 90%;
      text-align: center;
    }
  }
}
</style>
