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
  let isSettingsOpen = false
  let theme = 'dark'

  let soundEnabled = true

  /** @type {'en' | 'it' | 'pt'} */
  let language = 'en'

  let showInstallCard = true

  let logoutMessage = ''

  /** @type {any} */
  let installPrompt = null
  let canInstall = false

  /** @type {string | null} */
  let currentAnimatingCategory = null

  /** @type {number | null} */
  let currentPulseNumber = null

  /** @type {ReturnType<typeof setTimeout> | null} */
  let resetTimer = null

  /** @type {ReturnType<typeof setTimeout> | null} */
  let releaseTimer = null

  /** @type {{ subscription?: { unsubscribe: () => void } } | null} */
  let authListener = null

  /** @type {{ icon: string; key: string }[]} */

  const categories = [
  { key: 'love', icon: '/icons/love.png' },
  { key: 'friends', icon: '/icons/friends.png' },
  { key: 'work', icon: '/icons/work.png' },
  { key: 'nature', icon: '/icons/nature.png' },

  { key: 'family', icon: '/icons/family.png' },
  { key: 'money', icon: '/icons/money.png' },
  { key: 'creativity', icon: '/icons/create.png' },
  { key: 'surprise', icon: '/icons/surprise.png' },

  { key: 'culture', icon: '/icons/culture.png' },
  { key: 'activity', icon: '/icons/movement.png' },
  { key: 'treats', icon: '/icons/treats.png' },
  { key: 'other', icon: '/icons/other.png' }
]

  const tapSoundPaths = [
    '/sounds/fa.mp3',
    '/sounds/la.mp3',
    '/sounds/re.mp3',
    '/sounds/re2.mp3',
    '/sounds/si.mp3'
  ]

  /** @type {Record<string, any>} */
  const translations = {
    en: {
      loading: 'Loading...',
      appSubtitle: 'Catch your joyful moments, one butterfly at a time.',
      emailPlaceholder: 'Your email',
      sendLoginLink: 'Send me a login link',
      installHint: 'Your email is only used to access your Joyering account.',

      gardenTitle: 'The Joy Garden',
      gardenSubtitle: 'Catch a joyful moment!',
      collectionButton: 'See Your Joy Collection',

      collectionTitle: 'Collected Joy',
      collectionSubtitle: 'Release when you reach 21!',
      flyButton: 'Let Them Fly!',
      collectMoreJoy: 'Collect More Joy',

      installCardTitle: 'Keep Joyering close',
      installCardText: 'Install Joyering on your phone so it’s always there when a joyful moment appears.',
      installButton: 'Install Joyering',
      notNow: 'Not now',

        categories: {
  love: 'Love',
  friends: 'Friends',
  work: 'Work',
  nature: 'Nature',

  family: 'Family',
  money: 'Money',
  creativity: 'Creativity',
  surprise: 'Surprise',

  culture: 'Culture',
  activity: 'Activity',
  treats: 'Food & Drink',
  other: 'Other'
},

      settingsTitle: 'Settings',
      themeLabel: 'Theme',
      dark: 'Dark',
      light: 'Light',
      languageLabel: 'Language',
      soundLabel: 'Sound',
soundOn: 'On',
soundOff: 'Off',
      logout: 'Log out',

      joyfulMomentSingular: 'You have collected 1 joyful moment',
      joyfulMomentPlural: 'You have collected {count} joyful moments'
    },

    it: {
      loading: 'Caricamento...',
      appSubtitle: 'Raccogli i tuoi momenti gioiosi, una farfalla alla volta.',
      emailPlaceholder: 'La tua email',
      sendLoginLink: 'Mandami un link di accesso',
      installHint: 'La tua email viene usata solo per accedere a Joyering.',

      gardenTitle: 'Il Giardino della Gioia',
      gardenSubtitle: 'Cattura un momento gioioso!',
      collectionButton: 'Guarda la tua collezione di gioia',

      collectionTitle: 'Gioia raccolta',
      collectionSubtitle: 'Lasciale volare quando arrivi a 21!',
      flyButton: 'Lasciale volare!',
      collectMoreJoy: 'Raccogli altra gioia',

      installCardTitle: 'Tieni Joyering vicino',
      installCardText: 'Installa Joyering sul tuo telefono così sarà sempre lì quando arriva un momento gioioso.',
      installButton: 'Installa Joyering',
      notNow: 'Non ora',

      categories: {
  love: 'Amore',
  friends: 'Amici',
  work: 'Lavoro',
  nature: 'Natura',

  family: 'Famiglia',
  money: 'Denaro',
  creativity: 'Creatività',
  surprise: 'Sorpresa',

  culture: 'Cultura',
  activity: 'Attività',
  treats: 'Cibo e bevande',
  other: 'Altro'
},

      settingsTitle: 'Impostazioni',
      themeLabel: 'Tema',
      dark: 'Scuro',
      light: 'Chiaro',
      languageLabel: 'Lingua',
      soundLabel: 'Suono',
soundOn: 'On',
soundOff: 'Off',
      logout: 'Esci',

      joyfulMomentSingular: 'Hai raccolto 1 momento gioioso',
      joyfulMomentPlural: 'Hai raccolto {count} momenti gioiosi'
    },

    pt: {
      loading: 'Carregando...',
      appSubtitle: 'Recolha seus momentos felizes, uma borboleta de cada vez.',
      emailPlaceholder: 'Seu e-mail',
      sendLoginLink: 'Envie-me um link de acesso',
      installHint: 'Seu e-mail é usado apenas para acessar o Joyering.',

      gardenTitle: 'O Jardim da Alegria',
      gardenSubtitle: 'Capture um momento feliz!',
      collectionButton: 'Ver sua coleção de alegria',

      collectionTitle: 'Alegria recolhida',
      collectionSubtitle: 'Solte quando chegar a 21!',
      flyButton: 'Deixe voar!',
      collectMoreJoy: 'Recolher mais alegria',

      installCardTitle: 'Mantenha Joyering por perto',
      installCardText: 'Instale o Joyering no seu telefone para que ele esteja sempre ali quando um momento feliz aparecer.',
      installButton: 'Instalar Joyering',
      notNow: 'Agora não',

      categories: {
  love: 'Amor',
  friends: 'Amigos',
  work: 'Trabalho',
  nature: 'Natureza',

  family: 'Família',
  money: 'Dinheiro',
  creativity: 'Criatividade',
  surprise: 'Surpresa',

  culture: 'Cultura',
  activity: 'Atividade',
  treats: 'Comida e bebida',
  other: 'Outro'
},

      settingsTitle: 'Configurações',
      themeLabel: 'Tema',
      dark: 'Escuro',
      light: 'Claro',
      languageLabel: 'Idioma',
      soundLabel: 'Som',
soundOn: 'Ligado',
soundOff: 'Desligado',
      logout: 'Sair',

      joyfulMomentSingular: 'Você recolheu 1 momento feliz',
      joyfulMomentPlural: 'Você recolheu {count} momentos felizes'
    }
  }

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

