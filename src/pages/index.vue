<template>
  <div>
    <div class="container mx-auto px-4 py-8">
      <div class="flex justify-between items-center">
        <h1 class="text-2xl font-bold">GitHub Stars List Marker</h1>

        <div class="w-[290px] space-y-1 p-3 bg-amber-100 text-amber-600 border border-amber-200 rounded-sm">
          <div class="flex justify-between items-center">
            <span class="font-bold">GitHub API Rate Limit:</span>
            <span>
              <span class="text-amber-500">{{ rateLimit.remaining }}/{{ rateLimit.limit }}</span>
              <svg class="ml-1 inline-block align-bottom w-6 h-6" viewBox="0 0 32 32">
                <circle
                  class="fill-gray-300"
                  cx="16"
                  cy="16"
                  r="16"
                ></circle>
                <circle
                  class="fill-transparent stroke-amber-500"
                  cx="16"
                  cy="16"
                  r="8"
                  stroke-width="16"
                  :stroke-dasharray="`calc(${rateLimit.remaining} / ${rateLimit.limit} * 50.266) 50.266`"
                  transform="rotate(-90) translate(-32)"
                ></circle>
              </svg>
            </span>
          </div>
          <div class="flex justify-between items-center">
            <span class="font-bold">Resets at:</span>
            <span class="text-amber-500">{{ rateLimit.resetMinutes }} minutes left</span>
          </div>
        </div>
      </div>

      <div class="mt-8">
        <form @submit.prevent="fetchStarredRepos">
          <div class="flex justify-between items-center">
            <div class="text-lg font-bold space-x-4">
              <span>GitHub User: <input type="text" class="w-[180px] border border-gray-200 rounded-sm px-2 py-1" v-model="githubUsername" /></span>
              <span>Page: <input type="number" class="w-[80px] border border-gray-200 rounded-sm px-2 py-1" v-model="page" /></span>
            </div>

            <div class="space-x-3">
              <button type="submit" class="shrink-0 bg-emerald-400 enabled:hover:bg-emerald-500 disabled:bg-emerald-200 text-white text-sm px-4 py-2 rounded-sm" :disabled="isLoading">
                Fetch
              </button>

              <button type="button" class="shrink-0 bg-purple-500 enabled:hover:bg-purple-600 disabled:bg-purple-300 text-white text-sm px-4 py-2 rounded-sm" :disabled="isLoading" @click="openALlLinks">
                Open All Links
              </button>

              <button type="button" class="shrink-0 bg-sky-500 enabled:hover:bg-sky-600 disabled:bg-sky-300 text-white text-sm px-4 py-2 rounded-sm" :disabled="isLoading" @click="markAll">
                Mark All
              </button>
            </div>
          </div>
        </form>

        <div class="mt-4 relative border border-gray-200 rounded-sm">
          <ul class="divide-y divide-gray-200">
            <li
              v-for="(repo, index) in starredRepos"
              :key="repo.name"
              class="flex items-center justify-between"
              :class="{ 'bg-indigo-50': repo.marked }"
            >
              <a :href="`https://github.com/${repo.name}`" rel="noopener noreferrer" target="_blank" class="inline-block text-indigo-500 hover:text-indigo-600 px-4 py-2 break-all">
                {{ repo.name }}
              </a>
              <div class="p-2">
                <button
                  type="button"
                  class="shrink-0 bg-gray-400 hover:bg-gray-500 text-white text-sm px-4 py-2 rounded-sm"
                  :class="{
                    'bg-gray-400 hover:bg-gray-500': repo.marked,
                    'bg-indigo-500 hover:bg-indigo-600': !repo.marked,
                  }"
                  @click="repo.marked ? unmark(index) : mark(index)"
                >
                  {{ repo.marked ? 'Unmark' : 'Mark' }}
                </button>
              </div>
            </li>

            <li v-if="starredRepos.length === 0" class="py-4 text-center text-gray-400">
              No starred repositories found.
            </li>
          </ul>

          <div v-if="isLoading" class="absolute inset-0 bg-white/50 flex items-center justify-center">
            <svg class="size-10 animate-spin text-indigo-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
          </div>
        </div>

        <div class="flex justify-between items-center mt-6">
          <div>
            <button v-if="page > 1" type="button" class="bg-gray-400 enabled:hover:bg-gray-500 disabled:bg-gray-200 text-white text-sm px-4 py-2 rounded-sm" :disabled="isLoading" @click="prev">
              Previous
            </button>
          </div>
          <div>
            <button v-if="starredRepos.length > 1" type="button" class="bg-gray-400 enabled:hover:bg-gray-500 disabled:bg-gray-200 text-white text-sm px-4 py-2 rounded-sm" :disabled="isLoading" @click="next">
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { RemovableRef } from '@vueuse/shared'

