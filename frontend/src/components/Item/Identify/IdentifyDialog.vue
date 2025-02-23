<template>
  <v-dialog
    :model-value="model"
    :fullscreen="$vuetify.display.mobile"
    :height="$vuetify.display.mobile ? undefined : 'auto'"
    :width="$vuetify.display.mobile ? undefined : '70vw'"
    @after-leave="emit('close')">
    <v-card height="100%" class="d-flex flex-column" loading>
      <template #loader>
        <v-progress-linear v-model="progress" :indeterminate="isLoading" />
      </template>
      <v-toolbar color="transparent">
        <template #prepend>
          <v-btn icon :disabled="tabName === 'searchMenu'" @click="clear">
            <i-mdi-arrow-left />
          </v-btn>
        </template>
        <template #append>
          <v-btn icon @click="model = false">
            <i-mdi-close />
          </v-btn>
        </template>
        <v-toolbar-title>
          {{ $t('identify') }}
        </v-toolbar-title>
      </v-toolbar>
      <v-card-subtitle v-if="itemPath" class="pb-3">
        {{ itemPath }}
      </v-card-subtitle>

      <v-divider />

      <v-card-text
        class="pa-0 px-2 flex-grow-1"
        :class="{
          'd-flex': !$vuetify.display.mobile,
          'flex-row': !$vuetify.display.mobile
        }">
        <v-window v-model="tabName" class="pa-2 flex-fill">
          <v-window-item value="searchMenu">
            <v-col>
              <v-text-field
                v-for="(field, idx) in searchFields"
                :key="field.key"
                v-model="fieldsInputs[idx].value"
                variant="outlined"
                class="mb-2"
                :placeholder="field.value ?? undefined"
                :persistent-placeholder="field.value ? true : false"
                clearable
                :disabled="isLoading"
                :type="field.type"
                :label="field.title" />
            </v-col>
          </v-window-item>
          <v-window-item value="resultsMenu">
            <h3>{{ $t('results') }}</h3>
            <div class="mt-2 text-subtitle-1">
              {{ $t('identifyInstructResult') }}
            </div>
            <v-checkbox
              v-if="searchResults"
              v-model="replaceImage"
              class="d-flex mt-2"
              color="primary">
              <template #append>
                {{ $t('replaceExistingImages') }}
              </template>
            </v-checkbox>
            <v-divider />
            <identify-results
              v-if="Array.isArray(searchResults) && searchResults.length > 0"
              :items="searchResults"
              :item-type="item.Type"
              @select="applySelectedSearch" />
            <h3 v-else class="text-center my-4">
              {{ $t('searchNoResults') }}
            </h3>
          </v-window-item>
        </v-window>
      </v-card-text>

      <v-divider />
      <v-card-actions
        class="d-flex align-center pa-3"
        :class="{
          'justify-end': !$vuetify.display.mobile,
          'justify-center': $vuetify.display.mobile
        }">
        <v-btn
          v-if="tabName === 'searchMenu'"
          variant="flat"
          width="8em"
          color="primary"
          :loading="isLoading"
          @click="performSearch">
          {{ t('search.name') }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import {
  BaseItemDto,
  RemoteSearchResult
} from '@jellyfin/sdk/lib/generated-client';
import { getItemLookupApi } from '@jellyfin/sdk/lib/utils/api/item-lookup-api';
import { cloneDeep } from 'lodash-es';
import { computed, ref, toRaw } from 'vue';
import { useI18n } from 'vue-i18n';
import { useConfirmDialog, useRemote, useSnackbar } from '@/composables';

interface IdentifyField {
  key: string;
  title: string;
  type: string;
  value?: string | null;
}

const props = defineProps<{ item: BaseItemDto; mediaSourceIndex?: number }>();

const emit = defineEmits<{
  (e: 'close'): void;
}>();

const { t } = useI18n();
const remote = useRemote();

const availableProviders = (
  await remote.sdk.newUserApi(getItemLookupApi).getExternalIdInfos({
    itemId: props.item.Id ?? ''
  })
).data;

const model = ref(true);
const isLoading = ref(false);
const searchResults = ref<RemoteSearchResult[]>();
const replaceImage = ref(false);
const errorMessage = t('errors.anErrorHappened');
const searchFields = computed<IdentifyField[]>(() => {
  const result = [
    {
      key: 'search-name',
      title: t('name'),
      type: 'string',
      value: props.item.Name ?? ''
    }
  ];

  if (!['BoxSet', 'Person'].includes(props.item.Type || '')) {
    result.push({
      key: 'search-year',
      title: t('year'),
      type: 'number',
      value: props.item.ProductionYear ? String(props.item.ProductionYear) : ''
    });
  }

  /**
   * Providers that the item already has
   */
  if (props.item.ProviderIds) {
    for (const key in props.item.ProviderIds) {
      result.push({
        key,
        value: props.item.ProviderIds[key],
        title: `${key} ID`,
        type: 'string'
      });
    }
  }

  /**
   * Providers available for the item type, but not assigned to it
   */
  const populatedKeys = Object.keys(props.item.ProviderIds ?? {});
  const missingProviders = availableProviders
    .filter((p) => !populatedKeys.includes(p.Key ?? ''))
    .map((p) => p.Key)
    .filter((p): p is string => p !== undefined);

  for (const key of missingProviders) {
    result.push({
      key,
      value: '',
      title: `${key} ID`,
      type: 'string'
    });
  }

  return result;
});
const fieldsInputs = ref<IdentifyField[]>(cloneDeep(toRaw(searchFields.value)));
const tabName = computed(() =>
  searchResults.value === undefined ? 'searchMenu' : 'resultsMenu'
);
const progress = computed(() => {
  switch (tabName.value) {
    case 'searchMenu': {
      return 50;
    }
    case 'resultsMenu': {
      return 100;
    }
    default: {
      return 0;
    }
  }
});
const itemPath = computed<string | undefined>(() => {
  if (!props.item) {
    return;
  }

  if (props.mediaSourceIndex !== undefined) {
    return props.item.MediaSources?.[props.mediaSourceIndex]?.Path ?? undefined;
  }

  return props.item.Path ?? undefined;
});

/**
 * Get the remote search results for an item and search params.
 *
 * @param item - The item to search for.
 * @param searches - The search params to use.
 * @returns - An array of remote search results.
 */
async function getItemRemoteSearch(
  item: BaseItemDto,
  fields: IdentifyField[]
): Promise<RemoteSearchResult[] | undefined> {
  interface Query {
    Name?: string;
    Year?: number;
    ProviderIds: { [key: string]: string };
  }

  const remote = useRemote();
  const searcher = remote.sdk.newUserApi(getItemLookupApi);
  const itemId = item.Id;

  if (itemId === undefined) {
    return;
  }

  const searchQuery: Query = {
    Name: undefined,
    Year: undefined,
    ProviderIds: {}
  };

  /**
   * Our search fields are formatted with a key that denotes what
   * they should be used for in the query search.
   *
   * The `search-` prefix are used as the "information" that will be submitted
   * to the providers when searching (usually Name and Year).
   *
   * Every other key are the provider IDs, either prefilled or provided by user.
   * These provider IDs are basically directly use to pull information from said provider.
   */
  for (const field of fields) {
    const value =
      typeof field.value === 'string' ? field.value.trim() : field.value;

    if (field.key === 'search-name' && value) {
      searchQuery.Name = value;
    } else if (field.key === 'search-year' && value) {
      searchQuery.Year = Number(value);
    } else if (value) {
      searchQuery.ProviderIds[field.key] = value;
    }
  }

  switch (item.Type) {
    case 'Book': {
      return (
        await searcher.getBookRemoteSearchResults({
          bookInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'BoxSet': {
      return (
        await searcher.getBoxSetRemoteSearchResults({
          boxSetInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'Movie': {
      return (
        await searcher.getMovieRemoteSearchResults({
          movieInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'MusicAlbum': {
      return (
        await searcher.getMusicAlbumRemoteSearchResults({
          albumInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'MusicArtist': {
      return (
        await searcher.getMusicArtistRemoteSearchResults({
          artistInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'MusicVideo': {
      return (
        await searcher.getMusicVideoRemoteSearchResults({
          musicVideoInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'Person': {
      return (
        await searcher.getPersonRemoteSearchResults({
          personLookupInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'Series': {
      return (
        await searcher.getSeriesRemoteSearchResults({
          seriesInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
    case 'Trailer': {
      return (
        await searcher.getTrailerRemoteSearchResults({
          trailerInfoRemoteSearchQuery: {
            ItemId: itemId,
            SearchInfo: searchQuery
          }
        })
      ).data;
    }
  }
}

/**
 * Do a search for information on the item.
 */
async function performSearch(): Promise<void> {
  isLoading.value = true;

  try {
    const results = await getItemRemoteSearch(props.item, fieldsInputs.value);

    searchResults.value = Array.isArray(results) ? results : [];
  } catch (error) {
    console.error(error);

    useSnackbar(errorMessage, 'error');
  } finally {
    isLoading.value = false;
  }
}

/**
 * Apply the selected search result to the item.
 */
async function applySelectedSearch(result: RemoteSearchResult): Promise<void> {
  await useConfirmDialog(
    async () => {
      isLoading.value = true;

      try {
        await remote.sdk.newUserApi(getItemLookupApi).applySearchCriteria({
          itemId: props.item.Id ?? '',
          remoteSearchResult: result,
          replaceAllImages: replaceImage.value
        });
      } catch (error) {
        console.error(error);

        useSnackbar(errorMessage, 'error');
      } finally {
        isLoading.value = false;
        model.value = false;
      }
    },
    {
      title: '',
      text: t('identifyConfirmChanges', {
        originalItem: props.item.Name,
        newIdentifiedItem: result.Name
      }),
      confirmText: t('confirm')
    }
  );
}

/**
 * Clear search results
 */
function clear(): void {
  searchResults.value = undefined;
}
</script>
