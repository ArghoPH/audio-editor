<script setup>
import { computed, reactive, ref } from "vue";
import { parseBlob } from "music-metadata";
import { ID3Writer } from "browser-id3-writer";

const audioFile = ref(null);

const originalAudioUrl = ref("");
const editedAudioUrl = ref("");

const currentCoverUrl = ref("");
const newCoverUrl = ref("");

const currentCoverArrayBuffer = ref(null);
const newCoverFile = ref(null);
const removeCover = ref(false);

const originalMetadataJson = ref("");
const editedMetadataJson = ref("");

const nativeTags = ref([]);
const customTags = ref([]);

const status = ref("Upload your MP3 file.");
const successMessage = ref("");
const errorMessage = ref("");
const logs = ref([]);

const isLoading = ref(false);

const fileInfo = ref(null);
const editedFileInfo = ref(null);

const form = reactive({
  title: "",
  artists: "",
  album: "",
  albumArtist: "",
  year: "",
  genre: "",
  composer: "",
  publisher: "",
  copyright: "",
  legalUrl: "",
  artistUrl: "",
  isrc: "",
  track: "",
  disc: "",
  bpm: "",
  key: "",
  comment: "",
  lyrics: ""
});

const defaultFreshMetadata = {
  title: "Untitled Original Instrumental",
  artists: "Argho Chakma",
  album: "Original Music Collection",
  albumArtist: "Argho Chakma",
  year: "2026",
  genre: "Instrumental; Cinematic",
  composer: "Argho Chakma",
  publisher: "Argho Chakma Music",
  copyright: "© 2026 Argho Chakma. All rights reserved.",
  legalUrl: "",
  artistUrl: "",
  isrc: "",
  track: "1",
  disc: "1",
  bpm: "",
  key: "",
  comment: "",
  lyrics: ""
};

function addLog(message) {
  const time = new Date().toLocaleTimeString();
  logs.value.unshift(`[${time}] ${message}`);
}

function clearMessages() {
  successMessage.value = "";
  errorMessage.value = "";
}

function splitList(value) {
  return String(value || "")
    .split(";")
    .map((item) => item.trim())
    .filter(Boolean);
}

function joinList(value) {
  if (!value) return "";
  if (Array.isArray(value)) return value.join("; ");
  return String(value);
}

function cleanForJson(value) {
  if (value instanceof Uint8Array) {
    return `[binary ${value.byteLength} bytes]`;
  }

  if (value instanceof ArrayBuffer) {
    return `[binary ${value.byteLength} bytes]`;
  }

  if (Array.isArray(value)) {
    return value.map(cleanForJson);
  }

  if (value && typeof value === "object") {
    const obj = {};

    for (const key in value) {
      if (key.toLowerCase() === "data") {
        obj[key] = "[binary data hidden]";
      } else {
        obj[key] = cleanForJson(value[key]);
      }
    }

    return obj;
  }

  return value;
}

function sanitizeMetadata(metadata) {
  return {
    format: cleanForJson(metadata.format),
    common: cleanForJson({
      ...metadata.common,
      picture: metadata.common?.picture
        ? metadata.common.picture.map((pic) => ({
          format: pic.format,
          type: pic.type,
          description: pic.description,
          size: pic.data?.byteLength || pic.data?.length || 0
        }))
        : []
    }),
    native: cleanForJson(metadata.native)
  };
}

function normalizeNativeTags(native) {
  const result = [];

  for (const tagType in native || {}) {
    const tags = native[tagType] || [];

    tags.forEach((tag) => {
      let value = cleanForJson(tag.value);

      if (typeof value === "object") {
        value = JSON.stringify(value);
      }

      result.push({
        type: tagType,
        id: tag.id,
        value: String(value)
      });
    });
  }

  return result;
}

function uint8ArrayToArrayBuffer(uint8Array) {
  return uint8Array.buffer.slice(
    uint8Array.byteOffset,
    uint8Array.byteOffset + uint8Array.byteLength
  );
}

