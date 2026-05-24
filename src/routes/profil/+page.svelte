<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  // --- ÉTATS GLOBAUX ---
  let isLoading = $state(true);
  let isLoggedIn = $state(false);

  // --- ÉTATS : AUTHENTIFICATION ---
  let authMode = $state<'login' | 'register'>('login');
  let authPseudo = $state('');
  let authPassword = $state('');
  let authMessage = $state({ text: '', type: '' });
  let isAuthLoading = $state(false);

  // --- ÉTATS : PROFIL ---
  let isSaving = $state(false);
  let profileMessage = $state({ text: '', type: '' });
  let userId = $state('');
  let pseudo = $state('');
  let userCaracs = $state<Record<string, any>>({});
  let criteres = $state<any[]>([]);

  // --- INITIALISATION ---
  onMount(async () => {
    // Écoute des changements de connexion (se déclenche aussi au chargement)
    supabase.auth.onAuthStateChange(async (event, session) => {
      if (session) {
        isLoggedIn = true;
        userId = session.user.id;
        await fetchProfileData();
      } else {
        isLoggedIn = false;
        isLoading = false;
      }
    });
  });

  // --- LOGIQUE : AUTHENTIFICATION ---
  function getFakeEmail(p: string) {
    const cleanPseudo = p.toLowerCase().replace(/[^a-z0-9]/g, '');
    return `${cleanPseudo}_viewerdl_test@gmail.com`;
  }

  async function handleAuth(e: Event) {
    e.preventDefault();
    isAuthLoading = true;
    authMessage = { text: '', type: '' };

    if (!authPseudo || !authPassword) {
      authMessage = { text: "Veuillez remplir tous les champs.", type: 'error' };
      isAuthLoading = false;
      return;
    }

    const email = getFakeEmail(authPseudo);

    if (authMode === 'register') {
      const { error } = await supabase.auth.signUp({
        email,
        password: authPassword,
        options: {
          data: { pseudo_reel: authPseudo } // Important pour ton Trigger SQL !
        }
      });
      if (error) authMessage = { text: "Erreur d'inscription : " + error.message, type: 'error' };
      // Si succès, onAuthStateChange va s'en rendre compte et basculer la vue
    } else {
      const { error } = await supabase.auth.signInWithPassword({
        email,
        password: authPassword
      });
      if (error) authMessage = { text: "Identifiants incorrects.", type: 'error' };
    }

    isAuthLoading = false;
  }

  // --- LOGIQUE : PROFIL ---
  async function fetchProfileData() {
    isLoading = true;

    const { data: profileData } = await supabase
      .from('profil_viewer')
      .select('pseudo, caracteristiques')
      .eq('id', userId)
      .single();

    if (profileData) {
      pseudo = profileData.pseudo;
      userCaracs = profileData.caracteristiques || {};
    }

    const { data: critData } = await supabase
      .from('config_caracteristiques')
      .select('*')
      .eq('actif', true)
      .order('ordre', { ascending: true });

    if (critData) {
      criteres = critData;
      criteres.forEach(c => {
        if (userCaracs[c.cle] === undefined) {
          if (c.type_donnee === 'boolean') userCaracs[c.cle] = false;
          else userCaracs[c.cle] = '';
        }
      });
    }

    isLoading = false;
  }

  async function saveProfile() {
    isSaving = true;
    profileMessage = { text: '', type: '' };

    const { error } = await supabase
      .from('profil_viewer')
      .update({ caracteristiques: userCaracs })
      .eq('id', userId);

    if (error) {
      profileMessage = { text: "Erreur de synchronisation : " + error.message, type: 'error' };
    } else {
      profileMessage = { text: "Profil synchronisé avec la matrice !", type: 'success' };
    }
    isSaving = false;
  }

  async function logout() {
    await supabase.auth.signOut();
  }
</script>

