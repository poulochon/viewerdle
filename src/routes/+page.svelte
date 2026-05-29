<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0 });
  let authMessage = $state(''); // Message d'erreur d'authentification

  // NOUVEAU : Variables pour la messagerie
  let messageContent = $state('');
  let isSubmittingMessage = $state(false);
  let messageStatus = $state({ text: '', type: '' });

  onMount(async () => {
    isLoading = true;

    // Récupération du nombre du joueur (viewers) inscrits
    const { count: playersCount } = await supabase
      .from('profil_viewer')
      .select('*', { count: 'exact', head: true });

    // Récupération du nombre total de parties enregistrées
    const { count: gamesCount } = await supabase
      .from('historique')
      .select('*', { count: 'exact', head: true });

    stats = {
      totalPlayers: playersCount || 0,
      totalGames: gamesCount || 0
    };

    isLoading = false;
  });

  // Fonction de vérification avant d'accéder au jeu
  async function handlePlayClick() {
    authMessage = '';

    // 1. Vérification de la connexion
    const { data: { session } } = await supabase.auth.getSession();

    if (!session) {
      authMessage = "Veuillez d'abord vous identifier pour accéder au jeu.";
      return;
    }

    // 2. Vérification des droits (Rôles 1 et 3 autorisés)
    const { data: roleCheck } = await supabase
      .from('viewer_droit')
      .select('id_droit')
      .eq('id_viewer', session.user.id)
      .in('id_droit', [1, 3]);

    if (!roleCheck || roleCheck.length === 0) {
      authMessage = "Accès refusé : Votre niveau d'accréditation est insuffisant.";
      return;
    }

    // Si tout est bon, on l'envoie sur le jeu
    goto('/jouer');
  }

  // NOUVEAU : Fonction pour envoyer le message
  async function handleSendMessage(e: Event) {
    e.preventDefault();
    messageStatus = { text: '', type: '' };

    if (!messageContent.trim()) {
      messageStatus = { text: "Le message est vide.", type: 'error' };
      return;
    }

    isSubmittingMessage = true;

    const { error } = await supabase
      .from('messagerie')
      .insert({ contenu: messageContent.trim() });

    if (error) {
      messageStatus = { text: "Erreur de transmission : " + error.message, type: 'error' };
    } else {
      messageStatus = { text: "Message transmis aux administrateurs avec succès.", type: 'success' };
      messageContent = ''; // On vide le champ après l'envoi
    }

    isSubmittingMessage = false;
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center text-center text-white min-h-[70vh] justify-center animate-fade-in pb-20">

  <h1 class="text-6xl md:text-8xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-6 drop-shadow-[0_0_40px_rgba(99,102,241,0.2)]">
    ViewerDle
  </h1>

  <p class="text-indigo-200/70 max-w-xl mx-auto mb-16 leading-relaxed text-sm md:text-lg">
    Bienvenue sur le réseau d'identification. Analysez les indices, décryptez les caractéristiques et démasquez la cible du jour.
  </p>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse text-xs uppercase tracking-widest font-mono mb-12">
      Calcul des statistiques réseau...
    </p>
  {:else}
    <div class="flex flex-col md:flex-row gap-6 w-full max-w-2xl justify-center items-center mb-16 animate-fade-in">

      <div class="w-full relative bg-slate-900/80 backdrop-blur-md border border-fuchsia-500/30 rounded-3xl p-8 shadow-[0_0_30px_rgba(217,70,239,0.1)] group hover:border-fuchsia-400 transition-colors">
        <div class="absolute -top-3 left-1/2 -translate-x-1/2 bg-slate-950 px-4 py-1 rounded-full border border-fuchsia-500/30 text-[10px] text-fuchsia-300 font-bold uppercase tracking-widest">
          Activité globale
        </div>
        <span class="block text-6xl md:text-7xl font-black text-fuchsia-400 font-mono mb-2 drop-shadow-[0_0_15px_rgba(217,70,239,0.4)]">
          {stats.totalGames}
        </span>
        <span class="text-xs uppercase tracking-widest text-indigo-200/60 font-bold">
          Parties Jouées
        </span>
      </div>

      <div class="w-full relative bg-slate-900/60 backdrop-blur-md border border-teal-500/20 rounded-3xl p-8 shadow-xl group hover:border-teal-500/40 transition-colors">
        <div class="absolute -top-3 left-1/2 -translate-x-1/2 bg-slate-950 px-4 py-1 rounded-full border border-teal-500/20 text-[10px] text-teal-300 font-bold uppercase tracking-widest">
          Base de données
        </div>
        <span class="block text-5xl md:text-6xl font-black text-teal-300 font-mono mb-2 drop-shadow-[0_0_10px_rgba(45,212,191,0.3)]">
          {stats.totalPlayers}
        </span>
        <span class="text-xs uppercase tracking-widest text-indigo-200/60 font-bold">
          Joueurs Inscrits
        </span>
      </div>

    </div>
  {/if}

  <div class="flex flex-col items-center min-h-[100px] mb-20">
    <button
      onclick={handlePlayClick}
      class="text-sm font-black uppercase tracking-widest text-teal-300 bg-teal-500/10 border-2 border-teal-500/30 hover:border-teal-400 hover:bg-teal-500/20 px-8 py-4 rounded-2xl transition-all cursor-pointer shadow-[0_0_20px_rgba(45,212,191,0.1)] hover:shadow-[0_0_30px_rgba(45,212,191,0.3)] hover:scale-105"
    >
      Accéder à la grille de jeu
    </button>

    {#if authMessage}
      <div class="mt-6 px-6 py-3 bg-rose-500/10 border border-rose-500/50 rounded-xl text-rose-300 font-bold text-xs uppercase tracking-widest animate-fade-in shadow-[0_0_15px_rgba(244,63,94,0.2)]">
        {authMessage}
      </div>
    {/if}
  </div>

  <div class="w-full max-w-2xl bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 shadow-xl animate-fade-in text-left">
    <h2 class="text-xs font-black uppercase tracking-widest text-indigo-300/60 mb-6 flex items-center gap-3">
      <div class="w-2 h-2 bg-indigo-500 rounded-full animate-pulse"></div>
      Un message pour les admin ?
    </h2>

    <form onsubmit={handleSendMessage} class="flex flex-col gap-4">
      <textarea
        bind:value={messageContent}
        placeholder="Un retour, un bug ou une suggestion ? Laissez un message ici..."
        rows="4"
        class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-teal-400 transition-colors placeholder:text-slate-600 resize-y"
      ></textarea>

      <div class="flex justify-end items-center gap-4">
        {#if messageStatus.text}
          <span class="text-xs font-bold uppercase tracking-widest {messageStatus.type === 'error' ? 'text-rose-400' : 'text-teal-400'}">
            {messageStatus.text}
          </span>
        {/if}

        <button
          type="submit"
          disabled={isSubmittingMessage}
          class="bg-indigo-500/10 border border-indigo-500/40 hover:bg-indigo-500/20 text-indigo-300 font-bold uppercase tracking-widest text-xs px-6 py-3 rounded-xl transition-all shadow-[0_0_10px_rgba(99,102,241,0.1)] hover:shadow-[0_0_15px_rgba(99,102,241,0.3)] disabled:opacity-50 cursor-pointer"
        >
          {isSubmittingMessage ? 'Transmission...' : 'Envoyer'}
        </button>
      </div>
    </form>
  </div>

</div>