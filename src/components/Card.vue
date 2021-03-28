<template>
  <div class="rounded-sm bg-white shadow-sm px-6 py-4 flex flex-col w-200">
    <div>
      <h1 class="text-xl font-semibold">Android Locale Parser</h1>
      <p class="text-sm text-gray-500">Enter the message you want to parse</p>
      <button @click="createZipFile()" :disabled="outputs.length == 0" class="btn">Download ZIP</button>
      <button @click="loadExample()" class="btn-new ml-2">Load an example</button>
    </div>
    <textarea
      v-model="state.value"
      rows="10"
      class="text-sm outline-none rounded-lg px-3 text-gray-700 py-2 border my-4 resize-none"
    />
    <template :key="item.locale" v-for="item in outputs">
      <h1 class="text-lg text-green-600 font-semibold">{{ item.locale }}</h1>
      <textarea
        :value="item.value"
        rows="10"
        class="code bg-gray-50 font-semibold text-xs outline-none rounded-lg px-3 text-gray-600 py-2 border my-4 resize-none"
      />
    </template>
  </div>
</template>

<script setup>
import { computed, reactive } from "vue";
import JSZip from 'jszip'
import { saveAs } from 'file-saver';

const locales = {
  khmer: "km",
  english: "en",
  chinese: "zh",
};


function transformLocale(locale) {
  if (!locale) return null;
  if (typeof locale !== "string") return null;
  const key = locale.toLowerCase();
  return locales[key] || locale;
}

const state = reactive({
  value: "",
});

function extractNameFromTitle(value) {
  const result = /#(\w+)/i.exec(value);
  if (!result) {
    return "";
  }
  return result[1];
}

const outputs = computed(() => {
  const lines = state.value.split("\n");

  const translations = {};

  let prevLine;
  let childPrevLine;

  lines
    .map((line) => {
      if (line.startsWith('@')) return null;
      if (/\@file:\s*(\w+)/gm.exec(line)) {
        return null;
      }

      const regex = /^(\w+):[^\S\r\n]*(.+)/gm;
      const matches = regex.exec(line);

      
      if (!matches) {
        if (line.trim().length != 0) {
          if (
            (prevLine && line.trim().startsWith("-")) ||
            line.trim().startsWith("+")
          ) {
            let offset = 0;
            if (line.trim().endsWith(":")) offset = 1;
            childPrevLine = line.slice(1, line.trim().length - offset).trim();
          } else {
            prevLine = line;
            childPrevLine = null;
          }
        }

        return null;
      }

      const locale = transformLocale(matches[1]);
      const value = matches[2].trim();
      const params = [];

      if (value) {
        let paramsMatch;
        const paramRegex = /\<([^>]+)\>/g;
        while ((paramsMatch = paramRegex.exec(value))) {
          params.push({
            value: paramsMatch[0],
            key: paramsMatch[1],
          });
        }
      }

      let newValue = value;
      if (params.length > 0) {
        params.forEach((param, index) => {
          newValue = newValue.replace(param.value, `%${index + 1}$s`);
        });
      }

      let id = "";
      let xmlValue = "";
      if (prevLine) {
        if (childPrevLine) {
          id = extractNameFromTitle(childPrevLine);
          xmlValue = `<!--[${prevLine}] - ${childPrevLine
            .replace("#" + id, "")
            .trim()} -->\n`;
        } else {
          id = extractNameFromTitle(prevLine);
          xmlValue = `<!-- [${prevLine.replace("#" + id, "").trim()}] -->\n`;
        }
      }

      xmlValue += `<string name="${id}">${newValue}</string>\n`;

      return {
        locale,
        xmlValue,
        label: prevLine,
      };
    })
    .filter(Boolean)
    .sort((a, b) => {
      if (a.locale > b.locale) {
        return 1;
      } else if (a.locale < b.locale) {
        return -1;
      } else {
        return 0;
      }
    })
    .forEach((item) => {
      if (!translations[item.locale]) {
        translations[item.locale] = [];
      }
      translations[item.locale].push(item.xmlValue);
    });

  return Object.keys(translations).map((it) => {
    return { locale: it, value: translations[it].join("\n") };
  });
});


async function createZipFile() {
  let filename = 'strings.xml';
  const filenameResult = /\@file:\s*(\w+)/gm.exec(state.value)
  if (filenameResult) {
    filename = filenameResult[1] + '.xml';
  }  
  const zip = new JSZip();
  outputs.value.forEach(item => {
    let localeKey = item.locale;
    if (localeKey === 'en') {
      localeKey = 'values' 
    } else {
      localeKey =`values-${localeKey.toLowerCase()}`
    }
    zip.folder(localeKey).file(filename, item.value);
  });

  const blob = await zip.generateAsync({ type: 'blob' });
  saveAs(blob, 'locales.zip');
}

function loadExample() {  
  const content = `@file: accounts
My Profile

- #title_account Accounts
km: គណនី
en: Account

- #message_balance Balance Message
km: ទឹកប្រាក់លោកអ្នកគឺ <amount>
en: Your balance is <amount>
`
state.value = content;
}

</script>


<style lang="postcss" scoped>
.btn {
  @apply transition disabled:cursor-auto disabled:shadow-none disabled:text-gray-400 bg-green-500 disabled:bg-gray-200 font-semibold hover:bg-green-600 shadow outline-none text-white rounded-lg px-4 py-2 mt-2;
}

.btn-new {
  @apply transition disabled:cursor-auto disabled:shadow-none disabled:text-gray-400 bg-blue-500 disabled:bg-gray-200 font-semibold hover:bg-blue-600 shadow outline-none text-white rounded-lg px-4 py-2 mt-2;
}
</style>