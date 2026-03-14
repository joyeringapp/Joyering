<script>
  import { onMount, onDestroy } from 'svelte'
  import { supabase } from '$lib/supabase'

  let email = ''
  let isLoadingSession = true

  /** @type {import('@supabase/supabase-js').User | null} */
  let user = null

  /** @type {HTMLAudioElement[]} */
  let tapSounds = []

  /** @type {HTMLAudioElement | null} */
  let jarSound = null

  /** @type {HTMLAudioElement | null} */
  let releaseSound = null

  let butterflyCount = 0
  let screen = 'garden'
  let isReleasing = false
  // SETTINGS
// "settings" is the small panel opened by the gear icon.
// It is where the user can change app preferences such as
// theme (light/dark), language, or log out of the app.
let isSettingsOpen = false
  
  /** @type {string | null} */
  let currentAnimatingCategory = null

  /** @type {number | null} */
  let currentPulseNumber = null

  /** @type {ReturnType<typeof setTimeout> | null} */
  let resetTimer = null

  /** @type {ReturnType<typeof setTimeout> | null} */
  let releaseTimer = null

  /** @type {{ name: string; icon: string; key: string }[]} */
  const categories = [
    { name: 'Love', icon: '/icons/love.png', key: 'love' },
    { name: 'Friends', icon: '/icons/friends.png', key: 'friends' },
    { name: 'Success', icon: '/icons/success.png', key: 'success' },
    { name: 'Creativity', icon: '/icons/creativity.png', key: 'creativity' },
    { name: 'Work', icon: '/icons/work.png', key: 'work' },
    { name: 'Money', icon: '/icons/money.png', key: 'money' },
    { name: 'Goodies', icon: '/icons/goodies.png', key: 'goodies' },
    { name: 'Family', icon: '/icons/family.png', key: 'family' },
    { name: 'Exercise', icon: '/icons/exercise.png', key: 'exercise' },
    { name: 'Surprise', icon: '/icons/surprise.png', key: 'surprise' },
    { name: 'Nature', icon: '/icons/nature.png', key: 'nature' },
    { name: 'Other', icon: '/icons/other.png', key: 'other' }
  ]

  const tapSoundPaths = [
    '/sounds/fa.mp3',
    '/sounds/la.mp3',
    '/sounds/re.mp3',
    '/sounds/re2.mp3',
    '/sounds/si.mp3'
  ]

  async function login() {
    const { error } = await supabase.auth.signInWithOtp({
      email,
      options: {
        emailRedirectTo: window.location.origin
      }
    })

    if (error) {
      alert(error.message)
    } else {
      alert('Check your email for the login link!')
      email = ''
    }
  }

  async function logout() {
    const { error } = await supabase.auth.signOut()

    if (error) {
      alert(error.message)
    } else {
      isSettingsOpen = false
    }
  }

  /**
   * @param {string[]} paths
   */
  function preloadImages(paths) {
    for (const p of paths) {
      const img = new Image()
      img.src = p
    }
  }

  function setupTapSounds() {
    tapSounds = tapSoundPaths.map((path) => {
      const audio = new Audio(path)
      audio.preload = 'auto'
      audio.volume = 0.35
      return audio
    })

    jarSound = new Audio('/sounds/jar.mp3')
    jarSound.preload = 'auto'
    jarSound.volume = 0.35

    releaseSound = new Audio('/sounds/release.mp3')
    releaseSound.preload = 'auto'
    releaseSound.volume = 0.45
  }

  function playRandomTapSound() {
    if (!tapSounds.length) return

    const index = Math.floor(Math.random() * tapSounds.length)
    const sound = /** @type {HTMLAudioElement} */ (tapSounds[index].cloneNode())
    sound.volume = 0.35
    sound.play().catch(() => {})
  }

  async function ensureJoyStateRow() {
    if (!user) return

    const { data, error } = await supabase
      .from('joy_state')
      .select('user_id, butterfly_count')
      .eq('user_id', user.id)
      .maybeSingle()

    if (error) {
      console.error('Error checking joy state:', error)
      return
    }

    if (!data) {
      const { error: insertError } = await supabase
        .from('joy_state')
        .insert({
          user_id: user.id,
          butterfly_count: 0
        })

      if (insertError) {
        console.error('Error creating joy state:', insertError)
      }
    }
  }

  async function loadJoyState() {
    if (!user) return

    const { data, error } = await supabase
      .from('joy_state')
      .select('butterfly_count')
      .eq('user_id', user.id)
      .maybeSingle()

    if (error) {
      console.error('Error loading joy state:', error)
      return
    }

    butterflyCount = data?.butterfly_count ?? 0
  }

  async function saveJoyState() {
    if (!user) return

    const { error } = await supabase
      .from('joy_state')
      .upsert(
        {
          user_id: user.id,
          butterfly_count: butterflyCount,
          updated_at: new Date().toISOString()
        },
        { onConflict: 'user_id' }
      )

    if (error) {
      console.error('Error saving joy state:', error)
    }
  }

  async function restoreSessionAndState() {
    isLoadingSession = true

    const { data, error } = await supabase.auth.getSession()

    if (error) {
      console.error('Error restoring session:', error)
      user = null
      butterflyCount = 0
      isLoadingSession = false
      return
    }

    user = data.session?.user ?? null

    if (user) {
      await ensureJoyStateRow()
      await loadJoyState()
    } else {
      butterflyCount = 0
    }

    isLoadingSession = false
  }

  onMount(async () => {
    setupTapSounds()

    preloadImages([
      '/jars/jar-empty.gif',
      '/jars/jar-1.gif',
      '/jars/jar-2.gif',
      '/jars/jar-3.gif',
      '/jars/jar-4.gif',
      '/jars/jar-5.gif',
      '/jars/jar-6.gif',
      '/jars/jar-7.gif',
      '/jars/jar-8.gif',
      '/jars/jar-mid.gif',
      '/jars/jar-full.gif',
      '/animations/release-butterflies.gif'
    ])

    for (let i = 1; i <= 10; i++) {
      const img = new Image()
      img.src = `/butterflies/pulse${i}.gif`
    }

    await restoreSessionAndState()
  })

  /**
   * @param {{ name: string; icon: string; key: string }} category
   */
  function handleCategoryTap(category) {
    if (isReleasing || !user) return

    playRandomTapSound()

    butterflyCount += 1
    saveJoyState()

    currentPulseNumber = ((butterflyCount - 1) % 10) + 1
    currentAnimatingCategory = category.key

    if (resetTimer) {
      clearTimeout(resetTimer)
    }

    resetTimer = setTimeout(() => {
      currentAnimatingCategory = null
      currentPulseNumber = null
      resetTimer = null
    }, 1000)
  }

  function getJarImage() {
    if (butterflyCount === 0) return '/jars/jar-empty.gif'
    if (butterflyCount === 1) return '/jars/jar-1.gif'
    if (butterflyCount === 2) return '/jars/jar-2.gif'
    if (butterflyCount === 3) return '/jars/jar-3.gif'
    if (butterflyCount === 4) return '/jars/jar-4.gif'
    if (butterflyCount === 5) return '/jars/jar-5.gif'
    if (butterflyCount === 6) return '/jars/jar-6.gif'
    if (butterflyCount === 7) return '/jars/jar-7.gif'
    if (butterflyCount === 8) return '/jars/jar-8.gif'
    if (butterflyCount <= 15) return '/jars/jar-mid.gif'
    return '/jars/jar-full.gif'
  }

  function openCollection() {
    screen = 'collection'
    jarSound?.play().catch(() => {})
  }

  function letThemFly() {
    if (butterflyCount < 21 || isReleasing || !user) return

    releaseSound?.play().catch(() => {})

    if (resetTimer) {
      clearTimeout(resetTimer)
      resetTimer = null
    }

    currentAnimatingCategory = null
    currentPulseNumber = null
    isReleasing = true

    if (releaseTimer) {
      clearTimeout(releaseTimer)
    }

    releaseTimer = setTimeout(async () => {
      butterflyCount = 0
      await saveJoyState()
      isReleasing = false
      releaseTimer = null
    }, 3000)
  }

  onDestroy(() => {
    if (resetTimer) clearTimeout(resetTimer)
    if (releaseTimer) clearTimeout(releaseTimer)
  })
