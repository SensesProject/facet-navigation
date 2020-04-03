<template>
  <div class="facet facet-list">
    <FacetHeader
      :title="title"
      :isFiltered="isFiltered"
      :isInverted="isInverted"
      :isReverse="isReverse"
      :isAlphabetical="isAlphabetical"
      :tooltip="tooltip"
      :count="count"
      @removeFacet="() => removeFacet(id)"
      @toggleInvert="toggleInvert"
      @toggleReverse="toggleReverse"
      @toggleAlphabetical="toggleAlphabetical"
      @reset="reset" />
    <ul :class="['list-list', { isFiltered }]">
      <li
        v-for="{ key, total, isActive, n } in elements"
        :key="key"
        :class="['option', { isActive: selected.includes(key), nonRemaining: items[n].value === 0 }]">
        <span class="label" @click="() => selectItem(key)">{{ key }}</span>
        <span class="counter"><span v-if="items[n].value !== total">{{ items[n].value }}/</span>{{ total }}</span>
        <span class="action action-add" @click="() => addItem(key)">Include</span>
        <span class="action action-remove" @click="() => removeItem(key)">Remove</span>
      </li>
    </ul>
  </div>
</template>

<script>
import FacetHeader from './FacetHeader.vue'
import { mapActions } from 'vuex'
import { pull, map, without, sortBy, reverse, isEqual } from 'lodash'

const LIST_DEFAULT = []
const INVERTED_DEFAULT = false
const REVERSE_DEFAULT = false
const SORTING_DEFAULT = false

export default {
  name: 'FacetList',
  props: {
    items: {
      type: Array,
      default: () => []
    },
    init: {
      type: Object,
      default: () => {}
    },
    title: {
      type: String
    },
    id: {
      type: String
    },
    tooltip: {
      type: String
    }
  },
  components: {
    FacetHeader
  },
  data: function () {
    return {
      selected: LIST_DEFAULT, // List of selected items
      isInverted: INVERTED_DEFAULT, // Is the list of selection inverted
      isReverse: REVERSE_DEFAULT, // Is the sorting of the items reverse
      isAlphabetical: SORTING_DEFAULT // Sorting is alphabetical or by counts
    }
  },
  computed: {
    isFiltered () {
      return this.selected !== LIST_DEFAULT
    },
    keys () {
      // Get a list of all item to work with inverting
      return map(this.items, ({ key }) => key)
    },
    count () {
      // Count the number of options
      return this.keys.length
    },
    elements () {
      const { isAlphabetical, isReverse, init } = this
      // This is important: we get the position in the array by counting
      // Because later we will sort the array, but the original position is saved in the counter
      // This way, we can easily access the current value in the items
      // By doing this, we can safe time because we do not need to loop over this array all the time
      let n = -1
      let list = map(init, (total, key) => {
        n++
        return {
          n,
          key,
          total
        }
      })
      list = isAlphabetical ? sortBy(list, ['key', 'total']) : sortBy(list, ['total', 'key'])
      return isReverse ? list : reverse(list)
    }
  },
  methods: {
    ...mapActions('filter', [
      'filter',
      'removeFacet',
      'resetFilter'
    ]),
    toggleInvert () {
      this.isInverted = !this.isInverted
      this.apply()
    },
    toggleReverse () {
      this.isReverse = !this.isReverse
    },
    toggleAlphabetical () {
      this.isAlphabetical = !this.isAlphabetical
      this.isReverse = !this.isAlphabetical
    },
    reset () {
      this.selected = LIST_DEFAULT
      this.apply()
    },
    selectItem (value) {
      if (isEqual(this.selected, [value])) {
        this.reset()
      } else {
        this.selected = [value]
        this.apply()
      }
    },
    addItem (value) {
      // Check if item is not already in the list
      if (!this.selected.includes(value)) {
        this.selected.push(value)
        this.apply()
      }
    },
    removeItem (value) {
      // Check if item is in list
      if (this.selected.includes(value)) {
        pull(this.selected, value)
        this.apply()
      }
    },
    apply () {
      // Creating a temporary list to handle inverted list
      let items
      if (this.isInverted) {
        // If list is inverted, the list of all keys is subtracted by the selected list
        items = without(this.keys, ...this.selected)
      } else {
        items = this.selected
      }

      // Check number of items
      const n = items.length
      switch (n) {
        case 0:
        case this.keys.length:
          // If all or no key is selected, reset the filter
          this.resetFilter(this.id)
          break
        case 1:
          // If only one key is selected, we can use filterExact
          this.filter({ key: this.id, value: items[0] })
          break
        default:
          // Per default pass the whole array for filtering
          this.filter({ key: this.id, value: items })
          break
      }
    }
  }
}
</script>

<style lang="scss" scoped>
  @import "~@/assets/style/global";

  .isActive {
    font-weight: bold;
    cursor: default;
  }
  .nonRemaining {
    text-decoration: line-through;
  }

  .facet-list {
    .list-list {
      list-style: none;
      background-color: #fff;
      padding: 0.5rem;
      border: 1px solid $color-pale-gray;
      border-radius: $border-radius;
      box-shadow: inset 0 1px 1px 0 rgba(0, 0, 0, 0.05);
      overflow-y: scroll;
      height: calc(1.5rem * 10);

      .option {
        display: flex;
        position: relative;
        justify-content: space-between;
        white-space: nowrap;
        overflow: hidden;
        font-size: $size-smallest;
        color: $color-deep-gray;
        transition: color 0.2s;
        height: 1.3rem;

        .label {
          overflow: hidden;
          max-width: 70%;
          text-overflow: ellipsis;
          cursor: pointer;
        }

        .action {
          cursor: pointer;
        }

        .action-add, .action-remove {
          position: absolute;
          right: 0;
          opacity: 0;
          pointer-events: none;
        }

        &:hover {
          color: $color-accent;

          &.isActive {
            .action-remove {
              opacity: 1;
              pointer-events: all;
            }
            .counter {
              opacity: 0;
            }
          }
        }

        &.nonRemaining {
          opacity: 0.5;

          .label {
            text-decoration: line-through;
          }
        }
      }

      &.isFiltered {
        .option:hover:not(.isActive) {
          .action-add {
            opacity: 1;
            pointer-events: all;
          }
          .counter {
            opacity: 0;
          }
        }
      }
    }
  }
</style>