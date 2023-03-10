<template lang="pug">
header.global-navbar(:class='{ "not-at-top": notAtTop, hidden }')
  .flex
    a.side-nav-toggle.plain(@click='toggleSideNav')
      fa(icon='bars')

    .logo-area
      router-link.plain(to='/')
        img.site-logo(:src='LogoH')

    .flex.search-area
      .search-full.align-right.flex-1
        search-box
      .search-icon.align-right.flex-1
        button.pointer(@click='openSideNav')
          fa(icon='search')
          | &nbsp;搜索

    #global-nav__user-area.user-area
      .user-link
        a.dropdown-btn.plain.pointer(
          :class='{ "show-user": showUserDropdown }',
          @click='showUserDropdown = !showUserDropdown'
        )
          img.avatar(
            :src='userStore.isLoggedIn ? userStore.userProfileImg : API_BASE + "/~/common/images/no_profile.png"',
            :title='userStore.isLoggedIn ? userStore.userId + " (" + userStore.userPixivId + ")" : "未登入"'
          )
          fa(icon='angle-down')

        transition(
          name='fade',
          mode='out-in',
          enter-active-class='fade-in-up',
          leave-active-class='fade-out-down'
        )
          .dropdown-content(v-show='showUserDropdown')
            ul
              //- notLogIn
              li(v-if='!userStore.isLoggedIn')
                .nav-user-card
                  .top
                    .banner-bg
                    img.avatar(
                      :src='API_BASE + "/~/common/images/no_profile.png"'
                    )
                  .details
                    a.user-name 游客
                    .uid 绑定令牌，同步您的 Pixiv 信息！

              //- isLogedIn
              li(v-if='userStore.isLoggedIn')
                .nav-user-card
                  .top
                    .banner-bg
                    router-link.plain.name(:to='"/users/" + userStore.userId')
                      img.avatar(:src='API_BASE + userStore.userProfileImgBig')
                  .details
                    router-link.plain.user-name(:to='"/users/" + userStore.userId') {{ userStore.userName }}
                    .uid @{{ userStore.userPixivId }}

              li(v-if='$route.path !== "/login"')
                router-link.plain(:to='"/login?back=" + $route.path') {{ userStore.isLoggedIn ? "查看令牌" : "用户登入" }}
              li(v-if='userStore.isLoggedIn')
                a.plain(@click='logoutUser') 用户登出
</template>

<script lang="ts" setup>
import { ref, onMounted, watch } from 'vue'
import SearchBox from './SearchBox.vue'
import { API_BASE } from '../config'
import { logout } from './userData'
import LogoH from '../assets/LogoH.png'
import { useSideNavStore, useUserStore } from '../states'
import { useRouter } from 'vue-router'

const hidden = ref(false)
const notAtTop = ref(false)
const showUserDropdown = ref(false)
const router = useRouter()
const sideNavStore = useSideNavStore()
const userStore = useUserStore()

function toggleSideNav() {
  sideNavStore.open = !sideNavStore.open
}

function openSideNav() {
  sideNavStore.open = true
}

function logoutUser() {
  logout()
  userStore.logout()
  location.reload()
}

watch(hidden, (value) => {
  if (value) {
    document.body.classList.add('global-navbar_hidden')
  } else {
    document.body.classList.remove('global-navbar_hidden')
  }
})

onMounted(() => {
  window.addEventListener('scroll', () => {
    const newTop = document.documentElement.scrollTop
    if (newTop > 50) {
      notAtTop.value = true
    } else {
      notAtTop.value = false
    }
  })

  // Outside close user dropdown
  document
    .getElementById('global-nav__user-area')
    ?.addEventListener('click', (e) => e.stopPropagation())
  document.addEventListener('click', () => {
    if (showUserDropdown.value) showUserDropdown.value = false
  })
})
</script>

<style lang="sass">

.global-navbar
  background-color: var(--theme-accent-color)
  padding: 0.4rem 1rem
  color: var(--theme-background-color)
  display: flex
  align-items: center
  position: fixed
  height: 50px
  width: 100%
  box-sizing: border-box
  white-space: nowrap
  top: 0
  z-index: 100
  transition: all .8s ease

  .flex
    display: flex
    width: 100%
    gap: 1rem
    align-items: center

  &.not-at-top
    box-shadow: 0 0px 8px var(--theme-box-shadow-color)

  .side-nav-toggle
    font-size: 1.2rem
    text-align: center
    color: var(--theme-accent-link-color)
    cursor: pointer
    width: 2.4rem
    height: 2.4rem
    border-radius: 50%
    display: flex
    align-items: center

    &:hover
      background-color: rgba(255,255,255,0.2)

    [data-icon]
      margin: 0 auto

  .logo-area
    .site-logo
      height: 2.2rem
      width: auto

  .search-area
    flex: 1

  .user-area
    .avatar
      height: 2rem
      width: 2rem
      border-radius: 50%

    .user-link
      position: relative

      .dropdown-btn
        list-style: none
        display: flex
        align-items: center

      [data-icon]
        margin-left: 6px
        color: #fff
        transition: all 0.4s

      .dropdown-btn.show-user
        [data-icon]
          transform: rotateZ(180deg)

      .dropdown-content
        position: absolute
        top: 1.4rem
        right: 0
        padding: 0
        padding-top: 0.4rem
        width: 200px

        ul
          list-style: none
          padding: 4px
          background-color: #fff
          box-shadow: 0 0 4px #aaa
          border-radius: 4px

          li > *
            padding: 0.5rem

          li a
            display: block
            cursor: pointer

            &:hover
              background-color: var(--theme-tag-color)

  .nav-user-card
    border-bottom: 1px solid
    position: relative

    .top
      position: relative

      .banner-bg
        position: absolute
        top: calc(-0.4rem - 6px)
        left: -12px
        height: 56px
        width: calc(100% + 24px)
        background-color: rgba(var(--theme-accent-color--rgb), 0.1)
        z-index: 0

      a
        display: inline !important

    .avatar
      width: 68px
      height: 68px

    .details
      .user-name
        font-size: 1rem

      .uid
        font-size: 0.8rem
        color: #aaa

  .search-icon
    display: none

@media screen and (max-width: 450px)
  .global-navbar
    .search-full
      display: none
    .search-icon
      display: block
</style>
