<template lang="pug">
#artwork-view
  //- Loading
  section.align-center(v-if='loading')
    placeholder
    p {{ "正在读取作品 #" + $route.params.id }}

  //- Done
  section.illust-container(v-if='!error && !loading')
    #top-area
      gallery(:pages='gallery')

      .body-inner
        #meta-area
          .artwork-info
            h1(:class='illust.xRestrict ? "danger" : ""') {{ illust.illustTitle }}
            p.description.pre(v-html='illust.description')
            p.description.no-desc(
              v-if='!illust.description',
              :style='{ color: "#aaa" }'
            ) (无简介)
            p.canonical-link
              a.button(
                :href='illust?.extraData?.meta?.canonical || "#"',
                target='_blank',
                rel='noopener noreferrer'
              ) 在 Pixiv 上查看 →

            p.stats
              span.original(v-if='illust.isOriginal')
                fa(icon='laugh-wink')
                | 原创
              span.like-count(title='点赞')
                fa(icon='thumbs-up')
                | {{ illust.likeCount }}

              //- 收藏
              //- 未收藏/不可收藏
              span.bookmark-count(
                v-if='!illust.bookmarkData',
                @click='async () => await addArtworkBookmark()',
                :title='store.isLoggedIn ? "添加收藏" : "收藏"'
              )
                fa(icon='heart')
                | {{ illust.bookmarkCount }}
              //- 已收藏
              router-link.bookmark-count.bookmarked(
                v-if='illust.bookmarkData',
                :to='"/users/" + store.userId',
                title='查看收藏'
              )
                fa(icon='heart')
                | {{ illust.bookmarkCount }}

              span.view-count(title='浏览')
                fa(icon='eye')
                | {{ illust.viewCount }}
              span.count
                fa(icon='images')
                | {{ gallery.length }}张

            p.create-date {{ new Date(illust.createDate).toLocaleString() }}

          .artwork-tags
            span.x-restrict(v-if='illust.xRestrict', title='R-18') R-18
            art-tag(
              :key='_',
              v-for='(item, _) in illust.tags.tags',
              :tag='item.tag'
            )

        aside#author-area
          .author-info
            h2 作者
            .align-center(v-if='!user.userId')
              placeholder
            author-card(:user='user', v-if='user.userId')

        card.comments(title='评论')
          comments-area(
            :id='illust.id || illust.illustId',
            :count='illust.commentCount'
          )

    //- 相关推荐
    .recommend-works
      h2 相关推荐
      .align-center.loading(v-if='!recommend.length')
        placeholder
      artwork-list(:list='recommend')
      show-more(
        v-if='recommendNextIds.length',
        :text='recommendLoading ? "加载中" : "加载更多"',
        :method='async () => await getMoreRecommend()',
        :loading='recommendLoading'
      )

  //- Error
  section.error(v-if='error')
    error-page(title='出大问题', :description='error')
</template>

<script lang="ts" setup>
import { API_BASE } from '../config'

import AuthorCard from '../components/AuthorCard.vue'
import ArtTag from '../components/ArtTag.vue'
import ArtworkList from '../components/ArtworksList/ArtworkList.vue'
import Card from '../components/Card.vue'
import CommentsArea from '../components/Comment/CommentsArea.vue'
import ErrorPage from '../components/ErrorPage.vue'
import Gallery from '../components/Gallery.vue'
import Placeholder from '../components/Placeholder.vue'
import ShowMore from '../components/ShowMore.vue'
import { getCache, setCache } from './siteCache'

// Types
import type { Artwork, ArtworkInfo, ArtworkUrls, User } from '../types'

import { onMounted, ref } from 'vue'
import { onBeforeRouteUpdate, useRoute } from 'vue-router'
import { getJSON } from '../utils/fetch'
import { useUserStore } from '../states'
import { addBookmark, sortArtList } from '../utils/artworkActions'

type Gallery = {
  urls: ArtworkUrls & {
    thumb_mini: string
  }
  width: number
  height: number
}

const loading = ref(true)
const error = ref('')
const illust = ref<Artwork>({} as Artwork)
const gallery = ref<Gallery[]>([])
const user = ref<User>({} as User)
const recommend = ref<ArtworkInfo[]>([])
const recommendNextIds = ref<string[]>([])
const recommendLoading = ref(false)
const bookmarkLoading = ref(false)
const route = useRoute()
const store = useUserStore()

async function init(id: string): Promise<void> {
  loading.value = true
  const dataCache = getCache(`illust.${id}`)
  const pageCache = getCache(`illust.${id}.page`)
  if (dataCache && pageCache) {
    illust.value = dataCache
    gallery.value = pageCache
    loading.value = false
    document.title = `${dataCache.illustTitle} | Artwork | PixivNow`
    await getUser(dataCache.userId)
    await getFirstRecommend(id)
    return
  }

  try {
    const [illustData, illustPage] = await Promise.all([
      getJSON<Artwork>(`${API_BASE}/ajax/illust/${id}?full=1`),
      getJSON<Gallery[]>(`${API_BASE}/ajax/illust/${id}/pages`),
    ])
    document.title = `${illustData.illustTitle} | Artwork | PixivNow`
    setCache(`illust.${id}`, illustData)
    setCache(`illust.${id}.page`, illustPage)
    illust.value = illustData
    gallery.value = illustPage
    await getUser(illustData.userId)
    await getFirstRecommend(id)
  } catch (err) {
    console.warn('illust fetch error', `#${id}`, err)
    if (err instanceof Error) {
      error.value = err.message
    } else {
      error.value = '未知错误'
    }
  } finally {
    loading.value = false
  }
}

