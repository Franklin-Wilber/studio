
<template>

  <KModal
    :title="title"
    :submitText="$tr('saveAction')"
    :submitDisabled="!canSave"
    :cancelText="$tr('cancelAction')"
    data-test="edit-booleanMap-modal"
    @submit="handleSave"
    @cancel="close"
  >
    <p v-if="resourcesSelectedText.length > 0" data-test="resources-selected-message">
      {{ resourcesSelectedText }}
    </p>
    <p v-if="hasMixedCategories" data-test="mixed-categories-message">
      {{ hasMixedCategoriesMessage }}
    </p>
    <template v-if="isDescendantsUpdatable && isTopicSelected">
      <KCheckbox
        :checked="updateDescendants"
        data-test="update-descendants-checkbox"
        :label="$tr('updateDescendantCheckbox')"
        @change="(value) => { updateDescendants = value }"
      />
      <Divider />
    </template>
    <slot
      name="input"
      :value="selectedValues"
      :inputHandler="(value) => { selectedValues = value }"
    ></slot>

  </KModal>

</template>


<script>

  /**
   * EditBooleanMapModal
   * This component is a modal responsible for reusing the logic of saving
   * the edition of a boolean map field for multiple nodes.
   */
  import isEqual from 'lodash/isEqual';
  import { mapGetters, mapActions } from 'vuex';
  import { ContentKindsNames } from 'shared/leUtils/ContentKinds';
  import { getInvalidText } from 'shared/utils/validation';
  import commonStrings from 'shared/translator';

  export default {
    name: 'EditBooleanMapModal',
    props: {
      nodeIds: {
        type: Array,
        required: true,
      },
      field: {
        type: String,
        required: true,
      },
      title: {
        type: String,
        required: true,
      },
      isDescendantsUpdatable: {
        type: Boolean,
        default: false,
      },
      confirmationMessage: {
        type: String,
        required: true,
      },
      validators: {
        type: Array,
        default: () => [],
      },
      resourcesSelectedText: {
        type: String,
        default: '',
      },
    },
    data() {
      return {
        updateDescendants: false,
        error: '',
        /**
         * selectedValues is an object with the following structure:
         * {
         *  [optionId]: [nodeId1, nodeId2, ...]
         * }
         * Where nodeIds is the id of the nodes that have the option selected
         */
        selectedValues: {},
        changed: false,
      };
    },
    computed: {
      ...mapGetters('contentNode', ['getContentNodes', 'getContentNode']),
      nodes() {
        return this.getContentNodes(this.nodeIds);
      },
      isTopicSelected() {
        return this.nodes.some(node => node.kind === ContentKindsNames.TOPIC);
      },
      canSave() {
        return Object.values(this.selectedValues).some(value => value.length > 0);
      },
      hasMixedCategories() {
        const allSelectedCategories = new Set(Object.keys(this.selectedValues));
        for (const categoryId of allSelectedCategories) {
          const nodesWithThisCategory = this.selectedValues[categoryId].length;

          if (nodesWithThisCategory < this.nodes.length) {
            return true;
          }
        }
        return false;
      },
      hasMixedCategoriesMessage() {
        // eslint-disable-next-line kolibri/vue-no-undefined-string-uses
        return commonStrings.$tr('addAdditionalCatgoriesDescription');
      },
    },
    watch: {
      selectedValues(newValue, oldValue) {
        this.validate();
        this.changed = !isEqual(newValue, oldValue);
      },
    },
    created() {
      const optionsNodes = {};

      this.nodes.forEach(node => {
        Object.entries(node[this.field] || {})
          .filter(entry => entry[1] === true)
          .forEach(([optionId]) => {
            optionsNodes[optionId] = optionsNodes[optionId] || [];
            optionsNodes[optionId].push(node.id);
          });
      });

      this.selectedValues = optionsNodes;
      // reset
      this.$nextTick(() => {
        this.changed = false;
      });
    },
    methods: {
      ...mapActions('contentNode', ['updateContentNode', 'updateContentNodeDescendants']),
      close(changed = false) {
        this.$emit('close', {
          changed: this.error ? false : changed,
          updateDescendants: this.updateDescendants,
        });
      },
      validate() {
        if (this.validators && this.validators.length) {
          this.error = getInvalidText(
            this.validators,
            Object.keys(this.selectedValues).filter(
              key => this.selectedValues[key].length === this.nodes.length
            )
          );
        } else {
          this.error = '';
        }
      },
      async handleSave() {
        await Promise.all(
          this.nodes.map(async node => {
            const fieldValue = {};
            const currentNode = this.getContentNode(node.id);

            Object.entries(this.selectedValues).forEach(([key, value]) => {
              const existingValue = currentNode[this.field]?.[key] || false;
              fieldValue[key] = existingValue || value.includes(node.id);
            });

            if (this.updateDescendants && node.kind === ContentKindsNames.TOPIC) {
              return this.updateContentNodeDescendants({
                id: node.id,
                [this.field]: fieldValue,
              });
            }
            return this.updateContentNode({
              id: node.id,
              [this.field]: fieldValue,
            });
          })
        );
        this.$store.dispatch('showSnackbarSimple', this.confirmationMessage || '');
        this.close(this.changed);
      },
    },
    $trs: {
      saveAction: 'Save',
      cancelAction: 'Cancel',
      updateDescendantCheckbox:
        'Apply to all resources, folders, and subfolders contained within the selected folders.',
    },
  };

</script>

<style scoped>
</style>
