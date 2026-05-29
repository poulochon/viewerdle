<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let userProfile = $state<any>(null);
  let userPseudo = $state('');
  let criteres = $state<any[]>([]);

  // Messages de retour pour le profil
  let profileMessage = $state('');
  let profileError = $state('');

  // Variables pour la sécurité (Mot de passe)
  let confirmPseudo = $state('');
  let newPassword = $state('');
  let pwdMessage = $state('');
  let pwdError = $state('');

  onMount(async () => {
    // 1. Vérification de la session
    const { data: { session } } = await supabase.auth.getSession();
    if (!session) {
      goto('/');
      return;
    }

    // 2. Récupération des données du joueur
    const { data: profile } = await supabase
      .from('profil_viewer')
      .select('*')
      .eq('id', session.user.id)
      .single();

    if (profile) {
      userProfile = profile;
      userPseudo = profile.pseudo;
      // On s'assure que l'objet caractéristiques existe bien
      if (!userProfile.caracteristiques) {
        userProfile.caracteristiques = {};
      }
    }

    // 3. Récupération des critères actifs pour générer le formulaire
    const { data: critData } = await supabase
      .from('config_caracteristiques')
      .select('*')
      .eq('actif', true)
      .order('ordre', { ascending: true });

    criteres = critData || [];
    isLoading = false;
  });

  // Fonction pour changer le mot de passe
  async function handlePasswordUpdate() {
    pwdError = '';
    pwdMessage = '';

    if (confirmPseudo !== userPseudo) {
      pwdError = "Le pseudo de confirmation est incorrect.";
      return;
    }
    if (newPassword.length < 6) {
      pwdError = "Le mot de passe doit faire au moins 6 caractères.";
      return;
    }

    const { error } = await supabase.auth.updateUser({
      password: newPassword
    });

    if (error) {
      pwdError = "Erreur système : " + error.message;
    } else {
      pwdMessage = "Mot de passe modifié avec succès !";
      confirmPseudo = '';
      newPassword = '';
    }
  }

  // Fonction pour sauvegarder les caractéristiques
  async function handleSaveProfile() {
    profileMessage = '';
    profileError = '';

    const { error } = await supabase
      .from('profil_viewer')
      .update({ caracteristiques: userProfile.caracteristiques })
      .eq('id', userProfile.id);

    if (error) {
      profileError = "Impossible de sauvegarder : " + error.message;
    } else {
      profileMessage = "Vos caractéristiques ont bien été mises à jour !";
    }
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20 text-white">

  <div class="text-center mb-12">
    <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-2 drop-shadow-md">
      Mon Espace Joueur
    </h1>
    <p class="text-indigo-300/50 uppercase tracking-[0.3em] text-xs font-bold">
      Gestion du compte et des caractéristiques
    </p>
  </div>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest mt-10">Chargement des données du viewer...</p>
  {:else if userProfile}

    <div class="w-full mb-10 p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-rose-500/30 rounded-3xl shadow-[0_0_20px_rgba(244,63,94,0.1)]">
      <h2 class="text-xs font-black uppercase tracking-widest text-rose-300/70 mb-6 flex items-center gap-2">
        <div class="w-2 h-2 bg-rose-500 rounded-full animate-pulse"></div>
        Sécurité du compte
      </h2>

      <div class="flex flex-col md:flex-row gap-4 items-stretch">
        <input
          type="text"
          bind:value={confirmPseudo}
          placeholder="Confirmez votre Pseudo"
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
          class="bg-rose-500/10 border border-rose-500/50 hover:bg-rose-500/20 text-rose-300 font-bold uppercase tracking-widest text-xs px-6 py-3 rounded-xl transition-all whitespace-nowrap shadow-[0_0_10px_rgba(244,63,94,0.1)] hover:shadow-[0_0_15px_rgba(244,63,94,0.3)]"
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

    <div class="w-full p-6 md:p-8 bg-slate-900/80 backdrop-blur-md border border-teal-500/30 rounded-3xl shadow-[0_0_20px_rgba(45,212,191,0.05)]">

      <div class="flex justify-between items-center mb-8 border-b border-teal-500/20 pb-4">
        <div>
          <h2 class="text-sm font-black uppercase tracking-widest text-teal-300 flex items-center gap-3">
            <div class="w-2 h-2 bg-teal-500 rounded-full animate-pulse"></div>
            Profil de Jeu
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
              <select
                bind:value={userProfile.caracteristiques[crit.cle]}
                class="bg-slate-950 border border-indigo-500/30 text-white rounded-xl px-4 py-3 text-sm outline-none focus:border-teal-400 transition-colors cursor-pointer"
              >
                <option value={null} disabled selected={userProfile.caracteristiques[crit.cle] === undefined}>Sélectionner...</option>
                <option value={true}>Vrai</option>
                <option value={false}>Faux</option>
              </select>

            {:else if crit.type_donnee === 'number'}
              <input
                type="number"
                bind:value={userProfile.caracteristiques[crit.cle]}
                class="bg-slate-950 border border-indigo-500/30 text-white rounded-xl px-4 py-3 text-sm outline-none focus:border-teal-400 transition-colors placeholder:text-slate-700"
                placeholder="Ex: 1995"
              />

            {:else}
              <input
                type="text"
                bind:value={userProfile.caracteristiques[crit.cle]}
                class="bg-slate-950 border border-indigo-500/30 text-white rounded-xl px-4 py-3 text-sm outline-none focus:border-teal-400 transition-colors placeholder:text-slate-700"
                placeholder="Votre réponse..."
              />
            {/if}
          </div>
        {/each}
      </div>

      <div class="flex flex-col items-center border-t border-teal-500/20 pt-8">
        <button
          onclick={handleSaveProfile}
          class="bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest text-sm px-10 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.15)] hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] hover:scale-105"
        >
          Enregistrer le profil
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

  {/if}
</div>