<div class="w-full max-w-2xl mx-auto flex flex-col items-center animate-fade-in z-20 pb-20">

  {#if isLoading}
    <div class="text-teal-300 animate-pulse uppercase tracking-widest text-sm font-bold mt-20">
      Connexion à la matrice...
    </div>
  {:else if !isLoggedIn}
    <div class="w-full max-w-md mt-10 bg-slate-900/80 backdrop-blur-xl border border-indigo-500/20 rounded-3xl p-8 shadow-[0_0_40px_rgba(99,102,241,0.1)]">

      <div class="text-center mb-8">
        <h1 class="text-3xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-2">
          {authMode === 'login' ? 'Identification' : 'Enregistrement'}
        </h1>
        <p class="text-indigo-200/50 text-xs tracking-widest uppercase">Accès au réseau ViewerDle</p>
      </div>

      {#if authMessage.text}
        <div class="mb-6 p-4 rounded-xl text-xs font-bold text-center border backdrop-blur-sm bg-slate-950/50 {authMessage.type === 'error' ? 'border-rose-500/50 text-rose-300' : 'border-teal-500/50 text-teal-300'}">
          {authMessage.text}
        </div>
      {/if}

      <form onsubmit={handleAuth} class="flex flex-col gap-4">
        <div class="flex flex-col gap-1">
          <label class="text-[10px] font-bold uppercase tracking-widest text-indigo-300/70 pl-1">Pseudo Twitch</label>
          <input
            type="text"
            bind:value={authPseudo}
            placeholder="Ton pseudo..."
            class="bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all"
          />
        </div>

        <div class="flex flex-col gap-1">
          <label class="text-[10px] font-bold uppercase tracking-widest text-indigo-300/70 pl-1">Mot de passe (Jeu)</label>
          <input
            type="password"
            bind:value={authPassword}
            placeholder="••••••••"
            class="bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all"
          />
        </div>

        <button
          type="submit"
          disabled={isAuthLoading}
          class="mt-4 py-4 rounded-xl font-bold uppercase tracking-widest bg-indigo-500/20 text-indigo-200 border border-indigo-500/30 hover:bg-indigo-400/30 hover:border-indigo-300 hover:shadow-[0_0_20px_rgba(99,102,241,0.3)] transition-all cursor-pointer disabled:opacity-50"
        >
          {isAuthLoading ? 'Chargement...' : (authMode === 'login' ? 'Connexion' : 'Créer mon agent')}
        </button>
      </form>

      <div class="mt-6 text-center border-t border-indigo-500/20 pt-6">
        <button
          type="button"
          onclick={() => { authMode = authMode === 'login' ? 'register' : 'login'; authMessage = {text: '', type: ''}; }}
          class="text-xs text-indigo-300/50 hover:text-teal-300 uppercase tracking-widest transition-colors"
        >
          {authMode === 'login' ? "Pas encore d'agent ? Créer un compte" : "Déjà enregistré ? S'identifier"}
        </button>
      </div>
    </div>

  {:else}
    <div class="w-full flex items-end justify-between mb-8 border-b border-indigo-500/30 pb-4">
      <div>
        <h2 class="text-sm font-bold text-indigo-300/60 tracking-widest uppercase mb-1">Dossier Agent</h2>
        <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400">
          {pseudo}
        </h1>
      </div>
      <button
        onclick={logout}
        class="text-xs font-bold uppercase tracking-widest text-rose-400/60 hover:text-rose-400 border border-rose-500/20 hover:border-rose-500/50 hover:bg-rose-500/10 px-4 py-2 rounded-xl transition-all cursor-pointer"
      >
        Déconnexion
      </button>
    </div>

    {#if profileMessage.text}
      <div class="w-full p-4 mb-6 rounded-xl text-sm font-bold tracking-wide border backdrop-blur-sm bg-slate-950/50 text-center {profileMessage.type === 'error' ? 'border-rose-500/50 text-rose-300' : 'border-teal-500/50 text-teal-300 shadow-[0_0_15px_rgba(45,212,191,0.2)]'}">
        {profileMessage.text}
      </div>
    {/if}

    <div class="w-full bg-slate-900/80 backdrop-blur-xl border border-teal-500/20 rounded-3xl shadow-[0_0_40px_rgba(45,212,191,0.05)] overflow-hidden p-6 md:p-10">

      <p class="text-xs text-indigo-200/50 uppercase tracking-widest font-semibold mb-8 text-center">
        Veuillez renseigner vos caractéristiques. Ces données serviront pour le jeu.
      </p>

      {#if criteres.length === 0}
        <div class="text-center text-indigo-300/40 italic py-10">
          Aucun critère n'a encore été configuré par l'administrateur.
        </div>
      {:else}
        <div class="flex flex-col gap-6">
          {#each criteres as critere}
            <div class="flex flex-col gap-2">
              <label class="text-xs font-bold tracking-widest uppercase text-teal-300/80 pl-1">
                {critere.label}
              </label>

              {#if critere.type_donnee === 'text'}
                <input type="text" bind:value={userCaracs[critere.cle]} placeholder="..." class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all" />
              {:else if critere.type_donnee === 'number'}
                <input type="number" bind:value={userCaracs[critere.cle]} placeholder="0" class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 font-mono focus:outline-none focus:border-teal-400 transition-all" />
              {:else if critere.type_donnee === 'boolean'}
                <label class="flex items-center gap-4 bg-slate-950 border border-indigo-500/30 rounded-xl p-3 cursor-pointer hover:border-teal-400/50 transition-all">
                  <input type="checkbox" bind:checked={userCaracs[critere.cle]} class="accent-teal-400 w-5 h-5 cursor-pointer" />
                  <span class="text-indigo-200 text-sm">Oui, c'est mon cas</span>
                </label>
              {:else if critere.type_donnee === 'date'}
                <input type="date" bind:value={userCaracs[critere.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all color-scheme-dark" />
              {:else if critere.type_donnee === 'enumeration'}
                <select bind:value={userCaracs[critere.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all cursor-pointer">
                  <option value="" disabled selected>--- Sélectionnez une option ---</option>
                  {#each critere.options || [] as option}
                    <option value={option}>{option}</option>
                  {/each}
                </select>
              {/if}
            </div>
          {/each}
        </div>

        <button onclick={saveProfile} disabled={isSaving} class="mt-10 w-full py-4 rounded-xl font-black uppercase tracking-widest bg-teal-500/20 text-teal-200 border border-teal-500/30 hover:bg-teal-400/30 hover:border-teal-300 hover:shadow-[0_0_25px_rgba(45,212,191,0.4)] transition-all disabled:opacity-50 cursor-pointer">
          {isSaving ? 'Écriture dans la matrice...' : 'Sauvegarder le profil'}
        </button>
      {/if}
    </div>
  {/if}
</div>

<style>
  .color-scheme-dark { color-scheme: dark; }
</style>