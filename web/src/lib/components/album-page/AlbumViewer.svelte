<script lang="ts">
  import { shortcut } from '$lib/actions/shortcut';
  import AlbumMap from '$lib/components/album-page/AlbumMap.svelte';
  import DownloadAction from '$lib/components/timeline/actions/DownloadAction.svelte';
  import SelectAllAssets from '$lib/components/timeline/actions/SelectAllAction.svelte';
  import AssetSelectControlBar from '$lib/components/timeline/AssetSelectControlBar.svelte';
  import Timeline from '$lib/components/timeline/Timeline.svelte';
  import { assetMultiSelectManager } from '$lib/managers/asset-multi-select-manager.svelte';
  import { assetViewerManager } from '$lib/managers/asset-viewer-manager.svelte';
  import { featureFlagsManager } from '$lib/managers/feature-flags-manager.svelte';
  import { TimelineManager } from '$lib/managers/timeline-manager/timeline-manager.svelte';
  import { handleDownloadAlbum } from '$lib/services/album.service';
  import { getGlobalActions } from '$lib/services/app.service';
  import { dragAndDropFilesStore } from '$lib/stores/drag-and-drop-files.store';
  import { mediaQueryManager } from '$lib/stores/media-query-manager.svelte';
  import { SlideshowNavigation, SlideshowState, slideshowStore } from '$lib/stores/slideshow.store';
  import { handlePromiseError } from '$lib/utils';
  import { fileUploadHandler, openFileUploadDialog } from '$lib/utils/file-uploader';
  import type { AlbumResponseDto, SharedLinkResponseDto } from '@immich/sdk';
  import { ActionButton, IconButton } from '@immich/ui';
  import { mdiDownload, mdiFileImagePlusOutline, mdiPresentationPlay } from '@mdi/js';
  import { t } from 'svelte-i18n';
  import { wedding } from '$lib/config/wedding';
  import ControlAppBar from '../shared-components/ControlAppBar.svelte';
  import ThemeButton from '../shared-components/ThemeButton.svelte';
  import AlbumSummary from './AlbumSummary.svelte';

  interface Props {
    sharedLink: SharedLinkResponseDto;
  }

  let { sharedLink }: Props = $props();

  const album = sharedLink.album as AlbumResponseDto;

  let { slideshowState, slideshowNavigation } = slideshowStore;

  const options = $derived({ albumId: album.id, order: album.order });
  let timelineManager = $state<TimelineManager>() as TimelineManager;

  dragAndDropFilesStore.subscribe((value) => {
    if (!(value.isDragging && value.files.length > 0)) {
      return;
    }

    handlePromiseError(fileUploadHandler({ files: value.files, albumId: album.id }));
    dragAndDropFilesStore.set({ isDragging: false, files: [] });
  });

  const handleStartSlideshow = async () => {
    const asset =
      $slideshowNavigation === SlideshowNavigation.Shuffle
        ? await timelineManager.getRandomAsset()
        : timelineManager.months[0]?.timelineDays[0]?.viewerAssets[0]?.asset;
    if (asset) {
      handlePromiseError(
        assetViewerManager.setAssetId(asset.id).then(() => ($slideshowState = SlideshowState.PlaySlideshow)),
      );
    }
  };

  const { Cast } = $derived(getGlobalActions($t));
</script>

<svelte:document
  use:shortcut={{
    shortcut: { key: 'Escape' },
    onShortcut: () => {
      if (!assetViewerManager.isViewing && assetMultiSelectManager.selectionActive) {
        assetMultiSelectManager.clear();
      }
    },
  }}
/>

<main class="wedding-gallery relative h-dvh overflow-hidden px-2 pt-(--navbar-height) max-md:pt-(--navbar-height-md) md:px-6">
  <Timeline enableRouting={true} {album} bind:timelineManager {options} assetInteraction={assetMultiSelectManager}>
    <section class="wedding-gallery__hero px-2 pt-8 md:px-0 md:pt-24">
      <!-- ALBUM TITLE -->
      <div class="wedding-gallery__art" aria-hidden="true">
        <img src={wedding.heroImage} alt="" />
      </div>
      <p class="wedding-gallery__eyebrow">{wedding.galleryEyebrow}</p>
      <div class="wedding-gallery__title-row">
        <h1 class="wedding-gallery__title transition-all outline-none">{album.albumName}</h1>
      </div>
      <div class="wedding-gallery__meta"><span>{wedding.couple}</span><span>{wedding.date}</span></div>

      {#if sharedLink.allowUpload}
        <button
          class="wedding-upload-cta"
          type="button"
          onclick={() => openFileUploadDialog({ albumId: album.id })}
        >
          <span class="wedding-upload-cta__eyebrow">Você registrou um momento especial?</span>
          <span class="wedding-upload-cta__label">Enviar fotos para Rebeca &amp; Junior <span aria-hidden="true">↗</span></span>
          <span class="wedding-upload-cta__hint">Clique aqui para selecionar fotos ou vídeos</span>
        </button>
      {/if}

      {#if album.assetCount > 0}
        <AlbumSummary {album} />
      {/if}

      <!-- ALBUM DESCRIPTION -->
      {#if album.description}
        <p
          class="wedding-gallery__description mt-6 mb-12 w-full pb-2 text-start whitespace-pre-line"
        >
          {album.description}
        </p>
      {/if}
    </section>
  </Timeline>
</main>

<header>
  {#if assetMultiSelectManager.selectionActive}
    <AssetSelectControlBar>
      <SelectAllAssets {timelineManager} assetInteraction={assetMultiSelectManager} />
      {#if sharedLink.allowDownload}
        <DownloadAction filename="{album.albumName}.zip" />
      {/if}
    </AssetSelectControlBar>
  {:else}
    <ControlAppBar>
      {#snippet leading()}
        <a data-sveltekit-preload-data="hover" class="wedding-wordmark ms-4" href="/" aria-label={wedding.couple}>
          <img src={wedding.monogramImage} alt="" />
          {#if !mediaQueryManager.maxMd}<span>{wedding.couple}</span>{/if}
        </a>
      {/snippet}

      {#snippet trailing()}
        <ActionButton action={Cast} />

        {#if sharedLink.allowUpload}
          <IconButton
            shape="round"
            color="secondary"
            variant="ghost"
            aria-label={$t('add_photos')}
            onclick={() => openFileUploadDialog({ albumId: album.id })}
            icon={mdiFileImagePlusOutline}
          />
        {/if}

        {#if album.assetCount > 0 && sharedLink.allowDownload}
          <IconButton
            shape="round"
            variant="ghost"
            color="secondary"
            aria-label={$t('slideshow')}
            onclick={handleStartSlideshow}
            icon={mdiPresentationPlay}
          />
          <IconButton
            shape="round"
            color="secondary"
            variant="ghost"
            aria-label={$t('download')}
            onclick={() => handleDownloadAlbum(album)}
            icon={mdiDownload}
          />
        {/if}
        {#if sharedLink.showMetadata && featureFlagsManager.value.map}
          <AlbumMap {album} />
        {/if}
        <ThemeButton />
      {/snippet}
    </ControlAppBar>
  {/if}
</header>
