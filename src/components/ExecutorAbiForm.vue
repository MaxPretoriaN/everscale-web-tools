<script setup lang="ts">
import { nextTick, ref, shallowRef, watch } from 'vue';

import * as core from '@core';
import { unzipSync } from 'fflate';
import { convertAddress, rewriteAbiUrl } from '../common';

enum LoadAbiType {
  FROM_FILE,
  FROM_TEXT,
  FROM_LINK,
  FROM_ZIP
}

const DEFAULT_ABI_NAME = 'abi1';
const COLLATOR = new Intl.Collator(undefined, { numeric: true, sensitivity: 'base' });

const getAllAbis = () => Object.keys(localStorage).sort(COLLATOR.compare);

const props = defineProps<{
  address?: string;
  codeHash?: string;
}>();

const emit = defineEmits<{
  (e: 'change', abi: string): void;
}>();

const modalType = ref<LoadAbiType>();

const abiNameInput = ref(DEFAULT_ABI_NAME);
const fieldsInput = ref<{ file?: File; text?: string; link?: string, zip?: { file?: File, link?: string } }>({});
const everscanAbi = ref<string>();
const error = ref<string>();

const storedAbi = shallowRef<string[]>(getAllAbis());
const abiFilter = ref<string>('');
const abiFilterField = ref<HTMLDivElement>();
const selectedAbi = ref<string>();
const abiSelectorVisible = ref(false);

const inProgress = ref(false);

watch(
  () => [props.codeHash, props.address],
  ([codeHash, address], _, onCleanup) => {
    if (address == null || codeHash == null) {
      return;
    }

    const controller = new AbortController();

    const localState = { codeHashChanged: false };
    onCleanup(() => {
      everscanAbi.value = undefined;
      localState.codeHashChanged = true;
      controller.abort();
    });

    fetch(`https://verify.everscan.io/info/code_hash/${codeHash}`, {
      method: 'GET',
      signal: controller.signal
    })
      .then(res => res.json())
      .then(data => {
        if (localState.codeHashChanged) {
          return
        }

        if (data && data.abi) {
          everscanAbi.value = JSON.stringify(data.abi);
        }

        if (data && data.contract_name) {
          const shortName = data.contract_name.split('/').pop().replace(/\.(tsol|sol|fc|fif)$/, '');

          if (shortName) {
            document.title = `${shortName} ${convertAddress(address)} | Everscale tools`;
          }
        }
      })
      .catch(console.error);
  },
  {
    immediate: true
  }
);

watch(
  () => [selectedAbi.value, props.address],
  ([abiName, address]) => {
    if (abiName && address) {
      document.title = `${abiName} ${convertAddress(address)} | Everscale tools`;
    }
  },
  {
    immediate: true
  }
);

function onSelectAbi(name: string) {
  const text = localStorage.getItem(name);
  selectedAbi.value = text != null ? name : undefined;
  abiSelectorVisible.value = false;
  emit('change', text);
}

function onDeleteAbi(name: string) {
  localStorage.removeItem(name);
  storedAbi.value = getAllAbis();
}

function closeModal() {
  modalType.value = undefined;
  fieldsInput.value = {};
  error.value = undefined;
}

function onChangeFile(e: InputEvent) {
  const files = (e.target as HTMLInputElement).files;
  if (files == null || files.length == 0) {
    return;
  }
  fieldsInput.value = {
    file: files[0]
  };
}

function onChangeZipFile(e: InputEvent) {
  const files = (e.target as HTMLInputElement).files;
  if (files == null || files.length == 0) {
    return;
  }
  fieldsInput.value = {
    zip: {
      file: files[0]
    }
  };
}

function onChangeText(e: InputEvent) {
  fieldsInput.value = {
    text: (e.target as HTMLInputElement).value
  };
}

function onChangeLink(e: InputEvent) {
  fieldsInput.value = {
    link: (e.target as HTMLInputElement).value
  };
}

function onChangeZipLink(e: InputEvent) {
  fieldsInput.value = {
    zip: {
      link: (e.target as HTMLInputElement).value
    }
  };
}

async function loadAbiText(): Promise<string> {
  const { file, text, link } = fieldsInput.value;
  if (text != null) {
    return text;
  } else if (link != null) {
    const url = rewriteAbiUrl(new URL(link));
    return fetch(url.toString(), {}).then(res => res.text());
  } else if (file != null) {
    return new Promise<string | undefined>((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = event => {
        resolve(event?.target?.result as string | undefined);
      };
      reader.onerror = event => reject(event?.target?.error);
      reader.readAsText(file);
    }).then(content => content || '');
  } else {
    throw new Error('No ABI input specified');
  }
}

type ZipContent = { filename: string, files: { filename: string, text: string }[] };

