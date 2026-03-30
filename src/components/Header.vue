<script setup>
import { computed, ref } from 'vue';
import router from '../router/index.js';
import { store, loadData, unloadData, decompress } from '../store.js';
import { RouterLink, useRoute } from 'vue-router';
import ConfigMenu from './ConfigMenu.vue'
import SearchWindow from './SearchWindow.vue'
import { availableCoverageTypes, getPathChildren, getRate, hasTableForFile, pathType } from '../store.js';

const props = defineProps({
  date: String,
  branch: String,
  repo: String,
  commit: String,
  logo: String,
  title: String
})

const route = useRoute();
const exportStatus = ref('');

document.addEventListener("keyup", (e) => {
  if (e.ctrlKey && e.key == 'k') {
    e.preventDefault();
    store.showSearchWindow = !store.showSearchWindow;
  }
});

document.addEventListener("keydown", (e) => {
  if (e.ctrlKey && e.key == 'k') {
    e.preventDefault();
  }
});

const breadcrumbParts = computed(() => {
  const result = [];
  if (!route.params.path) {
    return result;
  }

  let accumulator = '';
  for (const part of route.params.path.split('/')) {
    result.push({
      name: part,
      target: `/${encodeURIComponent(accumulator + part)}`,
    });
    accumulator += `${part}/`;
  }
  return result;
});

// Filter out line/table highlight params when navigating
const queryWithoutHighlight = computed(() => {
  const { L, T, ...rest } = route.query;
  return rest;
});

const showFilePicker = Object.keys(originalFiles).length === 0;

const dataLoaded = computed(() => store.dataLoaded)
const currentPath = computed(() => route.params.path ?? "");

const currentViewKind = computed(() => {
  if (route.path === '/tables') {
    return 'tables';
  }
  if (!dataLoaded.value) {
    return 'none';
  }
  if (hasTableForFile(currentPath.value) && ((route.query?.showTable?.toLowerCase() === "true") ?? false)) {
    return 'table';
  }
  return pathType(currentPath.value) ?? 'overview';
});

const currentSummary = computed(() => {
  if (!dataLoaded.value) {
    return {};
  }
  return store.summaries[currentPath.value] ?? {};
});

function formatCoverageTable(summary) {
  const coverageTypes = availableCoverageTypes();
  const lines = [
    '| Coverage type | Hits | Total | Rate |',
    '| --- | ---: | ---: | ---: |',
  ];

  for (const type of coverageTypes) {
    const stats = summary[type] ?? { hits: 0, total: 0 };
    lines.push(`| ${type} | ${stats.hits ?? 0} | ${stats.total ?? 0} | ${getRate(stats)}% |`);
  }

  return lines.join('\n');
}

function formatChildEntriesTable(path) {
  const coverageTypes = availableCoverageTypes();
  const children = getPathChildren(path);
  const prefix = path ? `${path}/` : '';

  const items = children
    .map((name) => {
      const fullPath = `${prefix}${name}`;
      const summary = store.summaries[fullPath] ?? {};
      return {
        name,
        path: fullPath,
        kind: pathType(fullPath),
        summary,
      };
    })
    .sort((a, b) => {
      if (a.kind !== b.kind) {
        return a.kind === 'file' ? 1 : -1;
      }
      return a.name.localeCompare(b.name);
    });

  if (items.length === 0) {
    return '';
  }

  const header = ['Source', ...coverageTypes];
  const separator = ['---', ...coverageTypes.map(() => '---:')];
  const lines = [
    `| ${header.join(' | ')} |`,
    `| ${separator.join(' | ')} |`,
  ];

  for (const item of items) {
    const row = [`${item.kind === 'file' ? 'file' : 'dir'} \`${item.name}\``];
    for (const type of coverageTypes) {
      const stats = item.summary[type] ?? { hits: 0, total: 0 };
      row.push(`${getRate(stats)}%`);
    }
    lines.push(`| ${row.join(' | ')} |`);
  }

  return lines.join('\n');
}

function currentViewLabel() {
  switch (currentViewKind.value) {
    case 'tables':
      return 'All tables';
    case 'table':
      return `Table view: \`${currentPath.value}\``;
    case 'file':
      return `File view: \`${currentPath.value}\``;
    case 'dir':
      return currentPath.value ? `Directory view: \`${currentPath.value}\`` : 'Overview';
    default:
      return 'Overview';
  }
}

function currentSelectionLines() {
  const lines = [];

  if (store.selectedDataset) {
    lines.push(`- Dataset: \`${store.selectedDataset}\``);
  }
  if (route.query.L) {
    lines.push(`- Selected lines: \`${route.query.L}\``);
  }
  if (route.query.T) {
    lines.push(`- Selected table entry: \`${route.query.T}\``);
  }
  if (route.query.showTable === 'true' && currentPath.value) {
    lines.push(`- Table mode enabled for: \`${currentPath.value}\``);
  }

  return lines;
}