</script>

<svelte:head>
  <title>Joyering</title>
</svelte:head>

{#if isLoadingSession}
  <div class="auth-page">
    <div class="auth-card">
      <img
        class="auth-icon"
        src="/icon-192.png"
        alt="Joyering"
        draggable="false"
      />

      <h1>Joyering</h1>
      <p class="auth-subtitle">Loading...</p>
    </div>
  </div>

{:else if !user}
  <div class="auth-page">
    <div class="auth-card">
      <img
        class="auth-icon"
        src="/icon-192.png"
        alt="Joyering"
        draggable="false"
      />

      <h1>Joyering</h1>

      <p class="auth-subtitle">
        Catch your joyful moments, one butterfly at a time.
      </p>

      <div class="auth-form">
        <input
          type="email"
          placeholder="Your email"
          bind:value={email}
        />

        <button on:click={login}>
          Send me a login link
        </button>
      </div>
    </div>
  </div>

{:else}
  <div class="app-shell">
    {#if screen === 'garden'}
  <button
    class="settings-button"
    type="button"
    on:click={() => (isSettingsOpen = true)}
    aria-label="Open settings"
  >
    ⚙
  </button>
{/if}

    <div class="page">
      <div class="wrapper">
        {#if screen === 'garden'}
          <h1>The Joy Garden</h1>
          <p class="subtitle garden-subtitle">Catch a joyful moment!</p>

          <div class="grid">
            {#each categories as category}
              <button
                class="category"
                type="button"
                on:click={() => handleCategoryTap(category)}
              >
                <div class="visual-slot">
                  {#if currentAnimatingCategory === category.key && currentPulseNumber}
                    <img
                      class="pulse-butterfly"
                      src={`/butterflies/pulse${currentPulseNumber}.gif`}
                      alt=""
                      draggable="false"
                    />
                  {:else}
                    <img
                      class="category-icon"
                      src={category.icon}
                      alt={category.name}
                      draggable="false"
                    />
                  {/if}
                </div>

                <span class="category-label">{category.name}</span>
              </button>
            {/each}
          </div>

          <button
            class="collection-button"
            type="button"
            on:click={openCollection}
          >
            See Your Joy Collection
          </button>
        {:else}
          <div class="collection-screen">
            {#if isReleasing}
              <div class="release-screen">
                <img
                  class="release-animation"
                  src="/animations/release-butterflies.gif"
                  alt=""
                  draggable="false"
                />
              </div>
            {:else}
              <h1>Collected Joy</h1>
              <p class="subtitle collection-subtitle">Release when you reach 21!</p>

              <div class="jar-block">
                <div class="jar-area">
                  <img
                    class="jar-image"
                    src={getJarImage()}
                    alt="Collected joy jar"
                    draggable="false"
                  />
                </div>

                <p class="joy-count">
                  {#if butterflyCount === 1}
                    You have collected 1 joyful moment
                  {:else if butterflyCount > 1}
                    You have collected {butterflyCount} joyful moments
                  {/if}
                </p>
              </div>

              <div class="collection-buttons">
                {#if butterflyCount >= 21}
                  <button
                    class="fly-button"
                    type="button"
                    on:click={letThemFly}
                  >
                    Let Them Fly!
                  </button>
                {/if}

                <button
                  class="collection-button"
                  type="button"
                  on:click={() => (screen = 'garden')}
                >
                  Collect More Joy
                </button>
              </div>
            {/if}
          </div>
        {/if}
      </div>
    </div>

    {#if isSettingsOpen}
  <div
    class="settings-overlay"
    on:click={() => (isSettingsOpen = false)}
  >
    <div
      class="settings-modal"
      on:click|stopPropagation
    >
          <div class="settings-header">
            <h2>Settings</h2>

            <button
              class="settings-close"
              type="button"
              on:click={() => (isSettingsOpen = false)}
              aria-label="Close settings"
            >
              ×
            </button>
          </div>

          <div class="settings-section">
            <p class="settings-label">Theme</p>
            <p class="settings-placeholder">Coming soon</p>
          </div>

          <div class="settings-section">
            <p class="settings-label">Language</p>
            <p class="settings-placeholder">Coming soon</p>
          </div>

          <div class="settings-divider"></div>

          <button
            class="settings-logout"
            type="button"
            on:click={logout}
          >
            Log out
          </button>
        </div>
      </div>
    {/if}
  </div>
{/if}

<style>
  :global(html) {
    margin: 0;
    padding: 0;
    background: #000;
    min-height: 100%;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    min-height: 100vh;
    background: #000;
    color: #fff;
    font-family: Arial, Helvetica, sans-serif;
    -webkit-tap-highlight-color: transparent;
  }

  :global(*) {
    box-sizing: border-box;
    -webkit-tap-highlight-color: transparent;
  }

  .auth-page {
    min-height: 100vh;
    width: 100%;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 32px;
  }

  .auth-card {
    width: 100%;
    max-width: 480px;
    padding: 44px 32px;
    border-radius: 24px;
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid rgba(255, 255, 255, 0.08);
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    gap: 24px;
  }

  .auth-icon {
    width: 96px;
    height: 96px;
    object-fit: contain;
    margin-bottom: 4px;
    user-select: none;
    pointer-events: none;
  }

  .auth-subtitle {
    margin: 0;
    font-size: 1.08rem;
    line-height: 1.5;
    opacity: 0.92;
    max-width: 360px;
  }

  .auth-form {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  .auth-form input {
    width: 100%;
    height: 52px;
    border: none;
    border-radius: 12px;
    padding: 0 16px;
    font-size: 1rem;
    outline: none;
  }

  .app-shell {
    min-height: 100vh;
    width: 100%;
    background: #000;
    overflow: hidden;
  }

  .auth-form button,
  .collection-button,
  .fly-button {
    border: none;
    border-radius: 12px;
    color: white;
    font-size: 1rem;
    font-weight: 700;
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
    outline: none;
  }

  .auth-form button {
    width: 100%;
    min-height: 52px;
    padding: 0 16px;
    background: #62c7cf;
    box-shadow: 0 8px 20px rgba(98, 199, 207, 0.25);
  }

  .page {
    min-height: 100vh;
    width: 100%;
    background: #000;
    display: flex;
    justify-content: center;
    padding:
      env(safe-area-inset-top)
      20px
      calc(36px + env(safe-area-inset-bottom));
  }

  .wrapper {
    width: 100%;
    max-width: 760px;
    display: flex;
    flex-direction: column;
    align-items: center;
    margin: 0 auto;
  }

  h1 {
    margin: 0;
    font-size: 2.6rem;
    line-height: 1.06;
    font-weight: 700;
    text-align: center;
  }

  .subtitle {
    font-size: 1.1rem;
    font-weight: 500;
    line-height: 1.15;
    text-align: center;
    opacity: 0.96;
    max-width: 420px;
  }

  .garden-subtitle {
    margin: 22px 0 50px;
  }

  .collection-subtitle {
    margin: 22px 0 50px;
  }

  .grid {
    width: 100%;
    max-width: 560px;
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    justify-items: center;
    gap: 20px 10px;
    margin: 0 auto 42px;
  }

  .category {
    background: transparent;
    border: none;
    padding: 0;
    margin: 0;
    color: white;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    font: inherit;
    text-align: center;
    appearance: none;
    -webkit-appearance: none;
    outline: none;
    user-select: none;
    -webkit-user-select: none;
    touch-action: manipulation;
  }

  .category:focus,
  .category:focus-visible,
  .category:active,
  .auth-form button:focus,
  .auth-form button:focus-visible,
  .collection-button:focus,
  .collection-button:focus-visible,
  .fly-button:focus,
  .fly-button:focus-visible {
    outline: none;
    box-shadow: none;
  }

  .visual-slot {
    width: 78px;
    height: 78px;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    background: transparent;
  }

  .category-icon {
    width: 78px;
    height: 78px;
    border-radius: 50%;
    object-fit: cover;
    display: block;
    pointer-events: none;
    user-select: none;
    -webkit-user-select: none;
  }

  .pulse-butterfly {
    width: 88px;
    height: 88px;
    object-fit: contain;
    display: block;
    pointer-events: none;
    user-select: none;
    -webkit-user-select: none;
  }

  .category-label {
    font-size: 0.96rem;
    line-height: 1.1;
    color: white;
    pointer-events: none;
  }

  .collection-button,
  .fly-button {
    width: 100%;
    max-width: 320px;
    min-height: 46px;
    padding: 0 20px;
  }

  .collection-button {
    background: #62c7cf;
    box-shadow: 0 8px 20px rgba(98, 199, 207, 0.25);
  }

  .fly-button {
    background: #d86fa5;
    box-shadow: 0 8px 20px rgba(216, 111, 165, 0.3);
  }

  .auth-form button:hover,
  .collection-button:hover,
  .fly-button:hover {
    filter: brightness(1.03);
  }

  .collection-button:active,
  .fly-button:active,
  .auth-form button:active {
    transform: translateY(1px);
  }

  .collection-screen {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .jar-block {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 28px;
  }

  .jar-area {
    width: 100%;
    max-width: 360px;
    height: 300px;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }

  .jar-image {
    width: 270px;
    height: 270px;
    object-fit: contain;
    display: block;
    user-select: none;
    -webkit-user-select: none;
  }

  .joy-count {
    margin: 6px 0 0;
    font-size: 1.05rem;
    line-height: 1.2;
    text-align: center;
    opacity: 0.96;
    min-height: 22px;
  }

  .collection-buttons {
    width: 100%;
    max-width: 320px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    gap: 14px;
  }

  .release-screen {
    width: 100%;
    min-height: 78vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .release-animation {
    width: min(92vw, 720px);
    height: auto;
    display: block;
    user-select: none;
    -webkit-user-select: none;
    transform: translateY(-32px);
  }

  .settings-button {
    position: fixed;
    top: calc(env(safe-area-inset-top) + 14px);
    left: 14px;
    width: 42px;
    height: 42px;
    border: none;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.08);
    color: #fff;
    font-size: 1.2rem;
    line-height: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
    z-index: 1000;
    backdrop-filter: blur(8px);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.28);
  }

  .settings-button:focus,
  .settings-button:focus-visible,
  .settings-close:focus,
  .settings-close:focus-visible,
  .settings-logout:focus,
  .settings-logout:focus-visible {
    outline: none;
    box-shadow: none;
  }

  .settings-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.55);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    z-index: 1100;
  }

  .settings-modal {
    width: 100%;
    max-width: 360px;
    border-radius: 22px;
    background: rgba(18, 18, 18, 0.96);
    border: 1px solid rgba(255, 255, 255, 0.08);
    padding: 22px 20px 20px;
    color: #fff;
    box-shadow: 0 20px 50px rgba(0, 0, 0, 0.45);
  }

  .settings-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
  }

  .settings-header h2 {
    margin: 0;
    font-size: 1.35rem;
    line-height: 1.1;
    font-weight: 700;
  }

  .settings-close {
    border: none;
    background: transparent;
    color: #fff;
    font-size: 1.8rem;
    line-height: 1;
    cursor: pointer;
    padding: 0;
  }

  .settings-section {
    margin-bottom: 18px;
  }

  .settings-label {
    margin: 0 0 6px;
    font-size: 0.98rem;
    font-weight: 700;
  }

  .settings-placeholder {
    margin: 0;
    font-size: 0.96rem;
    opacity: 0.8;
  }

  .settings-divider {
    width: 100%;
    height: 1px;
    background: rgba(255, 255, 255, 0.1);
    margin: 10px 0 16px;
  }

  .settings-logout {
    width: 100%;
    min-height: 46px;
    border: none;
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.1);
    color: #fff;
    font-size: 1rem;
    font-weight: 700;
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
  }

  @media (max-width: 640px) {
    h1 {
      font-size: 2.05rem;
    }

    .subtitle {
      font-size: 1rem;
    }

    .auth-page {
      padding: 20px;
    }

    .auth-card {
      max-width: 420px;
      padding: 36px 22px;
      gap: 20px;
    }

    .auth-icon {
      width: 84px;
      height: 84px;
    }

    .auth-subtitle {
      font-size: 1rem;
      max-width: 300px;
    }

    .page {
      min-height: 100vh;
      width: 100%;
      background: #000;
      display: flex;
      justify-content: center;
      padding:
        env(safe-area-inset-top)
        14px
        calc(34px + env(safe-area-inset-bottom));
    }

    .wrapper {
      max-width: 360px;
      margin: 90px auto 0;
    }

    .garden-subtitle {
      margin: 14px 0 58px;
    }

    .collection-subtitle {
      margin: 16px 0 38px;
    }

    .grid {
      max-width: 360px;
      gap: 20px 8px;
      margin-bottom: 64px;
    }

    .visual-slot {
      width: 60px;
      height: 60px;
    }

    .category-icon {
      width: 60px;
      height: 60px;
    }

    .pulse-butterfly {
      width: 70px;
      height: 70px;
    }

    .category-label {
      font-size: 0.86rem;
    }

    .collection-buttons {
      max-width: 320px;
      gap: 14px;
    }

    .jar-block {
      margin-bottom: 26px;
    }

    .jar-area {
      height: 320px;
    }

    .jar-image {
      width: 280px;
      height: 280px;
    }

    .joy-count {
      font-size: 1rem;
      margin-top: 4px;
    }

    .release-screen {
      min-height: 72vh;
    }

    .release-animation {
      width: 96vw;
    }

    .settings-button {
      top: calc(env(safe-area-inset-top) + 10px);
      left: 10px;
      width: 40px;
      height: 40px;
    }

    .settings-overlay {
      padding: 18px;
    }

    .settings-modal {
      max-width: 340px;
      padding: 20px 18px 18px;
    }
  }
</style>