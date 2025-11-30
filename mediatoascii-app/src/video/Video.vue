<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from "vue";

import { invoke } from "@tauri-apps/api/core";
import { listen, type UnlistenFn } from "@tauri-apps/api/event";
import { open as openDialog } from "@tauri-apps/plugin-dialog";
import { openPath as openExternally } from "@tauri-apps/plugin-opener";
import { defaultVideoConfig, type VideoConfig } from "./video";

const STORAGE_KEY = "mediatoascii:video-config";

const config = ref<VideoConfig>(loadPersistedConfig());
const processError = ref<string | null>(null);
const processing = ref(false);
const progress = ref(0);
const status = ref<"idle" | "processing" | "done" | "error">("idle");
const statusDetail = ref("Waiting to start");
const outputPath = ref<string | null>(null);
const startedAt = ref<number | null>(null);
const progressListener = ref<UnlistenFn | null>(null);
const progressWatcherRunning = ref(false);

function loadPersistedConfig(): VideoConfig {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      return { ...defaultVideoConfig(), ...JSON.parse(saved) };
    } catch (_) {
      return defaultVideoConfig();
    }
  }
  return defaultVideoConfig();
}

watch(
  config,
  (value) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(value));
  },
  { deep: true }
);

onMounted(async () => {
  await attachProgressListener();
});

onUnmounted(() => {
  if (progressListener.value) {
    progressListener.value();
  }
});

async function attachProgressListener() {
  if (progressListener.value) return;
  progressListener.value = await listen<number>("video-progress", (event) => {
    progress.value = Math.min(100, Math.floor(event.payload * 100));
    if (progress.value >= 100 && status.value === "processing") {
      statusDetail.value = "Finishing up...";
    }
  });
}

const validationErrors = computed(() => {
  const errors: string[] = [];
  if (!config.value.video_path.trim()) {
    errors.push("Input video is required.");
  }
  if (
    !config.value.output_video_path ||
    config.value.output_video_path.trim().length === 0
  ) {
    errors.push("Output video path is required (MP4).");
  }
  if (config.value.scale_down <= 0) {
    errors.push("Scale down must be greater than 0.");
  }
  if (config.value.font_size <= 0) {
    errors.push("Font size must be greater than 0.");
  }
  if (config.value.height_sample_scale <= 0) {
    errors.push("Height sample scale must be greater than 0.");
  }
  if (config.value.max_fps <= 0) {
    errors.push("Max FPS must be greater than 0.");
  }
  if (config.value.rotate !== -1 && ![0, 1, 2].includes(config.value.rotate)) {
    errors.push(
      "Rotate must be -1 (no rotate), 0 (90 CW), 1 (180), or 2 (90 CCW)."
    );
  }
  return errors;
});

const canSubmit = computed(
  () => !processing.value && validationErrors.value.length === 0
);

async function pickVideoFile() {
  const selected = await openDialog({
    multiple: false,
    filters: [{ name: "Videos", extensions: ["mp4", "mov", "avi", "mkv"] }],
  });
  if (typeof selected === "string") {
    config.value.video_path = selected;
  }
}

async function pickOutputFile() {
  const selected = await openDialog({
    directory: true,
    multiple: false,
    defaultPath: config.value.output_video_path
      ? config.value.output_video_path.substring(
          0,
          config.value.output_video_path.lastIndexOf("/")
        )
      : undefined,
  });
  if (typeof selected === "string") {
    // Check if path ends with separator, if not add it
    const separator = selected.includes("\\") ? "\\" : "/";
    const path = selected.endsWith(separator) ? selected : selected + separator;
    config.value.output_video_path = path + "output.mp4";
  }
}

async function openOutput() {
  if (!outputPath.value) return;
  try {
    await openExternally(outputPath.value);
  } catch (err) {
    console.error("Could not open output file", err);
  }
}

function resetState() {
  processError.value = null;
  status.value = "idle";
  statusDetail.value = "Waiting to start";
  progress.value = 0;
  outputPath.value = null;
  startedAt.value = null;
}

async function processVideo() {
  processError.value = null;
  status.value = "processing";
  statusDetail.value = "Preparing video...";
  progress.value = 0;
  outputPath.value = null;
  startedAt.value = Date.now();

  const payload: VideoConfig = {
    ...config.value,
    output_video_path: config.value.output_video_path?.trim() || null,
  };

  processing.value = true;
  try {
    if (!progressWatcherRunning.value) {
      progressWatcherRunning.value = true;
      invoke("video_progress").finally(() => {
        progressWatcherRunning.value = false;
      });
    }
    await invoke("process_video", { config: payload });
    progress.value = 100;
    status.value = "done";
    statusDetail.value = "Finished writing output";
    outputPath.value = payload.output_video_path ?? null;
  } catch (error: any) {
    processError.value = error?.message || String(error);
    status.value = "error";
    statusDetail.value = "Something went wrong";
  } finally {
    processing.value = false;
  }
}