isSettingsOpen = false
logoutMessage = 'You have been logged out'

setTimeout(() => {
  logoutMessage = ''
}, 3000)

const { error } = await supabase.auth.signOut()

if (error) {
  alert(error.message)
}
}

  function loadSavedTheme() {
    const savedTheme = localStorage.getItem('joyering-theme')

    if (savedTheme === 'light' || savedTheme === 'dark') {
      theme = savedTheme
    }
  }

  /**
   * @param {'dark' | 'light'} newTheme
   */
  function setTheme(newTheme) {
    theme = newTheme
    localStorage.setItem('joyering-theme', newTheme)
  }

  /** @param {boolean} enabled */
function setSoundEnabled(enabled) {
  soundEnabled = enabled
  localStorage.setItem('joyering-sound', enabled ? 'on' : 'off')
}

  function loadSavedSoundSetting() {
  const savedSound = localStorage.getItem('joyering-sound')

  if (savedSound === 'off') {
    soundEnabled = false
  } else {
    soundEnabled = true
  }
}

  function loadSavedLanguage() {
    const savedLanguage = localStorage.getItem('joyering-language')

    if (savedLanguage === 'en' || savedLanguage === 'it' || savedLanguage === 'pt') {
      language = savedLanguage
      return
    }

    const browserLang = navigator.language || ''

    if (browserLang.startsWith('it')) {
      language = 'it'
    } else if (browserLang.startsWith('pt')) {
      language = 'pt'
    } else {
      language = 'en'
    }
  }

  /**
   * @param {'en' | 'it' | 'pt'} newLanguage
   */
  function setLanguage(newLanguage) {
    language = newLanguage
    localStorage.setItem('joyering-language', newLanguage)
  }

  /** @param {string} key */
  function t(key) {
    return translations[language]?.[key] || translations.en[key] || key
  }

  function joyfulMomentText() {
    if (butterflyCount === 1) {
      return t('joyfulMomentSingular')
    }

    if (butterflyCount > 1) {
      return t('joyfulMomentPlural').replace('{count}', String(butterflyCount))
    }

    return ''
  }

  function isStandaloneMode() {
    if (typeof window === 'undefined') return false

    const displayModeStandalone = window.matchMedia('(display-mode: standalone)').matches
    const iosStandalone =
      'standalone' in window.navigator && window.navigator.standalone === true

    return displayModeStandalone || iosStandalone
  }

  function showManualInstallHelp() {
    const ua = window.navigator.userAgent.toLowerCase()
    const isIOS = /iphone|ipad|ipod/.test(ua)
    const isAndroid = /android/.test(ua)
    const isChrome =
      /chrome/.test(ua) && !/edg|opr|opera|samsungbrowser/.test(ua)

    if (isIOS) {
      alert('To save Joyering on your phone, tap Share, then tap "Add to Home Screen".')
      return
    }

    if (isAndroid && isChrome) {
      alert('To save Joyering on your phone, open the browser menu and choose "Install app" or "Add to Home screen".')
      return
    }

    alert('To save Joyering on your phone, open your browser menu and look for an install option.')
  }

  async function installJoyering() {
    if (isStandaloneMode()) {
      showInstallCard = false
      return
    }

    if (installPrompt) {
      try {
        await installPrompt.prompt()
        const choiceResult = await installPrompt.userChoice

        installPrompt = null
        canInstall = false

        if (choiceResult?.outcome === 'accepted') {
          showInstallCard = false
          localStorage.setItem('joyering-install-dismissed', 'true')
        }
      } catch (error) {
        console.error('Install prompt error:', error)
      }

      return
    }

    showManualInstallHelp()
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
    if (!soundEnabled) return
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

  /** @param {import('@supabase/supabase-js').Session | null} session */
async function syncUserAndState(session) {
    const nextUser = session?.user ?? null
    user = nextUser

    if (nextUser) {
      await ensureJoyStateRow()
      await loadJoyState()
    } else {
      butterflyCount = 0
      screen = 'garden'
      isSettingsOpen = false
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

    await syncUserAndState(data.session ?? null)
    isLoadingSession = false
  }

  onMount(() => {
    loadSavedTheme()
loadSavedLanguage()
loadSavedSoundSetting()
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

    const dismissed = localStorage.getItem('joyering-install-dismissed')

    if (isStandaloneMode() || dismissed === 'true') {
      showInstallCard = false
    }

    /** @param {any} e */
    const handleBeforeInstallPrompt = (e) => {
      e.preventDefault()
      installPrompt = e
      canInstall = true
    }

    const handleAppInstalled = () => {
      installPrompt = null
      canInstall = false
      showInstallCard = false
      localStorage.setItem('joyering-install-dismissed', 'true')
    }

    window.addEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
    window.addEventListener('appinstalled', handleAppInstalled)

    restoreSessionAndState()

    const { data } = supabase.auth.onAuthStateChange(async (_event, session) => {
      await syncUserAndState(session)
      isLoadingSession = false
    })

    authListener = data

    return () => {
      window.removeEventListener('beforeinstallprompt', handleBeforeInstallPrompt)
      window.removeEventListener('appinstalled', handleAppInstalled)

      if (authListener?.subscription) {
        authListener.subscription.unsubscribe()
      }
    }
  })

  /**
 * @param {{ icon: string; key: string }} category
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

  if (soundEnabled) {
    jarSound?.play().catch(() => {})
  }
}

  function letThemFly() {
  if (butterflyCount < 21 || isReleasing || !user) return

  if (soundEnabled) {
  releaseSound?.play().catch(() => {})
}

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

    if (authListener?.subscription) {
      authListener.subscription.unsubscribe()
    }
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
      <p class="auth-subtitle">{t('loading')}</p>
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
        {t('appSubtitle')}
      </p>

      <div class="auth-form">
        <input
          type="email"
          placeholder={t('emailPlaceholder')}
          bind:value={email}
        />

        <button on:click={login}>
          {t('sendLoginLink')}
        </button>

        <p class="install-hint">
          {t('installHint')}
        </p>
      </div>
    </div>
  </div>

{:else}
  <div class={`app-shell theme-${theme}`}>
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
          {#if showInstallCard}
            <div class="install-card">
              <p class="install-title">{t('installCardTitle')}</p>

              <p class="install-text">
                {t('installCardText')}
              </p>

              <div class="install-buttons">
                <button
                  class="install-primary"
                  type="button"
                  on:click={installJoyering}
                >
                  {t('installButton')}
                </button>

                <button
                  class="install-secondary"
                  type="button"
                  on:click={() => {
                    showInstallCard = false
                    localStorage.setItem('joyering-install-dismissed', 'true')
                  }}
                >
                  {t('notNow')}
                </button>
              </div>
            </div>
          {/if}

          <h1>{t('gardenTitle')}</h1>
          <p class="subtitle garden-subtitle">{t('gardenSubtitle')}</p>

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
                      alt={translations[language].categories[category.key]}
                      draggable="false"
                    />
                  {/if}
                </div>

                <span class="category-label">{translations[language].categories[category.key]}</span>
              </button>
            {/each}
          </div>

          <button
            class="collection-button"
            type="button"
            on:click={openCollection}
          >
            {t('collectionButton')}
          </button>
        {:else}
          <div class="collection-screen">

            {#if isReleasing}
  <div class="release-screen">
    {#key isReleasing}
      <img
        class="release-animation"
        src="/animations/release-butterflies.gif"
        alt=""
        draggable="false"
      />
    {/key}
  </div>
{:else}
              <h1>{t('collectionTitle')}</h1>
              <p class="subtitle collection-subtitle">{t('collectionSubtitle')}</p>

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
                  {joyfulMomentText()}
                </p>
              </div>

              <div class="collection-buttons">
                {#if butterflyCount >= 21}
                  <button
                    class="fly-button"
                    type="button"
                    on:click={letThemFly}
                  >
                    {t('flyButton')}
                  </button>
                {/if}

                <button
                  class="collection-button"
                  type="button"
                  on:click={() => (screen = 'garden')}
                >
                  {t('collectMoreJoy')}
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
            <h2>{t('settingsTitle')}</h2>

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
            <p class="settings-label">{t('themeLabel')}</p>

            <div class="settings-choice-row">
              <button
                class:active-choice={theme === 'dark'}
                class="settings-choice"
                type="button"
                on:click={() => setTheme('dark')}
              >
                {t('dark')}
              </button>

              <button
                class:active-choice={theme === 'light'}
                class="settings-choice"
                type="button"
                on:click={() => setTheme('light')}
              >
                {t('light')}
              </button>
            </div>
          </div>

          <div class="settings-section">
            <p class="settings-label">{t('languageLabel')}</p>
          
            <div class="settings-choice-row">
              <button
                class:active-choice={language === 'en'}
                class="settings-choice"
                type="button"
                on:click={() => setLanguage('en')}
              >
                English
              </button>
          
              <button
                class:active-choice={language === 'it'}
                class="settings-choice"
                type="button"
                on:click={() => setLanguage('it')}
              >
                Italiano
              </button>
          
              <button
                class:active-choice={language === 'pt'}
                class="settings-choice"
                type="button"
                on:click={() => setLanguage('pt')}
              >
                Português
              </button>
            </div>
          </div>
          
 
          
          <div class="settings-section">
            <p class="settings-label">{t('soundLabel')}</p>
          
            <div class="settings-choice-row">
              <button
                class:active-choice={soundEnabled}
                class="settings-choice"
                type="button"
                on:click={() => setSoundEnabled(true)}
              >
                {t('soundOn')}
              </button>
          
              <button
                class:active-choice={!soundEnabled}
                class="settings-choice"
                type="button"
                on:click={() => setSoundEnabled(false)}
              >
                {t('soundOff')}
              </button>
            </div>
          </div>

          <div class="settings-divider"></div>

          <div class="settings-section legal-section">
            <p class="legal-brand">Joyering™</p>
            <p class="legal-copy">© 2026 Ulrika Torquato</p>
            <p class="legal-copy">All rights reserved.</p>
          </div>
          
          <div class="settings-divider"></div>
          
<button
  class="settings-logout"
  type="button"
  on:click={logout}
>
  {t('logout')}
</button>
        </div>
      </div>
    {/if}

  </div>
{/if}

{#if logoutMessage}
<div class="toast-message">
  {logoutMessage}
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

  .install-hint {
    margin-top: 10px;
    font-size: 0.9rem;
    line-height: 1.4;
    opacity: 0.75;
    max-width: 320px;
    text-align: center;
  }

  .app-shell {
    min-height: 100vh;
    width: 100%;
    background: #000;
    overflow: hidden;
  }

  .app-shell.theme-light {
    background: #f7f6f1;
    color: #151515;
  }

  .app-shell.theme-light .page,
  .app-shell.theme-light .collection-screen,
  .app-shell.theme-light .wrapper,
  .app-shell.theme-light .jar-block {
    background: transparent;
    color: #151515;
  }

  .app-shell.theme-light h1,
  .app-shell.theme-light .subtitle,
  .app-shell.theme-light .category-label,
  .app-shell.theme-light .joy-count {
    color: #151515;
  }

  .app-shell.theme-light .settings-button {
    background: rgba(0, 0, 0, 0.08);
    color: #151515;
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
  }

  .app-shell.theme-light .settings-modal {
    background: rgba(255, 255, 255, 0.96);
    border: 1px solid rgba(0, 0, 0, 0.08);
    color: #151515;
  }

  .app-shell.theme-light .settings-header h2,
  .app-shell.theme-light .settings-label,
  .app-shell.theme-light .settings-close {
    color: #151515;
  }

  .app-shell.theme-light .settings-divider {
    background: rgba(0, 0, 0, 0.08);
  }

  .app-shell.theme-light .settings-choice {
    background: rgba(0, 0, 0, 0.08);
    color: #151515;
  }

  .app-shell.theme-light .settings-logout {
    background: rgba(0, 0, 0, 0.08);
    color: #151515;
  }

  .app-shell.theme-light .install-card {
    background: rgba(0, 0, 0, 0.05);
    border: 1px solid rgba(0, 0, 0, 0.08);
  }

  .app-shell.theme-light .install-secondary {
    background: rgba(0, 0, 0, 0.08);
    color: #151515;
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
    background: transparent;
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
    font-size: 0.9rem;
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

.release-animation {
  animation: releaseFade 3s ease-in-out forwards;
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
  .settings-logout:focus-visible,
  .settings-choice:focus,
  .settings-choice:focus-visible {
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

  .legal-section {
  text-align: center;
  margin-bottom: 16px;
}

.legal-brand {
  margin: 0 0 6px;
  font-size: 0.98rem;
  font-weight: 700;
  opacity: 0.95;
}

.legal-copy {
  margin: 0;
  font-size: 0.82rem;
  line-height: 1.4;
  opacity: 0.72;
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

  .settings-choice-row {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  .settings-choice {
    min-height: 40px;
    padding: 0 16px;
    border: none;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.08);
    color: #fff;
    font-size: 0.95rem;
    font-weight: 700;
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
  }

  .active-choice {
    background: #62c7cf;
    color: #000;
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

  .install-card {
    width: 100%;
    max-width: 420px;
    margin-bottom: 26px;
    padding: 18px 18px 16px;
    border-radius: 18px;
    background: rgba(255,255,255,0.06);
    border: 1px solid rgba(255,255,255,0.08);
    text-align: center;
  }

  .install-title {
    margin: 0 0 6px;
    font-size: 1rem;
    font-weight: 700;
  }

  .install-text {
    margin: 0;
    font-size: 0.92rem;
    opacity: 0.9;
  }

  .install-buttons {
    display: flex;
    justify-content: center;
    gap: 10px;
    margin-top: 12px;
  }

  .install-primary {
    padding: 0 14px;
    height: 36px;
    border-radius: 999px;
    border: none;
    background: #62c7cf;
    font-weight: 700;
    cursor: pointer;
  }

  .install-secondary {
    padding: 0 14px;
    height: 36px;
    border-radius: 999px;
    border: none;
    background: rgba(255,255,255,0.1);
    color: white;
    cursor: pointer;
  }

  .toast-message {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0,0,0,0.85);
  color: white;
  padding: 12px 18px;
  border-radius: 10px;
  font-size: 0.95rem;
  z-index: 2000;
  animation: fadeInOut 3s ease;
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
      background: transparent;
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