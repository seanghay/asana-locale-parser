<template>
  <div class="rounded-sm bg-white shadow-sm px-6 py-4 flex flex-col w-200">
    <div>
      <h1 class="text-xl font-semibold">Android Locale Parser</h1>
      <p class="text-sm text-gray-500">Enter the message you want to parse</p>
      <div class="flex">
        <button
          @click="createZipFile()"
          :disabled="outputs.length == 0"
          class="btn"
        >
          Download ZIP
        </button>
        <button @click="loadExample()" class="btn-new ml-2">
          Load an example
        </button>
        <a href="https://git.io/JYZ9N" target="_blank" class="mt-4 ml-4">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
          >
            <path
              d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"
            />
          </svg>
        </a>
      </div>
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
import JSZip from "jszip";
import { saveAs } from "file-saver";

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
      if (line.startsWith("@")) return null;
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
  let filename = "strings.xml";
  const filenameResult = /\@file:\s*(\w+)/gm.exec(state.value);
  if (filenameResult) {
    filename = filenameResult[1] + ".xml";
  }
  const zip = new JSZip();
  outputs.value.forEach((item) => {
    let localeKey = item.locale;
    if (localeKey === "en") {
      localeKey = "values";
    } else {
      localeKey = `values-${localeKey.toLowerCase()}`;
    }

    const content = `<?xml version="1.0" encoding="utf-8"?>\n<resources>\n${item.value}</resources>`;
    zip.folder(localeKey).file(filename, content);
  });

  const blob = await zip.generateAsync({ type: "blob" });
  saveAs(blob, "locales.zip");
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
`;
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