async function sha256Blob(blob) {
  const buffer = await blob.arrayBuffer();
  const hashBuffer = await crypto.subtle.digest("SHA-256", buffer);
  const hashArray = Array.from(new Uint8Array(hashBuffer));

  return hashArray.map((b) => b.toString(16).padStart(2, "0")).join("");
}

function formatDuration(seconds) {
  if (!seconds) return "Unknown";

  const min = Math.floor(seconds / 60);
  const sec = Math.floor(seconds % 60);

  return `${min}:${String(sec).padStart(2, "0")}`;
}

function resetState() {
  clearMessages();

  if (originalAudioUrl.value) URL.revokeObjectURL(originalAudioUrl.value);
  if (editedAudioUrl.value) URL.revokeObjectURL(editedAudioUrl.value);
  if (currentCoverUrl.value) URL.revokeObjectURL(currentCoverUrl.value);
  if (newCoverUrl.value) URL.revokeObjectURL(newCoverUrl.value);

  originalAudioUrl.value = "";
  editedAudioUrl.value = "";
  currentCoverUrl.value = "";
  newCoverUrl.value = "";

  currentCoverArrayBuffer.value = null;
  newCoverFile.value = null;
  removeCover.value = false;

  originalMetadataJson.value = "";
  editedMetadataJson.value = "";

  nativeTags.value = [];
  customTags.value = [];

  fileInfo.value = null;
  editedFileInfo.value = null;

  Object.keys(form).forEach((key) => {
    form[key] = "";
  });
}

function fillFormFromMetadata(metadata) {
  const common = metadata.common || {};

  form.title = common.title || "";
  form.artists = joinList(common.artists || common.artist);
  form.album = common.album || "";
  form.albumArtist = common.albumartist || "";
  form.year = common.year || "";
  form.genre = joinList(common.genre);
  form.composer = joinList(common.composer);
  form.publisher = joinList(common.label || common.publisher);
  form.copyright = common.copyright || "";
  form.legalUrl = "";
  form.artistUrl = "";
  form.isrc = joinList(common.isrc);

  form.track = common.track?.no
    ? `${common.track.no}${common.track.of ? "/" + common.track.of : ""}`
    : "";

  form.disc = common.disk?.no
    ? `${common.disk.no}${common.disk.of ? "/" + common.disk.of : ""}`
    : "";

  form.bpm = common.bpm || "";
  form.key = common.key || "";
  form.comment = joinList(common.comment);
  form.lyrics = joinList(common.lyrics);
}

async function readAudioFile(event) {
  const file = event.target.files[0];

  if (!file) return;

  resetState();

  audioFile.value = file;
  originalAudioUrl.value = URL.createObjectURL(file);

  status.value = "Reading audio information...";
  isLoading.value = true;
  addLog("Audio selected.");

  try {
    const hash = await sha256Blob(file);
    const metadata = await parseBlob(file, {
      duration: true,
      skipCovers: false
    });

    fileInfo.value = {
      name: file.name,
      type: file.type || "Unknown",
      size: `${(file.size / 1024 / 1024).toFixed(2)} MB`,
      duration: formatDuration(metadata.format.duration),
      sampleRate: metadata.format.sampleRate || "Unknown",
      bitrate: metadata.format.bitrate
        ? `${Math.round(metadata.format.bitrate / 1000)} kbps`
        : "Unknown",
      codec: metadata.format.codec || "Unknown",
      container: metadata.format.container || "Unknown",
      tagTypes: metadata.format.tagTypes?.join(", ") || "Unknown",
      sha256: hash
    };

    originalMetadataJson.value = JSON.stringify(
      sanitizeMetadata(metadata),
      null,
      2
    );

    nativeTags.value = normalizeNativeTags(metadata.native);

    fillFormFromMetadata(metadata);

    const cover = metadata.common?.picture?.[0];

    if (cover) {
      const coverBlob = new Blob([cover.data], {
        type: cover.format || "image/jpeg"
      });

      currentCoverUrl.value = URL.createObjectURL(coverBlob);
      currentCoverArrayBuffer.value = uint8ArrayToArrayBuffer(cover.data);
      addLog("Cover image found.");
    } else {
      addLog("No cover image found.");
    }

    status.value = "Audio information loaded successfully.";
    successMessage.value = "Original audio metadata loaded successfully.";
    addLog("Original metadata JSON created.");
  } catch (error) {
    console.error(error);
    errorMessage.value = error.message;
    status.value = "Failed to read audio.";
    addLog("Error while reading audio.");
  } finally {
    isLoading.value = false;
  }
}

