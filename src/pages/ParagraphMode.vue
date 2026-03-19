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
} from "vue";
import { useStore } from "../store";
import { storeToRefs } from "pinia";

import rawArticles from "../utils/article.json";
import { getPinyinOf } from "../utils/hanzi";
import { matchSpToPinyin } from "../utils/keyboard";
import { TypingSummary } from "../utils/summary";

// 指标配置接口（持久化到 localStorage）
interface CriteriaConfig {
  open: boolean;
  speed: number;
  accuracy: number;
  pressPerHanzi: number;
  action: 'noop' | 'retry' | 'shuffle';
}

const criteria = ref<CriteriaConfig>({
  open: false,
  speed: 200,
  accuracy: 95,
  pressPerHanzi: 3,
  action: 'noop',
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
});

// 保存配置
watch(criteria, (val) => {
  localStorage.setItem('paragraph-criteria', JSON.stringify(val));
}, { deep: true });

// ---------- 原有 store 和状态 ----------
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
      history: [],       // 新增必需属性
      correctSum: 0,     // 新增必需属性
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

// 用于存储乱序后的文本（如果有）
const shuffledTextRef = ref<string | null>(null);

// 构建文章显示结构，显式标注元组类型
function buildArticleText(text: string): Array<Array<[string, number]>> {
  return text.split('\n').map(line => 
    line.split('').map((char, idx) => [char, idx] as [string, number])
  );
}

const article = computed(() => {
  const articleIndex = index.value % articles.value.length;
  const info = loadArticleText(articles.value[articleIndex]);

  info.progress.currentIndex = jumpToNextValidHanzi(
    info.progress.currentIndex,
    info.text
  );

  const currentHanzi = info.text[info.progress.currentIndex] ?? "";
  const pinyin = getPinyinOf(currentHanzi);

  // 使用乱序文本（如果有）否则使用原始文本
  const baseText = shuffledTextRef.value ?? info.text;
  const text = buildArticleText(baseText);

  return {
    type: info.type,
    text,
    currentHanzi,
    answer: [...new Set(pinyin)],
    spHints: (store.mode().py2sp.get(pinyin.at(0) ?? "") ?? "").split(""),
    progress: info.progress,
    name: info.name,
    originalText: info.text,
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
  shuffledTextRef.value = null;
  summary.value = new TypingSummary();
}

const pinyin = ref<string[]>([]);
const isValidPinyin = ref(false);

function onSeq([lead, follow]: [string?, string?]) {
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

// 监听完成事件
watch(() => article.value.progress.currentIndex, (newVal, oldVal) => {
  if (oldVal < article.value.progress.total && newVal >= article.value.progress.total) {
    handleArticleFinish();
  }
});

function handleArticleFinish() {
  if (!criteria.value.open) {
    article.value.progress.currentIndex = 0;
    summary.value = new TypingSummary();
    return;
  }

  const speed = summary.value.hanziPerMinutes;
  const accuracy = summary.value.totalAccuracy * 100;  // 使用 totalAccuracy
  const pressPerHanzi = summary.value.pressPerHanzi;

  let meet = true;
  if (criteria.value.speed > 0 && speed < criteria.value.speed) meet = false;
  if (criteria.value.accuracy > 0 && accuracy < criteria.value.accuracy) meet = false;
  if (criteria.value.pressPerHanzi > 0 && pressPerHanzi > criteria.value.pressPerHanzi) meet = false;

  if (!meet) {
    switch (criteria.value.action) {
      case 'retry':
        article.value.progress.currentIndex = 0;
        break;
      case 'shuffle':
        shuffleArticle();
        break;
      case 'noop':
        article.value.progress.currentIndex = 0;
        break;
    }
  } else {
    article.value.progress.currentIndex = 0;
  }

  summary.value = new TypingSummary();
}

function shuffleArticle() {
  const original = article.value.originalText;
  const paragraphs = original.split('\n');
  const lengths = paragraphs.map(p => p.length);

  const allChars = original.split('');
  for (let i = allChars.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [allChars[i], allChars[j]] = [allChars[j], allChars[i]];
  }

  let newParagraphs: string[] = [];
  let start = 0;
  for (let len of lengths) {
    const end = start + len;
    newParagraphs.push(allChars.slice(start, end).join(''));
    start = end;
  }

  const shuffled = newParagraphs.join('\n');
  shuffledTextRef.value = shuffled;
  article.value.progress.currentIndex = 0;
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
      history: [],       // 新增必需属性
      correctSum: 0,     // 新增必需属性
    },
  });

  editingTitle.value = "";
  editingContent.value = "";
  index.value = articles.value.length - 1;
}

function deleteArticle() {
  articles.value.splice(index.value, 1);
  onAriticleChange(index.value);
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
  <div class="p-mode">
    <!-- 指标设置栏 -->
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
    </div>

    <!-- 显示区域 -->
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
              {{ article.progress.currentIndex }} 字 /
              {{ article.progress.total }} 字
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

      <div v-if="!isEditing" class="text-area">
        <div class="scroll-area">
          <p v-for="(p, i) in article.text" :key="i">
            <span
              v-for="([s, t], si) in p"
              :key="si"
              class="bg-text"
              :class="t < 0 ? 'done-text' : t === 0 ? 'current-text' : ''"
              :id="t === 0 ? 'cursor' : ''"
            >
              {{ s }}
            </span>
          </p>
        </div>
      </div>

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

    <Keyboard v-if="!isEditing" :valid-seq="onSeq" :hints="article.spHints" />

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
  .criteria-bar {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 1rem;
    padding: 0.5rem 1rem;
    background: var(--gray-f8);
    border-bottom: 1px solid var(--gray-e0);
    font-size: 14px;

    @media (max-width: 576px) {
      padding: 0.5rem;
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

      .criteria-input {
        width: 70px;
        padding: 4px;
        border: 1px solid var(--gray-c);
        border-radius: 4px;
        background: var(--white);
        color: var(--black);
        font-size: 14px;

        @media (max-width: 576px) {
          width: 60px;
        }
      }

      .criteria-select {
        padding: 4px;
        border: 1px solid var(--gray-c);
        border-radius: 4px;
        background: var(--white);
        color: var(--black);
        font-size: 14px;
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

  // 以下为原有样式（略）
  .display-area {
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

      &:before {
        content: "";
        position: absolute;
        width: 100%;
        height: 100%;
        left: 0;
        top: 0;
        background: linear-gradient(
          0deg,
          var(--white) 0%,
          transparent 30%,
          transparent 70%,
          var(--white) 100%
        );
        pointer-events: none;
        z-index: 999;
      }

      .scroll-area {
        overflow-y: scroll;
        height: 144px;
        position: relative;
        margin: 8px 0;

        @media (max-width: 576px) {
          height: 30vh;
        }

        .bg-text {
          opacity: 0.4;
        }

        .done-text {
          opacity: 1;
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

  .summary {
    position: absolute;
    right: var(--app-padding);
    bottom: var(--app-padding);

    @media (max-width: 576px) {
      top: 36px;
    }
  }
}
</style>
