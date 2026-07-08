<script setup>
import { ref, computed, watch } from "vue";

// ─── 音楽理論データ ───────────────────────────────────────────
const NOTES = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];

// 楽器定義
//  - strings: 選択可能な弦数の範囲
//  - plainStrings: プレーン弦の本数(上=高音側から)
//  - tunings: 弦数ごとのチューニング。並びは表示上の順(index0=1弦=最高音 → 最終弦=最低音)。
//    値は音名の半音インデックス(0=C)。
//    ギター: 標準6弦に対し拡張弦は低音側へ完全4度ずつ追加(…E B → F# → C#)
//    ベース: 4弦=EADG、5弦=+低B、6弦=+低B & 高C
const INSTRUMENTS = {
  guitar: {
    label: "ギター",
    plainStrings: 3,
    strings: { min: 6, max: 9 },
    tunings: {
      6: [4, 11, 7, 2, 9, 4],              // E B G D A E
      7: [4, 11, 7, 2, 9, 4, 11],          // + 低B
      8: [4, 11, 7, 2, 9, 4, 11, 6],       // + 低F#
      9: [4, 11, 7, 2, 9, 4, 11, 6, 1],    // + 低C#
    },
  },
  bass: {
    label: "ベース",
    plainStrings: 0,
    strings: { min: 4, max: 6 },
    tunings: {
      4: [7, 2, 9, 4],                     // G D A E
      5: [7, 2, 9, 4, 11],                 // + 低B
      6: [0, 7, 2, 9, 4, 11],              // 高C + G D A E + 低B
    },
  },
};

// スケール定義 (カテゴリ別)
const SCALE_GROUPS = {
  "基本": {
    "メジャー":             { iv: [0,2,4,5,7,9,11], deg: ["R","2","3","4","5","6","7"] },
    "ナチュラルマイナー":   { iv: [0,2,3,5,7,8,10], deg: ["R","2","♭3","4","5","♭6","♭7"] },
    "ハーモニックマイナー": { iv: [0,2,3,5,7,8,11], deg: ["R","2","♭3","4","5","♭6","7"] },
    "メロディックマイナー": { iv: [0,2,3,5,7,9,11], deg: ["R","2","♭3","4","5","6","7"] },
    "ハーモニックメジャー": { iv: [0,2,4,5,7,8,11], deg: ["R","2","3","4","5","♭6","7"] },
  },
  "ペンタトニック / ブルース": {
    "メジャーペンタ":   { iv: [0,2,4,7,9],      deg: ["R","2","3","5","6"] },
    "マイナーペンタ":   { iv: [0,3,5,7,10],     deg: ["R","♭3","4","5","♭7"] },
    "マイナーブルース": { iv: [0,3,5,6,7,10],   deg: ["R","♭3","4","♭5","5","♭7"] },
    "メジャーブルース": { iv: [0,2,3,4,7,9],    deg: ["R","2","♭3","3","5","6"] },
  },
  "チャーチモード": {
    "ドリアン":         { iv: [0,2,3,5,7,9,10],  deg: ["R","2","♭3","4","5","6","♭7"] },
    "フリジアン":       { iv: [0,1,3,5,7,8,10],  deg: ["R","♭2","♭3","4","5","♭6","♭7"] },
    "リディアン":       { iv: [0,2,4,6,7,9,11],  deg: ["R","2","3","#4","5","6","7"] },
    "ミクソリディアン": { iv: [0,2,4,5,7,9,10],  deg: ["R","2","3","4","5","6","♭7"] },
    "ロクリアン":       { iv: [0,1,3,5,6,8,10],  deg: ["R","♭2","♭3","4","♭5","♭6","♭7"] },
  },
  "メロディックマイナー系 (ジャズ)": {
    "オルタード":             { iv: [0,1,3,4,6,8,10],  deg: ["R","♭9","#9","3","♭5","#5","♭7"] },
    "リディアン7th":          { iv: [0,2,4,6,7,9,10],  deg: ["R","2","3","#4","5","6","♭7"] },
    "リディアンオーグメント": { iv: [0,2,4,6,8,9,11],  deg: ["R","2","3","#4","#5","6","7"] },
    "ミクソリディアン♭6":     { iv: [0,2,4,5,7,8,10],  deg: ["R","2","3","4","5","♭6","♭7"] },
    "ドリアン♭2":             { iv: [0,1,3,5,7,9,10],  deg: ["R","♭2","♭3","4","5","6","♭7"] },
    "ロクリアン♮2":           { iv: [0,2,3,5,6,8,10],  deg: ["R","2","♭3","4","♭5","♭6","♭7"] },
  },
  "ハーモニックマイナー系": {
    "フリジアンドミナント":   { iv: [0,1,4,5,7,8,10],  deg: ["R","♭2","3","4","5","♭6","♭7"] },
    "ハンガリアンマイナー":   { iv: [0,2,3,6,7,8,11],  deg: ["R","2","♭3","#4","5","♭6","7"] },
  },
  "シンメトリック": {
    "ホールトーン":           { iv: [0,2,4,6,8,10],      deg: ["R","2","3","#4","#5","♭7"] },
    "ディミニッシュ (W-H)":   { iv: [0,2,3,5,6,8,9,11],  deg: ["R","2","♭3","4","♭5","#5","6","7"] },
    "コンディミ (H-W)":       { iv: [0,1,3,4,6,7,9,10],  deg: ["R","♭9","#9","3","♭5","5","6","♭7"] },
  },
  "ビバップ": {
    "ビバップメジャー":     { iv: [0,2,4,5,7,8,9,11],  deg: ["R","2","3","4","5","#5","6","7"] },
    "ビバップドミナント":   { iv: [0,2,4,5,7,9,10,11], deg: ["R","2","3","4","5","6","♭7","7"] },
  },
  "和風 / 民族": {
    "沖縄 (琉球)": { iv: [0,4,5,7,11], deg: ["R","3","4","5","7"] },
    "都節":        { iv: [0,1,5,7,8],  deg: ["R","♭2","4","5","♭6"] },
    "平調子":      { iv: [0,2,3,7,8],  deg: ["R","2","♭3","5","♭6"] },
  },
};

