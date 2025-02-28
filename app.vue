<template>
  <a-config-provider
      :theme="{algorithm:darkTheme?theme.darkAlgorithm:theme.defaultAlgorithm,
        token:{
          colorBgContainer:'transparent',
          colorPrimary:useCustomTheme.primaryColor.colorValue
        }
      }"
  >
    <NuxtLayout :name="layout">
      <NuxtLoadingIndicator/>
      <NuxtPage/>
    </NuxtLayout>

  </a-config-provider>
</template>
<script setup lang="ts">
import {theme} from 'ant-design-vue';
import {useAppCustomTheme, useAppDarkTheme, useAppSetting} from "~/composables/useApp";
import {useTheme} from "vuetify";
import type {LayoutKey} from "#build/types/layouts";



//获取页面配置
const appSettingRef = await useAppSetting()
const useCustomTheme = useAppCustomTheme()
const darkTheme = useAppDarkTheme()
const vuetifyTheme = useTheme();
const {smAndUp}=useDisplay()
/**
 * 动态布局
 * @type {ComputedRef<LayoutKey>}
 */
const layout=computed(()=>{
  const {path}=useRoute()
  if(path.includes('draw/') || path==='/'||path.includes('ai/')||path.includes('user/')||path.includes('pay/')||path.includes('about/')){
    if (!smAndUp.value) {
      return 'mobile-ui-layout' as LayoutKey
    }else {
      return 'main-layout' as LayoutKey
    }
  }else if(path.includes('admin/')) {
    return 'landing-layout' as LayoutKey
  }else if(path.includes('test/')){
    //return 'test-layout' as LayoutKey
    return 'ui-layout' as LayoutKey
  }
  else {
    return 'main-layout' as LayoutKey
  }
})

vuetifyTheme.global.name.value=darkTheme.value?'dark':'light' // 根据主题设置切换vuetify主题
// console.log("vuetifyTheme",vuetifyTheme,themes)
const appPageContent = computed(() => appSettingRef.value?.page_content || {});  //获取页面配置参数，动态修改标题
onBeforeMount(async () => {
  if(!appPageContent.value.website_title) {
    console.error("Title not loaded!");
  }
  useHead({
    title: {
      textContent: appPageContent.value.website_title?appPageContent.value.website_title:''

    },
    meta: [
      // <meta name="viewport" content="width=device-width, initial-scale=1">
      {name: 'google-site-verification', content: '4SOiCOVLrSZsGqZKhVt7dEKsatxjHCU5TfwcPkpvU1E'},
      {name: 'viewport', content: 'width=device-width, initial-scale=1,user-scalable=no'},
      {
        name: 'description',
        content: '一款跨平台的AI工具，只需要点点鼠标就可以完成各种自媒体和商业文案，商业大片、商品图片和模特写真，更有AI文生视频，助力自媒体创作'
      },
      {name: 'keywords', content: 'AI文案，AI工具，AI摄影，chatGPT,sora,AI视频，AI写真，AI电商'},
    ],
    // link: [
    //   {
    //     id: 'theme-link',
    //     rel: 'stylesheet',
    //     href: '/themes/aura-dark-green/theme.css'
    //   }
    // ]
  })
})



</script>
