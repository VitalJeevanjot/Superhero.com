<template>
  <div class="tips-and-comments">
    <BackButtonRibbon />
    <div
      v-if="record"
      class="tipped__url"
    >
      <Component
        :is="id ? 'TipComment' : 'TipRecord'"
        v-bind="{ [id ? 'comment' : 'tip']: record }"
      />
    </div>
    <div
      v-if="record"
      class="comment__section"
    >
      <p class="latest__comments">
        {{ $t('views.TipCommentsView.LatestReplies') }}
      </p>
      <SendComment
        :tip-id="tipId"
        :parent-id="id"
      />
    </div>
    <div class="comments__section">
      <Loading
        v-if="showLoading"
        :above-content="!!(record && nestedComments.length)"
      />
      <template v-if="record">
        <div
          v-if="nestedComments.length === 0 && !showLoading"
          class="no-results text-center w-100"
          :class="[error ? 'error' : '']"
        >
          {{ $t('views.TipCommentsView.NoResultsMsg') }}
        </div>

        <Component
          :is="id ? 'TipComment' : 'TipCommentList'"
          v-for="(nestedComment, index) in nestedComments"
          :key="index"
          :comment="nestedComment"
        />
      </template>
    </div>
  </div>
</template>

<script>
import TipComment from '../components/tipRecords/TipComment.vue';
import TipRecord from '../components/tipRecords/TipRecord.vue';
import TipCommentList from '../components/tipRecords/TipCommentList.vue';
import BackButtonRibbon from '../components/BackButtonRibbon.vue';
import Loading from '../components/Loading.vue';
import { EventBus } from '../utils/eventBus';
import backendAuthMixin from '../utils/backendAuthMixin';
import SendComment from '../components/SendComment.vue';

export default {
  components: {
    Loading,
    TipRecord,
    TipCommentList,
    BackButtonRibbon,
    SendComment,
    TipComment,
  },
  mixins: [backendAuthMixin()],
  props: {
    id: { type: [String, Number], default: '' },
    tipId: { type: [String, Number], required: true },
  },
  data: () => ({
    showLoading: false,
    error: false,
  }),
  computed: {
    record() {
      return this.id
        ? this.$store.state.backend.comment[this.id]
        : this.$store.state.backend.tip[this.tipId];
    },
    nestedComments() {
      return this.id ? this.record.children : this.record.comments;
    },
  },
  async mounted() {
    const handler = () => this.reloadData();
    this.$watch(({ id }) => id, handler, { immediate: true });
    EventBus.$on('reloadData', handler);
    const interval = setInterval(handler, 120 * 1000);

    this.$once('hook:destroyed', () => {
      EventBus.$off('reloadData', handler);
      clearInterval(interval);
    });
  },
  methods: {
    async handleBackendSucceedCall(methodName) {
      switch (methodName) {
        case 'unPinItem':
        case 'pinItem':
          await this.$store.dispatch('updatePinnedItems');
          break;
        case 'sendPostReport':
          await this.$store.dispatch('modals/open', {
            name: 'success',
            title: this.$t('components.tipRecords.TipRecord.reportPostTitle'),
            body: this.$t('components.tipRecords.TipRecord.reportPostBody'),
          });
          break;
        default:
      }
    },
    async reloadData() {
      this.showLoading = true;
      try {
        await this.$store.dispatch(this.id
          ? 'backend/reloadComment' : 'backend/reloadTip', this.id || this.tipId);
      } catch (error) {
        this.error = true;
        throw error;
      } finally {
        this.showLoading = false;
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.tips-and-comments {
  color: $light_font_color;
  font-size: 0.75rem;

  .tipped__url {
    .tip__record {
      margin-bottom: 0;

      &.row {
        background-color: $thumbnail_background_color;
      }
    }

    .tip-comment.row {
      background-color: $thumbnail_background_color;
      border-radius: 0;
      margin-bottom: 0;

      ::v-deep .body .note {
        overflow: visible;
      }
    }
  }

  .tipped__url .tip__record.row ::v-deep .tip__body .tip__article {
    background-color: $thumbnail_background_color_alt;

    .preview__image {
      background-color: $thumbnail_background_color_alt;
    }

    &:hover {
      background-color: #373843;

      .preview__image {
        background-color: #373843;
      }
    }
  }

  .comments__section {
    background-color: $thumbnail_background_color;
    padding: 1rem;
    position: relative;
  }

  .no-results {
    color: $standard_font_color;
    font-size: 0.75rem;
    text-align: center;

    &.error {
      color: $red_color;
    }
  }

  .comment__section {
    background-color: $thumbnail_background_color;
    padding: 0.75rem 1rem 0 1rem;

    p {
      font-size: 0.75rem;
      text-transform: capitalize;
      margin-bottom: 0.7rem;
      color: white;
      font-weight: 600;
    }
  }
}
</style>