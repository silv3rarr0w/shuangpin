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
  watch,
  nextTick,
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

// ========== 分段练习设置 ==========
const enableSegment = ref(false);
const segmentSize = ref(10);
const thresholdSpeed = ref(50);
const thresholdAccuracy = ref(100);
const thresholdPress = ref(3);
const thresholdAction = ref<"shuffle" | "retry" | "none">("shuffle");

const fullText = ref("");
const segments = ref<Array<{ start: number; end: number }>>([]);
const currentSegmentIndex = ref(0);
const segmentStartStats = ref({
  totalChars: 0,
  totalKeys: 0,
  totalErrors: 0,
  time: 0,
});

const totalSegments = computed(() => segments.value.length);

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

// 重置分段状态
function resetSegments() {
  fullText.value = "";
  segments.value = [];
  currentSegmentIndex.value = 0;
  segmentStartStats.value = {
    totalChars: 0,
    totalKeys: 0,
    totalErrors: 0,
    time: 0,
  };
}

function initSegmentsIfEnabled() {
  if (!enableSegment.value) {
    resetSegments();
    return;
  }
  const info = loadArticleText(articles.value[index.value]);
  fullText.value = info.text;
  segments.value = buildSegments(fullText.value, segmentSize.value);
  currentSegmentIndex.value = 0;
  // 重置进度索引到第一段起点
  article.value.progress.currentIndex = segments.value[0]?.start || 0;
  recordSegmentStart();
}

function recordSegmentStart() {
  segmentStartStats.value = {
    totalChars: summary.value.totalValidMatches || 0,
    totalKeys: summary.value.totalPressCount || 0,
    totalErrors: (summary.value.totalValidMatches || 0) - (summary.value.totalCorrectMatches || 0),
    time: Date.now(),
  };
}

function checkSegment达标() {
  const now = Date.now();
  const timeDelta = (now - segmentStartStats.value.time) / 1000 / 60; // 分钟
  if (timeDelta <= 0) return true; // 时间未流逝，视为达标

  const charsDelta = (summary.value.totalValidMatches || 0) - segmentStartStats.value.totalChars;
  const keysDelta = (summary.value.totalPressCount || 0) - segmentStartStats.value.totalKeys;
  const errorsDelta = ((summary.value.totalValidMatches || 0) - (summary.value.totalCorrectMatches || 0)) - segmentStartStats.value.totalErrors;

  if (charsDelta === 0) return true; // 未打任何字，视为达标

  const speed = charsDelta / timeDelta;
  const accuracy = (charsDelta - errorsDelta) / charsDelta;
  const pressPerChar = keysDelta / charsDelta;

  const speedOK = speed >= thresholdSpeed.value;
  const accOK = accuracy * 100 >= thresholdAccuracy.value;
  const pressOK = pressPerChar <= thresholdPress.value;

  return speedOK && accOK && pressOK;
}

function handleSegmentEnd() {
  const is达标 = checkSegment达标();
  if (!is达标) {
    switch (thresholdAction.value) {
      case "shuffle": {
        const remainingSegments = segments.value.slice(currentSegmentIndex.value + 1);
        shuffleArray(remainingSegments);
        segments.value = [
          ...segments.value.slice(0, currentSegmentIndex.value + 1),
          ...remainingSegments,
        ];
        moveToNextSegment();
        break;
      }
      case "retry":
        article.value.progress.currentIndex = segments.value[currentSegmentIndex.value].start;
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

function moveToNextSegment() {
  if (currentSegmentIndex.value + 1 < segments.value.length) {
    currentSegmentIndex.value++;
    article.value.progress.currentIndex = segments.value[currentSegmentIndex.value].start;
    recordSegmentStart();
  }
}

function shuffleArray<T>(arr: T[]) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
}
// ========== 结束 ==========

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

      // 检查是否到达段尾（仅当分段开启且分段存在时）
      if (enableSegment.value && segments.value.length > 0) {
        const currentSegment = segments.value[currentSegmentIndex.value];
        if (currentSegment && nextIndex >= currentSegment.end) {
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

// 监听 index 和 isEditing，当进入练习模式时初始化分段
watch([index, isEditing], async ([newIndex, editing]) => {
  if (!editing && enableSegment.value) {
    await nextTick(); // 等待 article 计算完成
    initSegmentsIfEnabled();
  } else {
    resetSegments(); // 离开练习模式时清除分段状态
  }
}, { immediate: true });
</script>

<template>
  <!-- 模板部分与之前相同，此处省略以节省篇幅，请保持原样 -->
</template>

<style lang="less" scoped>
/* 样式与之前相同，此处省略 */
</style>
