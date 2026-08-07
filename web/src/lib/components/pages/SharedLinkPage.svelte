<script lang="ts">
  import AlbumViewer from '$lib/components/album-page/AlbumViewer.svelte';
  import IndividualSharedViewer from '$lib/components/share-page/IndividualSharedViewer.svelte';
  import ControlAppBar from '$lib/components/shared-components/ControlAppBar.svelte';
  import ThemeButton from '$lib/components/shared-components/ThemeButton.svelte';
  import { assetViewerManager } from '$lib/managers/asset-viewer-manager.svelte';
  import { authManager } from '$lib/managers/auth-manager.svelte';
  import { setSharedLink } from '$lib/utils';
  import { handleError } from '$lib/utils/handle-error';
  import { navigate } from '$lib/utils/navigation';
  import { sharedLinkLogin, SharedLinkType, type AssetResponseDto, type SharedLinkResponseDto } from '@immich/sdk';
  import { Button, PasswordInput } from '@immich/ui';
  import { onDestroy, tick } from 'svelte';
  import { t } from 'svelte-i18n';
  import { wedding } from '$lib/config/wedding';

  type Props = {
    data: {
      meta: {
        title: string;
        description?: string;
        imageUrl?: string;
      };

      sharedLink?: SharedLinkResponseDto;
      key?: string;
      slug?: string;
      asset?: AssetResponseDto;
      passwordRequired?: boolean;
    };
  };

  const { data }: Props = $props();

  let { sharedLink, passwordRequired, key, slug, meta } = $state(data);
  let { title, description } = $state(meta);
  let isOwned = $derived(authManager.authenticated && authManager.user.id === sharedLink?.userId);
  let password = $state('');

  if (passwordRequired) {
    assetViewerManager.showAssetViewer(false);
  }

  const handlePasswordSubmit = async () => {
    try {
      sharedLink = await sharedLinkLogin({ key, slug, sharedLinkLoginDto: { password } });
      setSharedLink(sharedLink);
      passwordRequired = false;
      title = (sharedLink.album ? sharedLink.album.albumName : $t('public_share')) + ' - Immich';
      description =
        sharedLink.description ||
        $t('shared_photos_and_videos_count', { values: { assetCount: sharedLink.assets.length } });
      await tick();
      await navigate(
        { targetRoute: 'current', assetId: null, assetGridRouteSearchParams: assetViewerManager.gridScrollTarget },
        { forceNavigate: true, replaceState: true },
      );
    } catch (error) {
      handleError(error, $t('errors.unable_to_get_shared_link'));
    }
  };

  const onsubmit = async (event: Event) => {
    event.preventDefault();
    await handlePasswordSubmit();
  };

  onDestroy(() => {
    setSharedLink(undefined);
  });
</script>

<svelte:head>
  <title>{title}</title>
  <meta name="description" content={description} />
</svelte:head>
{#if passwordRequired}
  <main class="wedding-gate relative min-h-dvh overflow-hidden px-6 pt-(--navbar-height) sm:px-12 md:px-24 lg:px-40">
    <div class="wedding-gate__image" aria-hidden="true"><img src={wedding.heroImage} alt="" /></div>
    <div class="wedding-gate__veil"></div>
    <div class="wedding-gate__content">
      <p class="wedding-gate__eyebrow">{wedding.invitation}</p>
      <img class="wedding-gate__monogram" src={wedding.monogramImage} alt="" />
      <h1>Nosso<br />álbum</h1>
      <p>{wedding.couple}<br />{wedding.date}</p>
      <form class="wedding-gate__form" novalidate {onsubmit}>
        <PasswordInput autocomplete="off" bind:value={password} placeholder="Senha do convite" />
        <Button type="submit">Entrar</Button>
      </form>
      <p class="wedding-gate__footnote">{wedding.venue}<br />{wedding.address}</p>
    </div>
  </main>
  <header>
    <ControlAppBar>
      {#snippet leading()}
        <a data-sveltekit-preload-data="hover" class="wedding-wordmark ms-4" href="/" aria-label={wedding.couple}>
          <img src={wedding.monogramImage} alt="" />
          <span>{wedding.couple}</span>
        </a>
      {/snippet}

      {#snippet trailing()}
        <ThemeButton />
      {/snippet}
    </ControlAppBar>
  </header>
{/if}

{#if !passwordRequired && sharedLink?.type === SharedLinkType.Album}
  <AlbumViewer {sharedLink} />
{/if}
{#if !passwordRequired && sharedLink?.type === SharedLinkType.Individual}
  <div class="immich-scrollbar">
    <IndividualSharedViewer {sharedLink} {isOwned} />
  </div>
{/if}
