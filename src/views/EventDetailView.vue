<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ArrowLeft, ExternalLink, Facebook, Heart, Share2, Twitter } from 'lucide-vue-next'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { Avatar, AvatarFallback, AvatarImage } from '@/components/ui/avatar'
import { cn } from '@/lib/utils'
import { toast } from 'vue-sonner' // 1. Import toast
import { getArtistAlbums, getArtistInfo, getEvent, getSpotifyToken } from '@/api'
import type { ArtistInfo } from '@/api/types'
import { useFavorites } from '@/composables/useFavorites'
const { favoriteIds, isFavorite, removeFavorite, addFavorite } = useFavorites()

const route = useRoute()
const router = useRouter()

const eventId = route.params.id as string
const eventData = ref<any>(null)
const isLoading = ref(true)

const albRes = ref(null)


async function fetchEventDetails() {
  isLoading.value = true
  console.log(`Fetching details for event ID: ${eventId}`)
//   await new Promise(resolve => setTimeout(resolve, 500))
  const {data} = await getEvent(eventId)
  console.log(data);
  const res = await getArtistInfo(data._embedded.attractions[0].name)
  console.log(res);
  
  const artistInfo:ArtistInfo = res.data.artists?.items[0]
  if (artistInfo && artistInfo.id) {
    const res = await getArtistAlbums(artistInfo.id)
    albRes.value = res.data
  }
  

  
  

  eventData.value = {
    id: eventId,
    title: data.name,
    isLike: false,
    ticketUrl: data.url,
    date: data.dates.start.localDate + ' ' + data.dates.start.localTime,
    artists: data._embedded.attractions.map((artist: any) => artist.name),
    venue: {
      name: data._embedded.venues[0].name,
      address: data._embedded.venues[0]?.address?.line1+' '+data._embedded.venues[0]?.city?.name+' '+data._embedded.venues[0]?.state?.stateCode+' '+data._embedded.venues[0]?.country?.countryCode,
      image: data._embedded.venues[0]?.images?[0]?.url:'',
      rules: {
         parking: data._embedded.venues[0]?.parkingDetail,
         general: data._embedded.venues[0]?.generalInfo?.generalRule,
         child: data._embedded.venues[0]?.generalInfo?.childRule
      },
      url: data._embedded.venues[0]?.url,
    },
    classifications: {
      segment: data.classifications[0]?.segment?.name?data.classifications[0]?.segment?.name:'',
      genre: data.classifications[0]?.genre?.name?data.classifications[0]?.genre?.name:'',
      subGenre: data.classifications[0]?.subGenre?.name?data.classifications[0]?.subGenre?.name:'',
      type: data.classifications[0]?.type?.name?data.classifications[0]?.type?.name:'',
      subType: data.classifications[0]?.subType?.name?data.classifications[0]?.subType?.name:'',
    },
    ticketStatus: data.dates.status.code,
    seatmap: data.seatmap?.staticUrl?data.seatmap?.staticUrl:'',
    artistInfo: {
      name: artistInfo?.name,
      avatar: artistInfo?.images[0]?.url?artistInfo.images[0]?.url:'',
      followers: artistInfo?.followers.total,
      popularity: artistInfo?.popularity+"%",
      genres: artistInfo?.genres.join(', '),
      albums: albRes.value?.items?albRes.value?.items:[],
      uri: artistInfo?.external_urls.spotify
    }
  }
  isLoading.value = false
}

function handleToggleFavorite() {
  if (!eventData.value) return
  
  if (isFavorite(eventData.value.id).value) {
    removeFavorite(eventData.value)
  } else {
    addFavorite(eventData.value)
  }
}

// 2. 更新收藏处理函数以显示 toast
// function handleToggleFavorite() {
//   if (!eventData.value) return

//   // 假设这里是调用后端API的逻辑，并在成功后更新本地状态
//   // 为了模拟，我们直接切换本地状态
//   const currentIsLike = eventData.value.isLike
  
//   if (!currentIsLike) { // 如果当前是未收藏状态，要进行收藏操作
//     eventData.value.isLike = true // 立即更新UI为已收藏
//     toast.success('Added to Favorites', {
//       description: eventData.value.title,
//     })
//     // 实际项目中，这里会调用API `addToFavorites(eventId)`
//   } else { // 如果当前是已收藏状态，要进行取消收藏操作
//     // 立即更新UI为未收藏，但给用户撤销的机会
//     eventData.value.isLike = false 
    
