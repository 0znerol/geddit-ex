<template>
    <div v-if="!post" class="d-flex justify-content-center align-items-center cover-all position-absolute">
        <div class="d-flex circle md-background p-2">
            <div class="spinner-border text-4" role="status"></div>
        </div>
    </div>
    <div v-else>
     <FullPost :post="post" />
        <div class="d-flex dpx-16">
            <span class="title-small text-4">Comments</span>
        </div>
        <div class="list-container dpy-16">
            <div v-for="comment in comments">
                <div 
                  v-show="comment.kind == 't1' && parentHidden.get(comment.author+comment.depth) != true"
                  class="list-item-divider dpx-16"
                  :class="pressedComment === comment ? 'list-item' : 'list-item-full'"
                  @mousedown="startLongPress(comment)"
                  @mouseup="endLongPress"
                  @touchstart="startLongPress(comment)"
                  @touchend="endLongPress"
                  @mouseleave="endLongPress"
                  @touchcancel="endLongPress"
                >
                    <div v-show="comment.depth" class="comment-depth-container">
                        <div class="comment-depth" v-for="_ in comment.depth">
                            <div class="comment-depth-line"></div>
                        </div>
                    </div>
                    <div class="comment-body" >
                        <div class="d-flex align-items-center dpb-4">
                            <div class="list-item-leading-icon ps-0 dpe-8">
                                <span class="material-icons">{{ comment.author == 'AutoModerator' ? 'local_police' : 'face'
                                }}</span>
                            </div>
                            <span class="label-small text-10" @click.passive="open_user(comment.author)">{{
                                comment.author }}</span>
                        </div>
                        <span class="body-medium" v-show="pressedComment.get(comment.author+comment.depth) != comment.body" v-html="markdown(comment.body)"></span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeMount, onActivated, nextTick } from 'vue';
import { useRouter } from 'vue-router';
import FullPost from './FullPost.vue';
import { Geddit } from "/js/geddit.js";

const router = useRouter();
const geddit = new Geddit();

const post = ref(null);
const comments = ref([]);
const view = ref(document.querySelector('.content-view'));
const pressedComment = ref(new Map);
const parentHidden = ref(new Map)
const longPressTimer = ref(null)

async function setup() {
    let response = await geddit.getSubmissionComments(router.currentRoute.value.params.id);
    if (!response) return;

    // Get all replies for all comments in the post with Promise all as a single array
    Promise.all(response.comments.map(async (comment) => {
        return await get_all_replies(comment);
    }))
        .then(replies => {
            comments.value = replies.flat();
        })

    post.value = response.submission;
    await nextTick();
}

function decodeHtml(html) {
    let txt = document.createElement("textarea");
    txt.innerHTML = html;
    return txt.value;
}

function markdown(body) {
    return decodeHtml(body);
}

async function open_user(author) {
    router.push(`/u/${author}`);
}

async function get_all_replies(comment, depth = 0) {
    let replies = [];
    if (comment.kind == "more") {
        replies.push({
            kind: "more",
            children: comment.data.children,
        })
        return replies;
    }

    replies.push({
        kind: "t1",
        author: comment.data.author,
        body: comment.data.body_html,
        depth: depth,
    })

    if (!comment.data.replies) return replies;
    comment.data.replies.data.children.map(async (reply) => {
        replies.push(...await get_all_replies(reply, depth + 1));
    })
    return replies;
}

onBeforeMount(() => {
    if (!router.currentRoute.value.params.id) {
        router.back();
        return;
    }

    setup();

    // Scroll to top
    view.value.scroll({
        top: 0
    })
})

onActivated(() => {
    // Scroll to the last position
    let pages = JSON.parse(localStorage.getItem("pages"));
    let this_page = pages.find(page => page.path == window.location.pathname);
    if (this_page) {
        view.value.scrollTop = parseInt(this_page.scroll);
    }
})

;

const startLongPress = (comment) => {
    console.log(comments)
  longPressTimer.value = setTimeout(() => {
    if (pressedComment.value.get(comment.author+comment.depth) == comment.body) {
        pressedComment.value.delete(comment.author+comment.depth)
        let cutComments = comments.value.slice(comments.value.findIndex(c => c.author == comment.author && c.depth == comment.depth)+1, comments.value.length)
        for (let c of cutComments){
            if(c.depth == 0) return
            if (c.depth > comment.depth){
                parentHidden.value.delete(c.author+c.depth)
            }     
        }
    } else {
        pressedComment.value.set(comment.author+comment.depth, comment.body)
        let cutComments = comments.value.slice(comments.value.findIndex(c => c.author == comment.author && c.depth == comment.depth)+1, comments.value.length)
        for (let c of cutComments){
            if(c.depth == 0) return
            if (c.depth > comment.depth){
                parentHidden.value.set(c.author+c.depth, true)
            }     
        }
    }
  }, 500);
};

const endLongPress = () => {
  clearTimeout(longPressTimer.value);
  longPressTimer.value = null;
}

onBeforeMount(() => {
  clearTimeout(longPressTimer.value);
})


</script>