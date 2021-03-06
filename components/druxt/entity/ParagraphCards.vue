<template>
  <div>
    <div v-if="$fetchState.pending" class="w-full h-96 text-center p-14">
      <!-- <le-loading></le-loading> -->
      <font-awesome-icon icon="circle-notch" class="text-gray-200 animate-spin" style="font-size: 100px" />
    </div>
    <div v-else class="card-block block-space">
      <div class="container mx-auto">
        <slot name="subtitle"/>
        <ul v-if="gallery" class=" grid gap-4 lg:gap-8 grid-cols-1 md:grid-cols-2 lg:grid-cols-3">
          <li class="card__item" v-for="card in cards" :key="card.id">
            <le-card :card="card" :lightbox="lightbox"/>
          </li>
        </ul>
        <ul v-else class="card__list grid gap-4" :class="gridClass">
          <li class="card__item" v-for="card in cards" :key="card.id">
            <div class="card bg-gray-100 h-full" :class="getVariant">
              <div class="w-full bg-center bg-cover" :class="imageClass" v-bind:style="'background-image:url(' + card.cover + ')'"></div>
              <div class="card__content p-5 xl:p-6">
                <h3 class="card__title block__title">{{ card.heading }}</h3>
                <div class="card__content markup__html" v-html="card.content"></div>
                <NuxtLink v-if="card.link" class="button card__link" :to="card.link.uri">
                  See more
                </NuxtLink>
              </div>
            </div>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import { DruxtEntityMixin } from 'druxt-entity'
import { mapActions } from 'vuex'

export default {
  mixins: [DruxtEntityMixin],
  async fetch() {
    const util = require('util')

    for (const delta in this.fields.items.data.data) { // get fields.items.data.data... from Detected Vue tool --> data
      const item = this.fields.items.data.data[delta]
      const result = await this.getEntity({ id: item.id, type: item.type })
      let link = false
      let cover = false
      let images = false
      let imageArray = []
      let mediaArray = result.relationships._media.data

      if(mediaArray.length > 0) {
        // Get the first image as a cover
        let coverObject = await this.getResource(mediaArray[0])
        if(coverObject.relationships.media_image) {
          const coverImage = await this.getResource(
            coverObject.relationships.media_image.data
          )
          cover = this.leGetImage(coverImage, 'media_library')
        }

        for(let index in mediaArray) {
          let media = mediaArray[index]
          const imageObj = await this.getResource(media)
          if(imageObj.relationships.media_image) {
            const image = await this.getResource(
              imageObj.relationships.media_image.data
            )
            imageArray[index] = {
              src: this.leGetImage(image),
              title: this.leGetImageAlt(imageObj)
            }
          }
        }
        imageArray.length > 1 ? images = imageArray : ''
      }

      let heading = result.attributes._heading
      let content = result.attributes._markup
      let linkObj = result.attributes._link

      if (linkObj) {
        let linkText = linkObj.text
        let linkUri = linkObj.uri
        let linkLabel = linkText ? linkText : linkUri
        // Todo: Implemented the <front> page
        if(linkUri.includes("entity:")) {
          const util = require('util')
          const linkEntity = linkUri.replace(/entity:/gi,'')
          const route = await this.getRoute(linkEntity)

          let nodeId = route.props.uuid
          let nodeType = route.props.type
          
          const entity = await this.getEntity({ id: nodeId, type: nodeType })
          let desk = entity.attributes.deck ? entity.attributes.deck.value : ''

          linkLabel = linkText ? linkText : entity.attributes.title
          content = content ? content : desk
          linkUri = entity.attributes.path.alias

          if(!heading) {
            heading = entity.attributes.title
          }

          if(!cover && entity.relationships.hero.data) {
            // Get the image from hero field
            let media = entity.relationships.hero.data
            let mediaObj = await this.getResource(media)
            if(mediaObj.relationships.media_image) {
              const image = await this.getResource(
                mediaObj.relationships.media_image.data
              )
              cover = this.leGetImage(image, 'media_library')
            }
          }
        }

        link = {
          uri: linkUri,
          text: linkLabel
        }
      }

      if (!this.cards) this.cards = []
      this.cards[delta] = {
        props: false,
        heading: heading,
        content: content,
        link: link,
        cover: cover,
        images: images,
        lightbox: this.lightbox
      }
    }
  },
  data: () => ({
    cards: false,
    lightbox: false,
    gallery: false,
    imageClass: 'h-48',
    gridClass: 'grid-cols-1 md:grid-cols-2 lg:grid-cols-3'
  }),
  methods: {
    ...mapActions({
      getEntity: 'druxtRouter/getEntity',
      getRoute: 'druxtRouter/getRoute',
      getResource: 'druxtRouter/getEntity',
    })
  },
  computed: {
    getVariant() {
      let variant = this.entity.attributes.behavior_settings.lecapi_variants.variant
      if(variant == 'horizontal') {
        this.imageClass = 'h-full'
        this.gridClass = 'grid-cols-1 md:grid-cols-2'
        return "grid grid-cols-1 md:grid-cols-2 gap-2"
      }
      else if(variant == 'lightbox') {
        this.lightbox = true
        this.gallery = true
      }
      else if(variant == 'gallery') {
        this.lightbox = false
        this.gallery = true
      }
      else {
        return variant
      }
    }
  }
}
</script>
<style lang="scss" scoped>
  * {
    box-sizing: border-box;
  }
</style>