const elapsedText = computed(() => {
  if (!startedAt.value) return null;
  const ms = Date.now() - startedAt.value;
  const seconds = Math.floor(ms / 1000);
  if (seconds < 60) return `${seconds}s elapsed`;
  const minutes = Math.floor(seconds / 60);
  return `${minutes}m ${seconds % 60}s elapsed`;
});
</script>

<template>
  <div class="space-y-8 animate-in fade-in duration-500">
    <!-- Header Section -->
    <div class="flex flex-col gap-2">
      <h2 class="text-2xl font-bold text-white tracking-tight neon-text">
        Video Processing
      </h2>
      <p class="text-slate-400">
        Transform video clips into high-framerate ASCII art.
      </p>
    </div>

    <div class="grid gap-6 lg:grid-cols-2">
      <!-- Input/Output Section -->
      <div class="glass-panel rounded-2xl p-6 space-y-6">
        <div class="flex items-center gap-2 mb-4">
          <div
            class="h-1 w-8 bg-cyan-500 rounded-full shadow-[0_0_10px_rgba(6,182,212,0.5)]"
          ></div>
          <h3 class="text-sm font-bold uppercase tracking-widest text-cyan-400">
            Source & Destination
          </h3>
        </div>

        <!-- Input -->
        <div class="space-y-2">
          <div class="flex justify-between items-center">
            <label
              class="text-xs font-semibold text-slate-300 uppercase tracking-wider"
              >Input Video</label
            >
            <button
              type="button"
              class="text-xs text-cyan-400 hover:text-cyan-300 transition-colors"
              @click="pickVideoFile"
              :disabled="processing"
            >
              BROWSE FILES
            </button>
          </div>
          <div class="relative group">
            <input
              v-model="config.video_path"
              :disabled="processing"
              placeholder="Select video..."
              class="glass-input w-full rounded-lg px-4 py-3 text-sm placeholder-slate-500"
            />
            <div
              class="absolute inset-0 rounded-lg ring-1 ring-white/10 pointer-events-none group-hover:ring-white/20 transition-all"
            ></div>
          </div>
        </div>

        <!-- Output -->
        <div class="space-y-2">
          <div class="flex justify-between items-center">
            <label
              class="text-xs font-semibold text-slate-300 uppercase tracking-wider"
              >Output Path</label
            >
            <button
              type="button"
              class="text-xs text-cyan-400 hover:text-cyan-300 transition-colors"
              @click="pickOutputFile"
              :disabled="processing"
            >
              CHOOSE FOLDER
            </button>
          </div>
          <div class="relative group">
            <input
              v-model="config.output_video_path"
              :disabled="processing"
              placeholder="/path/to/output.mp4"
              class="glass-input w-full rounded-lg px-4 py-3 text-sm placeholder-slate-500"
            />
            <div
              class="absolute inset-0 rounded-lg ring-1 ring-white/10 pointer-events-none group-hover:ring-white/20 transition-all"
            ></div>
          </div>
        </div>

        <!-- Options -->
        <div class="flex flex-wrap gap-4 pt-2">
          <label class="flex items-center gap-3 cursor-pointer group">
            <div class="relative flex items-center">
              <input
                type="checkbox"
                v-model="config.overwrite"
                :disabled="processing"
                class="peer sr-only"
              />
              <div
                class="w-9 h-5 bg-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-cyan-600"
              ></div>
            </div>
            <span
              class="text-sm text-slate-300 group-hover:text-white transition-colors"
              >Overwrite existing</span
            >
          </label>
          <label class="flex items-center gap-3 cursor-pointer group">
            <div class="relative flex items-center">
              <input
                type="checkbox"
                v-model="config.use_max_fps_for_output_video"
                :disabled="processing"
                class="peer sr-only"
              />
              <div
                class="w-9 h-5 bg-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-cyan-600"
              ></div>
            </div>
            <span
              class="text-sm text-slate-300 group-hover:text-white transition-colors"
              >Force Max FPS</span
            >
          </label>
        </div>
      </div>

      <!-- Settings Section -->
      <div class="glass-panel rounded-2xl p-6 space-y-6">
        <div class="flex items-center gap-2 mb-4">
          <div
            class="h-1 w-8 bg-purple-500 rounded-full shadow-[0_0_10px_rgba(168,85,247,0.5)]"
          ></div>
          <h3
            class="text-sm font-bold uppercase tracking-widest text-purple-400"
          >
            Configuration
          </h3>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <label class="space-y-2">
            <span class="text-xs font-semibold text-slate-400">Scale Down</span>
            <input
              type="number"
              step="0.1"
              min="0.1"
              v-model.number="config.scale_down"
              :disabled="processing"
              class="glass-input w-full rounded-lg px-3 py-2 text-sm"
            />
          </label>
          <label class="space-y-2">
            <span class="text-xs font-semibold text-slate-400">Font Size</span>
            <input
              type="number"
              step="0.5"
              min="1"
              v-model.number="config.font_size"
              :disabled="processing"
              class="glass-input w-full rounded-lg px-3 py-2 text-sm"
            />
          </label>
          <label class="space-y-2">
            <span class="text-xs font-semibold text-slate-400"
              >Height Sample</span
            >
            <input
              type="number"
              step="0.01"
              min="0.1"
              v-model.number="config.height_sample_scale"
              :disabled="processing"
              class="glass-input w-full rounded-lg px-3 py-2 text-sm"
            />
          </label>
          <label class="space-y-2">
            <span class="text-xs font-semibold text-slate-400">Max FPS</span>
            <input
              type="number"
              step="1"
              min="1"
              v-model.number="config.max_fps"
              :disabled="processing"
              class="glass-input w-full rounded-lg px-3 py-2 text-sm"
            />
          </label>
        </div>

        <div class="pt-2">
          <label class="space-y-2 block">
            <span class="text-xs font-semibold text-slate-400">Rotation</span>
            <select
              v-model.number="config.rotate"
              :disabled="processing"
              class="glass-input w-full rounded-lg px-3 py-2 text-sm appearance-none cursor-pointer"
            >
              <option :value="-1" class="bg-slate-900">No rotation</option>
              <option :value="0" class="bg-slate-900">90° Clockwise</option>
              <option :value="1" class="bg-slate-900">180°</option>
              <option :value="2" class="bg-slate-900">
                90° Counter-Clockwise
              </option>
            </select>
          </label>
        </div>

        <label class="flex items-center gap-3 cursor-pointer group pt-2">
          <div class="relative flex items-center">
            <input
              type="checkbox"
              v-model="config.invert"
              :disabled="processing"
              class="peer sr-only"
            />
            <div
              class="w-9 h-5 bg-slate-700 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-purple-600"
            ></div>
          </div>
          <span
            class="text-sm text-slate-300 group-hover:text-white transition-colors"
            >Invert Colors</span
          >
        </label>
      </div>
    </div>

    <!-- Status & Actions -->
    <div class="glass-panel rounded-2xl p-6 border-t-2 border-t-cyan-500/20">
      <div class="flex flex-col md:flex-row gap-6 items-center justify-between">
        <div class="w-full md:w-2/3 space-y-3">
          <div class="flex justify-between items-end">
            <div>
              <h4 class="text-sm font-bold text-white uppercase tracking-wider">
                {{
                  status === "processing"
                    ? "RENDERING..."
                    : status === "done"
                    ? "COMPLETE"
                    : status === "error"
                    ? "FAILED"
                    : "READY"
                }}
              </h4>
              <p class="text-xs text-slate-400 mt-1 font-mono">
                {{ statusDetail }}
              </p>
            </div>
            <span v-if="elapsedText" class="text-xs font-mono text-cyan-400">{{
              elapsedText
            }}</span>
          </div>

          <div class="relative h-2 bg-slate-800 rounded-full overflow-hidden">
            <div
              class="absolute top-0 left-0 h-full bg-gradient-to-r from-cyan-500 to-purple-500 transition-all duration-300 ease-out shadow-[0_0_10px_rgba(6,182,212,0.5)]"
              :style="{ width: `${progress}%` }"
            ></div>
          </div>

          <div
            v-if="processError"
            class="text-xs text-red-400 bg-red-900/20 border border-red-900/50 p-2 rounded"
          >
            {{ processError }}
          </div>

          <div
            v-if="outputPath && status === 'done'"
            class="flex items-center gap-3 text-xs bg-emerald-900/20 border border-emerald-900/50 p-2 rounded text-emerald-300"
          >
            <span class="font-semibold">SAVED:</span>
            <span class="truncate flex-1 font-mono opacity-80">{{
              outputPath
            }}</span>
            <button
              @click="openOutput"
              class="hover:text-white underline decoration-emerald-500/50 hover:decoration-emerald-500"
            >
              OPEN
            </button>
          </div>

          <div
            v-if="validationErrors.length"
            class="text-xs text-amber-400 bg-amber-900/20 border border-amber-900/50 p-2 rounded"
          >
            <ul class="list-disc pl-4 space-y-0.5">
              <li v-for="err in validationErrors" :key="err">{{ err }}</li>
            </ul>
          </div>
        </div>

        <div class="flex gap-3 w-full md:w-auto">
          <button
            type="button"
            class="btn-secondary px-6 py-3 rounded-lg font-bold text-sm tracking-wide"
            @click="resetState"
            :disabled="processing"
          >
            RESET
          </button>
          <button
            type="button"
            class="btn-primary flex-1 md:flex-none px-8 py-3 rounded-lg shadow-lg shadow-cyan-500/20 disabled:opacity-50 disabled:cursor-not-allowed"
            :disabled="!canSubmit"
            @click="processVideo"
          >
            {{ processing ? "PROCESSING..." : "INITIATE" }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