// 名前 → 定義 のフラットなマップ (検索用)
const SCALES = Object.assign({}, ...Object.values(SCALE_GROUPS));

// コード: ベースタイプ + テンション(複数選択)
const CHORD_BASE = {
  major: { iv: [0, 4, 7], deg: ["R", "3", "5"],  label: "メジャー" },
  minor: { iv: [0, 3, 7], deg: ["R", "♭3", "5"], label: "マイナー" },
  sus2:  { iv: [0, 2, 7], deg: ["R", "2", "5"],  label: "sus2" },
  sus4:  { iv: [0, 5, 7], deg: ["R", "4", "5"],  label: "sus4" },
  dim:   { iv: [0, 3, 6], deg: ["R", "♭3", "♭5"], label: "dim" },
  aug:   { iv: [0, 4, 8], deg: ["R", "3", "#5"],  label: "aug" },
  power: { iv: [0, 7],    deg: ["R", "5"],        label: "5 (パワー)" },
};

const TENSIONS = [
  { id: "7",   semi: 10, deg: "♭7" },
  { id: "M7",  semi: 11, deg: "7" },
  { id: "6",   semi: 9,  deg: "6" },
  { id: "9",   semi: 2,  deg: "9" },
  { id: "♭9",  semi: 1,  deg: "♭9" },
  { id: "#9",  semi: 3,  deg: "#9" },
  { id: "11",  semi: 5,  deg: "11" },
  { id: "#11", semi: 6,  deg: "#11" },
  { id: "13",  semi: 9,  deg: "13" },
  { id: "♭13", semi: 8,  deg: "♭13" },
];

// 同じ度数のナチュラル/変化形は同時に成立しないため排他にする。
//  - 7th : ♭7 と M7
//  - 9th : ナチュラル9 と ♭9/#9 (♭9 と #9 の併用はオルタードで成立するため許可)
//  - 11th: ナチュラル11 と #11
//  - 13th: ナチュラル13 と ♭13
const TENSION_CONFLICTS = {
  "7":   ["M7"],
  "M7":  ["7"],
  "9":   ["♭9", "#9"],
  "♭9":  ["9"],
  "#9":  ["9"],
  "11":  ["#11"],
  "#11": ["11"],
  "13":  ["♭13"],
  "♭13": ["13"],
};

// 度数 → 色カテゴリ (ルート / 3度 / 5度 / 7度 / その他)。実際の色はCSSの .cat-* で定義。
// 3rd と 7th はコードの性質を決めるガイドトーンなので独立色を与える (7=M7 / ♭7 は同色)。
const catOf = (deg) => {
  if (deg === "R") return "root";
  if (deg === "3" || deg === "♭3") return "third";
  if (deg === "5" || deg === "♭5" || deg === "#5") return "fifth";
  if (deg === "7" || deg === "♭7") return "seventh";
  return "other";
};

