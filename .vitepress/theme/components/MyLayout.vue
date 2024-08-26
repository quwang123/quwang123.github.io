<template>
  <Layout>
    <template #doc-before>
      <Title />
      <Category />
    </template>
    <template #doc-after>
      <div>
        <button v-if="prevPost" @click="navigateTo(prevPost.regularPath)">下一篇: {{ prevPost.frontMatter.title }}</button>
        <button v-if="nextPost" @click="navigateTo(nextPost.regularPath)">上一篇: {{ nextPost.frontMatter.title }}</button>
      </div>
      <Comments />
    </template>
    <!-- Home slot-->
    <template #home-hero-before><HomeHero /> </template>
    <template #home-features-after> <Page /></template>
  </Layout>
  <!-- copywright -->
  <CopyWright />
</template>

<script lang="ts" setup>
import DefaultTheme from "vitepress/theme";
import HomeHero from "./HomeHero.vue";
import CopyWright from "./CopyWright.vue";
import Comments from "./Comments.vue";
import Page from "./Page.vue";
import Category from "./Category.vue";
import Title from "./Title.vue";
import { useData, withBase } from "vitepress";
import { computed } from "vue";
import { useYearSort } from "../utils";

const { Layout } = DefaultTheme;
const { theme, page } = useData();
const data = computed(() => {
  return useYearSort(theme.value.posts);
});

const currentIndex = computed(() => {
  return data.value.flat().findIndex(article => article.frontMatter.title === page.value.title);
});

const prevPost = computed(() => {
  const index = currentIndex.value;
  return index > 0 ? data.value.flat()[index - 1] : null;
});

const nextPost = computed(() => {
  const index = currentIndex.value;
  console.log(index, data.value.flat().length);
  return index < data.value.flat().length - 1 ? data.value.flat()[index + 1] : null;
});

const navigateTo = (path) => {
  window.location.href = withBase(path);
};
</script>

<style scoped>
button {
  display: inline-block;
  position: relative;
  color: var(--vp-c-color-d);
  cursor: pointer;
  font-size: 1.2em;
  font-weight: bold;
}

button::after {
  content: "";
  position: absolute;
  width: 100%;
  transform: scaleX(0);
  height: 2px;
  bottom: 0;
  left: 0;
  background-color: var(--vp-c-color-d);
  transform-origin: bottom right;
  transition: transform 0.25s ease-out;
}
button:hover::after {
  transform: scaleX(1);
  transform-origin: bottom left;
}
</style>