<script setup>
import { ref, reactive, onMounted, nextTick } from "vue";
import * as monaco from "monaco-editor";
import editorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker&inline";
import jsonWorker from "monaco-editor/esm/vs/language/json/json.worker?worker&inline";
import cssWorker from "monaco-editor/esm/vs/language/css/css.worker?worker&inline";
import htmlWorker from "monaco-editor/esm/vs/language/html/html.worker?worker&inline";
import tsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker&inline";
import uuid4 from "../uuid4.js";
import prettier from "prettier/standalone";
import prettierPluginHtml from "prettier/plugins/html";
import prettierPluginEstree from "prettier/plugins/estree";
import prettierPluginBabel from "prettier/plugins/babel";
import storageUtil from "../storage-util.js";

const fontSize = ref(storageUtil.getStorage("fontSize", 13));

self.MonacoEnvironment = {
  getWorker(_, label) {
    if (label === "json") {
      return new jsonWorker();
    }
    if (label === "css" || label === "scss" || label === "less") {
      return new cssWorker();
    }
    if (label === "html" || label === "handlebars" || label === "razor") {
      return new htmlWorker();
    }
    if (label === "typescript" || label === "javascript") {
      return new tsWorker();
    }
    return new editorWorker();
  },
};

// defineProps({
//   msg: String,
// })

const editorMonaco = ref(null);
let editorMonacoObj = null;

// set theme
monaco.editor.setTheme("vs-dark");

let ps;
let formatVue;
let formatJs;
(async () => {
  ps = prettier; //await import('prettier/standalone');
  const plugins = [
    prettierPluginHtml, //await import('prettier/plugins/html'),
    prettierPluginEstree, //await import('prettier/plugins/estree'),
    prettierPluginBabel, //await import('prettier/plugins/babel'),
  ];

  /*
  console.log(await ps.format(`
<div>
            <span>aaa</span>
</div>
`, { parser: "html", plugins, tabWidth: 2 }));


  console.log(await ps.format(`
function foo() {
          console.log(1234);
}
`, { parser: "babel", plugins, tabWidth: 2 }));
  */

  formatVue = async (srcText) => {
    return await ps.format(srcText, {
      parser: "vue",
      plugins,
      tabWidth: 2,
    });
  };

  formatJs = async (srcText) => {
    return await ps.format(srcText, {
      parser: "babel",
      plugins,
      tabWidth: 2,
    });
  };
})();

function createMonacoEditor(
  targetElement,
  onChange,
  language,
  initialValue = null,
) {
  let editor = monaco.editor.create(targetElement, {
    value: initialValue != null ? initialValue : "",
    language: language,
    automaticLayout: true,
    fontSize: fontSize.value,
    tabSize: 2,
  });

  // set tab width
  editor.getModel().updateOptions({ tabSize: 2 });

  // // add listener
  // editor.getModel().onDidChangeContent((event) => {
  //   onChange(editor, event);
  // });

  // setup formatter
  setTimeout(() => {
    console.log("vue formatter loaded!");
    monaco.languages.registerDocumentFormattingEditProvider("html", {
      async provideDocumentFormattingEdits(model) {
        let text = await formatVue(model.getValue());
        return [
          {
            range: model.getFullModelRange(),
            text,
          },
        ];
      },
    });
  }, 1000);

  setTimeout(() => {
    console.log("js formatter loaded!");
    monaco.languages.registerDocumentFormattingEditProvider("javascript", {
      async provideDocumentFormattingEdits(model) {
        let text = await formatJs(model.getValue());
        return [
          {
            range: model.getFullModelRange(),
            text,
          },
        ];
      },
    });
  }, 1000);

  return editor;
}

onMounted(() => {
  /*
  editorMonacoObj = createMonacoEditor(editorMonaco.value, 
    (ed, event) => {
      // console.log(ed.getValue());
      editor.text = ed.getValue();
      setSaveTimeout();
    },
    "javascript");
  */
});

function languageByExtension(extension) {
  let langs = {
    js: "javascript",
    html: "html",
    vue: "html",
    css: "css",
    txt: "txt",
  };
  return langs[extension];
}

const count = ref(0);

const treelist = ref([]);