// コードネーム生成 (例: Cm7(9,13), C7sus4, Cdim7, Cadd9)
//  - 7th/6th を含まずにテンションだけ足した場合は "add" 表記にする (add系コード)
function chordLabel(rootName, quality, tensions) {
  const has = (t) => tensions.includes(t);
  const seventh = has("M7") ? "M7" : has("7") ? "7" : "";
  const exts = tensions.filter((t) => !["7", "M7", "6"].includes(t));
  let core = "";

  switch (quality) {
    case "major":
      core = seventh;
      if (has("6")) { seventh ? exts.unshift("13") : (core = "6"); }
      break;
    case "minor":
      core = "m" + seventh;
      if (has("6")) { seventh ? exts.unshift("13") : (core = "m6"); }
      break;
    case "dim":
      // dim + ♭7 = ハーフディミニッシュ / dim + 6(𝄫7) = dim7
      if (has("6")) core = "dim7";
      else if (has("7")) core = "m7♭5";
      else if (has("M7")) core = "dimM7";
      else core = "dim";
      break;
    case "aug":
      core = "aug" + seventh;
      if (has("6")) exts.unshift("6");
      break;
    case "sus2":
    case "sus4":
      core = seventh + quality;
      if (has("6")) exts.unshift("6");
      break;
    case "power":
      core = "5";
      if (seventh) exts.unshift(seventh === "M7" ? "M7" : "♭7");
      if (has("6")) exts.unshift("6");
      break;
    default:
      core = seventh;
  }

  const uniq = [...new Set(exts)];
  if (!uniq.length) return rootName + core;

  // 7th か 6th があれば積み重ねの土台があるので通常のテンション(括弧内)表記。
  // どちらも無ければ追加音なので "add" 表記にする。
  const hasBackbone = seventh !== "" || has("6");
  if (hasBackbone) return rootName + core + `(${uniq.join(",")})`;

  // トライアド(メジャー/マイナー)に単一の追加音なら Cadd9 / Cmadd9 の形。
  // それ以外(複数 or sus/dim/aug/power)は括弧で addX を並べる。
  if (uniq.length === 1 && (core === "" || core === "m")) {
    return rootName + core + "add" + uniq[0];
  }
  return rootName + core + `(${uniq.map((e) => "add" + e).join(",")})`;
}

// ─── フレットボード描画定数 ─────────────────────────────────
const FRETS = 24;
const OPEN_W = 52;        // 開放弦エリア幅
const NUT_X = OPEN_W + 8; // ナット位置
const FRET_W = 62;
const STR_GAP = 36;
const TOP = 34;
const SVG_W = NUT_X + FRETS * FRET_W + 16;
const INLAYS = [3, 5, 7, 9, 12, 15, 17, 19, 21, 24];
const DOUBLE_INLAYS = [12, 24]; // 上下2点マーク

// ─── リアクティブ状態 ────────────────────────────────────────
const mode = ref("scale");          // "scale" | "chord"
const root = ref(0);                // 0 = C (スケール時はキー)
const scaleName = ref("メジャー");
const quality = ref("major");       // コードのベース
const tensions = ref([]);           // 選択中テンション
const labelMode = ref("note");      // "note" | "degree"
const instrument = ref("guitar");
const strings = ref(6);             // 弦数

const inst = computed(() => INSTRUMENTS[instrument.value]);
// 選択可能な弦数(min〜max)
const stringOptions = computed(() => {
  const { min, max } = inst.value.strings;
  const out = [];
  for (let n = min; n <= max; n++) out.push(n);
  return out;
});
// 現在のチューニング(範囲外選択時は最小弦数へフォールバック)
const tuning = computed(() => {
  const t = inst.value.tunings;
  return t[strings.value] || t[inst.value.strings.min];
});
// 低音→高音の音名表記 (例: E A D G B E)
const tuningLabel = computed(() => [...tuning.value].reverse().map((n) => NOTES[n]).join(" "));
const nStr = computed(() => tuning.value.length);
const boardH = computed(() => STR_GAP * (nStr.value - 1)); // 1弦〜最低弦の間隔
const SVG_H = computed(() => TOP + boardH.value + 46);

// 楽器を切り替えたら弦数をその楽器の既定(最小)本数へ設定 (ギター→6 / ベース→4)
watch(instrument, () => {
  strings.value = inst.value.strings.min;
});

const toggleTension = (id) => {
  const prev = tensions.value;
  if (prev.includes(id)) {
    tensions.value = prev.filter((t) => t !== id);
    return;
  }
  // 同じ度数の相反する変化形を外してから追加
  const conflicts = TENSION_CONFLICTS[id] || [];
  tensions.value = [...prev.filter((t) => !conflicts.includes(t)), id];
};

