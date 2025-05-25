<script setup lang="ts">
import { Site } from '@/api/types'
import api from '@/api'
import type { TorrentInfo, SiteCategory } from '@/api/types'
import { formatFileSize } from '@core/utils/formatters'
import AddDownloadDialog from '../dialog/AddDownloadDialog.vue'
import { useI18n } from 'vue-i18n'

// 国际化
const { t } = useI18n()

// 输入参数
const props = defineProps({
  site: Object as PropType<Site>,
})

// 关键字
const keyword = ref<string>()

// 选择分类
const selectCategory = ref<number[]>([])

// 全部分类
const siteCategoryList = ref<SiteCategory[]>()

// 分类选项
const categoryOptions = computed(() => {
  return siteCategoryList.value?.map(item => {
    return { title: item.desc, value: item.id }
  })
})

// 注册事件
const emit = defineEmits(['close'])

// 数据列表
const resourceDataList = ref<TorrentInfo[]>([])

// 搜索
const resourceSearch = ref('')

// 总条数
const resourceTotalItems = ref(0)

// 每页条数
const resourceItemsPerPage = ref(25)

// 加载状态
const resourceLoading = ref(false)

// 种子元数据
const torrent = ref<TorrentInfo>()

// 资源浏览表头
const resourceHeaders = [
  { title: t('dialog.siteResource.titleColumn'), key: 'title', sortable: false },
  { title: t('dialog.siteResource.timeColumn'), key: 'pubdate', sortable: true },
  { title: t('dialog.siteResource.sizeColumn'), key: 'size', sortable: true },
  { title: t('dialog.siteResource.seedersColumn'), key: 'seeders', sortable: true },
  { title: t('dialog.siteResource.peersColumn'), key: 'peers', sortable: true },
  { title: '', key: 'actions', sortable: false },
]

// 打开种子详情页面
function openTorrentDetail(page_url: string) {
  window.open(page_url, '_blank')
}

// 下载种子文件
async function downloadTorrentFile(enclosure: string) {
  window.open(enclosure, '_blank')
}

// 促销Chip类
function getVolumeFactorClass(downloadVolume: number, uploadVolume: number) {
  if (downloadVolume === 0) return 'text-white bg-lime-500'
  else if (downloadVolume < 1) return 'text-white bg-green-500'
  else if (uploadVolume !== 1) return 'text-white bg-sky-500'
  else return 'text-white bg-gray-500'
}

// 添加下载
async function addDownload(_torrent: any) {
  torrent.value = _torrent
  addDownloadDialog.value = true
}

// 添加下载对话框
const addDownloadDialog = ref(false)

// 添加下载成功
function addDownloadSuccess(url: string) {
  addDownloadDialog.value = false
}

// 添加下载失败
function addDownloadError(error: string) {
  addDownloadDialog.value = false
}

// 调用API，查询站点资源
async function getResourceList() {
  resourceLoading.value = true
  try {
    resourceDataList.value = await api.get(`site/resource/${props.site?.id}`, {
      params: {
        keyword: keyword.value,
        cat: selectCategory.value?.join(','),
      },
    })
  } catch (error) {
    console.error(error)
  }
  resourceLoading.value = false
}

// 加载站点分类
async function getSiteCategoryList() {
  try {
    siteCategoryList.value = await api.get(`site/category/${props.site?.id}`)
  } catch (error) {
    console.error(error)
  }
}

