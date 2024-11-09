<script setup>
  import { ref, reactive, onMounted } from "vue";
import * as monaco from "monaco-editor";
import editorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker&inline";
import jsonWorker from "monaco-editor/esm/vs/language/json/json.worker?worker&inline";
import cssWorker from "monaco-editor/esm/vs/language/css/css.worker?worker&inline";
import htmlWorker from "monaco-editor/esm/vs/language/html/html.worker?worker&inline";
import tsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker&inline";

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

function createMonacoEditor(targetElement, onChange, language, initialValue = null) {
  let editor = monaco.editor.create(targetElement, {
    value: initialValue != null ? initialValue : "",
    language: language,
    automaticLayout: true,
    fontSize: 13,
    tabSize: 2,
  });

  // set tab width
  editor.getModel().updateOptions({ tabSize: 2 });

  // set theme
  monaco.editor.setTheme("vs-dark");

  // add listener
  editor.getModel().onDidChangeContent((event) => {
    onChange(editor, event);
  });
  
  return editor;
}

onMounted(() => {
  editorMonacoObj = createMonacoEditor(editorMonaco.value, 
    (ed, event) => {
      // console.log(ed.getValue());
      editor.text = ed.getValue();
      setSaveTimeout();
    },
    "javascript");
});

function languageByExtension(extension) {
  let langs = {
    "js" : "javascript",
    "html" : "html",
    "vue" : "html",
    "css" : "css",
    "txt" : "txt",
  }
  return langs[extension];
}

const count = ref(0);

const treelist = ref([]);

const hdir = ref(null);
const editor = reactive({
  text: "",
  hfile: null,
  name: "untitled",
  entry: null,
});

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

async function openFile(item) {
  editor.hfile = item.handle;
  editor.name = item.name;
  editor.entry = item;

  const file = await item.handle.getFile();
  const reader = new FileReader();
  reader.addEventListener("load", () => {
    // editor.text = reader.result;
    let ext = item.name.split(".");
    if(ext.length > 0 && languageByExtension(ext.slice(-1)[0])) {
      var model = editorMonacoObj.getModel();
      monaco.editor.setModelLanguage(model, languageByExtension(ext.slice(-1)[0]));
    }
    editorMonacoObj.setValue(reader.result);
  });
  reader.readAsText(file);
}

async function deleteEntry() {
  if(!editor.entry) return;
  if(!editor.entry.file) return;
  if(!editor.entry.parent) return;

  if(!window.confirm("Delete opened file, are you sure?")) return;

  // editor.entry.parent.removeEntry(editor.entry.name);
  editor.entry.handle.remove();
}

async function createNewFile(filename, parentItem) {
  if(!parentItem) return;
  if(parentItem.file) return;

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

async function saveFile() {
  let writableStream = await editor.hfile.createWritable();
  await writableStream.write(editor.text);
  await writableStream.close();
  console.log("file saved");
}

async function expandTree(item) {
  let idx = treelist.value.indexOf(item);
  let subtree = [];
  for await (let [name, handle] of item.handle) {
    subtree.push({
      name: name,
      handle: handle,
      file: handle.kind === "file",
      level: item.level + 1,
      parent: item,
      expand: false,
    });
  }
  treelist.value.splice(idx + 1, 0, ...subtree);
  item.expand = true;
}

function prompt() {
  return window.prompt();
}

async function foldTree(item) {
  if(item.file) return;
  let children = treelist.value.filter(i => i.parent == item);
  children.forEach(foldTree);
  treelist.value = treelist.value.filter(i => i.parent != item);
  item.expand = false;
}

async function toggleTreeExpand(item) {
  if(item.expand) {
    foldTree(item);
  } else {
    expandTree(item);
  }
}

const selectedItem = ref(null);

async function clickTreeItem(item) {
  if(item.file) {
    openFile(item)
  //} else {
  //  toggleTreeExpand(item);
  }

  selectedItem.value = item;
}

const timeoutSave = ref(null);

function setSaveTimeout() {
  if(!autosaveEnabled.value) return;
  
  if(timeoutSave.value) {
    clearTimeout(timeoutSave.value);
  }

  timeoutSave.value = setTimeout(() => {
    saveFile();
    timeoutSave.value = null;
  }, 2000);
}

const autosaveEnabled = ref(false);

</script>

<template>
  <div class="container root flex-row">
    <div class="tree child w10 p1 bc-gray bl bt bb flex-col">
      <div class="child flex-row">
        <span class="clickable button1" @click="openDirectory">OPEN</span>
        <span class="clickable button1 ml1" @click="createNewFile(prompt(), selectedItem)">NEW</span>
        <span class="clickable button1 ml1" @click="deleteEntry">DEL</span>
      </div>
      <div>
        <label><input type="checkbox" v-model="autosaveEnabled"/>autosave</label>
      </div>
      <br />
      <div v-for="item in treelist" class="clickable" @click="clickTreeItem(item)">

        <span style="opacity:0.3">{{ "&nbsp;|&nbsp;".repeat(item.level) }}</span>
        <span v-if="!item.file" class="folder"
          @click.stop="toggleTreeExpand(item)">
          {{ (item.expand)? "-" : "+" }}&nbsp;
        </span>
        <span :style="{ textDecoration: (selectedItem == item)? 'underline' : '' }">&nbsp;{{ item.name }}</span>
        {{ item.file ? "" : "/" }}
      </div>
    </div>
    <div ref="editorMonaco" class="editor child w20 p1 bc-gray bl bt bb br" @keydown.ctrl.s.prevent="saveFile">
      <textarea
        ref="editorTextarea"
        class="editor-textarea"
        v-model="editor.text"
        spellcheck="false"
        @keydown.tab.prevent="onTab"
        @keydown.ctrl.s.prevent="saveFile"
        style="display:none"
      ></textarea>
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
</style>