function addCustomTag() {
  customTags.value.push({
    description: "",
    value: ""
  });

  addLog("New custom tag row added.");
}

function removeCustomTag(index) {
  customTags.value.splice(index, 1);
  addLog("Custom tag removed.");
}

function readNewCoverFile(event) {
  const file = event.target.files[0];

  if (!file) return;

  newCoverFile.value = file;
  removeCover.value = false;

  if (newCoverUrl.value) {
    URL.revokeObjectURL(newCoverUrl.value);
  }

  newCoverUrl.value = URL.createObjectURL(file);

  successMessage.value = "New cover image selected. Live JSON updated.";
  addLog("New cover image selected.");
}

function removeCoverImage() {
  newCoverFile.value = null;
  removeCover.value = true;

  if (newCoverUrl.value) {
    URL.revokeObjectURL(newCoverUrl.value);
  }

  newCoverUrl.value = "";

  successMessage.value = "Cover image will be removed when you save.";
  addLog("Cover image removal selected.");
}

function keepCurrentCover() {
  newCoverFile.value = null;
  removeCover.value = false;

  if (newCoverUrl.value) {
    URL.revokeObjectURL(newCoverUrl.value);
  }

  newCoverUrl.value = "";

  successMessage.value = "Current cover image will be kept.";
  addLog("Current cover image selected to keep.");
}

function applyFreshDefaultMetadata() {
  Object.keys(defaultFreshMetadata).forEach((key) => {
    form[key] = defaultFreshMetadata[key];
  });

  customTags.value = [
    {
      description: "Owner",
      value: "Argho Chakma"
    },
    {
      description: "FreshMetadata",
      value: "true"
    }
  ];

  newCoverFile.value = null;
  removeCover.value = true;

  if (newCoverUrl.value) {
    URL.revokeObjectURL(newCoverUrl.value);
  }

  newCoverUrl.value = "";

  editedMetadataJson.value = "";
  editedAudioUrl.value = "";
  editedFileInfo.value = null;

  successMessage.value =
    "Fresh default metadata applied successfully. Live New Metadata JSON updated instantly.";
  status.value = "Fresh metadata ready. Click Save Fresh MP3.";
  addLog("Fresh default metadata applied.");
}

function buildLiveMetadataObject() {
  return {
    action: "New metadata that will be written on save",
    common: {
      title: form.title,
      artists: splitList(form.artists),
      artist: form.artists,
      album: form.album,
      albumartist: form.albumArtist,
      year: form.year,
      genre: splitList(form.genre),
      composer: splitList(form.composer),
      publisher: form.publisher,
      copyright: form.copyright,
      legalUrl: form.legalUrl,
      artistUrl: form.artistUrl,
      isrc: form.isrc,
      track: form.track,
      disc: form.disc,
      bpm: form.bpm,
      key: form.key,
      comment: form.comment,
      lyrics: form.lyrics
    },
    cover: {
      action: newCoverFile.value
        ? "replace_cover"
        : removeCover.value
          ? "remove_cover"
          : currentCoverUrl.value
            ? "keep_current_cover"
            : "no_cover",
      newCoverFileName: newCoverFile.value ? newCoverFile.value.name : "",
      note: "Image binary data is not shown inside JSON."
    },
    customTags: customTags.value
      .filter((tag) => tag.description || tag.value)
      .map((tag) => ({
        description: tag.description,
        value: tag.value
      }))
  };
}

const liveNewMetadataJson = computed(() => {
  return JSON.stringify(buildLiveMetadataObject(), null, 2);
});