async function loadZipContent(): Promise<ZipContent> {
  const { zip } = fieldsInput.value;

  if (!zip?.file && !zip?.link) {
    throw new Error('No zip link or file specified');
  }

  const buffer = await (zip.file?.arrayBuffer() ?? (await fetch(zip.link)).arrayBuffer());

  const zipContents = unzipSync(new Uint8Array(buffer));

  const filesData = await Promise.all(Object.entries(zipContents).map((x) =>
    new Promise<{ filename: string, text: string }>((resolve, reject) => {
      const [filename, uint8] = x;
      const file = new File([uint8], filename);
      const reader = new FileReader();

      reader.onload = (event) => resolve({ filename: filename, text: (event?.target?.result as string ?? '') });
      reader.onerror = (event) => reject(event?.target?.error);
      reader.readAsText(file);
    })));

  const filename = (zip.file?.name ?? zip.link?.split('/').pop() ?? 'unnamed_archive');
  const files = filesData.sort((x, y) => x.filename.localeCompare(y.filename, undefined, { numeric: true, sensitivity: 'case' }));

  return {
    filename,
    files,
  };
}

type ZipContentProcessResult = { storageKey: string, text: string }[];

async function processZipContent(zipContent: ZipContent): Promise<ZipContentProcessResult> {
  if (zipContent.files.length === 0) {
    throw new Error(`Provided zip archive is empty`);
  }

  zipContent.files.forEach(abiFile => {
    try {
      core.checkAbi(abiFile.text);
    } catch (err: any) {
      throw new Error(`Processing error for '${abiFile.filename}' file. Internal ${err.toString()}`);
    }
  });

  const shortZipFilename = zipContent.filename.split('.').shift();
  const dateStr = new Date().toISOString().split('T')[0];
  const parentFormattedFilename = `${shortZipFilename}_${dateStr}`;

  // Store abis only after successful validation
  const abis = zipContent.files.map(abiFile => {
    const shortFilename = abiFile.filename.split('.').shift();
    const storageKey = `[${parentFormattedFilename}] ${shortFilename}`
    const text = abiFile.text;
    localStorage.setItem(storageKey, text);
    return {
      storageKey,
      text
    };
  });

  return abis;
};

function onSubmitEverscanAbi() {
  if (everscanAbi.value != null) {
    emit('change', everscanAbi.value);
  }
}

async function onSubmit() {
  if (inProgress.value) {
    return;
  }
  inProgress.value = true;
  try {
    const text = await loadAbiText();
    core.checkAbi(text);
    localStorage.setItem(abiNameInput.value, text);
    storedAbi.value = getAllAbis();
    selectedAbi.value = abiNameInput.value;
    emit('change', text);
    closeModal();
  } catch (e: any) {
    error.value = e.toString();
  } finally {
    inProgress.value = false;
  }
}

async function onSubmitZip() {
  if (inProgress.value) {
    return;
  }
  inProgress.value = true;
  try {
    const zipContent = await loadZipContent();
    const abis = await processZipContent(zipContent);
    storedAbi.value = getAllAbis();
    selectedAbi.value = abis[0]?.storageKey ?? undefined;
    emit('change', abis[0]?.text);
    closeModal();
  } catch (e: any) {
    error.value = e.toString();
  } finally {
    inProgress.value = false;
  }
}

function onToggleAbiSelector() {
  const visible = !abiSelectorVisible.value;
  abiSelectorVisible.value = visible;
  abiFilter.value = '';
  if (visible) {
    nextTick(() => abiFilterField.value?.focus());
  }
}

function onHideAbiSelector() {
  abiFilter.value = '';
  abiSelectorVisible.value = false;
}
</script>

