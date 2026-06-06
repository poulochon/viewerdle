<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let isLoading = $state(true);
  let userProfile = $state<any>(null);
  let userPseudo = $state('');
  let userEmail = $state('');
  let criteres = $state<any[]>([]);

  let authMode = $state<'login' | 'register'>('login');
  let authPassword = $state('');
  let authPseudo = $state('');
  let authError = $state('');
  let authMessage = $state('');

  // Feedbacks Caractéristiques
  let profileMessage = $state('');
  let profileError = $state('');

  // Feedbacks Indices
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
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout session")), 4000));
      const sessionPromise = supabase.auth.getSession();
      const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

      if (!session) {
        if (isComponentMounted) { userProfile = null; isLoading = false; }
        return;
      }

      if (isComponentMounted) userEmail = session.user.email || '';

      const dbRequests = Promise.all([
        supabase.from('profil_viewer').select('*').eq('id', session.user.id).single(),
        supabase.from('config_caracteristiques').select('*').eq('actif', true).order('ordre', { ascending: true })
      ]);

      const [profileReq, critReq] = await Promise.race([dbRequests, timeoutPromise]);

      if (isComponentMounted) {
        if (profileReq.data) {
          userProfile = profileReq.data;
          userPseudo = profileReq.data.pseudo;
          // Sécurités pour éviter les bugs si les colonnes sont vides
          if (!userProfile.caracteristiques) userProfile.caracteristiques = {};
          if (!userProfile.indices) userProfile.indices = [];
        }
        criteres = critReq.data || [];
      }
    } catch (error) {
      console.warn("Erreur réseau profil :", error);
      if (isComponentMounted) authError = "Instabilité réseau détectée.";
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  function generateFakeEmail(p: string) {
    const cleanPseudo = p.toLowerCase().replace(/[^a-z0-9]/g, '');
    return `${cleanPseudo}_viewerdl_test@gmail.com`;
  }

  async function handleLogin() {
    if (!isComponentMounted) return;
    authError = ''; authMessage = '';

    if (!authPseudo || !authPassword) {
      authError = "Veuillez renseigner vos identifiants."; return;
    }

    isLoading = true;
    try {
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout")), 4000));
      const loginPromise = supabase.auth.signInWithPassword({ email: generateFakeEmail(authPseudo), password: authPassword });
      const { error } = await Promise.race([loginPromise, timeoutPromise]);

      if (error && isComponentMounted) authError = "Identifiants invalides.";
    } catch (error) {
      if (isComponentMounted) authError = "Le serveur ne répond pas.";
    } finally {
      if (isComponentMounted && authError) isLoading = false;
    }
  }

  async function handleRegister() {
    if (!isComponentMounted) return;
    authError = ''; authMessage = '';

    if (!authPseudo || !authPassword) {
      authError = "Veuillez remplir tous les champs."; return;
    }

    isLoading = true;
    try {
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout")), 4000));
      const registerPromise = supabase.auth.signUp({
        email: generateFakeEmail(authPseudo), password: authPassword, options: { data: { pseudo_reel: authPseudo } }
      });
      const { error } = await Promise.race([registerPromise, timeoutPromise]);

      if (error && isComponentMounted) {
        if (error.message.includes('already registered') || error.message.includes('already exists')) {
          authError = "Ce pseudonyme est déjà utilisé par un autre joueur.";
        } else {
          authError = "Erreur de connexion : " + error.message;
        }
      }
    } catch (error) {
      if (isComponentMounted) authError = "Délai dépassé.";
    } finally {
      if (isComponentMounted && authError) isLoading = false;
    }
  }

  async function handleLogout() {
    if (!isComponentMounted) return;
    try {
      await supabase.auth.signOut();
      if (isComponentMounted) { authPassword = ''; authPseudo = ''; }
    } catch (error) {
      console.warn("Erreur déconnexion :", error);
    }
  }

  async function handlePasswordUpdate() {
    if (!isComponentMounted) return;
    pwdError = ''; pwdMessage = '';

    if (!oldPassword) { pwdError = "Ancien mot de passe requis."; return; }
    if (newPassword.length < 6) { pwdError = "6 caractères minimum."; return; }

    try {
      const { error: verifyError } = await supabase.auth.signInWithPassword({ email: userEmail, password: oldPassword });
      if (verifyError) { if (isComponentMounted) pwdError = "Ancien mot de passe incorrect."; return; }

      const { error: updateError } = await supabase.auth.updateUser({ password: newPassword });
      if (isComponentMounted) {
        if (updateError) pwdError = "Erreur système : " + updateError.message;
        else { pwdMessage = "Mot de passe modifié !"; oldPassword = ''; newPassword = ''; }
      }
    } catch (error) {
      if (isComponentMounted) pwdError = "Délai serveur dépassé.";
    }
  }

  async function handleSaveProfile() {
    if (!isComponentMounted) return;
    profileMessage = ''; profileError = '';

    try {
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout")), 4000));
      const updatePromise = supabase.from('profil_viewer').update({ caracteristiques: userProfile.caracteristiques }).eq('id', userProfile.id);
      const { error } = await Promise.race([updatePromise, timeoutPromise]);

      if (isComponentMounted) {
        if (error) profileError = "Impossible de sauvegarder : " + error.message;
        else profileMessage = "Vos caractéristiques ont bien été mises à jour !";
      }
    } catch (error) {
      if (isComponentMounted) profileError = "Réseau lent, réessayez plus tard.";
    }
  }

  /* --- NOUVELLES FONCTIONS POUR LES INDICES --- */

  function addIndice() {
    userProfile.indices.push({ texte: '', difficulte: 'simple' });
  }

  function removeIndice(index: number) {
    userProfile.indices.splice(index, 1);
  }

  async function handleSaveIndices() {
    if (!isComponentMounted) return;
    indicesMessage = ''; indicesError = '';

    // Vérification : on empêche la sauvegarde s'il y a un champ texte vide
    const hasEmpty = userProfile.indices.some((i: any) => !i.texte.trim());
    if (hasEmpty) {
      indicesError = "Veuillez remplir le texte de tous vos indices, ou supprimez les lignes vides.";
      return;
    }

    isSavingIndices = true;
    try {
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout")), 4000));
      const updatePromise = supabase.from('profil_viewer').update({ indices: userProfile.indices }).eq('id', userProfile.id);
      const { error } = await Promise.race([updatePromise, timeoutPromise]);

      if (isComponentMounted) {
        if (error) indicesError = "Erreur de sauvegarde : " + error.message;
        else indicesMessage = "Vos indices secrets ont bien été enregistrés !";
      }
    } catch (error) {
      if (isComponentMounted) indicesError = "Réseau lent, réessayez plus tard.";
    } finally {
      if (isComponentMounted) isSavingIndices = false;
    }
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20 text-white z-20 relative">

  <div class="text-center mb-12 flex flex-col items-center">
    <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-2 drop-shadow-md">
      Mon Espace Joueur
    </h1>
    <p class="text-indigo-300/50 uppercase tracking-[0.3em] text-xs font-bold mb-6">
      Gestion du compte et des caractéristiques
    </p>

    {#if !isLoading && userProfile}
      <button
        onclick={handleLogout}
        class="flex items-center gap-2 text-rose-400 hover:text-rose-300 border border-rose-500/30 hover:border-rose-400 bg-rose-500/10 px-6 py-2.5 rounded-xl text-xs font-bold uppercase tracking-widest transition-all shadow-[0_0_10px_rgba(244,63,94,0.1)] hover:shadow-[0_0_15px_rgba(244,63,94,0.3)] cursor-pointer"
      >
        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
          <path stroke-linecap="round" stroke-linejoin="round" d="M17 16l4-4m0 0l-4-4m4 4H7m6 4v1a3 3 0 01-3 3H6a3 3 0 01-3-3V7a3 3 0 013-3h4a3 3 0 013 3v1" />
        </svg>
        Déconnexion
      </button>
    {/if}
  </div>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest mt-10 text-sm font-bold">Connexion au réseau...</p>

  {:else if !userProfile}
    <div class="w-full max-w-md p-8 bg-slate-900/80 backdrop-blur-md border border-indigo-500/30 rounded-3xl shadow-[0_0_40px_rgba(99,102,241,0.1)] animate-fade-in">

      <div class="flex justify-center gap-6 mb-8 border-b border-indigo-500/20 pb-4">
        <button
          class="text-xs md:text-sm font-black uppercase tracking-widest transition-colors cursor-pointer {authMode === 'login' ? 'text-teal-300' : 'text-slate-500 hover:text-slate-400'}"
          onclick={() => { authMode = 'login'; authError = ''; authMessage = ''; }}
        >
          Connexion
        </button>
        <span class="text-slate-700">|</span>
        <button
          class="text-xs md:text-sm font-black uppercase tracking-widest transition-colors cursor-pointer {authMode === 'register' ? 'text-teal-300' : 'text-slate-500 hover:text-slate-400'}"
          onclick={() => { authMode = 'register'; authError = ''; authMessage = ''; }}
        >
          Inscription
        </button>
      </div>

      <form onsubmit={(e) => { e.preventDefault(); authMode === 'login' ? handleLogin() : handleRegister(); }} class="flex flex-col gap-4">
        <div class="flex flex-col gap-1">
          <label class="text-[10px] font-bold uppercase tracking-widest text-indigo-300/70 pl-1">Pseudo Twitch</label>
          <input
            type="text"
            bind:value={authPseudo}
            placeholder="Ton pseudo..."
            class="bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-teal-400 transition-colors placeholder:text-slate-600"
          />
        </div>

        <div class="flex flex-col gap-1">
          <label class="text-[10px] font-bold uppercase tracking-widest text-indigo-300/70 pl-1">Mot de passe</label>
          <input
            type="password"
            bind:value={authPassword}
            placeholder="••••••••"
            class="bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-teal-400 transition-colors placeholder:text-slate-600"
          />
        </div>

        <button
          type="submit"
          class="mt-4 bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.15)] hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] hover:scale-105 cursor-pointer"
        >
          {authMode === 'login' ? 'Initialiser la session' : 'Créer mon compte'}
        </button>
      </form>

      {#if authError}
        <div class="mt-6 p-4 w-full text-center bg-rose-500/10 border border-rose-500/30 rounded-xl">
          <p class="text-xs font-bold text-rose-400 uppercase tracking-widest">{authError}</p>
        </div>
      {:else if authMessage}
        <div class="mt-6 p-4 w-full text-center bg-teal-500/10 border border-teal-500/30 rounded-xl">
          <p class="text-xs font-bold text-teal-400 uppercase tracking-widest">{authMessage}</p>
        </div>
      {/if}
    </div>

  {:else}

    <div class="w-full mb-10 p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-rose-500/30 rounded-3xl shadow-[0_0_20px_rgba(244,63,94,0.1)] animate-fade-in">
      <h2 class="text-xs font-black uppercase tracking-widest text-rose-300/70 mb-6 flex items-center gap-2">
        <div class="w-2 h-2 bg-rose-500 rounded-full animate-pulse"></div>
        Sécurité du compte
      </h2>

      <div class="flex flex-col md:flex-row gap-4 items-stretch">
        <input
          type="password"
          bind:value={oldPassword}
          placeholder="Ancien mot de passe"
          class="flex-1 bg-slate-950 border border-rose-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-rose-400 placeholder:text-slate-600 transition-colors"
        />
        <input
          type="password"
          bind:value={newPassword}
          placeholder="Nouveau mot de passe"
          class="flex-1 bg-slate-950 border border-rose-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-rose-400 placeholder:text-slate-600 transition-colors"
        />
        <button
          onclick={handlePasswordUpdate}
          class="bg-rose-500/10 border border-rose-500/50 hover:bg-rose-500/20 text-rose-300 font-bold uppercase tracking-widest text-xs px-6 py-3 rounded-xl transition-all whitespace-nowrap shadow-[0_0_10px_rgba(244,63,94,0.1)] hover:shadow-[0_0_15px_rgba(244,63,94,0.3)] cursor-pointer"
        >
          Appliquer
        </button>
      </div>

      {#if pwdError}
        <div class="mt-4 p-3 bg-rose-500/10 border border-rose-500/30 rounded-xl">
          <p class="text-xs font-bold text-rose-400 uppercase tracking-widest">{pwdError}</p>
        </div>
      {:else if pwdMessage}
        <div class="mt-4 p-3 bg-teal-500/10 border border-teal-500/30 rounded-xl">
          <p class="text-xs font-bold text-teal-400 uppercase tracking-widest">{pwdMessage}</p>
        </div>
      {/if}
    </div>

    <div class="w-full p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-teal-500/30 rounded-3xl shadow-[0_0_40px_rgba(45,212,191,0.05)] animate-fade-in">

      <div class="flex justify-between items-center mb-8 border-b border-teal-500/20 pb-4">
        <div>
          <h2 class="text-sm font-black uppercase tracking-widest text-teal-300 flex items-center gap-3">
            <div class="w-2 h-2 bg-teal-500 rounded-full animate-pulse"></div>
            Caractéristiques
          </h2>
          <p class="text-xs text-indigo-300/60 mt-1">
            Ces informations permettront aux autres joueurs de vous identifier.
          </p>
        </div>
        <div class="text-right hidden md:block">
          <span class="text-[10px] uppercase tracking-widest text-indigo-300/40">Joueur connecté</span>
          <p class="font-bold text-indigo-200">{userPseudo}</p>
        </div>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
        {#each criteres as crit}
          <div class="flex flex-col gap-2">
            <label class="text-[10px] font-black uppercase tracking-widest text-indigo-300/70 ml-1">
              {crit.label}
            </label>

            {#if crit.type_donnee === 'boolean'}
              <label class="flex items-center gap-4 bg-slate-950 border border-indigo-500/30 rounded-xl p-3 cursor-pointer hover:border-teal-400/50 transition-all">
                <input type="checkbox" bind:checked={userProfile.caracteristiques[crit.cle]} class="accent-teal-400 w-5 h-5 cursor-pointer" />
                <span class="text-indigo-200 text-sm">Oui, c'est mon cas</span>
              </label>

            {:else if crit.type_donnee === 'enumeration'}
              <select bind:value={userProfile.caracteristiques[crit.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all cursor-pointer">
                <option value={null} disabled selected>--- Sélectionnez une option ---</option>
                {#each crit.options || [] as option}
                  <option value={option}>{option}</option>
                {/each}
              </select>

            {:else if crit.type_donnee === 'number'}
              <input type="number" bind:value={userProfile.caracteristiques[crit.cle]} placeholder="0" class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 font-mono focus:outline-none focus:border-teal-400 transition-all" />

            {:else if crit.type_donnee === 'date'}
              <input type="date" bind:value={userProfile.caracteristiques[crit.cle]} class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all" style="color-scheme: dark;" />

            {:else}
              <input type="text" bind:value={userProfile.caracteristiques[crit.cle]} placeholder="..." class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl p-3 text-indigo-100 focus:outline-none focus:border-teal-400 transition-all" />
            {/if}
          </div>
        {/each}
      </div>

      <div class="flex flex-col items-center border-t border-teal-500/20 pt-8">
        <button
          onclick={handleSaveProfile}
          class="bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest text-sm px-10 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.15)] hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] hover:scale-105 cursor-pointer"
        >
          Enregistrer les caractéristiques
        </button>

        {#if profileError}
          <div class="mt-6 p-4 w-full text-center bg-rose-500/10 border border-rose-500/30 rounded-xl">
            <p class="text-sm font-bold text-rose-400 uppercase tracking-widest">{profileError}</p>
          </div>
        {:else if profileMessage}
          <div class="mt-6 p-4 w-full text-center bg-teal-500/10 border border-teal-500/30 rounded-xl">
            <p class="text-sm font-bold text-teal-400 uppercase tracking-widest">{profileMessage}</p>
          </div>
        {/if}
      </div>
    </div>

    <div class="w-full mt-10 p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-amber-500/30 rounded-3xl shadow-[0_0_40px_rgba(245,158,11,0.05)] animate-fade-in">

      <div class="flex justify-between items-center mb-8 border-b border-amber-500/20 pb-4">
        <div>
          <h2 class="text-sm font-black uppercase tracking-widest text-amber-400 flex items-center gap-3">
            <div class="w-2 h-2 bg-amber-500 rounded-full animate-pulse"></div>
            Mes Indices Secrets
          </h2>
          <p class="text-xs text-amber-200/60 mt-1">
            Ajoutez des anecdotes pour aider les joueurs. Ils se débloqueront au fur et à mesure de la partie.
          </p>
        </div>
      </div>

      <div class="flex flex-col gap-4 mb-8">
        {#if userProfile.indices.length === 0}
          <p class="text-slate-500 text-sm italic text-center py-6 border border-dashed border-slate-700 rounded-2xl">
            Aucun indice configuré pour le moment.
          </p>
        {/if}

        {#each userProfile.indices as indice, i}
          <div class="flex flex-col md:flex-row gap-3 items-start md:items-center bg-slate-950/50 p-4 rounded-2xl border border-amber-500/10 hover:border-amber-500/30 transition-colors animate-fade-in">

            <span class="bg-amber-500/20 text-amber-300 text-[10px] font-black px-3 py-1.5 rounded-lg border border-amber-500/30">
              #{i + 1}
            </span>

            <input
              type="text"
              bind:value={indice.texte}
              placeholder="Ex: J'ai un chien nommé Rex..."
              class="flex-1 w-full bg-slate-900 border border-indigo-500/30 rounded-xl px-4 py-2 text-sm text-indigo-100 outline-none focus:border-amber-400 transition-colors placeholder:text-slate-600"
            />

            <select
              bind:value={indice.difficulte}
              class="bg-slate-900 border border-indigo-500/30 rounded-xl px-4 py-2 text-sm text-indigo-100 outline-none focus:border-amber-400 transition-colors cursor-pointer w-full md:w-auto"
            >
              <option value="simple">🟢 Simple</option>
              <option value="moyen">🟠 Moyen</option>
              <option value="niche">🔴 Niche</option>
            </select>

            <button
              onclick={() => removeIndice(i)}
              class="text-rose-400 hover:text-rose-300 p-2 bg-rose-500/10 hover:bg-rose-500/20 rounded-xl border border-rose-500/30 transition-colors cursor-pointer w-full md:w-auto flex justify-center"
              title="Supprimer cet indice"
            >
              <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
              </svg>
            </button>

          </div>
        {/each}

        <button
          onclick={addIndice}
          class="mt-2 w-full border border-dashed border-amber-500/40 hover:border-amber-400 text-amber-300/70 hover:text-amber-300 bg-amber-500/5 hover:bg-amber-500/10 py-4 rounded-2xl text-xs font-black uppercase tracking-widest transition-all cursor-pointer"
        >
          + Ajouter un indice
        </button>
      </div>

      <div class="flex flex-col items-center border-t border-amber-500/20 pt-8">
        <button
          onclick={handleSaveIndices}
          disabled={isSavingIndices}
          class="bg-amber-500/10 border-2 border-amber-500/40 hover:bg-amber-500/20 text-amber-400 font-black uppercase tracking-widest text-sm px-10 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(245,158,11,0.15)] hover:shadow-[0_0_25px_rgba(245,158,11,0.3)] hover:scale-105 cursor-pointer disabled:opacity-50"
        >
          {isSavingIndices ? 'Sauvegarde en cours...' : 'Enregistrer les indices'}
        </button>

        {#if indicesError}
          <div class="mt-6 p-4 w-full text-center bg-rose-500/10 border border-rose-500/30 rounded-xl">
            <p class="text-sm font-bold text-rose-400 uppercase tracking-widest">{indicesError}</p>
          </div>
        {:else if indicesMessage}
          <div class="mt-6 p-4 w-full text-center bg-amber-500/10 border border-amber-500/30 rounded-xl">
            <p class="text-sm font-bold text-amber-400 uppercase tracking-widest">{indicesMessage}</p>
          </div>
        {/if}
      </div>
    </div>

  {/if}
</div>