const copyrightMetadataCheck = computed(() => {
  const found = [];

  if (form.copyright) found.push(`Copyright: ${form.copyright}`);
  if (form.isrc) found.push(`ISRC: ${form.isrc}`);
  if (form.publisher) found.push(`Publisher/Label: ${form.publisher}`);
  if (form.legalUrl) found.push(`Legal URL: ${form.legalUrl}`);

  if (found.length === 0) {
    return "No copyright-related metadata field is currently filled.";
  }

  return `Metadata fields filled: ${found.join(" | ")}`;
});

function trySetFrame(writer, frame, value) {
  if (value === undefined || value === null || value === "") return;

  try {
    writer.setFrame(frame, value);
    addLog(`Frame written: ${frame}`);
  } catch (error) {
    addLog(`Frame skipped: ${frame} (${error.message})`);
  }
}

async function saveMetadata() {
  if (!audioFile.value) {
    alert("Please upload an MP3 file first.");
    return;
  }

  const isMp3 =
    audioFile.value.type === "audio/mpeg" ||
    audioFile.value.name.toLowerCase().endsWith(".mp3");

  if (!isMp3) {
    alert("This editor writes MP3 ID3 tags. Please use an MP3 file.");
    return;
  }

  clearMessages();
  isLoading.value = true;
  status.value = "Saving new metadata...";
  addLog("Metadata writing started.");

  try {
    const arrayBuffer = await audioFile.value.arrayBuffer();
    const writer = new ID3Writer(arrayBuffer);

    trySetFrame(writer, "TIT2", form.title);
    trySetFrame(writer, "TPE1", splitList(form.artists));
    trySetFrame(writer, "TALB", form.album);
    trySetFrame(writer, "TPE2", form.albumArtist);

    const yearNumber = parseInt(form.year, 10);
    if (!Number.isNaN(yearNumber)) {
      trySetFrame(writer, "TYER", yearNumber);
    }

    trySetFrame(writer, "TCON", splitList(form.genre));
    trySetFrame(writer, "TCOM", splitList(form.composer));
    trySetFrame(writer, "TPUB", form.publisher);
    trySetFrame(writer, "TCOP", form.copyright);
    trySetFrame(writer, "WCOP", form.legalUrl);
    trySetFrame(writer, "WOAR", form.artistUrl);

    const firstIsrc = splitList(form.isrc)[0] || form.isrc;
    trySetFrame(writer, "TSRC", firstIsrc);

    trySetFrame(writer, "TRCK", form.track);
    trySetFrame(writer, "TPOS", form.disc);

    const bpmNumber = parseInt(form.bpm, 10);
    if (!Number.isNaN(bpmNumber)) {
      trySetFrame(writer, "TBPM", bpmNumber);
    }

    trySetFrame(writer, "TKEY", form.key);

    if (form.comment) {
      trySetFrame(writer, "COMM", {
        description: "Comment",
        text: form.comment,
        language: "eng"
      });
    }

    if (form.lyrics) {
      trySetFrame(writer, "USLT", {
        description: "Lyrics",
        lyrics: form.lyrics,
        language: "eng"
      });
    }

    customTags.value.forEach((tag) => {
      if (!tag.description || !tag.value) return;

      trySetFrame(writer, "TXXX", {
        description: tag.description,
        value: tag.value
      });
    });

    let coverArrayBuffer = null;

    if (newCoverFile.value) {
      coverArrayBuffer = await newCoverFile.value.arrayBuffer();
      addLog("New cover image prepared.");
    } else if (!removeCover.value && currentCoverArrayBuffer.value) {
      coverArrayBuffer = currentCoverArrayBuffer.value;
      addLog("Current cover image will be kept.");
    } else {
      addLog("Cover image will not be written.");
    }

    if (coverArrayBuffer) {
      trySetFrame(writer, "APIC", {
        type: 3,
        data: coverArrayBuffer,
        description: "Cover image",
        useUnicodeEncoding: false
      });
    }

    writer.addTag();
    addLog("Fresh ID3 tag created.");

    const editedBlob = writer.getBlob();

    if (editedAudioUrl.value) {
      URL.revokeObjectURL(editedAudioUrl.value);
    }

    editedAudioUrl.value = URL.createObjectURL(editedBlob);

    const editedHash = await sha256Blob(editedBlob);

    editedFileInfo.value = {
      size: `${(editedBlob.size / 1024 / 1024).toFixed(2)} MB`,
      sha256: editedHash
    };

    const editedMetadata = await parseBlob(editedBlob, {
      duration: true,
      skipCovers: false
    });

    editedMetadataJson.value = JSON.stringify(
      sanitizeMetadata(editedMetadata),
      null,
      2
    );

    successMessage.value =
      "Success! Metadata saved. Edited audio preview and download button are ready.";
    status.value = "Metadata saved successfully.";
    addLog("Edited metadata JSON updated.");
    addLog("Download button ready.");
  } catch (error) {
    console.error(error);
    errorMessage.value = error.message;
    status.value = "Failed to save metadata.";
    addLog("Error while saving metadata.");
  } finally {
    isLoading.value = false;
  }
}

