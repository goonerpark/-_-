<template>
  <div class="app">
    <header class="topbar">
      <div>
        <h1>Drawing Navigator (Tree)</h1>
        <div class="sub">unit: {{ meta?.project?.unit ?? "-" }}</div>
      </div>

      <div class="actions">
        <button class="btn" @click="reload">Reload</button>
      </div>
    </header>

    <main class="layout">
      <!-- LEFT: TREE -->
      <section class="left">
        <div class="panel-title">도면 트리</div>

        <div v-if="loading" class="muted">loading...</div>
        <div v-else-if="errorMsg" class="error">{{ errorMsg }}</div>
        <div v-else class="tree">
          <TreeNode
            v-for="n in tree"
            :key="n.key"
            :node="n"
            :selected-key="selectedKey"
            @select="onSelect"
          />
        </div>
      </section>

      <!-- RIGHT: VIEWER -->
      <section class="right">
        <div class="panel-title">뷰어</div>

        <div v-if="!selectedNode" class="muted">
          왼쪽 트리에서 도면/공종/리비전을 선택하세요.
        </div>

        <div v-else class="viewer-wrap">
          <!-- Breadcrumb -->
          <div class="breadcrumb">
            <span v-for="(p, i) in selectedNode.path" :key="i">
              <span class="crumb">{{ p }}</span>
              <span v-if="i < selectedNode.path.length - 1" class="sep">›</span>
            </span>
          </div>

          <!-- MODE CONTROLS -->
          <div class="modebar">
            <label class="chk">
              <input type="checkbox" v-model="compareEnabled" />
              공종 간 간섭 확인(오버레이)
            </label>

            <div v-if="compareEnabled" class="compare-controls">
              <div class="row">
                <div class="field">
                  <div class="label">기준(현재 선택)</div>
                  <div class="value">{{ selectedNode.label }}</div>
                </div>

                <div class="field">
                  <div class="label">겹칠 대상 선택</div>
                  <select class="select" v-model="overlayKey">
                    <option value="">-- 선택 --</option>
                    <option
                      v-for="c in overlayCandidates"
                      :key="c.key"
                      :value="c.key"
                    >
                      {{ c.path.join(" › ") }}
                    </option>
                  </select>
                </div>
              </div>

              <div class="row sliders">
                <div class="field">
                  <div class="label">투명도</div>
                  <input type="range" min="0" max="100" v-model.number="overlayOpacity" />
                  <div class="value">{{ overlayOpacity }}%</div>
                </div>

                <div class="field">
                  <div class="label">미세 X(dx)</div>
                  <input type="range" min="-500" max="500" v-model.number="overlayDx" />
                  <div class="value">{{ overlayDx }}px</div>
                </div>

                <div class="field">
                  <div class="label">미세 Y(dy)</div>
                  <input type="range" min="-500" max="500" v-model.number="overlayDy" />
                  <div class="value">{{ overlayDy }}px</div>
                </div>

                <div class="field">
                  <div class="label">미세 Scale</div>
                  <input type="range" min="50" max="200" v-model.number="overlayScalePct" />
                  <div class="value">{{ overlayScalePct }}%</div>
                </div>

                <div class="field">
                  <div class="label">미세 Rot</div>
                  <input type="range" min="-180" max="180" v-model.number="overlayRotDeg" />
                  <div class="value">{{ overlayRotDeg }}°</div>
                </div>

                <button class="btn small" @click="resetOverlayTuning">Reset</button>
              </div>
            </div>
          </div>

          <!-- INFO -->
          <div class="info card">
            <div class="info-grid">
              <div class="k">Type</div>
              <div class="v">{{ selectedNode.kind }}</div>

              <div class="k">Drawing</div>
              <div class="v">{{ selectedNode.drawingId ?? "-" }}</div>

              <div class="k">Discipline</div>
              <div class="v">{{ selectedNode.discipline ?? "-" }}</div>

              <div class="k">Region</div>
              <div class="v">{{ selectedNode.region ?? "-" }}</div>

              <template v-if="selectedNode.rev">
                <div class="k">rev</div>
                <div class="v">{{ selectedNode.rev.version }}</div>

                <div class="k">Date</div>
                <div class="v">{{ selectedNode.rev.date }}</div>

                <div class="k">Desc</div>
                <div class="v">{{ selectedNode.rev.description }}</div>

                <div class="k">Changes</div>
                <div class="v">
                  <ul class="changes">
                    <li v-for="(c, i) in selectedNode.rev.changes" :key="i">{{ c }}</li>
                    <li v-if="selectedNode.rev.changes.length === 0" class="muted">-</li>
                  </ul>
                </div>
              </template>
            </div>
          </div>

          <!-- VIEWPORT -->
          <div ref="viewportRef" class="viewport card">
            <div class="canvas">
              <!-- base image -->
              <img
                v-if="baseImageUrl"
                class="img base"
                :src="baseImageUrl"
                alt="base"
                @load="onBaseLoad"
              />

              <!-- overlay image -->
              <img
                v-if="compareEnabled && overlayNode && overlayImageUrl"
                class="img overlay"
                :src="overlayImageUrl"
                alt="overlay"
                :style="overlayStyle"
                @load="onOverlayLoad"
              />
            </div>
          </div>

          <div v-if="compareEnabled" class="hint muted">
            팁: 자동 정렬이 어색하면 dx/dy/scale/rot로 미세 조정해서 “벽/코어/기둥” 같은 기준 요소를 맞춘 다음 간섭을 확인하세요.
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";