// 現在の構成音 (半音 → 度数ラベル)
const current = computed(() => {
  if (mode.value === "scale") {
    const s = SCALES[scaleName.value];
    return { iv: s.iv, deg: s.deg };
  }
  const base = CHORD_BASE[quality.value];
  const iv = [...base.iv];
  const deg = [...base.deg];
  TENSIONS.forEach((t) => {
    if (tensions.value.includes(t.id) && !iv.includes(t.semi)) {
      iv.push(t.semi);
      deg.push(t.deg);
    }
  });
  return { iv, deg };
});

const degMap = computed(() => {
  const m = {};
  current.value.iv.forEach((semi, i) => { m[semi % 12] = current.value.deg[i]; });
  return m;
});

const yOf = (s) => TOP + s * STR_GAP;
const xOfFret = (f) => (f === 0 ? OPEN_W / 2 : NUT_X + f * FRET_W - FRET_W / 2);

// プロットする点を列挙
const dots = computed(() => {
  const out = [];
  tuning.value.forEach((openNote, si) => {
    for (let f = 0; f <= FRETS; f++) {
      const semi = (openNote + f - root.value + 120) % 12;
      const deg = degMap.value[semi];
      if (deg !== undefined) {
        out.push({ x: xOfFret(f), y: yOf(si), deg, note: NOTES[(openNote + f) % 12], open: f === 0 });
      }
    }
  });
  return out;
});

const chordName = computed(() => chordLabel(NOTES[root.value], quality.value, tensions.value));

// コードネームを「ベース部分」と「テンション部分(括弧内)」に分割。
// 表示上テンションだけ小さくするため。例: "Cm7(9,13)" → { base: "Cm7", ext: "(9,13)" }
const chordNameParts = computed(() => {
  const full = chordName.value;
  const i = full.indexOf("(");
  return i === -1 ? { base: full, ext: "" } : { base: full.slice(0, i), ext: full.slice(i) };
});
</script>

