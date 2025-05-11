<template>
  <div
    v-if="subreddits.length && !posts.length"
    class="d-flex justify-content-center align-items-center cover-all position-absolute"
  >
    <div class="d-flex circle md-background p-2">
      <div class="spinner-border text-4" role="status"></div>
    </div>
  </div>
  <TopAppBar
    ref="topbar"
    subreddit="Popular"
    @params_changed="params_changed"
  />
  <div class="cards dpb-16" v-show="subreddits.length">
    <Post v-for="post in posts" :post="post.data" />
  </div>
  <div
    class="display-flex justify-content-center p-3"
    v-show="!subreddits.length"
  >
    <span class="title-medium text-4">Explore and follow some subreddits!</span>
  </div>
  <div v-show="!scroll_loaded" class="md-progress-container">
    <div class="md-progress">
      <div class="md-progress-indicator"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, onBeforeMount, onActivated, onDeactivated } from "vue";
import { Geddit } from "/js/geddit.js";
import Post from "./CompactPost.vue";
import TopAppBar from "./TopAppBar.vue";

const geddit = new Geddit();
const topbar = ref("hot");
const subreddits = ref([]);
const _subreddits = ref([]);

const posts = ref([]);
const after = ref(null);

const scroll_loaded = ref(true);

async function setup() {
  get_subreddits();

  let allPosts = [];
  if (!subreddits.value || subreddits.value.length == 0) {
    console.log("No subreddits found");
    return;
  }
  try {
    const fetchPromises = subreddits.value.map((subreddit) =>
      geddit
        .getSubmissions(topbar.value.sort, subreddit.display_name, {
          t: topbar.value.time,
        })
        .then((response) => response.posts)
        .catch((error) => {
          console.error(
            `Error fetching posts for subreddit ${subreddit.display_name}:`,
            error
          );
          return [];
        })
    );

    const results = await Promise.all(fetchPromises);

    allPosts = results.flat();

    allPosts.sort((a, b) => {
      return b.data.score - a.data.score;
    });
  } catch (error) {
    console.error("Error during setup:", error);
  }

  posts.value = allPosts;
  console.log(allPosts);
}

async function get_subreddits() {
  subreddits.value = JSON.parse(localStorage.getItem("subreddits"));
  _subreddits.value = subreddits.value;
}

async function get_posts() {
  get_subreddits();
  let allPosts = [];
  if (!subreddits.value || subreddits.value.length == 0) {
    console.log("No subreddits found");
    return;
  }
  try {
    console.log("fetching posts");
    const fetchPromises = subreddits.value.map((subreddit) =>
      geddit
        .getSubmissions(topbar.value.sort, subreddit.display_name, {
          t: topbar.value.time,
        })
        .then((response) => {
          response.posts;
        })
        .catch((error) => {
          console.error(
            `Error fetching posts for subreddit ${subreddit.display_name}:`,
            error
          );
          return [];
        })
    );

    const results = await Promise.all(fetchPromises);

    allPosts = results.flat();
    console.log(allPosts);
    allPosts.sort((a, b) => {
      return b.data.score - a.data.score;
    });
  } catch (error) {
    console.error("Error during setup:", error);
  }

  posts.value = allPosts;
}

async function scroll() {
  scroll_loaded.value = false;

  let response = await geddit.getSubmissions(topbar.value.sort, "popular", {
    after: after.value,
    t: topbar.value.time,
  });
  if (!response || !response.posts.length) {
    after.value = null;
    scroll_loaded.value = true;
    return;
  }

  posts.value.push(...response.posts);
  after.value = response.after;

  scroll_loaded.value = true;
}

async function params_changed() {
  posts.value = [];
  after.value = null;
  scroll_loaded.value = true;
  get_posts();
}

function scroll_handle(el) {
  if (
    el.target.scrollTop + el.target.clientHeight >=
      el.target.scrollHeight - window.innerWidth &&
    scroll_loaded.value &&
    after.value
  ) {
    scroll();
  }
}

onBeforeMount(() => {
  setup();
});

onActivated(() => {
  // Add the scroll event listener
  let view = document.querySelector(".content-view");
  view.addEventListener("scroll", scroll_handle);

  // Scroll to the last position
  let pages = JSON.parse(localStorage.getItem("pages"));
  let this_page = pages.find((page) => page.path == window.location.pathname);
  if (this_page) {
    document.querySelector(".content-view").scrollTop = parseInt(
      this_page.scroll
    );
  }
});

onDeactivated(() => {
  // Disable scroll event listener
  let view = document.querySelector(".content-view");
  view.removeEventListener("scroll", scroll_handle);
});
</script>