type NodeKind = "project" | "drawing" | "discipline" | "region" | "revision" | "base";

type Vec2 = [number, number];

type ImageTransform = {
  relativeTo?: string;
  x: number;
  y: number;
  scale: number;
  rotation: number;
};

type Revision = {
  version: string;
  image: string;
  date: string;
  description: string;
  changes: string[];
  imageTransform?: ImageTransform;
  polygon?: unknown;
};

type Region = {
  polygon?: unknown;
  revisions: Revision[];
};

type Discipline = {
  image?: string;
  imageTransform?: ImageTransform;
  polygon?: unknown;
  revisions?: Revision[];
  regions?: Record<string, Region>;
};

type Drawing = {
  id: string;
  name: string;
  image: string;
  parent: string | null;
  position: {
    vertices: Vec2[] | null;
    imageTransform: ImageTransform | null;
  } | null;
  disciplines?: Record<string, Discipline>;
};

type Meta = {
  project: { name: string; unit: string };
  disciplines: { name: string }[];
  drawings: Record<string, Drawing>;
};

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
  imageTransform?: ImageTransform;
  rev?: Revision;
};

import TreeNode from "./components/TreeNode.vue";

const loading = ref(false);
const errorMsg = ref<string | null>(null);

const meta = ref<Meta | null>(null);
const tree = ref<TreeNodeData[]>([]);
const allNodes = ref<TreeNodeData[]>([]);

const selectedKey = ref<string>("");
const selectedNode = computed(() => allNodes.value.find((n) => n.key === selectedKey.value) ?? null);

const compareEnabled = ref(false);
const overlayKey = ref<string>("");

const overlayOpacity = ref(45);
const overlayDx = ref(0);
const overlayDy = ref(0);
const overlayScalePct = ref(100);
const overlayRotDeg = ref(0);

function resetOverlayTuning() {
  overlayOpacity.value = 45;
  overlayDx.value = 0;
  overlayDy.value = 0;
  overlayScalePct.value = 100;
  overlayRotDeg.value = 0;
}

async function loadMeta() {
  loading.value = true;
  errorMsg.value = null;
  try {
    const res = await fetch("/data/metadata.json");
    if (!res.ok) throw new Error(`metadata.json load failed: ${res.status}`);
    const json = (await res.json()) as Meta;
    meta.value = json;

    const built = buildTree(json);
    tree.value = built.tree;
    allNodes.value = built.flat;

    const first = allNodes.value.find((n) => !!n.imageFile);
    if (first) selectedKey.value = first.key;
  } catch (e: any) {
    errorMsg.value = e?.message ?? "알 수 없는 오류";
  } finally {
    loading.value = false;
  }
}

function reload() {
  loadMeta();
}

