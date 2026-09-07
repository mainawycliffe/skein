<script setup lang="ts">
// The LLM-facing affordance on every doc page: hand this page to a model, or point one at the whole
// site. The menu is deliberately short — three things a reader can do today, and no MCP entry until
// `/mcp` exists (roadmap), because an entry that installs a server which answers nothing is worse
// than no entry.
//
// Nothing here knows where the files come from. The markdown twin is this page's own route plus
// `.md`, which is what `docs/.vitepress/llm-assets.ts` writes on build and serves in dev — one rule,
// stated once, so the client and the server cannot disagree about a path.
import { computed, nextTick, onBeforeUnmount, ref, useTemplateRef } from "vue";
import { useData, withBase } from "vitepress";

const { page } = useData();

/**
 * `relativePath` is the *served* path (post-`rewrites`), so appending nothing but a leading slash
 * gives the twin's URL by construction.
 *
 * The leading slash is load-bearing, not tidiness: `withBase` is documented as appending the base to
 * "internal (non-relative) urls" and **returns anything not starting with `/` unchanged**. Without
 * it these stay relative and resolve against the current directory — which is right by accident at
 * the top level (`/why` + `why.md` → `/why.md`) and wrong one level down, where `/recipes/memory` +
 * `recipes/memory.md` becomes `/recipes/recipes/memory.md`. That URL answers **200 with the SPA
 * HTML shell** rather than 404, so a copy would silently put HTML on the clipboard and no
 * `response.ok` check would notice.
 */
const markdownUrl = computed(() => withBase(`/${page.value.relativePath}`));
const llmsUrl = withBase("/llms.txt");
const llmsFullUrl = withBase("/llms-full.txt");

const open = ref(false);
/** `copied` and `failed` are shown on the button itself — a menu that closes with no feedback reads as a no-op. */
const status = ref<"idle" | "copied" | "failed">("idle");
const root = useTemplateRef<HTMLElement>("root");
const toggle = useTemplateRef<HTMLButtonElement>("toggle");

let resetTimer: ReturnType<typeof setTimeout> | undefined;
const report = (outcome: "copied" | "failed") => {
  status.value = outcome;
  clearTimeout(resetTimer);
  resetTimer = setTimeout(() => (status.value = "idle"), 2000);
};

const write = async (text: string) => {
  try {
    // Secure-context only, which localhost satisfies — so this works under `pnpm docs:dev` as well
    // as on Pages, and there is no separate dev path to keep working.
    await navigator.clipboard.writeText(text);
    report("copied");
  } catch {
    report("failed");
  }
};

const copyMarkdown = async () => {
  close();
  try {
    const response = await fetch(markdownUrl.value);
    if (!response.ok) throw new Error(String(response.status));
    await write(await response.text());
  } catch {
    report("failed");
  }
};

const copyHtml = async () => {
  close();
  // `.vp-doc` is the rendered article and nothing else: this component renders in the `doc-before`
  // slot, which VitePress puts *outside* `.vp-doc`, so the copied HTML never contains this button.
  const article = document.querySelector(".vp-doc");
  if (!article) return report("failed");
  await write(article.innerHTML);
};

const close = () => {
  open.value = false;
};

const onDocumentPointerDown = (event: PointerEvent) => {
  if (!root.value?.contains(event.target as Node)) close();
};

const onToggle = () => {
  open.value = !open.value;
  if (open.value) {
    document.addEventListener("pointerdown", onDocumentPointerDown);
    // `role="menu"` is a promise that arrow keys work and that opening lands focus inside. Making the
    // claim without the behaviour is worse than not making it — a screen reader announces a menu and
    // then the keys it just told the reader to use do nothing.
    void nextTick(() => items()[0]?.focus());
  } else {
    document.removeEventListener("pointerdown", onDocumentPointerDown);
  }
};

/** The focusable menu items, in DOM order — read live, so adding an entry needs nothing here. */
const items = () => [...(root.value?.querySelectorAll<HTMLElement>(".copy-page-item") ?? [])];

const onArrow = (step: 1 | -1) => {
  if (!open.value) return;
  const all = items();
  if (all.length === 0) return;
  const from = all.indexOf(document.activeElement as HTMLElement);
  // Wraps, and an ArrowUp with focus still on the toggle enters at the end rather than doing nothing.
  const next =
    from === -1 ? (step === 1 ? 0 : all.length - 1) : (from + step + all.length) % all.length;
  all[next]?.focus();
};

const onEscape = () => {
  if (!open.value) return;
  close();
  // Focus goes back to what opened the menu, or a keyboard reader is dropped at the top of the page.
  toggle.value?.focus();
};

onBeforeUnmount(() => {
  clearTimeout(resetTimer);
  document.removeEventListener("pointerdown", onDocumentPointerDown);
});
</script>