async function getUser(userId: string): Promise<void> {
  const value = getCache(`user.${userId}`)
  if (value) {
    user.value = value
    return
  }

  try {
    const [userData, profileData] = await Promise.all([
      getJSON<User>(`${API_BASE}/ajax/user/${userId}?full=1`),
      getJSON<{ illusts: Record<string, ArtworkInfo> }>(
        `${API_BASE}/ajax/user/${userId}/profile/top`
      ),
    ])
    const { illusts } = profileData
    const userValue = {
      ...userData,
      illusts: sortArtList(illusts),
    }
    user.value = userValue
    setCache(`user.${userId}`, userValue)
  } catch (err) {
    console.warn('User fetch error', err)
  }
}

async function getFirstRecommend(id: string): Promise<void> {
  if (recommendLoading.value) return
  try {
    recommendLoading.value = true
    console.log('init recommend')
    const data = await getJSON<{ illusts: ArtworkInfo[]; nextIds: string[] }>(
      `${API_BASE}/ajax/illust/${id}/recommend/init?limit=18`
    )
    recommend.value = data.illusts
    recommendNextIds.value = data.nextIds
  } catch (err) {
    console.error('recommend fetch error', err)
  } finally {
    recommendLoading.value = false
  }
}

async function getMoreRecommend(): Promise<void> {
  if (recommendLoading.value) return
  if (!recommendNextIds.value.length) {
    console.log('no more recommend')
    return
  }

  try {
    recommendLoading.value = true
    console.log('get more recommend')
    const requestIds = recommendNextIds.value.splice(0, 18)
    const searchParams = new URLSearchParams()
    for (const id of requestIds) {
      searchParams.append('illust_ids', id)
    }
    const data = await getJSON<{ illusts: ArtworkInfo[]; nextIds: string[] }>(
      `${API_BASE}/ajax/illust/recommend/illusts?${searchParams.toString()}`
    )
    recommend.value = recommend.value.concat(data.illusts)
    recommendNextIds.value = recommendNextIds.value.concat(data.nextIds)
  } catch (err) {
    console.error('recommend fetch error', err)
  } finally {
    recommendLoading.value = false
  }
}

async function addArtworkBookmark(): Promise<void> {
  if (!store.isLoggedIn) {
    console.log('需要登录才可以添加收藏')
    return
  }
  if (!illust.value.isBookmarkable) {
    console.log('无法添加收藏')
    return
  }
  if (illust.value.bookmarkData) {
    console.log('已经收藏过啦')
    return
  }
  if (bookmarkLoading.value) return
  try {
    bookmarkLoading.value = true
    const data = await addBookmark(illust.value.illustId)
    if (data.last_bookmark_id) {
      illust.value.bookmarkData = data
      illust.value.bookmarkCount++
    }
  } catch (err) {
    console.error('bookmark add error:', err)
  } finally {
    bookmarkLoading.value = false
  }
}

onBeforeRouteUpdate(async (to) => {
  await init(to.params.id as string)
})

onMounted(async () => {
  document.title = 'Artwork | PixivNow'
  await init(route.params.id as string)
})
</script>

<style scoped lang="sass">

.gallery
  margin: 1rem auto

.artwork-tags
  margin: 1rem 0

h1
  // display: inline-block
  box-shadow: none
  background: linear-gradient(90deg, var(--theme-accent-color), rgba(255,255,255,0))
  background-position: 0 1em
  background-repeat: no-repeat
  margin: 0

  &.danger
    background: linear-gradient(90deg, var(--theme-danger-color), rgba(255,255,255,0))
    background-position: 0 1em
    background-repeat: no-repeat

.x-restrict
  font-weight: bold
  color: #c00
  margin-right: 1rem

.stats
  > span, > a
    margin-right: 0.5rem
    color: #aaa

    [data-icon]
      margin-right: 4px

  .original
    color: inherit
    font-weight: 600

  .bookmark-count
    cursor: pointer

    &.bookmarked
      color: var(--theme-bookmark-color)
      font-weight: 600

.create-date
  color: #aaa
  font-size: 0.85rem

.breadcrumb
  margin-top: 1rem

.user-illusts
  ul
    margin-left: -1rem
    margin-right: -1rem
    background-color: var(--theme-background-color)

.load-more
  a.plain
    color: var(--theme-text-color)
    cursor: pointer

  .top .inner
    border-radius: 8px
    width: 100%
    padding: 28% 0
    background-color: var(--theme-box-shadow-color)
    text-align: center

  .bottom .author
    font-size: 0.8rem
</style>