function buildTree(m: Meta): { tree: TreeNodeData[]; flat: TreeNodeData[] } {
  const flat: TreeNodeData[] = [];

  const byId = m.drawings;
  const roots: Drawing[] = Object.values(byId).filter((d) => d.parent === null);

  const childrenMap = new Map<string, Drawing[]>();
  for (const d of Object.values(byId)) {
    if (!d.parent) continue;
    const arr = childrenMap.get(d.parent) ?? [];
    arr.push(d);
    childrenMap.set(d.parent, arr);
  }

  function push(node: TreeNodeData) {
    flat.push(node);
    return node;
  }

  function walkDrawing(d: Drawing, path: string[]): TreeNodeData {
    const drawingPath = [...path, `${d.id}. ${d.name}`];

    const drawingNode = push({
      key: `drawing:${d.id}`,
      label: `${d.id}. ${d.name}`,
      kind: "drawing",
      path: drawingPath,
      drawingId: d.id,
      imageFile: d.image,
      children: [],
    });

    const disc = d.disciplines ?? {};
    for (const [discName, discObj] of Object.entries(disc)) {
      const discPath = [...drawingPath, discName];

      const discImageFile = discObj.image ?? undefined;

      const discNode = push({
        key: `disc:${d.id}:${discName}`,
        label: discName,
        kind: "discipline",
        path: discPath,
        drawingId: d.id,
        discipline: discName,
        imageFile: discImageFile,
        imageTransform: discObj.imageTransform,
        children: [],
      });

      if (discObj.revisions && discObj.revisions.length > 0) {
        for (const rev of discObj.revisions) {
          const revNode = push({
            key: `rev:${d.id}:${discName}:${rev.version}`,
            label: rev.version,
            kind: "revision",
            path: [...discPath, rev.version],
            drawingId: d.id,
            discipline: discName,
            imageFile: rev.image,
            imageTransform: rev.imageTransform ?? discObj.imageTransform,
            rev,
          });
          discNode.children!.push(revNode);
        }

        if (!discNode.imageFile) {
          discNode.imageFile = discObj.revisions[discObj.revisions.length - 1]!.image;
        }
      }

      if (discObj.regions) {
        if (discObj.image) {
          const baseNode = push({
            key: `base:${d.id}:${discName}`,
            label: "BASE",
            kind: "base",
            path: [...discPath, "BASE"],
            drawingId: d.id,
            discipline: discName,
            imageFile: discObj.image,
            imageTransform: discObj.imageTransform,
          });
          discNode.children!.push(baseNode);
        }

        for (const [regionName, regionObj] of Object.entries(discObj.regions)) {
          const regionPath = [...discPath, `Region ${regionName}`];
          const regionNode = push({
            key: `region:${d.id}:${discName}:${regionName}`,
            label: `Region ${regionName}`,
            kind: "region",
            path: regionPath,
            drawingId: d.id,
            discipline: discName,
            region: regionName,
            children: [],
          });

          for (const rev of regionObj.revisions) {
            const rn = push({
              key: `rev:${d.id}:${discName}:${regionName}:${rev.version}`,
              label: rev.version,
              kind: "revision",
              path: [...regionPath, rev.version],
              drawingId: d.id,
              discipline: discName,
              region: regionName,
              imageFile: rev.image,
              imageTransform: rev.imageTransform ?? discObj.imageTransform,
              rev,
            });
            regionNode.children!.push(rn);
          }

          discNode.children!.push(regionNode);
        }
      }

      if ((discNode.children?.length ?? 0) === 0 && !discNode.imageFile) {
      }

      drawingNode.children!.push(discNode);
    }

    const kids = (childrenMap.get(d.id) ?? []).sort((a, b) => a.id.localeCompare(b.id));
    for (const child of kids) {
      drawingNode.children!.push(walkDrawing(child, drawingPath));
    }

    return drawingNode;
  }

  const projectName = m.project?.name ?? "Project";

  const projectNode: TreeNodeData = push({
    key: "project",
    label: projectName,
    kind: "project",
    path: [projectName],
    children: [],
  });

  for (const r of roots.sort((a, b) => a.id.localeCompare(b.id))) {
    projectNode.children!.push(walkDrawing(r, [projectName]));
  }

  return { tree: [projectNode], flat };
}

function onSelect(key: string) {
  selectedKey.value = key;
  overlayKey.value = "";
  resetOverlayTuning();
}

const baseDrawingId = computed(() => selectedNode.value?.drawingId ?? null);

const overlayCandidates = computed(() => {
  const base = selectedNode.value;
  if (!base || !base.drawingId) return [];
  return allNodes.value.filter((n) => {
    if (!n.imageFile) return false;
    if (n.key === base.key) return false;
    return n.drawingId === base.drawingId && (n.kind === "discipline" || n.kind === "revision" || n.kind === "base");
  });
});

const overlayNode = computed(() => {
  if (!overlayKey.value) return null;
  return overlayCandidates.value.find((n) => n.key === overlayKey.value) ?? null;
});

const baseImageUrl = computed(() => {
  const n = selectedNode.value;
  if (!n?.imageFile) return "";
  return `/data/drawings/${encodeURIComponent(n.imageFile)}`;
});

const overlayImageUrl = computed(() => {
  const n = overlayNode.value;
  if (!n?.imageFile) return "";
  return `/data/drawings/${encodeURIComponent(n.imageFile)}`;
});

const viewportRef = ref<HTMLElement | null>(null);
const baseNatural = ref({ w: 0, h: 0 });
const overlayNatural = ref({ w: 0, h: 0 });

function onBaseLoad(ev: Event) {
  const img = ev.target as HTMLImageElement;
  baseNatural.value = { w: img.naturalWidth, h: img.naturalHeight };
}

function onOverlayLoad(ev: Event) {
  const img = ev.target as HTMLImageElement;
  overlayNatural.value = { w: img.naturalWidth, h: img.naturalHeight };
}

function radToDeg(r: number) {
  return (r * 180) / Math.PI;
}
function getTransform(n: TreeNodeData | null): ImageTransform | null {
  if (!n?.imageTransform) return null;
  const t = n.imageTransform;
  if (typeof t.x !== "number" || typeof t.y !== "number") return null;
  return t;
}