<template>
  <div
    ref="root"
    class="copy-page"
    @keydown.esc="onEscape"
    @keydown.down.prevent="onArrow(1)"
    @keydown.up.prevent="onArrow(-1)"
  >
    <div class="copy-page-group">
      <!-- The primary action is the one the menu's first item names, so the common case is one click.
           No `aria-label`: it would override the visible text, and the visible text is what changes to
           "Copied" — labelling this would make the one piece of feedback the control has unannounced. -->
      <button class="copy-page-action" type="button" @click="copyMarkdown">
        <svg
          v-if="status === 'copied'"
          class="copy-page-icon"
          viewBox="0 0 16 16"
          aria-hidden="true"
        >
          <path
            d="M3 8.5l3.2 3.2L13 5"
            fill="none"
            stroke="currentColor"
            stroke-width="1.6"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
        <svg v-else class="copy-page-icon" viewBox="0 0 16 16" aria-hidden="true">
          <rect
            x="5.25"
            y="5.25"
            width="8"
            height="8"
            rx="1.75"
            fill="none"
            stroke="currentColor"
            stroke-width="1.4"
          />
          <path
            d="M10.75 3.4A1.9 1.9 0 0 0 9 2.25H4.5A2.25 2.25 0 0 0 2.25 4.5V9c0 .8.49 1.48 1.19 1.77"
            fill="none"
            stroke="currentColor"
            stroke-width="1.4"
            stroke-linecap="round"
          />
        </svg>
        <span>{{
          status === "copied" ? "Copied" : status === "failed" ? "Copy failed" : "Copy page"
        }}</span>
      </button>

      <button
        ref="toggle"
        class="copy-page-chevron"
        type="button"
        aria-haspopup="menu"
        :aria-expanded="open"
        aria-label="More ways to read this page"
        @click="onToggle"
      >
        <svg class="copy-page-icon" viewBox="0 0 16 16" aria-hidden="true">
          <path
            :d="open ? 'M4.5 9.75L8 6.25l3.5 3.5' : 'M4.5 6.25L8 9.75l3.5-3.5'"
            fill="none"
            stroke="currentColor"
            stroke-width="1.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
      </button>
    </div>

    <div v-show="open" class="copy-page-menu" role="menu">
      <button class="copy-page-item" type="button" role="menuitem" @click="copyMarkdown">
        <span class="copy-page-item-icon" aria-hidden="true">
          <svg viewBox="0 0 16 16">
            <rect
              x="5.25"
              y="5.25"
              width="8"
              height="8"
              rx="1.75"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
            />
            <path
              d="M10.75 3.4A1.9 1.9 0 0 0 9 2.25H4.5A2.25 2.25 0 0 0 2.25 4.5V9c0 .8.49 1.48 1.19 1.77"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
              stroke-linecap="round"
            />
          </svg>
        </span>
        <span class="copy-page-item-text">
          <span class="copy-page-item-title">Copy page</span>
          <span class="copy-page-item-hint">Copy this page as Markdown, for an LLM</span>
        </span>
      </button>

      <a
        class="copy-page-item"
        role="menuitem"
        :href="markdownUrl"
        target="_blank"
        rel="noreferrer"
        @click="close"
      >
        <span class="copy-page-item-icon" aria-hidden="true">
          <svg viewBox="0 0 16 16">
            <rect
              x="1.75"
              y="3.75"
              width="12.5"
              height="8.5"
              rx="1.75"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
            />
            <path
              d="M4.4 10.1V5.9l1.9 2.3 1.9-2.3v4.2M10.4 5.9v4.2M10.4 10.1l1.5-1.7M10.4 10.1L8.9 8.4"
              fill="none"
              stroke="currentColor"
              stroke-width="1.3"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </span>
        <span class="copy-page-item-text">
          <span class="copy-page-item-title"
            >View as Markdown <span class="copy-page-out">↗</span></span
          >
          <span class="copy-page-item-hint">Open this page as plain text</span>
        </span>
      </a>

      <button class="copy-page-item" type="button" role="menuitem" @click="copyHtml">
        <span class="copy-page-item-icon" aria-hidden="true">
          <svg viewBox="0 0 16 16">
            <path
              d="M6.1 4.4L2.6 8l3.5 3.6M9.9 4.4L13.4 8l-3.5 3.6"
              fill="none"
              stroke="currentColor"
              stroke-width="1.5"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </span>
        <span class="copy-page-item-text">
          <span class="copy-page-item-title">Copy as HTML</span>
          <span class="copy-page-item-hint">Copy the rendered page markup</span>
        </span>
      </button>

      <div class="copy-page-divider" role="separator"></div>

      <a
        class="copy-page-item"
        role="menuitem"
        :href="llmsUrl"
        target="_blank"
        rel="noreferrer"
        @click="close"
      >
        <span class="copy-page-item-icon" aria-hidden="true">
          <svg viewBox="0 0 16 16">
            <path
              d="M3.75 2.25h5.4l3.1 3.1v8.4a.9.9 0 0 1-.9.9h-7.6a.9.9 0 0 1-.9-.9V3.15a.9.9 0 0 1 .9-.9z"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
              stroke-linejoin="round"
            />
            <path
              d="M9 2.4v3.1h3.1M5.6 8.6h4.8M5.6 11h3.2"
              fill="none"
              stroke="currentColor"
              stroke-width="1.3"
              stroke-linecap="round"
            />
          </svg>
        </span>
        <span class="copy-page-item-text">
          <span class="copy-page-item-title">llms.txt <span class="copy-page-out">↗</span></span>
          <span class="copy-page-item-hint">The index of every doc, for an agent</span>
        </span>
      </a>

      <a
        class="copy-page-item"
        role="menuitem"
        :href="llmsFullUrl"
        target="_blank"
        rel="noreferrer"
        @click="close"
      >
        <span class="copy-page-item-icon" aria-hidden="true">
          <svg viewBox="0 0 16 16">
            <path
              d="M5.25 1.75h4.4l2.6 2.6v7.4a.9.9 0 0 1-.9.9h-6.1a.9.9 0 0 1-.9-.9V2.65a.9.9 0 0 1 .9-.9z"
              fill="none"
              stroke="currentColor"
              stroke-width="1.4"
              stroke-linejoin="round"
            />
            <path
              d="M3.4 4.4v9.05a.9.9 0 0 0 .9.9h6.1"
              fill="none"
              stroke="currentColor"
              stroke-width="1.3"
              stroke-linecap="round"
            />
          </svg>
        </span>
        <span class="copy-page-item-text">
          <span class="copy-page-item-title"
            >llms-full.txt <span class="copy-page-out">↗</span></span
          >
          <span class="copy-page-item-hint">Every doc as one file, in reading order</span>
        </span>
      </a>
    </div>
  </div>
