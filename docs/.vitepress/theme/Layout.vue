<script setup lang="ts">
// A wrapper around the default layout rather than a fork of it — the one slot this site fills.
// `doc-before` is a doc-layout slot, so the button appears on every prose page and on none of the
// pages that have no prose: the `layout: home` landing page and 404 never render it, with no
// frontmatter check here to keep in sync.
import DefaultTheme from "vitepress/theme-without-fonts";

import CopyPage from "./components/CopyPage.vue";

const { Layout } = DefaultTheme;
</script>

<template>
  <Layout>
    <template #doc-before>
      <CopyPage />
    </template>
  </Layout>
</template>
