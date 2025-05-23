<template>
  <div class="layout" v-if="post && author">
    <!-- MAIN POST -->
    <article class="post">
      <header class="author">
        <img v-if="author.avatarUrl" :src="avatarSrc" class="avatar" alt="avatar" />
        <div class="author-info">
          <h2>{{ post.description }}</h2>
          <p>by {{ `${author.username}` }}</p>
        </div>
      </header>

      <img :src="imageSrc" class="post-img" />

      <section class="caption">
        <time :datetime="post.createdAt.toISOString()">
          Posted at {{ formatDate(post.createdAt) }} - Inspire Web
        </time>
      </section>

      <section class="audio-wrapper">
        <SoundCloudPlayer
          v-if="post.soundcloudSong"
          :songUrl="post.soundcloudSong"
          :startTime="post.soundcloudSongStart || 0"
          :startVolume="30"
          :autoPlay="false"
          :volumeControl="false"
        />
      </section>

      <button class="close-btn" @click="goBack">× Close</button>
    </article>

    <!-- SIMILAR IMAGES -->
    <aside class="similar" v-if="similarPosts.length" @scroll="handleScroll">
      <h1 style="margin-bottom: 10px">More like this</h1>
      <div class="similar-grid">
        <router-link
          v-for="p in similarPosts"
          :key="p.postId"
          :to="{
            name: 'PostPage',
            params: { postId: p.postId },
            query: { ...route.query }, // keeps ?modal=1 if present
          }"
        >
          <img :src="thumbSrc(p)" alt="similar image" class="similar-img" />
        </router-link>
      </div>
    </aside>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, ref, computed, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getPostByID, getPostsByIDs } from '@/services/posts.ts'
import { searchPostsByPostID } from '@/services/search.ts'
import { getUserByID } from '@/services/auth.ts'
import type { Post } from '@/models/Post'
import type { User } from '@/models/User'
import SoundCloudPlayer from '@/components/SoundCloudPlayer.vue'

const page = ref(0)
const pageSize = 20

/* --- stores & router --- */
const route = useRoute()
const router = useRouter()

/* --- reactive state --- */
const post = ref<Post>()
const author = ref<User>()
const similarPosts = ref<Post[]>([])

/* --- helpers --- */
const mainImage = computed(
  () => post.value?.images.find((i) => i.variantType === 'ORIGINAL') ?? post.value?.images[0],
)

const imageSrc = computed(() =>
  mainImage.value ? `${import.meta.env.VITE_IMAGE_BASE_PATH}${mainImage.value.url}` : '',
)

const avatarSrc = computed(() =>
  author.value?.avatarUrl ? `${import.meta.env.VITE_IMAGE_BASE_PATH}${author.value.avatarUrl}` : '',
)

function thumbSrc(p: Post) {
  const img = p.images.find((i) => i.variantType === 'THUMBNAIL') ?? p.images[0]
  return `${import.meta.env.VITE_IMAGE_BASE_PATH}${img.url}`
}

function formatDate(d: Date) {
  return new Intl.DateTimeFormat('en', {
    dateStyle: 'medium',
    timeStyle: 'short',
  }).format(d)
}

function goBack() {
  router.back()
}

function handleScroll(e: Event) {
  const el = e.target as HTMLElement
  const nearBottom = el.scrollTop + el.clientHeight >= el.scrollHeight - 200

  if (nearBottom) {
    page.value++
    loadSimilarPosts()
  }
}

async function loadSimilarPosts(reset = false) {
  const id = route.params.postId as string
  const res = await searchPostsByPostID(id, pageSize, page.value * pageSize)
  const posts = await getPostsByIDs(res.map((r) => r.postId))
  let filtered = posts.filter((p) => p.postId !== id)
  if (filtered.length % 2 === 1) {
    filtered = filtered.slice(0, -1)
  }
  if (reset) {
    similarPosts.value = filtered
  } else {
    similarPosts.value.push(...filtered)
  }
}

/* --- load data --- */
onMounted(async () => {
  const id = route.params.postId as string

  post.value = await getPostByID(id)
  author.value = await getUserByID(post.value.authorId)

  page.value = 0
  await loadSimilarPosts(true)
})

// Watch for route changes (if user clicks another similar post)
watch(
  () => route.params.postId,
  async (newId) => {
    post.value = await getPostByID(newId)
    author.value = await getUserByID(post.value.authorId)

    const res = await searchPostsByPostID(newId as string, 20, 0)
    const posts = await getPostsByIDs(res.map((res) => res.postId))
    similarPosts.value = posts.filter((p: Post) => p.postId !== newId)
  },
)
</script>

<style scoped>
/* --- layout ---------------------------------------------------------- */
.layout {
  display: flex;
  justify-content: center;
  gap: 2rem;
  width: 100%;
  max-width: 1600px;
  margin-inline: auto;
}

.post {
  flex: 1 1 0;
  max-width: 800px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* --- author header --------------------------------------------------- */
.author {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  width: 100%;
  margin: 1rem 0;
}

.avatar {
  width: 48px;
  height: 48px;
  object-fit: cover;
  border-radius: 50%;
}

/* --- post image ------------------------------------------------------ */
.post-img {
  width: 100%;
  aspect-ratio: 1 / 1;
  object-fit: cover;
  border-radius: 0.5rem;
}

/* --- caption --------------------------------------------------------- */
.caption {
  width: 100%;
  padding: 1rem 0.5rem;
}

/* --- close button ---------------------------------------------------- */
.close-btn {
  margin-top: 1.5rem;
  padding: 0.75rem 1.25rem;
  border: none;
  border-radius: 0.5rem;
  background: transparent;
  font-weight: 500;
  font-size: 1rem;
  cursor: pointer;
  transition:
    transform 0.3s ease,
    background 0.2s ease;
}
.close-btn:hover {
  background: rgba(255 255 255 / 0.9);
}
.close-btn:active {
  transform: scale(0.97);
}
.close-btn:disabled {
  opacity: 0.6;
  cursor: default;
}

/* --- similar rail ---------------------------------------------------- */
.similar {
  width: 40%;
  max-height: 90vh;
  overflow-y: auto;
  padding-right: 0.5rem;

  /* Hide scrollbar for WebKit (Chrome, Safari) */
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE/Edge */
}

.similar::-webkit-scrollbar {
  display: none; /* WebKit */
}

.similar h3 {
  font-size: 1rem;
  margin-bottom: 0.5rem;
}

.similar-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0.5rem;
}

.similar-img {
  width: 100%;
  aspect-ratio: 1 / 1;
  object-fit: cover;
  border-radius: 0.5rem;
  transition: transform 0.15s ease;
}

.similar-img:hover {
  transform: scale(1.04);
}

/* --- responsive ------------------------------------------------------ */
@media (max-width: 900px) {
  .layout {
    flex-direction: column;
    padding: 0 1rem;
  }
  .similar {
    width: 100%;
    order: 2;
  }
  .similar-grid {
    grid-template-columns: repeat(
      auto-fill,
      minmax(110px, 1fr)
    ); /* Already good for responsiveness */
  }
}

.audio-wrapper {
  margin-top: 1rem;
  width: 100%;
}
</style>