async function resetAndSaveFreshMp3() {
  if (!audioFile.value) {
    alert("Please upload an MP3 file first.");
    return;
  }

  applyFreshDefaultMetadata();
  await saveMetadata();
}

function downloadEditedAudio() {
  if (!editedAudioUrl.value || !audioFile.value) return;

  const baseName = audioFile.value.name.replace(/\.[^/.]+$/, "");
  const a = document.createElement("a");

  a.href = editedAudioUrl.value;
  a.download = `${baseName}-fresh-metadata.mp3`;
  a.click();

  successMessage.value = "Download started.";
  addLog("Edited MP3 download started.");
}
</script>

<template>
  <div class="page">
    <h1>MP3 Metadata + Cover Editor</h1>

    <div class="card">
      <h2>1. Upload MP3</h2>

      <input type="file" accept="audio/mpeg,audio/mp3,.mp3" @change="readAudioFile" />

      <p class="status">{{ status }}</p>

      <p v-if="successMessage" class="success">
        {{ successMessage }}
      </p>

      <p v-if="errorMessage" class="error">
        {{ errorMessage }}
      </p>

      <p v-if="isLoading" class="loading">
        Processing... please wait.
      </p>
    </div>

    <div v-if="logs.length" class="card">
      <h2>Live Process Update</h2>

      <div class="logs">
        <p v-for="(log, index) in logs" :key="index">
          {{ log }}
        </p>
      </div>
    </div>

    <div v-if="originalAudioUrl" class="card">
      <h2>Original Audio Preview</h2>
      <audio :src="originalAudioUrl" controls></audio>
    </div>

    <div v-if="fileInfo" class="card">
      <h2>Original File Information</h2>

      <div class="info-grid">
        <p><b>Name:</b> {{ fileInfo.name }}</p>
        <p><b>Type:</b> {{ fileInfo.type }}</p>
        <p><b>Size:</b> {{ fileInfo.size }}</p>
        <p><b>Duration:</b> {{ fileInfo.duration }}</p>
        <p><b>Sample Rate:</b> {{ fileInfo.sampleRate }}</p>
        <p><b>Bitrate:</b> {{ fileInfo.bitrate }}</p>
        <p><b>Codec:</b> {{ fileInfo.codec }}</p>
        <p><b>Container:</b> {{ fileInfo.container }}</p>
        <p><b>Tag Types:</b> {{ fileInfo.tagTypes }}</p>
      </div>

      <p class="hash">
        <b>Original SHA-256:</b> {{ fileInfo.sha256 }}
      </p>
    </div>

    <div v-if="audioFile" class="card">
      <h2>2. Cover Image</h2>

      <div class="cover-row">
        <div v-if="currentCoverUrl">
          <h3>Current Cover</h3>
          <img :src="currentCoverUrl" class="cover" />
        </div>

        <div v-if="newCoverUrl">
          <h3>New Cover</h3>
          <img :src="newCoverUrl" class="cover" />
        </div>

        <div v-if="!currentCoverUrl && !newCoverUrl">
          <p>No cover image found.</p>
        </div>
      </div>

      <input type="file" accept="image/jpeg,image/png" @change="readNewCoverFile" />

      <div class="button-row">
        <button type="button" @click="keepCurrentCover">
          Keep Current Cover
        </button>

        <button type="button" @click="removeCoverImage">
          Remove Cover
        </button>
      </div>
    </div>

    <div v-if="audioFile" class="card">
      <h2>3. Copyright Metadata Check</h2>

      <p>{{ copyrightMetadataCheck }}</p>

      <small>
        Note: Filling copyright-related metadata fields is important for protecting your rights as a creator and
        ensuring proper attribution. Consider adding relevant information such as copyright, ISRC, publisher/label, and
        legal URL to help safeguard your work and make it easier for others to find and credit you.
      </small>
    </div>

    <div v-if="audioFile" class="card form">
      <h2>4. Change Metadata Tags</h2>

      <label>
        Title
        <input v-model="form.title" placeholder="Song title" />
      </label>

      <label>
        Artists
        <input v-model="form.artists" placeholder="Argho Chakma; Another Artist" />
      </label>

      <label>
        Album
        <input v-model="form.album" placeholder="Album name" />
      </label>

      <label>
        Album Artist
        <input v-model="form.albumArtist" placeholder="Album artist" />
      </label>

      <label>
        Year
        <input v-model="form.year" placeholder="2026" />
      </label>

      <label>
        Genre
        <input v-model="form.genre" placeholder="Instrumental; Cinematic" />
      </label>

      <label>
        Composer
        <input v-model="form.composer" placeholder="Argho Chakma" />
      </label>

      <label>
        Publisher / Label
        <input v-model="form.publisher" placeholder="Argho Chakma Music" />
      </label>

      <label>
        Copyright
        <input v-model="form.copyright" placeholder="© 2026 Argho Chakma. All rights reserved." />
      </label>

      <label>
        Legal / Copyright URL
        <input v-model="form.legalUrl" placeholder="https://example.com/copyright" />
      </label>

      <label>
        Artist URL
        <input v-model="form.artistUrl" placeholder="https://example.com" />
      </label>

      <label>
        ISRC
        <input v-model="form.isrc" placeholder="Example: BDXXX2600001" />
      </label>

      <label>
        Track
        <input v-model="form.track" placeholder="1/10" />
      </label>

      <label>
        Disc
        <input v-model="form.disc" placeholder="1/1" />
      </label>

      <label>
        BPM
        <input v-model="form.bpm" placeholder="100" />
      </label>

      <label>
        Key
        <input v-model="form.key" placeholder="F#m" />
      </label>

      <label>
        Comment
        <textarea v-model="form.comment" placeholder="Comment"></textarea>
      </label>

      <label>
        Lyrics / Description
        <textarea v-model="form.lyrics" placeholder="Lyrics or description"></textarea>
      </label>
    </div>

    <div v-if="audioFile" class="card">
      <h2>5. Custom Tags</h2>

      <button type="button" @click="addCustomTag">
        Add Custom Tag
      </button>

      <div v-for="(tag, index) in customTags" :key="index" class="custom-tag-row">
        <input v-model="tag.description" placeholder="Tag name e.g. Owner" />

        <input v-model="tag.value" placeholder="Tag value e.g. Argho Chakma" />

        <button type="button" @click="removeCustomTag(index)">
          Remove
        </button>
      </div>
    </div>

    <div v-if="audioFile" class="card">
      <h2>6. Fresh Metadata / Save</h2>

      <div class="button-row">
        <button type="button" @click="applyFreshDefaultMetadata">
          Create Fresh Default Metadata
        </button>

        <button type="button" @click="saveMetadata" :disabled="isLoading">
          Save Current Metadata
        </button>

        <button type="button" @click="resetAndSaveFreshMp3" :disabled="isLoading">
          Reset + Save Fresh MP3
        </button>
      </div>
    </div>

    <div v-if="audioFile" class="card">
      <h2>Live New Metadata JSON</h2>

      <p>
        Form, custom tags, বা cover image change করলেই এই JSON instant update হবে।
      </p>

      <textarea class="json-box" :value="liveNewMetadataJson" readonly></textarea>
    </div>

    <div v-if="editedAudioUrl" class="card">
      <h2>Edited Audio Preview</h2>

      <audio :src="editedAudioUrl" controls></audio>

      <div v-if="editedFileInfo" class="info-grid">
        <p><b>Edited Size:</b> {{ editedFileInfo.size }}</p>
        <p><b>Edited SHA-256:</b> {{ editedFileInfo.sha256 }}</p>
      </div>

      <button type="button" class="download" @click="downloadEditedAudio">
        Download Edited MP3
      </button>
    </div>

    <div v-if="nativeTags.length" class="card">
      <h2>Original Native Tags Found</h2>

      <div class="table">
        <div class="row head">
          <b>Type</b>
          <b>ID</b>
          <b>Value</b>
        </div>

        <div v-for="(tag, index) in nativeTags" :key="index" class="row">
          <span>{{ tag.type }}</span>
          <span>{{ tag.id }}</span>
          <span>{{ tag.value }}</span>
        </div>
      </div>
    </div>

    <div v-if="originalMetadataJson" class="card">
      <h2>Original Full Metadata JSON</h2>
      <textarea class="json-box" :value="originalMetadataJson" readonly></textarea>
    </div>

    <div v-if="editedMetadataJson" class="card">
      <h2>Edited Full Metadata JSON</h2>
      <textarea class="json-box" :value="editedMetadataJson" readonly></textarea>
    </div>
  </div>
