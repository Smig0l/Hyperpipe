<script setup>
import { reactive, ref, onMounted, onUnmounted } from 'vue';

import { useResults } from '@/stores/results.js';

import { getJsonHyp } from '@/scripts/fetch.js';
import { useRoute } from '@/scripts/util.js';
import { useI18n } from '@/stores/misc.js';

import AlbumItem from './AlbumItem.vue';

const props = defineProps(['id']),
  { getAlbum } = useResults(),
  { t } = useI18n();

const data = reactive({
    title: '',
    spotlight: [],
    featured: [],
    community: [],
  }),
  btns = reactive({
    moods: [],
    genres: [],
  }),
  todo = ['title', 'community', 'spotlight', 'featured'];

function get(id) {
  if (props.id || typeof id == 'string') {
    getJsonHyp('/genres/' + (props.id || id)).then(res => {
      console.log(res);

      for (let i of todo) {
        data[i] = res[i];
      }

      useRoute('/explore/' + (props.id || id));
    });
  } else {
    getJsonHyp('/genres').then(res => {
      console.log(res);

      for (let i of todo) {
        data[i] = undefined;
      }

      for (let i of ['moods', 'genres']) {
        btns[i] = res[i];
      }

      useRoute('/explore/');
    });
  }
}

onMounted(get);
</script>

<template>
  <template v-if="data.title">
    <i class="bi bi-arrow-left back" @click="get"> {{ t('action.back') }}</i>

    <h2 class="head">{{ data.title }}</h2>

    <template v-for="type in ['featured', 'spotlight', 'community']">
      <h3 class="head">{{ t('title.' + type) }}</h3>
      <div class="grid-3">
        <template v-for="i in data[type]">
          <AlbumItem
            :name="i.title"
            :author="i.subtitle"
            :art="i.thumbnails[0].url"
            @open-album="
              getAlbum('/playlist?list=' + i.id);
              nav.state.page = 'home';
            " />
        </template>
      </div>
    </template>
  </template>

  <template v-else>
    <h2 v-if="btns.moods.length > 0">{{ t('title.moods') }}</h2>

    <div class="btn-grid">
      <button
        v-for="i in btns.moods"
        class="btn"
        :style="`--btn-color: ${i.subtitle};`"
        @click="get(i.id)">
        {{ i.title }}
      </button>
    </div>

    <h2 v-if="btns.genres.length > 0">{{ t('title.genres') }}</h2>

    <div class="btn-grid">
      <button
        v-for="i in btns.genres"
        class="btn"
        :style="`--btn-color: ${i.subtitle};`"
        @click="get(i.id)">
        {{ i.title }}
      </button>
    </div>
  </template>
</template>

<style scoped>
.back {
  width: fit-content;
}
.btn-grid {
  display: grid;
  grid-template-columns: calc(100% / 3) calc(100% / 3) calc(100% / 3);
  grid-auto-rows: 1fr;
  grid-gap: 0.125rem;
  padding: 1rem 0;
}
.btn {
  aspect-ratio: 1;
  padding: 0.75rem;
  border-radius: 0.25rem;
  font-size: 1rem;
  font-weight: bold;
  letter-spacing: 0.125rem;
  background-color: var(--color-background-mute);
  color: var(--btn-color);
  transition: background-color 0.1s ease;
}
.btn:hover {
  background-color: var(--color-background-soft);
}
.btn:active {
  background-color: var(--color-border);
}
.head {
  margin-top: 1.75rem;
  text-align: center;
  text-transform: capitalize;
}

@media (min-width: 760px) {
  .btn-grid {
    grid-template-columns: calc(100% / 4) calc(100% / 4) calc(100% / 4) calc(
        100% / 4
      );
  }
}

@media (min-width: 1024px) {
  .btn-grid {
    grid-template-columns:
      calc(100% / 5) calc(100% / 5) calc(100% / 5) calc(100% / 5)
      calc(100% / 5);
  }
}
</style>
