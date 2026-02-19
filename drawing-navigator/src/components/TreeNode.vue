<template>
  <div class="node">
    <div
      class="row"
      :class="{ selected: isSelected }"
      @click.stop="selectThis"
    >
      <!-- indent -->
      <span class="indent" :style="{ width: `${depth * 14}px` }"></span>

      <!-- caret -->
      <button
        v-if="hasChildren"
        class="caret"
        type="button"
        @click.stop="toggle"
        :aria-label="expanded ? 'collapse' : 'expand'"
      >
        {{ expanded ? "▾" : "▸" }}
      </button>
      <span v-else class="caret placeholder">•</span>

      <!-- icon -->
      <span class="icon">
        {{ icon }}
      </span>

      <!-- label -->
      <span class="label">{{ safeLabel }}</span>
    </div>

    <div v-if="hasChildren && expanded" class="children">
      <TreeNode
        v-for="c in node.children"
        :key="c.key"
        :node="c"
        :selected-key="selectedKey"
        :depth="depth + 1"
        @select="emitSelect"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from "vue";

type NodeKind = "project" | "drawing" | "discipline" | "region" | "revision" | "base";

type TreeNodeData = {
  key: string;
  label: string;
  kind: NodeKind;
  children?: TreeNodeData[];
  path: string[];
  drawingId?: string;
  discipline?: string;
  region?: string;
  imageFile?: string;
  rev?: any;
  imageTransform?: any;
};

// 재귀 컴포넌트 이름 지정
defineOptions({ name: "TreeNode" });

const props = defineProps<{
  node: TreeNodeData;
  selectedKey: string;
  depth?: number;
}>();

const emit = defineEmits<{
  (e: "select", key: string): void;
}>();

const depth = computed(() => props.depth ?? 0);
const hasChildren = computed(() => (props.node.children?.length ?? 0) > 0);
const expanded = ref(true);

const safeLabel = computed(() => {
  const v = props.node?.label;
  return typeof v === "string" && v.length > 0 ? v : "(untitled)";
});

const isSelected = computed(() => props.selectedKey === props.node.key);

const icon = computed(() => {
  switch (props.node.kind) {
    case "project":
      return "📁";
    case "drawing":
      return "🗂️";
    case "discipline":
      return "🧩";
    case "region":
      return "🧱";
    case "base":
      return "🧾";
    case "revision":
      return "🕒";
    default:
      return "•";
  }
});

function toggle() {
  expanded.value = !expanded.value;
}

function selectThis() {
  emit("select", props.node.key);
  if (hasChildren.value) expanded.value = true;
}

function emitSelect(key: string) {
  emit("select", key);
}
</script>

<style scoped>
.row {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 8px;
  border-radius: 10px;
  cursor: pointer;
  user-select: none;
}
.row:hover {
  background: #f5f5f5;
}
.row.selected {
  background: #e9f2ff;
  outline: 1px solid #cfe3ff;
}

.indent {
  display: inline-block;
  flex: 0 0 auto;
}

.caret {
  width: 22px;
  height: 22px;
  border: none;
  background: transparent;
  cursor: pointer;
  border-radius: 6px;
}
.caret:hover {
  background: #eee;
}
.caret.placeholder {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  opacity: 0.35;
  width: 22px;
}

.icon {
  width: 20px;
  display: inline-flex;
  justify-content: center;
  opacity: 0.9;
}
.label {
  font-size: 13px;
  color: #222;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.children {
  margin-left: 0;
}
</style>