function buildCurrentViewMarkdown() {
  const lines = [
    `# ${props.title || 'Coverview'}`,
    '',
    `## ${currentViewLabel()}`,
    '',
  ];

  const selectionLines = currentSelectionLines();
  lines.push(...selectionLines);
  if (selectionLines.length > 0) {
    lines.push('');
  }

  if (dataLoaded.value && Object.keys(currentSummary.value).length > 0) {
    lines.push(formatCoverageTable(currentSummary.value));
    lines.push('');
  }

  if (currentViewKind.value === 'overview' || currentViewKind.value === 'dir') {
    const childTable = formatChildEntriesTable(currentPath.value);
    if (childTable) {
      lines.push('### Entries');
      lines.push('');
      lines.push(childTable);
      lines.push('');
    }
  }

  lines.push(`- Route: \`${route.fullPath}\``);
  if (props.repo) {
    lines.push(`- Repo: \`${props.repo}\``);
  }
  if (props.branch) {
    lines.push(`- Branch: \`${props.branch}\``);
  }
  if (props.commit) {
    lines.push(`- Commit: \`${props.commit.substring(0, 8)}\``);
  }

  return `${lines.join('\n').trim()}\n`;
}

async function copyCurrentViewMarkdown() {
  const markdown = buildCurrentViewMarkdown();

  try {
    await navigator.clipboard.writeText(markdown);
  } catch (_) {
    const textArea = document.createElement('textarea');
    textArea.value = markdown;
    textArea.style.position = 'fixed';
    textArea.style.opacity = '0';
    document.body.appendChild(textArea);
    textArea.focus();
    textArea.select();
    document.execCommand('copy');
    document.body.removeChild(textArea);
  }

  exportStatus.value = 'Copied markdown';
  window.setTimeout(() => {
    exportStatus.value = '';
  }, 2000);
}

async function onFileUpload(event) {
  const file = event.target.files[0];
  if (!file) {
    return;
  }

  let ext = file.name.split('.').pop();

  if (ext === 'zip' || ext == 'xz') {
    loadData(await decompress(file.stream(), ext), true)
  } else {
    let p = new Promise((resolve) => {
      const reader = new FileReader();
      reader.onloadend = () => resolve(reader.result);
      reader.readAsText(file);
    });
    loadData({ [file.name]: await p }, true)
  }

  store.loadedFromFile = true;

  // need to handle "404"
  router.push({path: '/', query: route.query });
};

function reset() {
  unloadData();
  router.push({path: "/", replace: true});
}

const returnFromAllTables = () => {
  const previous = window.history.state?.back;
  if (!previous || previous.startsWith('/tables')) {
    // Go back to the Overview page, if we would end up on `/tables` again
    // or exit Coverview entirely
    router.push({ path: '/', query: route.query });
  } else {
    router.go(-1);
  }
}
</script>