const hdir = ref(null);
/*
const editor = reactive({
  text: "",
  hfile: null,
  name: "untitled",
  entry: null,
});
*/

const cssExplorerWidth = ref(`18em`);
const cssEditorWidth = ref(`calc(100% - 18em)`);

async function openDirectory() {
  hdir.value = await window.showDirectoryPicker({ mode: "readwrite" });

  treelist.value = [];
  let workspace = {
    name: "workspace",
    handle: hdir.value,
    file: false,
    level: 0,
    parent: null,
    expand: false,
  };
  treelist.value.push(workspace);

  expandTree(workspace);

  // for await (let [name, handle] of hdir.value) {
  //   treelist.value.push({
  //     name: name,
  //     handle: handle,
  //     file: handle.kind === "file",
  //     level: workspace.level + 1,
  //     parent: workspace,
  //     expand: false
  //   });
  // }
}

async function deleteEntry() {
  if (!editor.entry) return;
  if (!editor.entry.file) return;
  if (!editor.entry.parent) return;

  if (!window.confirm("Delete opened file, are you sure?")) return;

  // editor.entry.parent.removeEntry(editor.entry.name);
  editor.entry.handle.remove();
}

async function createNewFile(filename, parentItem) {
  if (!parentItem) return;
  if (parentItem.file) return;

  let hfile = await parentItem.handle.getFileHandle(filename, { create: true });
  let writableStream = await hfile.createWritable();
  await writableStream.write("");
  await writableStream.close();
  console.log("file created");
  await foldTree(parentItem);
  await expandTree(parentItem);
  return hfile;
}

const editorTextarea = ref(null);
function onTab() {
  let obj = editorTextarea.value;
  var cursorPosition = obj.selectionStart;
  var cursorLeft = obj.value.substr(0, cursorPosition);
  var cursorRight = obj.value.substr(cursorPosition, obj.value.length);
  obj.value = cursorLeft + "  " + cursorRight;
  obj.selectionEnd = cursorPosition + 2;
}

async function saveFile(hfile, text) {
  let writableStream = await hfile.createWritable();
  await writableStream.write(text);
  await writableStream.close();
  console.log("file saved");
}

async function expandTree(item) {
  let idx = treelist.value.indexOf(item);
  let subtree = [];
  let subtreeDirs = [];
  for await (let [name, handle] of item.handle) {
    let addTarget = handle.kind === "file" ? subtree : subtreeDirs;
    addTarget.push({
      name: name,
      handle: handle,
      file: handle.kind === "file",
      level: item.level + 1,
      parent: item,
      expand: false,
    });
  }

  subtree.sort((a, b) => {
    return a.name === b.name ? 0 : a.name > b.name ? 1 : -1;
  });
  subtreeDirs.sort((a, b) => {
    return a.name === b.name ? 0 : a.name > b.name ? 1 : -1;
  });

  treelist.value.splice(idx + 1, 0, ...[...subtreeDirs, ...subtree]);
  item.expand = true;
}

function prompt() {
  return window.prompt();
}

async function foldTree(item) {
  if (item.file) return;
  let children = treelist.value.filter((i) => i.parent == item);
  children.forEach(foldTree);
  treelist.value = treelist.value.filter((i) => i.parent != item);
  item.expand = false;
}

async function toggleTreeExpand(item) {
  if (item.expand) {
    foldTree(item);
  } else {
    expandTree(item);
  }
}

const selectedItem = ref(null);

async function clickTreeItem(item) {
  if (item.file) {
    openFile(item);
  }

  selectedItem.value = item;
}

function setSaveTimeout(tabitem, force = false) {
  if (!autosaveEnabled.value && !force) return;

  if (tabitem.timeoutSave) {
    clearTimeout(tabitem.timeoutSave);
  }

  tabitem.timeoutSave = setTimeout(() => {
    saveFile(tabitem.entry.handle, tabitem.editor.getValue());
    tabitem.timeoutSave = null;
    let ti = tabs.tabitems[tabs.tabitems.indexOf(tabitem)];
    ti.modified = false;
  }, 2000);
}

const autosaveEnabled = ref(false);

