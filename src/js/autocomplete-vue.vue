<style lang="sass">
   .autocomplete {
        position: relative;
        input {
            width: 100%;
        }
    }

    .autocomplete__suggestions {
        position: absolute;
        top: 100%;
        background-color: #fff;
        width: 100%;
        z-index: 2;
    }

    .autocomplete__entry {
        &:hover {
            background-color: #f7f7f9;
            cursor: default;
        }
    }
    
    .autocomplete__selected {
        background-color: darken(#f7f7f9, 5%);
    }
</style>

<template>
  <div :class="classPrefix" @mousedown="mousefocus = true" @mouseout="mousefocus = false">
    <input
      type="text"
      @change="changeEvent"
      @blur="focused=false"
      @focus="focusEvent"
      v-model="search"
      :placeholder="placeholder"
      :class="inputClass"
      @keydown.down.prevent.stop="moveDown()"
      @keydown.up.prevent.stop="moveUp()"
      @keydown.enter.prevent.stop="select(selectedIndex)"
      @keydown.tab="mousefocus = false"
      ref="input"
      :required="required"
    />
    <div v-if="showSuggestions" :class="classPrefix + '__suggestions'">
      <div
        v-for="(entry, index) in filteredEntries"
        @mousedown="selected=true"
        @click="select(index)"
        :class="[classPrefix + '__entry', selectedClass(index)]"
      >{{ entry[matchedProperty] }}</div>
    </div>
  </div>
</template>
<script>
import { autocompleteBus } from "./autocompleteBus.js";

export default {
  data() {
    return {
      entries: [],
      search: "",
      focused: false,
      mousefocus: false,
      selectedIndex: 0,
      properties: [],
      matchedProperty: null,
      selected: null
    };
  },
  computed: {
    filteredEntries() {
      if (this.search.length <= this.threshold) {
        return [];
      } else {
        return this.entries
          .filter(entry => {
            if (this.ignoreCase) {
              return this.properties.find(prop => {
                if (
                  entry[prop].toLowerCase().indexOf(this.search.toLowerCase()) >
                  -1
                ) {
                  this.matchedProperty = prop;
                  return true;
                }
              });
            }

            return this.properties.find(prop => {
              if (entry[prop].indexOf(this.search) > -1) {
                this.matchedProperty = prop;
                return true;
              }
            });
          })
          .slice(0, this.limit);
      }
    },
    hasSuggestions() {
      if (this.search.length <= this.threshold) {
        return false;
      }

      return this.filteredEntries.length > 0;
    },
    showSuggestions() {
      if (!this.hasSuggestions) {
        return false;
      }

      if (this.focused || this.mousefocus) {
        return true;
      }

      return false;
    }
  },
  created() {
    this.search = this.value;
    if (this.list !== undefined) {
      this.setEntries(this.list);
    } else if (this.url !== undefined && this.requestType !== undefined) {
      this.getListAjax();
    }
  },
  mounted() {
    this.properties = !Array.isArray(this.property)
      ? [this.property]
      : this.property;
  },
  methods: {
    select(index) {
      this.selected = true;
      if (this.hasSuggestions) {
        this.search = this.filteredEntries[index][this.matchedProperty];
        autocompleteBus.$emit("autocomplete-select", {
          selected: this.search,
          fullObject: this.filteredEntries[0]
        });
        this.$emit("selected", {
          selected: this.search,
          fullObject: this.filteredEntries[0]
        });

        if (this.autoHide) {
          this.$nextTick(() => {
            this.mousefocus = false;
            this.focused = false;
            this.$refs.input.blur();
          });
        } else {
          this.$nextTick(() => {
            this.$refs.input.focus();
          });
        }
      }
    },
    setEntries(list) {
      if (list) {
        list.then(entry => {
          this.entries = entry;
        });
      }
    },
    moveUp() {
      if (this.selectedIndex - 1 < 0) {
        this.selectedIndex = this.filteredEntries.length - 1;
      } else {
        this.selectedIndex -= 1;
      }
    },
    moveDown() {
      if (this.selectedIndex + 1 > this.filteredEntries.length - 1) {
        this.selectedIndex = 0;
      } else {
        this.selectedIndex += 1;
      }
    },
    selectedClass(index) {
      if (index === this.selectedIndex) {
        return this.classPrefix + "__selected";
      }
      return "";
    },
    getListAjax() {
      return this.$http[this.requestType](this.url).then(response => {
        this.setEntries(response.data);
      });
    },
    changeEvent() {
      this.focused = false;
      if (!this.selected) this.$emit("no-selection");
    },
    focusEvent() {
      this.focused = true;
      this.selected = false;
    }
  },
  props: {
    classPrefix: {
      type: String,
      required: false,
      default: "autocomplete"
    },
    url: {
      type: String,
      required: false
    },
    requestType: {
      type: String,
      required: false,
      default: "get"
    },
    list: {
      type: Array | Promise,
      required: false
    },
    placeholder: {
      type: String,
      required: false
    },
    property: {
      type: String | Array,
      required: false,
      default: "name"
    },
    inputClass: {
      type: String,
      required: false
    },
    required: {
      type: Boolean,
      required: false,
      default: false
    },
    ignoreCase: {
      type: Boolean,
      required: false,
      default: true
    },
    threshold: {
      type: Number,
      required: false,
      default: 0
    },
    value: {
      required: false,
      default: ""
    },
    autoHide: {
      type: Boolean,
      required: false,
      default: true
    },
    limit: {
      type: Number,
      required: false
    }
  },
  watch: {
    filteredEntries(value) {
      if (this.selectedIndex > value.length - 1) {
        this.selectedIndex = 0;
      }
    },
    search(value) {
      this.$emit("input", value);
    },
    value(newValue) {
      this.search = newValue;
    },
    list(_list) {
      this.setEntries(_list);
    }
  }
};
</script>
