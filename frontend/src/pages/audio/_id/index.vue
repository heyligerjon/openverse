<template>
  <main :id="skipToContentTargetId" tabindex="-1">
    <div v-if="backToSearchPath" class="w-full px-2 py-2 md:px-6">
      <VBackToSearchResultsLink
        :id="$route.params.id"
        :href="backToSearchPath"
      />
    </div>

    <template v-if="audio">
      <VAudioTrack
        layout="full"
        :audio="audio"
        class="main-track"
        @interacted="sendAudioEvent($event, 'AudioDetailPage')"
      />
      <div
        class="mx-auto mt-10 flex flex-col gap-10 px-6 lg:mt-16 lg:max-w-5xl lg:gap-16"
      >
        <VMediaReuse data-testid="audio-attribution" :media="audio" />
        <VAudioDetails data-testid="audio-info" :audio="audio" />
        <VRelatedAudio
          v-if="relatedMedia.length || relatedFetchState.isFetching"
          :media="relatedMedia"
          :fetch-state="relatedFetchState"
          @interacted="sendAudioEvent($event, 'VRelatedAudio')"
        />
      </div>
    </template>
  </main>
</template>

<script lang="ts">
import { computed } from "vue"
import { defineComponent, useMeta } from "@nuxtjs/composition-api"

import { AUDIO } from "~/constants/media"
import { skipToContentTargetId } from "~/constants/window"
import type { AudioDetail } from "~/types/media"
import type { AudioInteractionData } from "~/types/analytics"
import { useAnalytics } from "~/composables/use-analytics"
import { singleResultMiddleware } from "~/middleware/single-result"
import { useRelatedMediaStore } from "~/stores/media/related-media"
import { useSingleResultStore } from "~/stores/media/single-result"
import { useSearchStore } from "~/stores/search"
import { createDetailPageMeta } from "~/utils/og"

import VAudioDetails from "~/components/VAudioDetails/VAudioDetails.vue"
import VAudioTrack from "~/components/VAudioTrack/VAudioTrack.vue"
import VBackToSearchResultsLink from "~/components/VBackToSearchResultsLink.vue"
import VMediaReuse from "~/components/VMediaInfo/VMediaReuse.vue"
import VRelatedAudio from "~/components/VAudioDetails/VRelatedAudio.vue"

export default defineComponent({
  name: "AudioDetailPage",
  components: {
    VAudioDetails,
    VAudioTrack,
    VBackToSearchResultsLink,
    VMediaReuse,
    VRelatedAudio,
  },
  layout: "content-layout",
  middleware: singleResultMiddleware,
  setup() {
    const singleResultStore = useSingleResultStore()
    const relatedMediaStore = useRelatedMediaStore()
    const searchStore = useSearchStore()

    const { sendCustomEvent } = useAnalytics()

    const audio = computed(() =>
      singleResultStore.mediaType === AUDIO
        ? (singleResultStore.mediaItem as AudioDetail)
        : null
    )
    const relatedMedia = computed(
      () => relatedMediaStore.media as AudioDetail[]
    )
    const relatedFetchState = computed(() => relatedMediaStore.fetchState)
    const backToSearchPath = computed(() => searchStore.backToSearchPath)

    const sendAudioEvent = (
      data: Omit<AudioInteractionData, "component">,
      component: "AudioDetailPage" | "VRelatedAudio"
    ) => {
      sendCustomEvent("AUDIO_INTERACTION", {
        ...data,
        component,
      })
    }

    useMeta(createDetailPageMeta(audio.value?.title, audio.value?.url))

    return {
      audio,
      backToSearchPath,
      relatedMedia,
      relatedFetchState,

      sendAudioEvent,

      skipToContentTargetId,
    }
  },
  async asyncData({ route, error, app, $pinia }) {
    const audioId = route.params.id
    const singleResultStore = useSingleResultStore($pinia)

    try {
      await singleResultStore.fetch(AUDIO, audioId)
    } catch (err) {
      error({
        statusCode: 404,
        message: app.i18n.t("error.media-not-found", {
          mediaType: AUDIO,
          id: route.params.id,
        }),
      })
    }
  },
  head: {},
})
</script>
<style>
.audio-page {
  --wp-max-width: 940px;
}
.audio-page section,
.audio-page aside {
  max-width: var(--wp-max-width);
  margin-right: auto;
  margin-left: auto;
}
.audio-page .full-track .mx-16 {
  @apply mt-6;
  @apply px-4 md:px-0;
  max-width: var(--wp-max-width);
  margin-right: auto;
  margin-left: auto;
}
</style>
