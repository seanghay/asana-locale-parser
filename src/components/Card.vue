<template>
  <div class="rounded-sm bg-white shadow-sm px-6 py-4 flex flex-col w-200">
    <h1 class="text-xl font-semibold">Asana Locale Parser</h1>
    <p class="text-sm text-gray-500">Enter the message you want to parse</p>
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

const locales = {
  khmer: "km",
  english: "en",
  chinese: "zh",
};

function transformLocale(locale) {
  if (!locale) return null;
  if (typeof locale !== "string") return null;
  const key = locale.toLowerCase();
  return locales[key];
}

const state = reactive({
  value: "",
});

const outputs = computed(() => {
  const lines = state.value.split("\n");

  const translations = {};

  let prevLine;
  let childPrevLine;

  lines
    .map((line) => {
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

      let xmlValue = `<string name="">${newValue}</string>\n`;

      if (prevLine) {
        if (childPrevLine) {
          xmlValue = `<!--[${prevLine}] - ${childPrevLine} -->\n` + xmlValue;
        } else {
          xmlValue = `<!-- [${prevLine}] -->\n` + xmlValue;
        }
      }

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
</script>