<template>
  <div class="field has-addons">
    <div class="control">
      <button class="button is-static">Load ABI</button>
    </div>
    <div class="control">
      <button class="button" :disabled="inProgress" @click="modalType = LoadAbiType.FROM_FILE">From file</button>
    </div>
    <div class="control">
      <button class="button" :disabled="inProgress" @click="modalType = LoadAbiType.FROM_TEXT">From text</button>
    </div>
    <div class="control">
      <button class="button" :disabled="inProgress" @click="modalType = LoadAbiType.FROM_LINK">From link</button>
    </div>
    <div class="control">
      <button class="button" :disabled="inProgress" @click="modalType = LoadAbiType.FROM_ZIP">From zip</button>
    </div>
    <div class="control">
      <button class="button" :disabled="inProgress || everscanAbi == null" @click="onSubmitEverscanAbi">
        From Everscan
      </button>
    </div>
    <div v-if="storedAbi.length > 0" :class="['dropdown', { 'is-active': abiSelectorVisible }]">
      <div class="dropdown-trigger">
        <button class="button" aria-haspopup="true" aria-controls="select-abi-dropdown" @click="onToggleAbiSelector">
          <span>{{ selectedAbi == null ? 'Select ABI...' : selectedAbi }}</span>
          <span class="icon is-small">
            <i :class="['fas', abiSelectorVisible ? 'fa-angle-up' : 'fa-angle-down']" aria-hidden="true" />
          </span>
        </button>
      </div>
      <div class="dropdown-menu" id="select-abi-dropdown" role="menu">
        <div class="dropdown-content">
          <input type="text" class="dropdown-item is-flex is-align-items-center input is-small pr-4" spellcheck="false"
            v-model="abiFilter" ref="abiFilterField" @blur="onHideAbiSelector">

          <a v-for="(name, i) in storedAbi" :key="i"
            :class="['dropdown-item', 'is-align-items-center', 'pr-4', name.toLowerCase().includes(abiFilter.toLowerCase()) ? 'is-flex' : 'is-hidden']"
            @mousedown="onSelectAbi(name)">
            <span class="mr-5">{{ name }}</span>
            <button class="delete is-small ml-auto" @mousedown="onDeleteAbi(name)"></button>
          </a>
        </div>
      </div>
    </div>
  </div>

  <Teleport to="body">
    <div :class="['modal', { 'is-active': modalType != null }]">
      <div class="modal-background" @click="closeModal" />
      <div class="modal-card">
        <header class="modal-card-head">
          <p class="modal-card-title">Load ABI</p>
          <button class="delete" aria-label="close" @click="closeModal" />
        </header>

        <section class="modal-card-body">
          <div v-if="modalType !== LoadAbiType.FROM_ZIP" class="field">
            <label for="abi-name" class="label">ABI name:</label>
            <input class="input" name="abi-name" type="text" spellcheck="false" v-model="abiNameInput"
              :disabled="inProgress" />
          </div>

          <div v-if="modalType === LoadAbiType.FROM_FILE" class="file">
            <label class="file-label">
              <input class="file-input" name="resume" type="file" @change="onChangeFile" />
              <span class="file-cta">
                <span class="file-icon">
                  <i class="fas fa-upload" />
                </span>
                <span class="file-label">{{
                  fieldsInput.file?.name != null ? fieldsInput.file.name : 'Choose a file…'
                }}</span>
              </span>
            </label>
            <p v-if="error != null" class="help is-danger">{{ error }}</p>
          </div>

          <div v-else-if="modalType === LoadAbiType.FROM_TEXT" class="field">
            <label for="abi-text" class="label">JSON ABI:</label>
            <div class="control">
              <textarea :class="['textarea', { 'is-danger': error != null }]" name="abi-text" spellcheck="true" rows="5"
                :disabled="inProgress" :value="fieldsInput.text || ''" @input="onChangeText" />
            </div>
            <p v-if="error != null" class="help is-danger">{{ error }}</p>
          </div>

          <div v-else-if="modalType === LoadAbiType.FROM_LINK" class="field">
            <label for="abi-link" class="label">Link to JSON ABI:</label>
            <div class="control">
              <input type="text" class="input" name="abi-link" spellcheck="false" :disabled="inProgress"
                :value="fieldsInput.link || ''" @input="onChangeLink" />
            </div>
            <p v-if="error != null" class="help is-danger">{{ error }}</p>
          </div>

          <div v-else-if="modalType === LoadAbiType.FROM_ZIP">
            <div class="field">
              <label for="zip-link" class="label">Link to zip archive:</label>
              <div class="control">
                <input type="text" class="input" name="zip-link" spellcheck="false" :disabled="inProgress"
                  :value="fieldsInput.zip?.link || ''" @input="onChangeZipLink" />
              </div>
            </div>

            <div class="file">
              <label class="file-label">
                <input class="file-input" name="resume" type="file" @change="onChangeZipFile" />
                <span class="file-cta">
                  <span class="file-icon">
                    <i class="fas fa-upload" />
                  </span>
                  <span class="file-label">{{
                    fieldsInput.zip?.file?.name != null ? fieldsInput.zip?.file?.name : 'Choose a file…'
                    }}</span>
                </span>
              </label>
            </div>

            <p v-if="error != null" class="help is-danger">{{ error }}</p>
          </div>

        </section>

        <footer class="modal-card-foot">
          <button v-if="modalType !== LoadAbiType.FROM_ZIP" :class="['button is-success', { 'is-loading': inProgress }]"
            :disabled="inProgress" @click="onSubmit">
            Submit
          </button>
          <button v-if="modalType === LoadAbiType.FROM_ZIP" :class="['button is-success', { 'is-loading': inProgress }]"
            :disabled="inProgress" @click="onSubmitZip">
            Submit
          </button>
          <button class="button" @click="closeModal">Cancel</button>
        </footer>
      </div>
    </div>
  </Teleport>
</template>
