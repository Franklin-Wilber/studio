<template>

  <LoadingText v-if="loading" />
  <VLayout
    v-else-if="node && !children.length"
    class="pa-4"
    justify-center
    fill-height
  >
    <VFlex v-if="isRoot && canEdit" class="text-xs-center">
      <h1 class="font-weight-bold headline mb-2">
        {{ $tr('emptyChannelText') }}
      </h1>
      <p class="subheading">
        {{ $tr('emptyChannelSubText') }}
      </p>
    </VFlex>
    <VFlex v-else-if="isRoot" class="subheading text-xs-center">
      {{ $tr('emptyViewOnlyChannelText') }}
    </VFlex>
    <VFlex v-else class="subheading text-xs-center">
      {{ $tr('emptyTopicText') }}
    </VFlex>
  </VLayout>
  <div v-else class="node-list" @scroll="$emit('scroll', $event)">
    <VList class="py-0">
      <template
        v-for="child in children"
      >
        <ContentNodeEditListItem
          :key="child.id"
          :nodeId="child.id"
          :compact="isCompactViewMode"
          :comfortable="isComfortableViewMode"
          :select="selected.indexOf(child.id) >= 0"
          :previewing="$route.params.detailNodeId === child.id"
          :hasSelection="selected.length > 0"
          data-test="node-list-item"
          @select="$emit('select', child.id)"
          @deselect="$emit('deselect', child.id)"
          @infoClick="goToNodeDetail(child.id)"
          @topicChevronClick="goToTopic(child.id)"
          @dblclick.native="onNodeDoubleClick(child)"
          @editTitleDescription="$emit('editTitleDescription', child.id)"
        />
      </template>
    </VList>
    <div class="pagination-container">
      <KButton v-if="more" :disabled="moreLoading" @click="loadMore">
        {{ $tr('showMore') }}
      </KButton>
    </div>
  </div>

</template>

<script>

  import { mapActions, mapGetters } from 'vuex';

  import ContentNodeEditListItem from '../components/ContentNodeEditListItem';
  import { RouteNames } from '../constants';
  import { ContentKindsNames } from 'shared/leUtils/ContentKinds';
  import LoadingText from 'shared/views/LoadingText';

  export default {
    name: 'NodePanel',
    components: {
      ContentNodeEditListItem,
      LoadingText,
    },
    props: {
      parentId: {
        type: String,
        required: true,
      },
      selected: {
        type: Array,
        default() {
          return [];
        },
      },
    },
    data() {
      return {
        loading: false,
        more: null,
        moreLoading: false,
      };
    },
    computed: {
      ...mapGetters(['isCompactViewMode', 'isComfortableViewMode']),
      ...mapGetters('currentChannel', ['rootId', 'canEdit']),
      ...mapGetters('contentNode', [
        'getContentNode',
        'getContentNodeChildren',
        'isNodeInCopyingState',
      ]),
      node() {
        return this.getContentNode(this.parentId);
      },
      children() {
        return this.getContentNodeChildren(this.parentId);
      },
      isRoot() {
        return this.rootId === this.parentId;
      },
    },
    created() {
      this.loading = true;
      this.loadChildren({ parent: this.parentId }).then(childrenResponse => {
        this.loading = false;
        this.more = childrenResponse.more || null;
      });
    },
    methods: {
      ...mapActions('contentNode', ['loadChildren', 'loadContentNodes']),
      goToNodeDetail(nodeId) {
        if (
          this.$route.params.nodeId === this.parentId &&
          this.$route.params.detailNodeId === nodeId
        ) {
          return;
        }

        this.$router.push({
          name: RouteNames.TREE_VIEW,
          params: {
            nodeId: this.parentId,
            detailNodeId: nodeId,
          },
        });
      },
      goToTopic(topicId) {
        if (this.$route.params.nodeId === topicId && !this.$route.params.detailNodeId) {
          return;
        }

        this.$router.push({
          name: RouteNames.TREE_VIEW,
          params: {
            nodeId: topicId,
            detailNodeId: null,
          },
        });

        this.$analytics.trackAction('channel_editor', 'Open topic', {
          eventLabel: this.node.title,
        });
      },
      onNodeDoubleClick(node) {
        // Don't try to navigate to nodes that are still copying
        if (!this.isNodeInCopyingState(node.id)) {
          if (node.kind === ContentKindsNames.TOPIC) {
            this.goToTopic(node.id);
          } else {
            this.goToNodeDetail(node.id);
          }
        }
      },
      loadMore() {
        if (this.more && !this.moreLoading) {
          this.moreLoading = true;
          this.loadContentNodes(this.more).then(response => {
            this.more = response.more || null;
            this.moreLoading = false;
          });
        }
      },
    },
    $trs: {
      emptyViewOnlyChannelText: 'Nothing in this channel yet',
      emptyTopicText: 'Nothing in this folder yet',
      emptyChannelText: 'Click "ADD" to start building your channel',
      emptyChannelSubText: 'Create, upload, or import resources from other channels',
      showMore: 'Show more',
    },
  };

</script>

<style lang="less" scoped>

  .node-list {
    width: 100%;
    height: max-content;
    min-height: 100%;
    padding: 0;
    padding-bottom: 88px;
    /* stylelint-disable-next-line custom-property-pattern */
    background-color: var(--v-backgroundColor-base);
  }

  .pagination-container {
    display: flex;
    justify-content: flex-start;
    margin: 4px;
  }

</style>