interface Repo {
  name: string
  marked: boolean
}

const githubUsername = useLocalStorage('gh-stars-list-username', '')
const page = useLocalStorage('gh-stars-list-page', 1)
const maredList = useLocalStorage('gh-stars-list-marked-list', [] as string[])
const starredRepos = useLocalStorage('gh-stars-list-starred-repos', []) as RemovableRef<Repo[]>

const isLoading = ref(false)

const rateLimit = reactive({
  limit: 60,
  remaining: 0,
  resetMinutes: 0,
})

async function fetchStarredRepos() {
  if (!githubUsername.value) {
    alert('Please enter a GitHub username.')
    return
  }

  if (rateLimit.remaining <= 0) {
    alert(`Rate limit exceeded. Please wait ${rateLimit.resetMinutes} minutes before trying again.`)
    return
  }

  isLoading.value = true

  const res = await fetch(`https://api.github.com/users/${githubUsername.value}/starred?sort=created&direction=asc&per_page=10&page=${page.value}`, {
    cache: 'no-cache',
  })
  if (res.ok) {
    const data = await res.json()
    starredRepos.value = data.map((repo: any) => ({
      name: repo.full_name,
      marked: isMarked(repo.full_name),
    } satisfies Repo))
  } else {
    starredRepos.value = []
    console.error('Failed to fetch starred repos:', res.statusText)
    alert('Failed to fetch starred repos. Please check the username or try again later.')
  }

  isLoading.value = false

  await updateRateLImit()
}

function mark(index: number) {
  const repo = starredRepos.value[index]
  repo.marked = true

  if (!maredList.value.includes(repo.name)) {
    maredList.value.push(repo.name)
  }
}

function unmark(index: number) {
  const repo = starredRepos.value[index]
  repo.marked = false

  const indexToRemove = maredList.value.indexOf(repo.name)
  if (indexToRemove !== -1) {
    maredList.value.splice(indexToRemove, 1)
  }
}

function isMarked(repo: string) {
  return maredList.value.includes(repo)
}

function markAll() {
  starredRepos.value.forEach((_, index) => mark(index))
}

function prev() {
  if (page.value > 1) {
    page.value -= 1
    fetchStarredRepos()
  }
}

function next() {
  if (starredRepos.value.length > 1) {
    page.value += 1
    fetchStarredRepos()
  }
}

function openALlLinks() {
  starredRepos.value.forEach(repo => {
    window.open(`https://github.com/${repo.name}`, '_blank')
  })
}

async function updateRateLImit() {
  const res = await fetch('https://api.github.com/rate_limit')
  if (res.ok) {
    const data = await res.json()
    rateLimit.limit = data.rate.limit
    rateLimit.remaining = data.rate.remaining
    rateLimit.resetMinutes = Math.floor((data.rate.reset - Date.now() / 1000) / 60)
  } else {
    console.error('Failed to fetch rate limit:', res.statusText)
  }
}

onMounted(async () => {
  await updateRateLImit()
})
</script>
