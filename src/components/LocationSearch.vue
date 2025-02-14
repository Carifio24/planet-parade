<template>
  <div
    class="forward-geocoding-container"
    :style="cssStyles"
  >
    <div
      class="forward-geocoding-input-row"
    >
      <v-text-field
        v-show="searchOpen"
        v-model="searchText"
        :class="['forward-geocoding-input', locationJustUpdated ? 'geocode-success' : '', small ? 'forward-geocoding-input-small' : '']"
        :label="locationJustUpdated ? 'Location Updated' : 'Enter a location'"
        bg-color='black'
        density="compact"
        hide-details
        variant="solo"
        :color="accentColor"
        @keydown.stop
        @keyup.enter="() => performForwardGeocodingSearch()"
        @keyup.esc="searchResults = null"
        @click:clear="searchResults = null"
        :error-messages="searchErrorMessage"
      ></v-text-field>
      <font-awesome-icon
        class="geocoding-search-icon"
        icon="magnifying-glass"
        :size="searchOpen ? 'xl' : buttonSize"
        :color="!searchOpen || (searchText && searchText.length > 2) ? accentColor : 'gray'"
        @click="() => {
          if (searchOpen) {
            performForwardGeocodingSearch();
          } else {
            searchOpen = true;
          }
        }"
      ></font-awesome-icon>

      <slot name="append-icon" class="geocode-icon"></slot>

      <font-awesome-icon
        class="geocoding-close-icon"
        v-show="searchOpen && !stayOpen"
        icon="circle-xmark"
        :size="searchOpen ? 'xl' : '1x'"
        color="gray"
        @click="() => {
          clearSearchData();
        }"
      ></font-awesome-icon>
      
    </div>
    
    <div
      class="forward-geocoding-results"
      :class="[small ? 'results-small' : '']"
      v-if="searchResults !== null"
    >
      <div
        v-for="(feature, index) in (searchResults !== null ?  searchResults.features : [])"
        class="forward-geocoding-result"
        :key="index"
        @click="() => setLocationFromSearchFeature(feature)"
      >
        {{ feature.place_name }}
      </div>
    </div>
  </div>
</template> 

<script setup lang="ts">
import { computed, ref } from "vue";
import { MapBoxFeature, MapBoxFeatureCollection } from "@cosmicds/vue-toolkit/src/mapbox";
import { watch } from "vue";

type SearchProvider = (searchText: string) => Promise<MapBoxFeatureCollection | null>;

export interface LocationSearchProps {
  searchProvider?: SearchProvider;
  modelValue?: boolean;
  stayOpen?: boolean;
  accentColor?: string;
  small?: boolean;
  theme?: string;
  buttonSize?: string;
}

const props = withDefaults(defineProps<LocationSearchProps>(), {
  searchProvider: (_searchText: string) => null,
  modelValue: true,
  stayOpen: false,
  accentColor: "white",
  small: false,
  theme: "dark",
  buttonSize: "1x",
});

const emit = defineEmits<{
  (e: "error", message: string): void
  (e: "set-location", feature: MapBoxFeature): void
  (e: "update:modelValue", value: boolean): void
}>();

const searchOpen = ref(props.modelValue || props.stayOpen);
const searchText = ref<string | null>(null);
const searchResults = ref<MapBoxFeatureCollection | null>(null);
const searchErrorMessage = ref<string | null>(null);
const locationJustUpdated = ref(false);

const cssStyles = computed(() => {
  return {
    "--accent-color": props.accentColor,
    "--bg-color": "black",
    "--fg-container-padding": searchOpen.value ? (props.small ? "0px 5px 0px 0px" : "5px 10px 12px 10px") : "0px",
    "--border-radius": searchOpen.value ? "7px" : "20px",
  };
});

function performForwardGeocodingSearch() {
  const search = searchText.value;
  if (search === null || search.length < 3) {
    return;
  }

  props.searchProvider(search).then(info => {
    if (info !== null && info.features.length === 1) {
      setLocationFromSearchFeature(info.features[0]);
    } else if (info !== null && info.features.length === 0) {
      searchErrorMessage.value = "No matching places were found";
      emit("error", searchErrorMessage.value); 
    } else {
      searchResults.value = info;
    }
  });
}

function setLocationFromSearchFeature(feature: MapBoxFeature) {
  timedJustUpdatedLocation();
  clearSearchData();
  emit("set-location", feature);
}

function clearSearchData() {
  searchResults.value = null;
  searchText.value = null;
  searchErrorMessage.value = null;
}

function timedJustUpdatedLocation() {
  locationJustUpdated.value = true;
  setTimeout(() => {
    locationJustUpdated.value = false;
  }, 5000);
}

watch(() => props.modelValue, (value: boolean) => { searchOpen.value = value; });

watch(searchOpen, (value: boolean) => emit("update:modelValue", value));

watch(searchText, function(text: string | null) {
  if (searchErrorMessage.value) {
    searchErrorMessage.value = null;
  }
  if (!text || text.length === 0) {
    searchResults.value = null;
  }
});
</script>

<style lang="less">
.forward-geocoding-container {
  width: fit-content;
  color: var(--accent-color);
  background-color: black;
  border: 2px solid var(--accent-color);
  border-radius: 7px;
  padding: var(--fg-container-padding);

  .v-text-field {
    min-width: 150px;
    width: min(200px, 20vw);
  }

  .forward-geocoding-input > .v-input__control > .v-field {
    border-radius: var(--border-radius);
  }
  
  .forward-geocoding-input.geocode-success label {
    color: var(--accent-color);
    opacity: 1;
  }

  .forward-geocoding-input-row {
    display: flex;
    flex-direction: row;
    justify-content: space-around;
    gap: 10px;
    align-items: center;
  }

  .geocoding-search-icon {
    padding-inline: calc(0.3 * var(--default-line-height));
    padding-block: calc(0.4 * var(--default-line-height));
  }

  .geocoding-search-icon:hover, .geocoding-close-icon:hover {
    cursor: pointer;
  }

  // For some reason setting width: 100% makes the search results 2px too small
  // It's probably some Vuetify styling thing
  // Maybe there's a better workaround, but this gets the job done for now
  .forward-geocoding-results {
    position: absolute;
    top: 42px;
    left: -1px;
    width: calc(100% + 2px);
    background: black;
    border: 2px solid var(--accent-color);
    border-top: 0px;
    border-bottom-left-radius: 10px;
    border-bottom-right-radius: 10px;
    padding: 0px 10px;

    &.results-small {
      top: 37px;
      width: calc(100% + 4px);
      left: -2px;
    }

    .forward-geocoding-result {
      border-top: 2px solid var(--accent-color);
      font-size: 12pt;
      pointer-events: auto;

      &:hover {
        cursor: pointer;
      }
    }
  }
}
</style>
