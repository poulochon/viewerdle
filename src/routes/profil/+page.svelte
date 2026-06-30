<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { fade, fly } from 'svelte/transition';
  import { backOut } from 'svelte/easing';

  let isLoading = $state(true);
  let userProfile = $state<any>(null);
  let userPseudo = $state('');
  let userEmail = $state('');
  let criteres = $state<any[]>([]);

  // Authentification
  let authMode = $state<'login' | 'register'>('login');
  let authPassword = $state('');
  let authPseudo = $state('');
  let authError = $state('');
  let authMessage = $state('');

  // Médailles
  let userMedals = $state<any[]>([]);

  // Feedbacks
  let profileMessage = $state('');
  let profileError = $state('');
  let indicesMessage = $state('');
  let indicesError = $state('');
  let isSavingIndices = $state(false);
  let oldPassword = $state('');
  let newPassword = $state('');
  let pwdMessage = $state('');
  let pwdError = $state('');

  let isComponentMounted = false;

  onMount(() => {
    isComponentMounted = true;

    const { data: { subscription } } = supabase.auth.onAuthStateChange(async (event, session) => {
      if (!isComponentMounted) return;
      if (session) {
        if (!userProfile) await checkSession();
      } else {
        userProfile = null;
        isLoading = false;
      }
    });

    checkSession();
    return () => {
      isComponentMounted = false;
      subscription.unsubscribe();
    };
  });

  async function checkSession() {
    if (!isComponentMounted) return;
    isLoading = true;

    try {
      const { data: { session } } = await supabase.auth.getSession();

      if (!session) {
        if (isComponentMounted) { userProfile = null; isLoading = false; }
        return;
      }

      if (isComponentMounted) userEmail = session.user.email || '';

      const [profileReq, critReq] = await Promise.all([
        supabase.from('profil_viewer').select('*').eq('id', session.user.id).single(),
        supabase.from('config_caracteristiques').select('*').eq('actif', true).order('ordre', { ascending: true })
      ]);

      if (isComponentMounted) {
        if (profileReq.data) {
          userProfile = profileReq.data;
          userPseudo = profileReq.data.pseudo;
          if (!userProfile.caracteristiques) userProfile.caracteristiques = {};
          if (!userProfile.indices) userProfile.indices = [];

          // Charger les médailles
          fetchMedals(session.user.id);
        }
        criteres = critReq.data || [];
      }
    } catch (error) {
      console.warn("Erreur profil :", error);
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  async function fetchMedals(userId: string) {
    const { data } = await supabase
      .from('user_medals')
      .select('*')
      .eq('id_compte', userId)
      .order('mois_annee', { ascending: false });
    if (isComponentMounted) userMedals = data || [];
  }

  function generateFakeEmail(p: string) {
    const cleanPseudo = p.toLowerCase().replace(/[^a-z0-9]/g, '');
    return `${cleanPseudo}_viewerdl_test@gmail.com`;
  }

  async function handleLogin() {
    authError = ''; authMessage = '';
    if (!authPseudo || !authPassword) { authError = "Identifiants requis."; return; }
    isLoading = true;
    const { error } = await supabase.auth.signInWithPassword({ email: generateFakeEmail(authPseudo), password: authPassword });
    if (error && isComponentMounted) { authError = "Identifiants invalides."; isLoading = false; }
  }

  async function handleRegister() {
    authError = ''; authMessage = '';
    if (!authPseudo || !authPassword) { authError = "Champs requis."; return; }
    isLoading = true;
    const { error } = await supabase.auth.signUp({
      email: generateFakeEmail(authPseudo), password: authPassword, options: { data: { pseudo_reel: authPseudo } }
    });
    if (error && isComponentMounted) { authError = error.message; isLoading = false; }
  }

  async function handleLogout() {
    await supabase.auth.signOut();
    if (isComponentMounted) { authPassword = ''; authPseudo = ''; userMedals = []; }
  }

  async function handlePasswordUpdate() {
    pwdError = ''; pwdMessage = '';
    const { error: verifyError } = await supabase.auth.signInWithPassword({ email: userEmail, password: oldPassword });
    if (verifyError) { pwdError = "Ancien mot de passe incorrect."; return; }
    const { error: updateError } = await supabase.auth.updateUser({ password: newPassword });
    if (updateError) pwdError = updateError.message;
    else { pwdMessage = "Modifié !"; oldPassword = ''; newPassword = ''; }
  }

  async function handleSaveProfile() {
    profileMessage = ''; profileError = '';
    const { error } = await supabase.from('profil_viewer').update({ caracteristiques: userProfile.caracteristiques }).eq('id', userProfile.id);
    if (error) profileError = error.message;
    else profileMessage = "Mis à jour !";
  }

  function addIndice() { userProfile.indices.push({ texte: '', difficulte: 'simple' }); }
  function removeIndice(index: number) { userProfile.indices.splice(index, 1); }

  async function handleSaveIndices() {
    indicesMessage = ''; indicesError = '';
    const hasEmpty = userProfile.indices.some((i: any) => !i.texte.trim());
    if (hasEmpty) { indicesError = "Veuillez remplir tous les indices."; return; }
    isSavingIndices = true;
    const { error } = await supabase.from('profil_viewer').update({ indices: userProfile.indices }).eq('id', userProfile.id);
    if (error) indicesError = error.message;
    else indicesMessage = "Enregistrés !";
    isSavingIndices = false;
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20 text-white z-20 relative">

  <div class="text-center mb-12 flex flex-col items-center">
    <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-2 drop-shadow-md">
      Mon Espace Joueur
    </h1>
    {#if !isLoading && userProfile}
      <button onclick={handleLogout} class="mt-4 text-rose-400 hover:text-rose-300 border border-rose-500/30 px-6 py-2 rounded-xl text-xs font-bold uppercase transition-all cursor-pointer">
        Déconnexion
      </button>
    {/if}
  </div>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest mt-10 text-sm font-bold">Connexion au réseau...</p>

  {:else if !userProfile}
    <div class="w-full max-w-md p-8 bg-slate-900/80 backdrop-blur-md border border-indigo-500/30 rounded-3xl shadow-xl">
      <div class="flex justify-center gap-6 mb-8 border-b border-indigo-500/20 pb-4">
        <button class="text-xs font-black uppercase {authMode === 'login' ? 'text-teal-300' : 'text-slate-500'}" onclick={() => authMode = 'login'}>Connexion</button>
        <button class="text-xs font-black uppercase {authMode === 'register' ? 'text-teal-300' : 'text-slate-500'}" onclick={() => authMode = 'register'}>Inscription</button>
      </div>
      <form onsubmit={(e) => { e.preventDefault(); authMode === 'login' ? handleLogin() : handleRegister(); }} class="flex flex-col gap-4">
        <input type="text" bind:value={authPseudo} placeholder="Pseudo Twitch" class="bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm outline-none focus:border-teal-400" />
        <input type="password" bind:value={authPassword} placeholder="Mot de passe" class="bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm outline-none focus:border-teal-400" />
        <button type="submit" class="mt-4 bg-teal-500/10 border border-teal-500/40 text-teal-300 font-black uppercase py-4 rounded-xl hover:bg-teal-500/20 transition-all">
          {authMode === 'login' ? 'Se connecter' : 'Créer un compte'}
        </button>
      </form>
      {#if authError}<p class="mt-4 text-rose-400 text-xs font-bold text-center">{authError}</p>{/if}
    </div>

  {:else}
    <div class="w-full mb-10 p-6 bg-slate-900/80 border border-rose-500/30 rounded-3xl">
      <h2 class="text-xs font-black uppercase text-rose-300/70 mb-6 flex items-center gap-2">
        <div class="w-2 h-2 bg-rose-500 rounded-full animate-pulse"></div> Sécurité
      </h2>
      <div class="flex flex-col md:flex-row gap-4">
        <input type="password" bind:value={oldPassword} placeholder="Ancien MDP" class="flex-1 bg-slate-950 border border-rose-500/30 rounded-xl px-4 py-3 text-sm" />
        <input type="password" bind:value={newPassword} placeholder="Nouveau MDP" class="flex-1 bg-slate-950 border border-rose-500/30 rounded-xl px-4 py-3 text-sm" />
        <button onclick={handlePasswordUpdate} class="bg-rose-500/10 border border-rose-500/50 text-rose-300 px-6 py-3 rounded-xl text-xs font-bold uppercase">Appliquer</button>
      </div>
      {#if pwdMessage}<p class="mt-4 text-teal-400 text-xs font-bold">{pwdMessage}</p>{/if}
    </div>

    <div class="w-full p-6 md:p-8 bg-slate-900/80 border border-teal-500/30 rounded-3xl mb-10">
      <h2 class="text-sm font-black uppercase text-teal-300 mb-6 flex items-center gap-3">
        <div class="w-2 h-2 bg-teal-500 rounded-full animate-pulse"></div> Caractéristiques
      </h2>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
        {#each criteres as crit}
          <div class="flex flex-col gap-2">
            <label class="text-[10px] font-black uppercase text-indigo-300/70">{crit.label}</label>
            {#if crit.type_donnee === 'boolean'}
              <label class="flex items-center gap-4 bg-slate-950 border border-indigo-500/30 rounded-xl p-3 cursor-pointer">
                <input type="checkbox" bind:checked={userProfile.caracteristiques[crit.cle]} class="accent-teal-400 w-5 h-5" />
                <span class="text-indigo-200 text-sm">Oui</span>
              </label>
            {:else if crit.type_donnee === 'enumeration'}
              <select bind:value={userProfile.caracteristiques[crit.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 outline-none">
                <option value={null}>--- Choisir ---</option>
                {#each crit.options || [] as opt}<option value={opt}>{opt}</option>{/each}
              </select>
            {:else}
              <input type={crit.type_donnee} bind:value={userProfile.caracteristiques[crit.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100" />
            {/if}
          </div>
        {/each}
      </div>
      <button onclick={handleSaveProfile} class="w-full bg-teal-500/10 border-2 border-teal-500/40 text-teal-300 font-black uppercase py-4 rounded-xl hover:bg-teal-500/20">Enregistrer</button>
    </div>

    <div class="w-full p-6 md:p-8 bg-slate-900/80 border border-amber-500/30 rounded-3xl mb-10">
      <h2 class="text-sm font-black uppercase text-amber-400 mb-6 flex items-center gap-3">
        <div class="w-2 h-2 bg-amber-500 rounded-full animate-pulse"></div> Mes Indices Secrets
      </h2>
      <div class="flex flex-col gap-4 mb-8">
        {#each userProfile.indices as indice, i}
          <div class="flex flex-col md:flex-row gap-3 bg-slate-950/50 p-4 rounded-2xl border border-amber-500/10">
            <span class="bg-amber-500/20 text-amber-300 text-[10px] font-black px-3 py-1.5 rounded-lg border border-amber-500/30 flex items-center">#{i+1}</span>
            <input type="text" bind:value={indice.texte} placeholder="Ton anecdote..." class="flex-1 bg-slate-900 border border-indigo-500/30 rounded-xl px-4 py-2 text-sm text-indigo-100" />
            <select bind:value={indice.difficulte} class="bg-slate-900 border border-indigo-500/30 rounded-xl px-4 py-2 text-sm text-indigo-100">
              <option value="simple">🟢 Simple</option><option value="moyen">🟠 Moyen</option><option value="niche">🔴 Niche</option>
            </select>
            <button onclick={() => removeIndice(i)} class="text-rose-400 p-2 bg-rose-500/10 rounded-xl">✕</button>
          </div>
        {/each}
        <button onclick={addIndice} class="w-full border border-dashed border-amber-500/40 text-amber-300/70 py-4 rounded-2xl text-xs font-black uppercase">+ Ajouter un indice</button>
      </div>
      <button onclick={handleSaveIndices} class="w-full bg-amber-500/10 border-2 border-amber-500/40 text-amber-400 font-black uppercase py-4 rounded-xl hover:bg-amber-500/20">Enregistrer les indices</button>
    </div>

    <div class="w-full p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-yellow-500/30 rounded-3xl shadow-xl animate-fade-in">
      <div class="flex justify-between items-center mb-8 border-b border-yellow-500/20 pb-4">
        <div>
          <h2 class="text-sm font-black uppercase tracking-widest text-yellow-400 flex items-center gap-3">
            <div class="w-2 h-2 bg-yellow-500 rounded-full animate-pulse"></div> Mon Palmarès
          </h2>
          <p class="text-xs text-yellow-200/60 mt-1">Vos exploits mensuels immortalisés.</p>
        </div>
      </div>

      {#if userMedals.length > 0}
        <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-4">
          {#each userMedals as medal}
            <div class="flex items-center gap-4 bg-slate-950/50 p-4 rounded-2xl border {medal.rank === 1 ? 'border-yellow-500/40 bg-yellow-500/5' : 'border-slate-700/50'} transition-all hover:scale-[1.02]">
              <span class="text-3xl">{medal.rank === 1 ? '🥇' : medal.rank === 2 ? '🥈' : '🥉'}</span>
              <div class="flex flex-col">
                <span class="text-xs font-black uppercase tracking-widest
                  {medal.categorie === 'createur' ? 'text-teal-400' : medal.categorie === 'streameuse' ? 'text-pink-400' : 'text-slate-200'}">
                  {
                    {
                      'assiduite': 'Assidu',
                      'createur': 'Créateur',
                      'streameuse': 'Streameuse',
                      'acharnes': 'Pro du Hasard'
                    }[medal.categorie] || medal.categorie
                  }
                </span>
                <span class="text-[10px] font-mono text-slate-500">{medal.mois_annee}</span>
              </div>
            </div>
          {/each}
        </div>
      {:else}
        <p class="text-slate-500 text-sm italic text-center py-10 border border-dashed border-slate-700 rounded-2xl">Aucune médaille remportée. À vos jeux !</p>
      {/if}
    </div>

  {/if}
</div>

<style>
  :global(.animate-fade-in) {
    animation: fadeIn 0.5s ease-out forwards;
  }
  @keyframes fadeIn {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
  }
</style>