const overlayStyle = computed(() => {

  const base = selectedNode.value;
  const over = overlayNode.value;
  if (!compareEnabled.value || !base || !over) {
    return { opacity: "0" };
  }

  const v = viewportRef.value;
  const bw = baseNatural.value.w;
  const bh = baseNatural.value.h;
  if (!v || bw <= 0 || bh <= 0) {
    return { opacity: String(overlayOpacity.value / 100) };
  }

  const cw = v.clientWidth;
  const ch = v.clientHeight;

  const displayScale = Math.min(cw / bw, ch / bh);

  const bt = getTransform(base);
  const ot = getTransform(over);

  const baseAnchor = bt ? { x: bt.x, y: bt.y, scale: bt.scale, rot: bt.rotation } : { x: bw / 2, y: bh / 2, scale: 1, rot: 0 };
  const overAnchor = ot ? { x: ot.x, y: ot.y, scale: ot.scale, rot: ot.rotation } : { x: bw / 2, y: bh / 2, scale: 1, rot: 0 };

  const dx = (overAnchor.x - baseAnchor.x) * displayScale + overlayDx.value;
  const dy = (overAnchor.y - baseAnchor.y) * displayScale + overlayDy.value;

  const scaleRatio = (overAnchor.scale / baseAnchor.scale) * (overlayScalePct.value / 100);
  const rotDiffDeg = radToDeg(overAnchor.rot - baseAnchor.rot) + overlayRotDeg.value;

  return {
    opacity: String(overlayOpacity.value / 100),
    transformOrigin: "0 0",
    transform: `translate(${dx}px, ${dy}px) scale(${scaleRatio}) rotate(${rotDiffDeg}deg)`,
  } as Record<string, string>;
});

onMounted(() => {
  loadMeta();
});
</script>

<style scoped>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}

.topbar {
  padding: 14px 16px;
  border-bottom: 1px solid #ddd;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.topbar h1 {
  margin: 0;
  font-size: 18px;
}

.sub {
  font-size: 12px;
  color: #666;
  margin-top: 2px;
}

.actions {
  display: flex;
  gap: 8px;
}

.btn {
  border: 1px solid #cfcfcf;
  background: #fff;
  padding: 8px 12px;
  border-radius: 10px;
  cursor: pointer;
}
.btn.small {
  padding: 6px 10px;
  border-radius: 10px;
}

.layout {
  flex: 1;
  display: grid;
  grid-template-columns: 380px 1fr;
  min-height: 0;
}

.left,
.right {
  min-height: 0;
  padding: 12px;
}

.left {
  border-right: 1px solid #ddd;
}

.panel-title {
  font-weight: 700;
  margin-bottom: 10px;
}

.tree {
  overflow: auto;
  height: calc(100vh - 120px);
  padding-right: 6px;
}

.viewer-wrap {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 0;
}

.breadcrumb {
  padding: 10px 12px;
  border: 1px solid #ddd;
  border-radius: 12px;
  font-size: 12px;
  color: #444;
  background: #fff;
}

.crumb {
  white-space: nowrap;
}
.sep {
  margin: 0 6px;
  color: #aaa;
}

.modebar {
  border: 1px solid #ddd;
  border-radius: 12px;
  background: #fff;
  padding: 10px 12px;
}

.chk {
  display: inline-flex;
  gap: 8px;
  align-items: center;
  font-size: 13px;
}

.compare-controls {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}
.row.sliders {
  grid-template-columns: 1fr 1fr 1fr 1fr 1fr auto;
  align-items: end;
}

.field .label {
  font-size: 12px;
  color: #666;
  margin-bottom: 6px;
}
.field .value {
  font-size: 12px;
  color: #333;
}

.select {
  width: 100%;
  border: 1px solid #cfcfcf;
  border-radius: 10px;
  padding: 8px 10px;
  background: #fff;
}

.card {
  border: 1px solid #ddd;
  border-radius: 12px;
  background: #fff;
}

.info {
  padding: 10px 12px;
}

.info-grid {
  display: grid;
  grid-template-columns: 110px 1fr;
  gap: 8px 10px;
  font-size: 12px;
}

.k {
  color: #666;
}
.v {
  color: #222;
}

.changes {
  margin: 0;
  padding-left: 16px;
}

.viewport {
  flex: 1;
  min-height: 0;
  padding: 10px;
}

.canvas {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 420px;
  overflow: hidden;
}

.img {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  user-select: none;
  pointer-events: none;
}

.img.overlay {
    /* transform은 computed에서 */
}

.muted {
  color: #777;
  font-size: 13px;
}

.error {
  color: #b00020;
  font-size: 13px;
}

.hint {
  font-size: 12px;
}
</style>