// let tabfirst = { entry: { name: "Main.vue", hfile: null, }, preview: false, };
const tabs = reactive({
  tabitems: [
    // tabfirst,
    // { entry: { name: "index.html", hfile: null, }, preview: false, },
    // { entry: { name: "vite.svg", hfile: null, }, preview: false, },
  ],
  selected: null, //tabfirst,
});

async function addTab(item) {
  const file = await item.handle.getFile();
  const reader = new FileReader();
  reader.addEventListener("load", () => {
    // editor.text = reader.result;
    let ext = item.name.split(".");
    if (ext.length > 0 && languageByExtension(ext.slice(-1)[0])) {
      var model = editorMonacoObj.getModel();
      monaco.editor.setModelLanguage(
        model,
        languageByExtension(ext.slice(-1)[0]),
      );
    }
    editorMonacoObj.setValue(reader.result);
  });
  reader.readAsText(file);
}

async function openFile(item) {
  //editor.hfile = item.handle;
  //editor.name = item.name;
  //editor.entry = item;

  let existing = tabs.tabitems.filter((t) => t.entry == item);
  if (existing.length > 0) {
    tabs.selected = existing[0];
    return;
  }

  let tabitem = {
    oid: uuid4(),
    entry: item,
    preview: false,
    modified: false,
  };

  tabs.tabitems.push(tabitem);
  tabs.selected = tabitem;
  let ti = tabs.tabitems[tabs.tabitems.indexOf(tabitem)];

  nextTick(async () => {
    const file = await item.handle.getFile();
    const reader = new FileReader();
    reader.addEventListener("load", () => {
      // editor.text = reader.result;

      let ext = item.name.split(".");

      let language = "text";

      if (ext.length > 0 && languageByExtension(ext.slice(-1)[0])) {
        // var model = editor.getModel();
        // monaco.editor.setModelLanguage(model, languageByExtension(ext.slice(-1)[0]));
        language = languageByExtension(ext.slice(-1)[0]);
      }

      let editor = createMonacoEditor(
        /*editorMonaco.value[tabs.tabitems.indexOf(tabitem)], */
        document.querySelector(`#editor-${ti.oid}`),
        (ed, val) => {
          ti.modified = true;
          setSaveTimeout(tabitem);
        },
        language,
      );
      tabitem.editor = editor;

      editor.setValue(reader.result);

      // add listener
      editor.getModel().onDidChangeContent((event) => {
        ((ed, val) => {
          ti.modified = true;
          setSaveTimeout(tabitem);
        })(editor, event);
      });
    });
    reader.readAsText(file);
  });
}

function clickTab(tabitem) {
  tabs.selected = tabitem;
}

function deleteTab(tabitem) {
  let idx = tabs.tabitems.indexOf(tabitem);
  tabs.tabitems = tabs.tabitems.filter((t) => t != tabitem);
  nextTick(() => {
    // tabitem.editor.getModel().dispose();
    // tabitem.editor.dispose();
    if (idx > 0) {
      tabs.selected = tabs.tabitems[idx - 1];
    } else if (tabs.tabitems.length > 0) {
      tabs.selected = tabs.tabitems[0];
    }
  });
}

const showConfig = ref(false);

function changeFontSize(value) {
  try {
    let ivalue = parseInt(value, 10);
    fontSize.value = ivalue;
    storageUtil.setStorage("fontSize", fontSize.value);
  } catch (ex) {}
}

function openConfig() {
  showConfig.value = true;
}

function closeConfig() {
  showConfig.value = false;
}
</script>

