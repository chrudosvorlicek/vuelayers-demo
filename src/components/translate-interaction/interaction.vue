<script>
  /** @module translate-interaction/interaction */
  import TranslateInteraction from 'ol/interaction/Translate'
  import { Source } from 'ol/source'
  import { interaction } from 'vuelayers/lib/mixin'
  import { makeWatchers } from 'vuelayers/lib/util/vue-helpers'
  import observableFromOlEvent from 'vuelayers/lib/rx-ext/from-ol-event'
  import { isFunction } from 'vuelayers/lib/util/minilo'
  import { hasInteraction, instanceOf } from 'vuelayers/lib/util/assert'
  /**
   * @vueProto
   * @alias module:translate-interaction/interaction
   * @title vl-interaction-translate
  */
  export default {
    name: 'vl-interaction-translate',
    mixins: [interaction],
    props: {
      /**
       * Source or collection identifier from IdentityMap.
       * @type {String}
       */
      source: {
        type: String,
        required: false,
      },
      features: {
        type: Array,
        default: undefined,
      },
      /**
       * Hit-detection tolerance. Pixels inside the radius around the given position will be checked for features.
       * This only works for the canvas renderer and not for WebGL.
       * @type {number}
       */
      hitTolerance: {
        type: Number,
        default: 0,
      },
      priority: {
        type: Number,
        default: 1000,
      },
    },
    methods: {
      /**
       * @return {Promise<ol.interaction.Translate>}
       * @protected
       */
      async createInteraction () {
        if (this.source !== undefined) {
          let source = await this.getInstance(this.source)
          instanceOf(source, Source, `Source "${this.source}" doesn't exists in the identity map.`)
          if (isFunction(source.getFeatures)) {
            this.features = source.getFeatures()
          }
        }
        return new TranslateInteraction({
          features: this.features.length > 0 ? this.features : undefined,
          hitTolerance: this.hitTolerance,
        })
      },
      /**
       * @return {void}
       * @protected
       */
      mount () {
        this::interaction.methods.mount()
      },
      /**
       * @return {void}
       * @protected
       */
      unmount () {
        this::interaction.methods.unmount()
      },
      /**
       * @return {void}
       * @protected
       */
      subscribeAll () {
        this::interaction.methods.subscribeAll()
        this::subscribeToInteractionChanges()
      },
    },
    watch: {
      ...makeWatchers(['source'], () => function () {
        this.recreate()
      }),
    },
  }
  
  /**
   * @private
   */
  function subscribeToInteractionChanges () {
    hasInteraction(this)
    const translateEvents = observableFromOlEvent(this.$interaction, ['translatestart', 'translateend'])
    this.subscribeTo(translateEvents, evt => {
      ++this.rev
      this.$emit(evt.type, evt)
    })
  }
</script>