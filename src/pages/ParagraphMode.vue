<script setup lang="ts">
import Pinyin from "../components/Pinyin.vue";
import Keyboard from "../components/Keyboard.vue";
import TypeSummary from "../components/TypeSummary.vue";

import {
  ref,
  watchPostEffect,
  onActivated,
  onDeactivated,
  watchEffect,
  computed,
  watch, // 新增导入
} from "vue";
import { useStore } from "../store";
import { storeToRefs } from "pinia";

import rawArticles from "../utils/article.json";
import { getPinyinOf, isValidHanzi } from "../utils/hanzi";
import { matchSpToPinyin } from "../utils/keyboard";
import { TypingSummary } from "../utils/summary";
import MenuList from "../components/MenuList.vue";

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

(function checkArticles() {
  const rawNames = new Set(Object.keys(rawArticles));
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

/**
 * 跳转到下一个有效的汉字（即在拼音库中存在的汉字）
 */
function jumpToNextValidHanzi(index: number, text: string) {
  while (index < text.length && !isValidHanzi(text[index])) {
    index += 1;
  }
  return index;
}

const index = storeToRefs(store).currentArticleIndex;
const article = computed(() => {
  const articleIndex = index.value % articles.value.length;

  const info = loadArticleText(articles.value[articleIndex]);

  info.progress.currentIndex = jumpToNextValidHanzi(
    info.progress.currentIndex,
    info.text,
  );

  const currentHanzi = info.text[info.progress.currentIndex] ?? "";
  const pinyin = getPinyinOf(currentHanzi);

  // 分段
  let text: [[string, number][]] = [[]];
  for (let i = 0; i < info.text.length; ++i) {
    const char = info.text[i];
    if (char === "\n") {
      text.push([]);
    } else {
      text.at(-1)?.push([char, i - info.progress.currentIndex]);
    }
  }

  return {
    type: info.type,
    text,
    currentHanzi,
    answer: [...new Set(pinyin)],
    spHints: (store.mode().py2sp.get(pinyin.at(0) ?? "") ?? "").split(""),
    progress: info.progress,
    name: info.name,
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

// ========== 新增：分段练习设置 ==========
const enableSegment = ref(false);               // 是否开启分段练习
const segmentSize = ref(50);                    // 每段字数（默认50）
const thresholdSpeed = ref(100);                 // 速度下限（字/分）
const thresholdAccuracy = ref(95);               // 准确率下限（%）
const thresholdPress = ref(2.5);                 // 平均击键上限（次/字）
const thresholdAction = ref<"shuffle" | "retry" | "none">("shuffle"); // 未达标操作

// 分段相关状态（仅在练习时使用）
const fullText = ref("");                         // 原始文章全文
const segments = ref<Array<{ start: number; end: number }>>([]); // 分段起止索引
const currentSegmentIndex = ref(0);                // 当前段索引
const segmentStartStats = ref({                     // 段开始时记录的统计快照
  totalChars: 0,
  totalKeys: 0,
  totalErrors: 0,
  time: 0,
});

// 计算总段数
const totalSegments = computed(() => segments.value.length);

// 根据全文和每段字数生成分段
function buildSegments(text: string, size: number) {
  const segs = [];
  let start = 0;
  while (start < text.length) {
    const end = Math.min(start + size, text.length);
    segs.push({ start, end });
    start = end;
  }
  return segs;
}

// 在开始练习时初始化分段（如果开启）
function initSegmentsIfEnabled() {
  if (!enableSegment.value) return;
  // 从当前文章获取全文
  const info = loadArticleText(articles.value[index.value]);
  fullText.value = info.text;
  segments.value = buildSegments(fullText.value, segmentSize.value);
  currentSegmentIndex.value = 0;
  // 重置进度索引到当前段起点
  article.value.progress.currentIndex = segments.value[0].start;
  // 记录段起始统计
  recordSegmentStart();
}

// 记录当前段起始时的统计快照
function recordSegmentStart() {
  segmentStartStats.value = {
    totalChars: summary.value.totalValidMatches || 0,   // 已输入字数
    totalKeys: summary.value.totalPressCount || 0,      // 总按键数
    totalErrors: (summary.value.totalValidMatches || 0) - (summary.value.totalCorrectMatches || 0), // 错误次数
    time: Date.now(),
  };
}

// 检查当前段是否达标，返回是否达标
function checkSegment达标() {
  const now = Date.now();
  const timeDelta = (now - segmentStartStats.value.time) / 1000 / 60; // 分钟
  const charsDelta = (summary.value.totalValidMatches || 0) - segmentStartStats.value.totalChars;
  const keysDelta = (summary.value.totalPressCount || 0) - segmentStartStats.value.totalKeys;
  const errorsDelta = ((summary.value.totalValidMatches || 0) - (summary.value.totalCorrectMatches || 0)) - segmentStartStats.value.totalErrors;

  if (charsDelta === 0) return true; // 未打任何字，视为达标（避免除零）

  const speed = charsDelta / timeDelta;                 // 字/分
  const accuracy = (charsDelta - errorsDelta) / charsDelta; // 准确率
  const pressPerChar = keysDelta / charsDelta;          // 平均击键

  const speedOK = speed >= thresholdSpeed.value;
  const accOK = accuracy * 100 >= thresholdAccuracy.value;
  const pressOK = pressPerChar <= thresholdPress.value;

  return speedOK && accOK && pressOK;
}

// 处理段结束：根据阈值执行操作
function handleSegmentEnd() {
  const is达标 = checkSegment达标();
  if (!is达标) {
    switch (thresholdAction.value) {
      case "shuffle":
        // 乱序剩余段落（不包括当前已完成段）
        const remainingSegments = segments.value.slice(currentSegmentIndex.value + 1);
        shuffleArray(remainingSegments);
        segments.value = [
          ...segments.value.slice(0, currentSegmentIndex.value + 1),
          ...remainingSegments,
        ];
        // 继续进入下一段
        moveToNextSegment();
        break;
      case "retry":
        // 重打当前段：重置索引到当前段起点
        article.value.progress.currentIndex = segments.value[currentSegmentIndex.value].start;
        // 清空输入缓存（pinyin 重置由外部处理）
        break;
      case "none":
      default:
        moveToNextSegment();
        break;
    }
  } else {
    moveToNextSegment();
  }
}

// 进入下一段
function moveToNextSegment() {
  if (currentSegmentIndex.value + 1 < segments.value.length) {
    currentSegmentIndex.value++;
    article.value.progress.currentIndex = segments.value[currentSegmentIndex.value].start;
    recordSegmentStart();
  } else {
    // 所有段落完成，可触发结束（现有逻辑会处理）
  }
}

// 工具：打乱数组
function shuffleArray<T>(arr: T[]) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
}
// ========== 新增结束 ==========

function onAriticleChange(i: number) {
  index.value = i;
  isEditing.value = i >= articles.value.length;
}

const pinyin = ref<string[]>([]);
const isValidPinyin = ref(false);

function onSeq([lead, follow]: [string?, string?]) {
  for (const answer of article.value.answer) {
    const res = matchSpToPinyin(
      store.mode(),
      lead as Char,
      follow as Char,
      answer,
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
      const nextIndex = article.value.progress.currentIndex + 1;
      article.value.progress.currentIndex = nextIndex;
      isValidPinyin.value = false;

      // 新增：如果开启了分段练习，检查是否到达段尾
      if (enableSegment.value && segments.value.length > 0) {
        const currentSegment = segments.value[currentSegmentIndex.value];
        if (nextIndex >= currentSegment.end) {
          // 当前段结束
          handleSegmentEnd();
        }
      }
    }, 30);
  }
});

watchEffect(() => {
  if (article.value.progress.currentIndex >= article.value.progress.total) {
    article.value.progress.currentIndex = 0;
  }
});

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

// 当文章切换或开始练习时，如果开启分段则初始化分段
watch(index, () => {
  if (!isEditing.value && enableSegment.value) {
    // 延迟等待 article 计算完成
    setTimeout(() => {
      initSegmentsIfEnabled();
    }, 0);
  }
});
</script>

<template>
  <div class="p-mode">
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

      <!-- 新增：分段练习设置面板（仅在编辑时显示） -->
      <div v-if="isEditing" class="segment-settings">
        <h4>分段练习设置</h4>
        <div class="setting-row">
          <span class="setting-label">开启分段练习</span>
          <el-switch v-model="enableSegment" />
        </div>
        <template v-if="enableSegment">
          <div class="setting-row">
            <span class="setting-label">每段字数</span>
            <el-input-number v-model="segmentSize" :min="10" :max="1000" size="small" />
          </div>
          <div class="setting-row">
            <span class="setting-label">速度下限（字/分）</span>
            <el-input-number v-model="thresholdSpeed" :min="0" :max="500" size="small" />
          </div>
          <div class="setting-row">
            <span class="setting-label">准确率下限（%）</span>
            <el-input-number v-model="thresholdAccuracy" :min="0" :max="100" size="small" />
          </div>
          <div class="setting-row">
            <span class="setting-label">平均击键上限（次/字）</span>
            <el-input-number v-model="thresholdPress" :min="0" :max="10" :step="0.1" size="small" />
          </div>
          <div class="setting-row">
            <span class="setting-label">未达标时操作</span>
            <el-select v-model="thresholdAction" size="small">
              <el-option label="乱序" value="shuffle" />
              <el-option label="重打当前段" value="retry" />
              <el-option label="不处理" value="none" />
            </el-select>
          </div>
          <div class="setting-note">
            * 当任何一项指标未达标时触发所选操作。
          </div>
        </template>
      </div>

      <div v-if="!isEditing" class="text-area">
        <div class="scroll-area">
          <p
            v-for="(p, i) in article.text"
            :key="i"
            :style="{ fontSize: settings.fontSize + 'px' }"
          >
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
        <!-- 新增：显示当前分段进度（仅当分段练习开启时） -->
        <div v-if="enableSegment" class="segment-progress">
          第 {{ currentSegmentIndex + 1 }} / {{ totalSegments }} 段
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

          @border: 1px solid var(--black);
          border-top: @border;
          border-bottom: @border;
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

    // 新增：分段设置面板样式
    .segment-settings {
      background-color: var(--white);
      border: 1px solid var(--gray-010);
      padding: 16px;
      margin-left: 20px;
      border-radius: 4px;
      font-size: 14px;

      h4 {
        margin: 0 0 12px 0;
        font-size: 16px;
        font-weight: bold;
      }

      .setting-row {
        display: flex;
        align-items: center;
        margin-bottom: 12px;

        .setting-label {
          width: 120px;
          flex-shrink: 0;
        }

        .el-input-number,
        .el-select {
          width: 140px;
        }
      }

      .setting-note {
        color: var(--gray-6);
        font-size: 12px;
        margin-top: 8px;
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

        p {
          line-height: 1.5;
          margin-bottom: 0.8em;
          word-break: break-all;
        }

        .bg-text {
          opacity: 0.4;
        }

        .done-text {
          opacity: 1;
        }

        .current-text {
          text-decoration: underline;
          text-underline-offset: 4px;
          opacity: 1;
          font-weight: 900;
          color: @primary-color;
        }
      }

      .segment-progress {
        text-align: right;
        font-size: 12px;
        color: var(--gray-6);
        margin-top: 4px;
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