<template>
  <div class="app">
    <div class="shell">
      <!-- ヘッダー: アンプヘッドのフロント -->
      <header class="amp-head">
        <div>
          <div class="brand-name">Fretglyph</div>
          <div class="brand-sub">SCALE &amp; CHORD VISUALIZER</div>
        </div>
        <div class="power">
          <span class="power-lbl">POWER</span>
          <span class="power-led" />
        </div>
      </header>

      <!-- コントロールパネル: ゴールドのブラッシュドメタル -->
      <div class="panel">
        <!-- 1段目: 楽器 + 表示パターン -->
        <div class="row mb16">
          <div>
            <div class="panel-lbl">Instrument / 楽器</div>
            <select v-model="instrument" class="sel sel-md">
              <option v-for="(def, id) in INSTRUMENTS" :key="id" :value="id">{{ def.label }}</option>
            </select>
          </div>
          <div>
            <div class="panel-lbl">Strings / 弦数</div>
            <select v-model.number="strings" class="sel">
              <option v-for="n in stringOptions" :key="n" :value="n">{{ n }}弦</option>
            </select>
          </div>
          <div class="mode-field">
            <div class="panel-lbl">Mode / 表示パターン</div>
            <div class="seg-group">
              <button
                class="seg-btn seg-btn--fill"
                :class="{ 'is-active': mode === 'scale' }"
                @click="mode = 'scale'"
              >
                スケール
              </button>
              <button
                class="seg-btn seg-btn--fill"
                :class="{ 'is-active': mode === 'chord' }"
                @click="mode = 'chord'"
              >
                コード
              </button>
            </div>
          </div>
        </div>

        <!-- 2段目: モード別UI -->
        <!-- ── スケール: キー + スケール選択 ── -->
        <div v-if="mode === 'scale'" class="row">
          <div>
            <div class="panel-lbl">Key / キー</div>
            <select v-model.number="root" class="sel">
              <option v-for="(n, i) in NOTES" :key="n" :value="i">{{ n }}</option>
            </select>
          </div>
          <div>
            <div class="panel-lbl">Scale / スケール</div>
            <select v-model="scaleName" class="sel sel-lg">
              <optgroup v-for="(scales, group) in SCALE_GROUPS" :key="group" :label="group">
                <option v-for="n in Object.keys(scales)" :key="n" :value="n">{{ n }}</option>
              </optgroup>
            </select>
          </div>
        </div>

        <!-- ── コード: コードネーム(左) + ルート/タイプ + テンション(右) ── -->
        <div v-else class="chord-grid">
          <!-- コードネーム表示: アンプの銘板風 (左側) -->
          <div class="chord-name">
            <div class="chord-name__cap">CHORD NAME</div>
            <div class="chord-name__value">{{ chordNameParts.base }}<span class="chord-name__ext">{{ chordNameParts.ext }}</span></div>
          </div>

          <!-- ルート/タイプ + テンション (右側) -->
          <div class="chord-right">
            <!-- ルート + タイプ (横並び) -->
            <div class="row">
              <div>
                <div class="panel-lbl">Root / ルート</div>
                <select v-model.number="root" class="sel">
                  <option v-for="(n, i) in NOTES" :key="n" :value="i">{{ n }}</option>
                </select>
              </div>
              <div>
                <div class="panel-lbl">Type / タイプ</div>
                <select v-model="quality" class="sel sel-md">
                  <option v-for="(def, id) in CHORD_BASE" :key="id" :value="id">{{ def.label }}</option>
                </select>
              </div>
            </div>

            <!-- テンション -->
            <div>
              <div class="panel-lbl">Tension / テンション (複数選択可)</div>
              <div class="tension-list">
                <button
                  v-for="t in TENSIONS"
                  :key="t.id"
                  class="tension-btn"
                  :class="{ 'is-on': tensions.includes(t.id) }"
                  @click="toggleTension(t.id)"
                >
                  {{ t.id }}
                </button>
                <button v-if="tensions.length > 0" class="tension-clear" @click="tensions = []">
                  クリア
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 構成音チップ + 凡例 -->
      <div class="chips">
        <div
          v-for="(semi, i) in current.iv"
          :key="i"
          class="chip"
          :class="'cat-' + catOf(current.deg[i])"
        >
          <span class="chip__badge">{{ current.deg[i] }}</span>
          <span class="chip__note">{{ NOTES[(root + semi) % 12] }}</span>
        </div>
        <div class="legend">
          <span><span class="legend-dot cat-root" /> ルート</span>
          <span><span class="legend-dot cat-third" /> 3度</span>
          <span><span class="legend-dot cat-fifth" /> 5度</span>
          <span><span class="legend-dot cat-seventh" /> 7度</span>
          <span><span class="legend-dot cat-other" /> その他</span>
        </div>
      </div>

      <!-- フレットボード直上: ラベル切替 -->
      <div class="label-bar">
        <span class="label-bar__cap">LABEL / ラベル</span>
        <div class="seg-group seg-group--gold">
          <button
            v-for="[id, t] in [['note', '音名'], ['degree', '度数']]"
            :key="id"
            class="seg-btn"
            :class="{ 'is-active': labelMode === id }"
            @click="labelMode = id"
          >
            {{ t }}
          </button>
        </div>
      </div>

      <!-- フレットボード: キャビネット風フレーム -->
      <div class="board-wrap">
        <svg :width="SVG_W" :height="SVG_H" class="board" :style="{ minWidth: SVG_W + 'px' }">
          <defs>
            <linearGradient id="wood" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stop-color="#3d2b1e" />
              <stop offset="50%" stop-color="#332317" />
              <stop offset="100%" stop-color="#291b11" />
            </linearGradient>
          </defs>

          <!-- 指板 -->
          <rect
            :x="NUT_X" :y="TOP - 20"
            :width="FRETS * FRET_W" :height="boardH + 40"
            :rx="4" fill="url(#wood)"
          />
          <!-- 木目 -->
          <line
            v-for="n in 7"
            :key="'grain-' + n"
            :x1="NUT_X" :x2="NUT_X + FRETS * FRET_W"
            :y1="TOP - 14 + (n - 1) * ((boardH + 28) / 6)"
            :y2="TOP - 17 + (n - 1) * ((boardH + 30) / 6)"
            stroke="#00000030" :stroke-width="1.4"
          />

          <!-- ポジションマーク -->
          <template v-for="f in INLAYS" :key="'inlay-' + f">
            <g v-if="DOUBLE_INLAYS.includes(f)">
              <circle :cx="NUT_X + f * FRET_W - FRET_W / 2" :cy="TOP + boardH * 0.22" :r="7" fill="#d8cfc0" :opacity="0.5" />
              <circle :cx="NUT_X + f * FRET_W - FRET_W / 2" :cy="TOP + boardH * 0.78" :r="7" fill="#d8cfc0" :opacity="0.5" />
            </g>
            <circle v-else :cx="NUT_X + f * FRET_W - FRET_W / 2" :cy="TOP + boardH / 2" :r="7" fill="#d8cfc0" :opacity="0.4" />
          </template>

          <!-- ナット -->
          <rect :x="NUT_X - 7" :y="TOP - 20" :width="7" :height="boardH + 40" :rx="2" fill="#e8e0d0" />

          <!-- フレットワイヤー -->
          <line
            v-for="n in FRETS"
            :key="'fret-' + n"
            :x1="NUT_X + n * FRET_W" :x2="NUT_X + n * FRET_W"
            :y1="TOP - 20" :y2="TOP + boardH + 20"
            stroke="#9aa3ad" :stroke-width="2.5"
          />

          <!-- 弦 (低音弦ほど太く) -->
          <line
            v-for="(openNote, i) in tuning"
            :key="'string-' + i"
            :x1="8" :x2="NUT_X + FRETS * FRET_W"
            :y1="yOf(i)" :y2="yOf(i)"
            :stroke="i < inst.plainStrings ? '#c8ccd4' : '#b0a488'"
            :stroke-width="instrument === 'guitar' ? 0.8 + i * 0.45 : 1.8 + i * 0.55"
          />

          <!-- 開放弦名 -->
          <text
            v-for="(openNote, i) in tuning"
            :key="'openlabel-' + i"
            :x="SVG_W - 4" :y="yOf(i) + 4"
            :font-size="10" fill="#8a7d5e" text-anchor="end"
            font-family="ui-monospace, monospace"
          >
            {{ NOTES[openNote] }}
          </text>

          <!-- フレット番号 -->
          <text
            v-for="n in FRETS"
            :key="'fretnum-' + n"
            :x="NUT_X + n * FRET_W - FRET_W / 2"
            :y="SVG_H - 8"
            :font-size="12"
            :fill="INLAYS.includes(n) ? '#d4af4e' : '#8a7d5e'"
            :font-weight="INLAYS.includes(n) ? 700 : 400"
            text-anchor="middle"
            font-family="ui-monospace, monospace"
          >
            {{ n }}
          </text>
          <text :x="OPEN_W / 2" :y="SVG_H - 8" :font-size="11" fill="#8a7d5e" text-anchor="middle" font-family="ui-monospace, monospace">
            開放
          </text>

          <!-- ノート (色は .cat-* クラスで指定) -->
          <g
            v-for="(d, i) in dots"
            :key="'dot-' + i"
            class="note"
            :class="['cat-' + catOf(d.deg), { 'note--open': d.open }]"
          >
            <circle :cx="d.x" :cy="d.y" :r="13.5" />
            <text
              :x="d.x" :y="d.y + 4"
              :font-size="(labelMode === 'note' ? d.note : d.deg).length > 1 ? 9.5 : 11"
            >
              {{ labelMode === 'note' ? d.note : d.deg }}
            </text>
          </g>
        </svg>
      </div>

      <p class="caption">
        {{ inst.label }} {{ strings }}弦 ・ チューニング: {{ tuningLabel }} ・ 上段が1弦(高音側) ・ 横スクロールで24フレットまで表示
      </p>

      <footer class="copyright">© 2026 Studio Jin Doe</footer>
    </div>
  </div>