// 装载时查询站点图标
onMounted(() => {
  getSiteCategoryList()
  getResourceList()
})
</script>
<template>
  <VDialog scrollable fullscreen :scrim="false" transition="dialog-bottom-transition">
    <VCard>
      <!-- Toolbar -->
      <div>
        <VToolbar color="primary">
          <VToolbarTitle>{{ t('dialog.siteResource.browseTitle', { name: props.site?.name }) }}</VToolbarTitle>
          <VSpacer />
          <VToolbarItems>
            <VBtn icon @click="emit('close')" class="me-3">
              <VIcon size="large" color="white" icon="ri-close-line" />
            </VBtn>
          </VToolbarItems>
        </VToolbar>
      </div>
      <div class="p-3">
        <VRow>
          <VCol cols="6" md="5">
            <VTextField
              v-model="keyword"
              size="small"
              density="compact"
              :label="t('dialog.siteResource.searchKeyword')"
              clearable
            />
          </VCol>
          <VCol cols="6" md="5">
            <VSelect
              v-model="selectCategory"
              :items="categoryOptions"
              size="small"
              density="compact"
              chips
              :label="t('dialog.siteResource.resourceCategory')"
              multiple
              clearable
            />
          </VCol>
          <VCol cols="12" md="2" class="text-center">
            <VBtn block prepend-icon="mdi-magnify" @click="getResourceList">{{ t('dialog.siteResource.search') }}</VBtn>
          </VCol>
        </VRow>
      </div>
      <VCardText class="px-0 py-0 my-0">
        <VDataTable
          v-model:items-per-page="resourceItemsPerPage"
          :headers="resourceHeaders"
          :items="resourceDataList"
          :items-length="resourceTotalItems"
          :search="resourceSearch"
          :loading="resourceLoading"
          density="compact"
          item-value="title"
          return-object
          fixed-header
          hover
          :items-per-page-text="t('dialog.siteResource.itemsPerPage')"
          :loading-text="t('dialog.siteResource.loading')"
          class="h-full"
        >
          <template #item.title="{ item }">
            <a href="javascript:void(0)" @click.stop="addDownload(item)">
              <div class="text-high-emphasis pt-1">
                {{ item.title }}
              </div>
              <div class="text-sm my-1">
                {{ item.description }}
              </div>
              <VChip v-if="item.hit_and_run" variant="elevated" size="small" class="me-1 mb-1 text-white bg-black">
                H&R
              </VChip>
              <VChip v-if="item.freedate_diff" variant="elevated" color="secondary" size="small" class="me-1 mb-1">
                {{ item.freedate_diff }}
              </VChip>
              <VChip
                v-for="(label, index) in item.labels"
                :key="index"
                variant="elevated"
                size="small"
                color="primary"
                class="me-1 mb-1"
              >
                {{ label }}
              </VChip>
              <VChip
                v-if="item.downloadvolumefactor !== 1 || item.uploadvolumefactor !== 1"
                :class="getVolumeFactorClass(item.downloadvolumefactor, item.uploadvolumefactor)"
                variant="elevated"
                size="small"
                class="me-1 mb-1"
              >
                {{ item.volume_factor }}
              </VChip>
            </a>
          </template>
          <template #item.pubdate="{ item }">
            <div>{{ item.date_elapsed }}</div>
            <div class="text-sm">
              {{ item.pubdate }}
            </div>
          </template>
          <template #item.size="{ item }">
            <div class="text-nowrap whitespace-nowrap">
              {{ formatFileSize(item.size) }}
            </div>
          </template>
          <template #item.seeders="{ item }">
            <div>{{ item.seeders }}</div>
          </template>
          <template #item.peers="{ item }">
            <div>{{ item.peers }}</div>
          </template>
          <template #item.actions="{ item }">
            <div class="me-n3">
              <IconBtn>
                <VIcon icon="mdi-dots-vertical" />
                <VMenu activator="parent" close-on-content-click>
                  <VList>
                    <VListItem @click="openTorrentDetail(item.page_url || '')">
                      <template #prepend>
                        <VIcon icon="mdi-information" />
                      </template>
                      <VListItemTitle>{{ t('dialog.siteResource.viewDetails') }}</VListItemTitle>
                    </VListItem>
                    <VListItem v-if="item.enclosure?.startsWith('http')" @click="downloadTorrentFile(item.enclosure)">
                      <template #prepend>
                        <VIcon icon="mdi-download" />
                      </template>
                      <VListItemTitle>{{ t('dialog.siteResource.downloadTorrent') }}</VListItemTitle>
                    </VListItem>
                  </VList>
                </VMenu>
              </IconBtn>
            </div>
          </template>
          <template #no-data>{{ t('dialog.siteResource.noData') }}</template>
        </VDataTable>
      </VCardText>
    </VCard>
    <!-- 添加下载对话框 -->
    <AddDownloadDialog
      v-if="addDownloadDialog"
      v-model="addDownloadDialog"
      :torrent="torrent"
      @done="addDownloadSuccess"
      @error="addDownloadError"
      @close="addDownloadDialog = false"
    />
  </VDialog>
</template>

<style lang="scss" scoped>
.v-table th {
  white-space: nowrap;
}
</style>