</template>

<style scoped>
/* Floated so the page's own H1 sets beside it, which is where a reader looks for a page-level
   action. `doc-before` is a sibling of <main>, so the float intrudes into it and the heading's line
   boxes shorten around this — no absolute positioning, nothing to keep in sync with the theme. */
.copy-page {
  position: relative;
  float: right;
  margin: 0 0 12px 20px;
  /* Above the heading and the sticky aside, or the open menu renders behind the outline. */
  z-index: 20;
}

/* Narrow screens have no room beside the title, and a float there would squeeze the H1 into a
   two-word column. Unfloated, it sits above the title, right-aligned. */
@media (max-width: 720px) {
  .copy-page {
    float: none;
    display: flex;
    justify-content: flex-end;
    margin: 0 0 16px;
  }
}

.copy-page-group {
  display: inline-flex;
  align-items: stretch;
  border: 1px solid var(--vp-c-border);
  border-radius: 8px;
  background-color: var(--vp-c-bg);
  overflow: hidden;
}

.copy-page-action,
.copy-page-chevron {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 0 10px;
  height: 32px;
  font-size: 13px;
  font-weight: 500;
  line-height: 1;
  color: var(--vp-c-text-1);
  background-color: transparent;
  transition:
    background-color 0.15s,
    color 0.15s;
  white-space: nowrap;
}

.copy-page-action:hover,
.copy-page-chevron:hover {
  background-color: var(--vp-c-bg-soft);
}

/* The hairline between the two halves — the same 1px the rest of the surface separates with. */
.copy-page-chevron {
  padding: 0 6px;
  border-left: 1px solid var(--vp-c-border);
  color: var(--vp-c-text-2);
}

.copy-page-icon {
  width: 15px;
  height: 15px;
  flex-shrink: 0;
}

.copy-page-menu {
  position: absolute;
  top: calc(100% + 6px);
  right: 0;
  width: 310px;
  /* The menu is right-anchored to the button, which sits at the right gutter — so on a 320px screen
     a fixed width hangs 14px off the left edge and is clipped, with no scrollbar to reveal it. */
  max-width: calc(100vw - 32px);
  padding: 6px;
  border: 1px solid var(--vp-c-border);
  border-radius: 10px;
  background-color: var(--vp-c-bg-elv);
  box-shadow: var(--vp-shadow-3);
}

.copy-page-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  width: 100%;
  padding: 7px 8px;
  border-radius: 7px;
  text-align: left;
  color: var(--vp-c-text-1);
  transition: background-color 0.15s;
}

.copy-page-item:hover {
  background-color: var(--vp-c-bg-soft);
}

/* Links inside the menu are menu items, not prose — the theme's underline and accent colour would
   make five of them read as a paragraph of links. */
.copy-page-item,
.copy-page-item:hover {
  text-decoration: none;
}

.copy-page-item-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  flex-shrink: 0;
  margin-top: 1px;
  border: 1px solid var(--vp-c-border);
  border-radius: 6px;
  color: var(--vp-c-text-2);
}

.copy-page-item-icon svg {
  width: 15px;
  height: 15px;
}

.copy-page-item-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.copy-page-item-title {
  font-size: 13px;
  font-weight: 500;
  line-height: 1.35;
}

.copy-page-item-hint {
  font-size: 12px;
  line-height: 1.35;
  color: var(--vp-c-text-3);
}

.copy-page-out {
  font-size: 11px;
  color: var(--vp-c-text-3);
}

.copy-page-divider {
  height: 1px;
  margin: 5px 8px;
  background-color: var(--vp-c-divider);
}
</style>