</template>

<style scoped>
/* ─── デザイントークン (Marshall風) ─────────────────────────── */
.app {
  --gold-panel: linear-gradient(180deg, #e9d283 0%, #d4af4e 38%, #b8912e 52%, #cfa943 70%, #e2c368 100%);
  --brushed: repeating-linear-gradient(90deg, rgba(255,255,255,0.10) 0px, rgba(255,255,255,0.10) 1px, rgba(0,0,0,0.04) 1px, rgba(0,0,0,0.04) 2px);
  --tolex: radial-gradient(rgba(255,255,255,0.028) 1px, transparent 1px), radial-gradient(rgba(0,0,0,0.5) 1px, transparent 1px), linear-gradient(180deg, #17140f, #100e0a);
  --panel-text: #1c1405;
  --piping: #e9e2ce;

  min-height: 100vh;
  background: var(--tolex);
  background-size: 5px 5px, 5px 5px, 100% 100%;
  background-position: 0 0, 2.5px 2.5px, 0 0;
  color: #e9e2ce;
  font-family: 'Hiragino Kaku Gothic ProN', 'Yu Gothic', system-ui, sans-serif;
  padding: 28px 20px 48px;
}

/* ボード自然幅(約1564px)＋枠を収められる幅。広い画面では全24フレットが表示され、
   狭い画面では .board-wrap が横スクロールになる(フレットのサイズ感は不変) */
.shell { max-width: 1600px; margin: 0 auto; }

/* ─── ヘッダー ───────────────────────────────────────────────── */
.amp-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  border: 2px solid var(--piping);
  border-radius: 10px;
  padding: 18px 24px;
  margin-bottom: 20px;
  background: linear-gradient(180deg, #1a160f, #0f0d09);
  box-shadow: 0 4px 16px rgba(0,0,0,0.6), inset 0 1px 0 rgba(255,255,255,0.06);
}
.brand-name {
  font-family: 'Brush Script MT', 'Segoe Script', 'Snell Roundhand', cursive;
  font-style: italic;
  font-size: 40px;
  line-height: 1.1;
  color: #f5f0e0;
  text-shadow: 0 2px 6px rgba(0,0,0,0.8);
}
.brand-sub { font-size: 10px; letter-spacing: 0.34em; color: #c9a02e; font-weight: 700; margin-top: 2px; }
.power { display: flex; align-items: center; gap: 8px; }
.power-lbl { font-size: 9px; letter-spacing: 0.22em; color: #7d7461; font-weight: 700; }
.power-led {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: radial-gradient(circle at 35% 30%, #ff8a8a, #e01818 60%, #7a0000);
  box-shadow: 0 0 10px #ff2a2a, 0 0 22px rgba(255,42,42,0.5);
  border: 1px solid #3a0000;
}

/* ─── コントロールパネル ─────────────────────────────────────── */
.panel {
  background: var(--brushed), var(--gold-panel);
  border: 1px solid #7a5d14;
  border-radius: 10px;
  padding: 18px 20px;
  margin-bottom: 18px;
  box-shadow: 0 3px 12px rgba(0,0,0,0.55), inset 0 1px 0 rgba(255,255,255,0.5), inset 0 -2px 6px rgba(0,0,0,0.25);
}

.row { display: flex; flex-wrap: wrap; gap: 14px; align-items: flex-end; }
/* 1段目はコントロールの高さが異なる(トグルが背高)。各フィールドを縦フレックスにし
   セレクトを一番背の高いコントロールまで伸ばして、タイトル(上端)とコントロール(下端)を揃える */
.row.mb16 { margin-bottom: 16px; align-items: stretch; }
.row.mb16 > div { display: flex; flex-direction: column; }
.row.mb16 > div > .sel,
.row.mb16 > div > .seg-group { flex: 1; }
.mode-field { min-width: 200px; }

.panel-lbl {
  font-size: 10px;
  letter-spacing: 0.18em;
  color: var(--panel-text);
  font-weight: 800;
  margin-bottom: 6px;
  text-transform: uppercase;
}

/* セレクトボックス */
.sel {
  background: #171207;
  color: #e9d283;
  border: 1px solid #6d5518;
  border-radius: 6px;
  padding: 9px 12px;
  font-size: 14px;
  font-weight: 600;
  outline: none;
  min-width: 110px;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.6);
}
.sel-md { min-width: 150px; }
.sel-lg { min-width: 220px; }

/* セグメントコントロール */
.seg-group {
  display: flex;
  gap: 4px;
  background: rgba(23,18,7,0.25);
  padding: 4px;
  border-radius: 8px;
  border: 1px solid rgba(0,0,0,0.25);
}
.seg-group--gold {
  background: var(--brushed), var(--gold-panel);
  border: 1px solid #7a5d14;
  box-shadow: 0 2px 8px rgba(0,0,0,0.5);
}
.seg-btn {
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: #8a6d1f;
  background: transparent;
  border: 1px solid transparent;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
}
.seg-btn--fill { flex: 1; }
.seg-btn.is-active {
  color: #f3e6b2;
  background: #171207;
  border: 1px solid #4a3a10;
  box-shadow: inset 0 2px 5px rgba(0,0,0,0.7);
}

/* コードモード: 2カラム */
.chord-grid { display: flex; flex-wrap: wrap; gap: 16px; align-items: stretch; }
.chord-name {
  display: flex;
  flex-direction: column;
  justify-content: center;
  background: linear-gradient(180deg, #14100a, #0d0b07);
  border: 1px solid #4a3a10;
  border-radius: 8px;
  padding: 10px 18px;
  text-align: center;
  box-shadow: inset 0 3px 8px rgba(0,0,0,0.8);
  min-width: 180px;
}
.chord-name__cap { font-size: 9px; letter-spacing: 0.3em; color: #8a6d1f; font-weight: 800; margin-bottom: 2px; }
.chord-name__value {
  font-size: 30px;
  font-weight: 700;
  color: #e9d283;
  font-family: ui-monospace, monospace;
  letter-spacing: 0.02em;
  text-shadow: 0 0 12px rgba(233,210,131,0.35);
}
/* テンション(括弧内)は控えめに小さく */
.chord-name__ext { font-size: 0.6em; }
.chord-right { display: flex; flex-direction: column; gap: 14px; flex: 1; min-width: 260px; }

/* テンションボタン */
.tension-list { display: flex; flex-wrap: wrap; gap: 8px; }
.tension-btn {
  padding: 8px 14px;
  font-size: 13px;
  font-weight: 700;
  font-family: ui-monospace, monospace;
  color: #5c4a14;
  background: rgba(23,18,7,0.14);
  border: 1px solid rgba(0,0,0,0.3);
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}
.tension-btn.is-on {
  color: #f3e6b2;
  background: linear-gradient(180deg, #1c160a, #0f0c06);
  border: 1px solid #4a3a10;
  box-shadow: inset 0 2px 6px rgba(0,0,0,0.8), 0 0 8px rgba(233,210,131,0.25);
}
.tension-clear {
  padding: 8px 14px;
  font-size: 12px;
  font-weight: 700;
  color: #5c4a14;
  background: transparent;
  border: 1px dashed rgba(0,0,0,0.35);
  border-radius: 6px;
  cursor: pointer;
}

/* ─── 構成音チップ + 凡例 ────────────────────────────────────── */
.chips { display: flex; flex-wrap: wrap; gap: 10px; align-items: center; margin-bottom: 16px; }
.chip {
  display: flex;
  align-items: center;
  gap: 7px;
  background: linear-gradient(180deg, #1a160f, #12100a);
  border: 1px solid var(--stroke-soft);
  border-radius: 999px;
  padding: 5px 12px 5px 6px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.5);
}
.chip__badge {
  min-width: 24px;
  height: 24px;
  border-radius: 999px;
  padding: 0 5px;
  background: var(--fill);
  color: var(--text);
  display: grid;
  place-items: center;
  font-size: 11px;
  font-weight: 700;
  font-family: ui-monospace, monospace;
}
.chip__note { font-size: 14px; font-weight: 600; color: #e9e2ce; }

.legend { margin-left: auto; font-size: 12px; color: #a89a76; display: flex; gap: 14px; }
.legend-dot {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: var(--fill);
  margin-right: 4px;
  vertical-align: -1px;
}

/* ─── ラベル切替バー ─────────────────────────────────────────── */
.label-bar { display: flex; justify-content: flex-end; align-items: center; gap: 8px; margin-bottom: 8px; }
.label-bar__cap { font-size: 10px; letter-spacing: 0.2em; color: #a89a76; font-weight: 800; }

/* ─── フレットボード枠 ───────────────────────────────────────── */
.board-wrap {
  overflow-x: auto;
  background: linear-gradient(180deg, #16130d, #0e0c08);
  border: 2px solid var(--piping);
  outline: 1px solid #7a5d14;
  outline-offset: -6px;
  border-radius: 10px;
  padding: 16px 12px;
  box-shadow: 0 6px 20px rgba(0,0,0,0.65), inset 0 1px 0 rgba(255,255,255,0.05);
}
/* フレットボードは自然サイズ固定。コンテナより広い場合は .board-wrap が横スクロール */
.board { display: block; }

.caption { font-size: 12px; color: #8a7d5e; margin-top: 12px; }

.copyright {
  margin-top: 20px;
  padding-top: 14px;
  border-top: 1px solid rgba(233,210,131,0.12);
  text-align: center;
  font-size: 11px;
  letter-spacing: 0.14em;
  color: #6f6650;
}

/* ─── 度数カラー (ルート:赤 / 3度:青 / 5度:緑 / その他:琥珀) ──── */
.cat-root  { --fill: #e5484d; --stroke: #ff7b7f; --stroke-soft: rgba(255,123,127,0.4); --text: #ffffff; }
.cat-third { --fill: #4671e8; --stroke: #82a3ff; --stroke-soft: rgba(130,163,255,0.4); --text: #ffffff; }
.cat-fifth { --fill: #2ea36b; --stroke: #63d69e; --stroke-soft: rgba(99,214,158,0.4);  --text: #ffffff; }
.cat-seventh { --fill: #8b5cf6; --stroke: #b79dff; --stroke-soft: rgba(183,157,255,0.4); --text: #ffffff; }
.cat-other { --fill: #c99a4b; --stroke: #e8c184; --stroke-soft: rgba(232,193,132,0.4); --text: #241a08; }

/* SVGノート */
.note circle { fill: var(--fill); stroke: var(--stroke); stroke-width: 1.2; }
.note.cat-root circle { stroke-width: 2.5; }
.note--open circle { opacity: 0.92; }
.note text {
  fill: var(--text);
  font-weight: 700;
  text-anchor: middle;
  font-family: ui-monospace, monospace;
}
</style>