<template>
  <header>
    <nav class="navbar">
      <div class="info">
        <div class="nav-left">
          <a v-if="logo" href="./">
            <template class="logo" v-html="logo" v-inline-svg></template>
          </a>
          <span class="path-text">{{ title }}</span>
        </div>
        <div class="file-input-holder">
          <label v-if="showFilePicker" for="fileInput" class="file-input">
            <img src="../assets/upload.svg" alt="upload">
            <input type="file" id="fileInput" accept=".xz, .zip, .info" @change="onFileUpload($event)">
          </label>
          <button class="header-button" v-if="store.loadedFromFile" @click="reset"><img src="../assets/block.svg"></button>
        </div>
        <div class="nav-right">
          <span class="info-item" v-if="Object.entries(store?.tables ?? {}).length > 0">
            <img src="../assets/table.svg" alt="All tables" />
            <RouterLink class="info-text" :to="{ path: '/tables', query: queryWithoutHighlight }" v-if="route.path !== '/tables'">Show all tables</RouterLink>
            <a @click="returnFromAllTables" style="cursor: pointer;" v-else>Hide all tables</a>
          </span>
          <span class="info-item date" v-if="date">
            <img src="../assets/date.svg" alt="Date icon" />
            <span class="info-text">{{ date ? ((new Date(date)).toLocaleDateString('sv') + ' ' + (new Date(date)).toLocaleTimeString('sv')) : '?' }}</span>
          </span>
          <span class="info-item repo" v-if="repo">
            <img src="../assets/repo.svg" alt="Repo icon" />
            <a :href="repo" v-if="repo?.startsWith('https://') || repo?.startsWith('http://')">{{ repo?.split("/").pop() }}</a>
            <span class="info-text branch" v-else>{{ repo?.split("/").pop() }}</span>
          </span>
          <span class="info-item branch" v-if="branch">
            <img src="../assets/branch.svg" alt="Branch icon" />
            <span class="info-text branch">{{ branch }}</span>
          </span>
          <span class="info-item commit" v-if="commit">
            <img src="../assets/commit.svg" alt="Commit icon" />
            <span class="info-text commit">{{ commit.substring(0, 8) }}</span>
          </span>
          <span class="info-item metadata" v-if="store?.metadata?._additional">
            <img src="../assets/fingerprint.svg" alt="Metadata icon" :title="store.metadata._additional" />
          </span>
          <button
            class="header-button"
            v-if="dataLoaded"
            @click="copyCurrentViewMarkdown"
            :title="exportStatus || 'Copy current view as Markdown'"
            :aria-label="exportStatus || 'Copy current view as Markdown'"
          >
            <img class="copy-icon" src="../assets/copy.svg" alt="Copy Markdown">
          </button>
          <span v-if="exportStatus" class="copy-status">{{ exportStatus }}</span>
          <button class="header-button" v-if="dataLoaded" @click="store.showSearchWindow = true">
            <img src="../assets/search.svg">
          </button>
          <ConfigMenu v-if="dataLoaded"/>
        </div>
      </div>
      <ul class="breadcrumbs">
        <li>
          <RouterLink :to="{ path: '/', query: queryWithoutHighlight }">{{ store.metadata.repo?.split("/").pop() || 'Overview' }}</RouterLink>
        </li>
        <li v-for="part in breadcrumbParts">
          <img src="../assets/caret.svg" alt="caret" />
          <RouterLink :to="{ path: part.target, query: queryWithoutHighlight }">{{ part.name }}</RouterLink>
        </li>
      </ul>
    </nav>
  </header>
  <SearchWindow v-if="store.showSearchWindow"/>
</template>

<style scoped>
.navbar {
  display: flex;
  flex-direction: column;
  background-color: var(--bg-primary);
  border-bottom: 1px solid var(--border-primary);
}

.info {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 1rem;
  padding: 1rem;
 }

.nav-left {
  display: flex;
  align-items: center;
  gap: 1rem;
  min-width: 200px;
}

.file-input-holder {
  display: flex;
  gap: 1rem;
}

input[type="file"] {
  display: none;
}

.header-button {
  background: none;
  border: none;
  color: #d4d4d8;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  padding: 0;
}

.header-button:hover {
  color: #f5f5f5;
}

.header-button img {
  width: 1.25rem;
  height: 1.25rem;
}

.copy-icon {
  filter: brightness(0) saturate(100%) invert(88%) sepia(7%) saturate(174%) hue-rotate(202deg) brightness(94%) contrast(91%);
}

.header-button:hover .copy-icon {
  filter: brightness(0) saturate(100%) invert(98%) sepia(3%) saturate(271%) hue-rotate(269deg) brightness(113%) contrast(92%);
}

.copy-status {
  color: #86efac;
  font-size: 0.8rem;
  white-space: nowrap;
}

.logo {
  height: 2.5rem;
  width: auto;
}

.path-text {
  color: var(--text-primary);
  font-size: 0.875rem;
}

.nav-right {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.info-item {
  color: var(--text-muted);
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  text-decoration: none;
}

.info-item img {
  width: 1.25rem;
  height: 1.25rem;
}

@media (max-width: 768px) {
  .nav-left {
    min-width: auto;
  }

  .logo {
    height: 2.5rem;
  }

  .info-text.branch,
  .info-text.commit {
    display: none;
  }

  .info-item {
    padding: 0.5rem;
    border-radius: 4px;
  }

  .info-item img {
    width: 1rem;
    height: 1rem;
  }
}

ul {
  display: flex;
  padding: 1rem;
  list-style: none;
  border-top: 1px solid #27272A;
  color: #A1A1AA;
  gap: 0.375rem;
}

li {
  display: flex;
  gap: 0.375rem;
  align-items: center;
}

a {
  white-space: nowrap;
  color: #A1A1AA;
  text-decoration: none;
}

a:hover {
  color: #F4F4F5;
  text-decoration: underline;
}

.last {
  color: #F4F4F5;
}

li > img {
  width: 1rem;
  height: 1rem;
}

@media (max-width: 480px) {
  .path-text {
    display: none;
  }

  .nav-right {
    gap: 0.5rem;
    flex-wrap: wrap;
  }
}
</style>

<style>
header :is(button, .file-input) {
  border: none;
  background: none;
  padding: 2px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

header :is(button, .file-input):hover {
  cursor: pointer;
  background: var(--border-primary);
}

header :is(button, .file-input) img {
  width: 1.375rem;
  height: 1.375rem;
}
</style>