//     // 弹出带有 Undo 按钮的 toast
//     toast('Removed from Favorites', {
//       description: eventData.value.title,
//       action: {
//         label: 'Undo',
//         onClick: () => {
//           // 用户点击 Undo，将状态恢复为已收藏
//           if (eventData.value) {
//             eventData.value.isLike = true
//             toast.info('Canceled unfavorite.', {
//               description: eventData.value.title + ' has been kept in your favorites.',
//               duration: 2000 // Undo 成功后显示一个短时间提示
//             })
//           }
//           // 实际项目中，这里会再次调用API `addToFavorites(eventId)` 来撤销删除
//         },
//       },
//       duration: 3000, // 提示持续时间，给用户足够时间点击 Undo
//       // onDismiss: () => {
//       //   // 如果 toast 消失了（用户没点击Undo），则确认取消收藏
//       //   // 实际项目中，这里可以调用API `removeFromFavorites(eventId)`
//       //   // 确保只有在没有点击 Undo 的情况下才执行
//       //   console.log('Unfavorite confirmed for:', eventData.value.title);
//       // }
//     })
//   }
// }

const orderedGenres = computed(() => {
  if (!eventData.value?.classifications) return []
  const order = ['segment', 'genre', 'subGenre', 'type', 'subType']
  const classifications = eventData.value.classifications
  return order
    .map(key => classifications[key])
    .filter(item => item && item !== 'Undefined')
})

const ticketStatusClasses = computed(() => {
  if (!eventData.value?.ticketStatus) return ''
  const status = eventData.value.ticketStatus.toLowerCase()
  switch (status) {
    case 'onsale':
      return 'text-white border-green-600 bg-green-600'
    case 'offsale':
      return 'text-white border-red-600 bg-red-600'
    case 'canceled':
      return 'text-white border-black bg-black'
    case 'postponed':
    case 'rescheduled':
      return 'text-white border-orange-600 bg-orange-600'
    default:
      return 'border-gray-400'
  }
})

onMounted(fetchEventDetails)
</script>