</template>

<style>
.page {
  max-width: 1000px;
  margin: 30px auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  color: #f5f5f5;
}

.card {
  margin-top: 18px;
  padding: 18px;
  border: 1px solid #333;
  border-radius: 12px;
  background: #171717;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-weight: bold;
}

input,
textarea,
button {
  padding: 11px;
  border-radius: 7px;
  border: 1px solid #444;
  font-size: 14px;
}

input,
textarea {
  background: #0d0d0d;
  color: #f5f5f5;
}

button {
  cursor: pointer;
  font-weight: bold;
  background: #f5f5f5;
  color: #111;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

audio {
  width: 100%;
}

.status {
  color: #d8d8d8;
}

.success {
  padding: 10px;
  border-radius: 8px;
  background: #113d1f;
  color: #9ff2b2;
}

.error {
  padding: 10px;
  border-radius: 8px;
  background: #4d1111;
  color: #ffb5b5;
}

.loading {
  color: #ffd37a;
}

.logs {
  max-height: 220px;
  overflow: auto;
  background: #0d0d0d;
  border-radius: 8px;
  padding: 10px;
}

.logs p {
  margin: 6px 0;
  font-family: Consolas, monospace;
  font-size: 13px;
}

.info-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px 18px;
}

.hash {
  word-break: break-word;
}

.cover-row {
  display: flex;
  gap: 22px;
  flex-wrap: wrap;
  margin-bottom: 15px;
}

.cover {
  width: 190px;
  height: 190px;
  object-fit: cover;
  border-radius: 12px;
  border: 1px solid #444;
}

.button-row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-top: 12px;
}

.custom-tag-row {
  display: grid;
  grid-template-columns: 1fr 1fr auto;
  gap: 10px;
  margin-top: 10px;
}

.json-box {
  width: 100%;
  min-height: 380px;
  background: #0d0d0d;
  color: #f5f5f5;
  font-family: Consolas, monospace;
  font-size: 13px;
  white-space: pre;
}

.table {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.row {
  display: grid;
  grid-template-columns: 150px 150px 1fr;
  gap: 10px;
  padding: 9px;
  border-bottom: 1px solid #333;
  word-break: break-word;
}

.head {
  background: #222;
}

.download {
  margin-top: 15px;
  width: 100%;
  background: #9ff2b2;
  color: #111;
}

@media (max-width: 700px) {

  .info-grid,
  .custom-tag-row,
  .row {
    grid-template-columns: 1fr;
  }
}
</style>