<template>
  <el-container class="mx-auto px-5 py-20" style="width: 850px;">
    <el-header class="h-auto">
      <h1 class="font-bold text-xl">Web3 Heatmap</h1>
      <p class="mt-3 text-sm text-gray-500">This is a tool to generate your Web3 activity as a heatmap.</p>
      <p class="mt-1 mb-2 text-sm text-gray-500">This project is free and <a href="https://github.com/NaturalSelectionLabs/Web3-Heatmap" target="_blank">open source</a> for everyone can get their own heatmap.</p>
    </el-header>
    <el-main>
      <el-form :model="form" label-width="120px" @submit.prevent>
        <el-form-item label="ENS or Address">
          <el-input v-model="form.ensOrAddress" />
        </el-form-item>
        <el-form-item>
          <el-button :loading="loading" class="btn" type="primary" @click="generate">Generate</el-button>
        </el-form-item>
      </el-form>
      <el-divider />
      <div ref="heatmap" class="p-3">
        <h2 class="font-bold text-gray-700 pb-3">{{ form.ensOrAddress || 'Someone' }} on Web3</h2>
        <div v-for="calendar in calendars">
          <p class="mt-5 text-gray-700 text-sm mb-2">{{ calendar.year }}: {{ calendar.count }} activities</p>
          <div class="flex gap-1">
            <div v-for="week in calendar.calendar" class="flex gap-1 flex-col">
              <el-tooltip
                v-for="day in week" 
                :content="day.dayjs.format('YYYY-MM-DD')"
                placement="top"
                :show-after="100"
              >
                <div class="item bg-gray-200 cursor-pointer" :class="{
                  'bg-gray-200': day.count === 0,
                  'bg-blue-200': day.count > 0 && day.count < 2,
                  'bg-blue-400': day.count >= 2 && day.count < 5,
                  'bg-blue-600': day.count >= 5 && day.count < 10,
                  'bg-blue-900': day.count >= 10,
                }"></div>
              </el-tooltip>
            </div>
          </div>
        </div>
      </div>
      <el-button :loading="loading" class="btn mt-5" @click="download">Download</el-button>
      <el-button :loading="loading" type="primary" class="btn mt-5" @click="tweet">Share to Twitter</el-button>
    </el-main>
    <el-footer class="text-gray-500 text-xs mt-5">Powered by <a class="underline" href="https://rss3.io/">RSS3</a>.</el-footer>
  </el-container>
</template>

<script setup lang="ts">
import { reactive, ref } from 'vue'
import axios from 'axios'
import html2canvas from 'html2canvas'
import dayjs from 'dayjs'
import dayOfYear from 'dayjs/plugin/dayOfYear'
import weekOfYear from 'dayjs/plugin/weekOfYear'
dayjs.extend(dayOfYear)
dayjs.extend(weekOfYear)

const form = reactive({
  ensOrAddress: new URL(window.location.href).searchParams.get('id') || '',
});
const loading = ref(false);

const getYearCalendar = (year: number) => {
  const length = year % 4 === 0 ? 366 : 365;
  const calendar: any = [];
  for (let i = 1; i < length + 1; i++) {
    const day = dayjs(year + '').dayOfYear(i);
    if (day.diff(dayjs()) > 0) {
      break;
    }
    let week = day.week();
    if (i > 7 && week === 1) {
      if (calendar[calendar.length - 1].length === 7) {
        week = calendar.length + 1;
      } else {
        week = calendar.length;
      }
    }
    if (!calendar[week - 1]) {
      calendar[week - 1] = [];
    }
    calendar[week - 1].push({
      dayjs: day,
      count: 0,
    });
  }
  return calendar;
};

const currentYear = dayjs().year();

const calendars = ref([{
  year: currentYear,
  calendar: getYearCalendar(currentYear),
  count: 0,
}, {
  year: currentYear - 1,
  calendar: getYearCalendar(currentYear - 1),
  count: 0,
}]);

async function generate() {
  loading.value = true;

  history.pushState({}, '', '?id=' + form.ensOrAddress);

  calendars.value = [{
    year: currentYear,
    calendar: getYearCalendar(currentYear),
    count: 0,
  }, {
    year: currentYear - 1,
    calendar: getYearCalendar(currentYear - 1),
    count: 0,
  }];

  let address;
  if (!/^0x[a-fA-F0-9]{40}$/.test(form.ensOrAddress)) {
    let name = form.ensOrAddress;
    if (!/\.eth$/.test(name)) {
      name = name + '.eth';
    }
    address = (await axios.get(`https://rss3.domains/name/${name}`)).data.address;
  } else {
    address = form.ensOrAddress;
  }
  if (address) {
    let lastTime = dayjs();
    const expectedTime = dayjs(currentYear - 1 + '')
    let items: any = [];
    let last;
    while (lastTime.diff(expectedTime) > 0) {
      const response: any = (await axios.get(`https://pregod.rss3.dev/v0.4.0/account:${address}@ethereum/notes?limit=1000`, {
        params: {
          limit: 1000,
          last_identifier: last,
        }
      })).data;
      if (response.identifier_next) {
        last = new URL(response.identifier_next).searchParams.get('last_identifier')
      } else {
        last = null;
      }
      const newList = response.list;
      if (newList.length) {
        lastTime = dayjs(newList[newList.length - 1].date_created);
        items = items.concat(newList);
      }
      if (!last) {
        break;
      }
    }
    const calendarList = calendars.value;
    for (let i = 0; i < items.length; i++) {
      const day = dayjs(items[i].date_created);
      let week = day.week();
      let calendar;
      if (day.year() === currentYear) {
        calendar = calendarList[0];
      } else if (day.year() === currentYear - 1) {
        calendar = calendarList[1];
      } else {
        break;
      }
      const list = calendar.calendar;
      if (day.dayOfYear() > 7 && week === 1) {
        if (list[list.length - 1].length === 7) {
          week = list.length + 1;
        } else {
          week = list.length;
        }
      }
      const result = list[week - 1].find((item: any) => item.dayjs.isSame(day, 'day'))
      result.count++;
      calendar.count++;
    }
    calendars.value = calendarList;
  }
  loading.value = false;
}

if (form.ensOrAddress) {
  generate();
}

const heatmap = ref<HTMLElement | null>(null)
function download() {
  html2canvas(heatmap.value!).then(function (canvas) {
    const link = document.createElement('a');
    link.download = 'heatmap.png';
    link.href = canvas.toDataURL()
    link.click();
  });
}

function tweet() {
  window.open(`https://twitter.com/intent/tweet?text=Check%20out%20my%20Web3%20Heatmap&url=${window.location.href}&via=rss3_`, '_blank');
}
</script>

<style>
button.btn {
  background-color: var(--el-button-bg-color);
}

.item {
  width: 10px;
  height: 10px;
}
</style>