<template>
  <div class="container py-8">
    <div v-if="isLoading" class="text-center py-20">
      <p class="text-muted-foreground">Loading event details...</p>
    </div>
    
    <div v-if="!isLoading && eventData">
      <!-- 顶部导航和操作 -->
      <div class="flex flex-col layout-phone justify-between items-start sm:items-center mb-6 gap-4">
        <div>
          <Button @click="router.back()" variant="link" class="p-0 h-auto text-muted-foreground hover:no-underline">
            <ArrowLeft class="mr-2 h-4 w-4" />
            Back to Search
          </Button>
          <h1 class="text-3xl font-bold tracking-tight mt-1 flex  flex-wrap items-center">
            {{ eventData.title }}
          </h1>
        </div>
        <div class="flex items-center gap-2">
          <Button as="a" :href="eventData.ticketUrl" target="_blank" rel="noopener noreferrer">
            Buy Tickets
            <ExternalLink class="ml-2 h-4 w-4" />
          </Button>
          <!-- 1. 修改：为收藏按钮添加事件和动态样式 -->
          <Button variant="outline" size="icon" @click="handleToggleFavorite">
            <Heart 
            :class="cn(
              'h-4 w-4 transition-all', 
              isFavorite(eventData) ? 'text-red-500 fill-red-500' : 'text-foreground'
            )" 
          />
          </Button>
        </div>
      </div>

      <!-- Tabs -->
      <Tabs default-value="info" class="w-full">
        <TabsList class="grid w-full grid-cols-3">
          <TabsTrigger value="info">Info</TabsTrigger>
          <TabsTrigger value="artist" :disabled="eventData.classifications.segment !== 'Music'">Artist</TabsTrigger>
          <TabsTrigger value="venue">Venue</TabsTrigger>
        </TabsList>

        <!-- Info Tab Content -->
        <TabsContent value="info" class="mt-6">
          <div class="grid md:grid-cols-2 gap-8">
            <!-- 左侧信息 -->
            <div class="space-y-6">
              <div>
                <h3 class="font-semibold mb-2 text-base">Date & Time</h3>
                <p class="text-muted-foreground">{{ eventData.date }}</p>
              </div>
              <div>
                <h3 class="font-semibold mb-2 text-base">Artist/Team</h3>
                <p class="text-muted-foreground">{{ eventData.artists.join(', ') }}</p>
              </div>
              <div>
                <h3 class="font-semibold mb-2 text-base">Venue</h3>
                <p class="text-muted-foreground">{{ eventData.venue.name }}</p>
              </div>
              <!-- 2. 修改：使用新的 orderedGenres 计算属性来渲染徽章 -->
              <div>
                <h3 class="font-semibold mb-2 text-base">Genres</h3>
                <div class="flex flex-wrap gap-2">
                  <Badge v-for="genre in orderedGenres" :key="genre" variant="secondary" >{{ genre }}</Badge>
                </div>
              </div>
              <!-- 3. 修改：使用动态类来设置票务状态的颜色 -->
              <div>
                <h3 class="font-semibold mb-2 text-base">Ticket Status</h3>
                <Badge  :class="ticketStatusClasses">{{ eventData.ticketStatus }}</Badge>
              </div>
              <div>
                <h3 class="font-semibold mb-2 text-base">Share</h3>
                <Button variant="outline" size="icon" as="a" target="_blank" :href="`https://twitter.com/intent/tweet?url=${encodeURIComponent(eventData.ticketUrl)}&text=${encodeURIComponent(`Check out this event: ${eventData.title}!`)}`">
                  <Twitter class="h-4 w-4" />
                </Button>
                <Button variant="outline" size="icon" as="a" target="_blank" :href="`https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(eventData.ticketUrl)}`">
                  <Facebook class="h-4 w-4" />
                </Button>
              </div>
            </div>
            <!-- 右侧座位图 -->
            <div>
              <h3 class="font-semibold mb-2 text-base">Seating Map</h3>
              <div class="border rounded-lg overflow-hidden">
                <img :src="eventData.seatmap" alt="Seating map" class="w-full h-auto" v-if="eventData.seatmap" />
              </div>
            </div>
          </div>
        </TabsContent>

        <!-- Artist Tab Content -->
        <TabsContent value="artist" class="mt-6" >
          <div class="flex items-start gap-6 mb-8">
            <Avatar size="lg" class="w-24 h-24">
              <AvatarImage :src="eventData.artistInfo.avatar" />
              <AvatarFallback>{{ eventData.artistInfo.name.substring(0, 2) }}</AvatarFallback>
            </Avatar>
            <div>
              <h2 class="text-2xl font-bold">{{ eventData.artistInfo.name }}</h2>
              <p class="text-muted-foreground">Followers: {{ eventData.artistInfo.followers }} &middot; Popularity: {{ eventData.artistInfo.popularity }}</p>
              <p class="text-muted-foreground capitalize">Genres: {{ eventData.artistInfo.genres }}</p>
              <Button size="sm" as="a" :href="eventData.artistInfo.uri" target="_blank" class="mt-2">Open in Spotify</Button>
            </div>
          </div>
          <h3 class="text-xl font-semibold mb-4">Albums</h3>
          <div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-4">
            <div v-for="album in eventData.artistInfo.albums" :key="album.id" class="rounded-md border-2 overflow-hidden ">
              <img :src="album.images[0].url" class=" aspect-square object-cover mb-2" />
              <p class="font-medium text-sm truncate px-2">{{ album.name }}</p>
              <p class="text-xs text-muted-foreground px-2">{{ album.release_date }} </p>
              <p class="text-xs text-muted-foreground px-2 mb-2">{{ album.total_tracks }} tracks</p>
            </div>
          </div>
        </TabsContent>

        <!-- Venue Tab Content -->
        <TabsContent value="venue" class="mt-6">
          <div class="flex justify-between items-center mb-4">
            <h2 class="text-2xl font-bold">{{ eventData.venue.name }}</h2>
            <Button variant="link" class="p-0 h-auto" as="a" :href="eventData.venue.url" target="_blank">
              See Events <ExternalLink class="ml-2 h-4 w-4" />
            </Button>
          </div>
          <p class="text-muted-foreground mb-6">
            {{ eventData.venue.address }}
            <Button variant="link" class="p-0 h-auto ml-4" as="a" :href="`https://www.google.com/maps/search/?api=1&query=${eventData.venue.address}`" target="_blank">

                <ExternalLink class="ml-2 h-4 w-4"></ExternalLink>
            </Button>
        </p>
          <div class="grid md:grid-cols-2 gap-8 items-start">
            <div class="border rounded-lg p-4 flex justify-center items-center">
              <img :src="eventData.venue.image" alt="Venue Logo" class="max-w-xs w-full" v-if="eventData.venue.image" />
            </div>
            <div class="space-y-4">
              <div>
                <h4 class="font-semibold">Parking</h4>
                <p class="text-sm text-muted-foreground">{{ eventData.venue.rules.parking }}</p>
              </div>
              <div>
                <h4 class="font-semibold">General Rule</h4>
                <p class="text-sm text-muted-foreground">{{ eventData.venue.rules.general }}</p>
              </div>
              <div>
                <h4 class="font-semibold">Child Rule</h4>
                <p class="text-sm text-muted-foreground">{{ eventData.venue.rules.child }}</p>
              </div>
            </div>
          </div>
        </TabsContent>
        
      </Tabs>
    </div>
  </div>
</template>

<style scoped>
.layout-phone{
    @media (min-width: 375px) {
        flex-direction: row;
        align-items: last baseline;
    }
}
</style>