<template>
  <div class="container root flex-row" :style="{ fontSize: `${fontSize}px` }">
    <div class="tree child w10 p1 bc-gray bl bt bb flex-col">
      <div class="child flex-row">
        <span class="clickable button1" @click="openDirectory">Open</span>
        <span
          class="clickable button1 ml1"
          @click="createNewFile(prompt(), selectedItem)"
          >New</span
        >
        <span class="clickable button1 ml1" @click="deleteEntry">Delete</span>
        <span class="clickable button1 ml1" @click="openConfig">Config</span>
      </div>
      <div>
        <label
          ><input type="checkbox" v-model="autosaveEnabled" />Autosave</label
        >
      </div>
      <br />
      <div
        v-for="item in treelist"
        class="clickable"
        @click="clickTreeItem(item)"
      >
        <span style="opacity:0.3">{{
          "&nbsp;|&nbsp;".repeat(item.level)
        }}</span>
        <span
          v-if="!item.file"
          class="folder"
          @click.stop="toggleTreeExpand(item)"
        >
          {{ item.expand ? "-" : "+" }}&nbsp;
        </span>
        <span
          :style="{ textDecoration: selectedItem == item ? 'underline' : '' }"
          >&nbsp;{{ item.name }}</span
        >
        {{ item.file ? "" : "/" }}
      </div>
    </div>
    <div class="editor child w20 p1 bc-gray bl bt bb br flex-col">
      <div class="child flex-row">
        <div
          :class="[
            'tabitem',
            'clickable',
            'br',
            'bc-gray',
            tabitem != tabs.selected ? 'bb' : '',
          ]"
          v-for="tabitem in tabs.tabitems"
          @click="clickTab(tabitem)"
        >
          <span class="tabname"
            >{{ tabitem.modified ? "*" : "" }}{{ tabitem.entry.name }}</span
          >
          <span v-if="tabs.selected == tabitem" @click.stop="deleteTab(tabitem)"
            >X</span
          >
        </div>
        <!--
        <div class="tabitem br bb bc-gray">index.html</div>
        <div class="tabitem br bb bc-gray">vite.svg</div>
        -->
      </div>
      <template v-for="tabitem in tabs.tabitems" :key="tabitem.oid">
        <div
          :id="`editor-${tabitem.oid}`"
          :style="{
            width: `100%`,
            height: `calc(100% - 1.5em)`,
            display: tabitem == tabs.selected ? `inherit` : `none`,
          }"
          @keydown.ctrl.s.prevent.stop="setSaveTimeout(tabitem, true)"
        ></div>
      </template>
    </div>
  </div>

  <div v-if="showConfig" class="config-screen-container">
    <div class="config-container">
      <span>Configuration</span>
      <br />
      <label
        >FontSize:&nbsp;
        <input
          :value="fontSize"
          @change="changeFontSize($event.target.value)"
        />
      </label>
      <br />
      <span class="clickable button1" @click="closeConfig">Close</span>
    </div>
  </div>
</template>

<style scoped>
.root {
  width: 100%;
  height: 100%;
}

.container {
  display: flex;
}

.child {
  display: flex;
}

.w10 {
  width: 10em;
}

.w20 {
  width: 20em;
}

.h10 {
  height: 10em;
}

.h20 {
  height: 20em;
}

.p1 {
  padding: 1em;
}

.bg-red {
  background-color: red;
}

.bc-light {
  border-color: #ccc !important;
}

.bc-gray {
  border-color: #888 !important;
}

.bl {
  border-left: solid 1px;
}

.br {
  border-right: solid 1px;
}

.bt {
  border-top: solid 1px;
}

.bb {
  border-bottom: solid 1px;
}

.flex-row {
  flex-direction: row;
}

.flex-col {
  flex-direction: column;
}

.tree {
  width: v-bind(cssExplorerWidth);
  /* font-family: monospace; */
  overflow: auto;
  white-space: pre;
}

.editor {
  width: v-bind(cssEditorWidth);
  padding: 0;
}

.editor-textarea {
  width: 100%;
  height: 100%;
  border: none;
  background-color: #333;
  resize: none;
  outline: none;
  white-space: pre;
  font-size: 0.95em;
  line-height: 1.2em;
}

.clickable {
  cursor: pointer;
}

.button1 {
  display: inline-block;
  min-width: 3em;
  background-color: #444;
  text-align: center;
  vertical-align: middle;
  font-size: 0.9em;
  padding: 3px;
}

.ml1 {
  margin-left: 1em;
}

.mr1 {
  margin-right: 1em;
}

.mt1 {
  margin-top: 1em;
}

.mb1 {
  margin-bottom: 1em;
}

.folder {
  display: inline-block;
  width: 1em;
  text-align: center;
}

.tabitem {
  display: inline-block;
  height: 1.5em;
  text-align: center;
  vertical-align: middle;
  box-sizing: border-box;
  padding: 0 1em;
}

.tabname {
  padding-right: 0.5em;
}

.config-screen-container {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.config-container {
  background-color: #333;
  border: solid 1px #555;
  z-index: 99999;
  padding: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
</style>
