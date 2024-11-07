<script setup>
import { ref, reactive } from 'vue'

// defineProps({
//   msg: String,
// })

const count = ref(0);

const treelist = ref([
  { name: "foo", handle: null },
  { name: "bar", handle: null },
]);

const hdir = ref(null);
const editor = reactive({
  text: "",
  hfile: null,
  name: "untitled",
});

const cssExplorerWidth = ref(`15em`);
const cssEditorWidth = ref(`calc(100% - 15em)`);

async function openDirectory() {
  hdir.value = await window.showDirectoryPicker({ mode: "readwrite" });

  treelist.value = [];
  for await (let [name, handle] of hdir.value) {
    treelist.value.push({ 
      name: name, 
      handle: handle, 
      file: handle.kind === "file",
      level: 0,
    });
  }
}

async function openFile(item) {
  editor.hfile = item.handle;
  editor.name = item.name;

  const file = await item.handle.getFile();
  const reader = new FileReader();
  reader.addEventListener("load", () => {
    editor.text = reader.result;
  });
  reader.readAsText(file);
}

const editorTextarea = ref(null);
function onTab() {
  let obj = editorTextarea.value;
	var cursorPosition = obj.selectionStart;
	var cursorLeft     = obj.value.substr( 0, cursorPosition );
	var cursorRight    = obj.value.substr( cursorPosition, obj.value.length );
	obj.value = cursorLeft+"  "+cursorRight;
	obj.selectionEnd = cursorPosition+2;
}

async function saveFile() {
  let writableStream = await editor.hfile.createWritable();
  await writableStream.write(editor.text);
  await writableStream.close();
  console.log("file saved");
}

</script>

<template>
  <div class="container root flex-row">
    <div class="tree child w10 p1 bc-gray bl bt bb flex-col">
      <div>explorer <span class="clickable" @click="openDirectory">[open dir]</span></div>
      <br/>
      <div v-for="item in treelist" class="clickable" @click="openFile(item)">
        {{ "&nbsp;".repeat(item.level) }}{{ item.name }} {{ (item.file)? "" : "/" }}
      </div>
    </div>
    <div class="editor child w20 p1 bc-gray bl bt bb br">
      <textarea ref="editorTextarea" class="editor-textarea" 
        v-model="editor.text"
        spellcheck="false"
        @keydown.tab.prevent="onTab"
        @keydown.ctrl.s.prevent="saveFile"></textarea>
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

.w10 { width: 10em; }
.w20 { width: 20em; }
.h10 { height: 10em; }
.h20 { height: 20em; }

.p1 { padding: 1em; }

.bg-red { background-color: red; }

.bc-light { border-color: #ccc !important; }
.bc-gray { border-color: #888 !important; }

.bl { border-left: solid 1px; }
.br { border-right: solid 1px; }
.bt { border-top: solid 1px; }
.bb { border-bottom: solid 1px; }

.flex-row { flex-direction: row; }
.flex-col { flex-direction: column; }

.tree {
  width: v-bind(cssExplorerWidth);
}

.editor {
  width: v-bind(cssEditorWidth);
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
</style>
