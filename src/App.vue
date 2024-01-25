<script setup lang="ts">
import { getVersion, getTauriVersion, getName } from "@tauri-apps/api/app"
import { save, open, message, confirm } from "@tauri-apps/api/dialog"
import { readTextFile } from "@tauri-apps/api/fs"
import { invoke } from "@tauri-apps/api/tauri"
import { ResponseType, fetch } from '@tauri-apps/api/http'

import { marked } from "marked"
import { ref, computed } from "vue"

import type { Ref, ComputedRef } from "vue"

// i could probably format this better
const input: Ref<string> = ref("# markdown\n\na simple markdown editor.\n\neditor on the right, preview on the left.")
const output: ComputedRef<string | Promise<string>> = computed(() => marked(input.value))

const httpLink: Ref<string> = ref("")

const update = (e: any) => {
  input.value = e.target.value
}

const saveFile = async () => {
  try {
    const savePath = await save({
      filters: [{
        name: "Markdown",
        extensions: ["md"]
      }]
    });

    if (!savePath) {
      return;
    }

    await invoke("save_file", {
      path: savePath,
      contents: input.value
    })
  } catch (err) {
    console.error(err)
  }
}

const readFile = async () => {
  try {
    const filePath = await open({
      multiple: false,
      filters: [{
        name: 'Markdown',
        extensions: ["md"]
      }]
    });

    if (!filePath) {
      return;
    }
    
    input.value = await readTextFile(filePath as string)
  } catch(err) {
    console.error(err)
  }
}

const openAppInfo = async () => {
  const name = await getName();
  const version = await getVersion();
  const tauriVersion = await getTauriVersion();

  message(`${name} v${version}\ntauri v${tauriVersion}\n\nby fakexale\nbuilt with tauri and vue `, `About ${name}`);
}

const requestHttp = async () => {
  const confirmed = await confirm("If you have any text in editor, it will be replaced by what the HTTP request fetches.\n\nContinue?", { "title": "markdown", "type": "warning" })

  if (!confirmed) {
    return
  }

  const request = await fetch(httpLink.value, {
    method: "GET",
    timeout: 60,
    responseType: ResponseType.Text 
  })

  const result = request.data as string
  console.log(result)

  input.value = result
}
</script>

<template>
  <div class="fileModal">
      <button class="button" @click="saveFile">Save</button>
      <button class="button" @click="readFile">Open</button>
      
      <button class="button" @click="openAppInfo">About</button>
  </div>

  <div class="requestModal">
    <input v-model="httpLink" class="textbox" placeholder="Enter a link to fetch markdown from...">
    <button class="button" @click="requestHttp">Fetch</button>
  </div>
  
  <div class="editor">
    <textarea class="input" :value="input" @input="update"></textarea>
    <div class="output" v-html="output"></div>
  </div>
</template>
