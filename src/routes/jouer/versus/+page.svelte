<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let currentUser = $state<any>(null);
  let allPlayers = $state<any[]>([]);
  let waitingGames = $state<any[]>([]);
  let myActiveGame = $state<any>(null);
  let errorMessage = $state('');

  let lobbyChannel: any;
  let isComponentMounted = false;

  onMount(() => {
    isComponentMounted = true;
    initLobby();

    return () => {
      isComponentMounted = false;
      if (lobbyChannel) supabase.removeChannel(lobbyChannel);
    };
  });

  async function initLobby() {
    isLoading = true;
    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) { goto('/'); return; }

      const { data: roleCheck } = await supabase.from('viewer_droit').select('id_droit').eq('id_viewer', session.user.id).in('id_droit', [1, 3]);
      if (!roleCheck || roleCheck.length === 0) { goto('/'); return; }

      currentUser = session.user;

      // Récupérer les pseudos pour un affichage propre
      const { data: players } = await supabase.from('profil_viewer').select('id, pseudo');
      allPlayers = players || [];

      await fetchGames();
      setupRealtime();
    } catch (e: any) {
      errorMessage = "Erreur de connexion au serveur d'identification.";
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  async function fetchGames() {
    if (!isComponentMounted) return;

    // On cherche les parties en attente ou celles du joueur en cours de sélection/jeu
    const { data: games, error } = await supabase
      .from('versus_parties')
      .select('*')
      .in('status', ['waiting', 'selecting', 'playing'])
      .order('created_at', { ascending: false });

    if (error) {
      console.error("Erreur chargement parties:", error);
      return;
    }

    if (isComponentMounted) {
      // Trouver si le joueur a une partie active
      myActiveGame = games.find((g: any) =>
        (g.joueur1_id === currentUser.id || g.joueur2_id === currentUser.id) &&
        ['waiting', 'selecting', 'playing'].includes(g.status)
      );

      // Si le statut passe à 'selecting' ou 'playing', on téléporte vers le plateau !
      if (myActiveGame && (myActiveGame.status === 'selecting' || myActiveGame.status === 'playing')) {
        goto(`/jouer/versus/${myActiveGame.id}`);
        return;
      }

      // Filtrer les parties en attente créées par d'autres
      waitingGames = games.filter((g: any) => g.status === 'waiting' && g.joueur1_id !== currentUser.id);
    }
  }

  function setupRealtime() {
    lobbyChannel = supabase.channel('lobby_updates')
      .on('postgres_changes', {
        event: '*',
        schema: 'public',
        table: 'versus_parties'
      }, () => {
        // Dès qu'une partie est créée, rejointe ou annulée, on met à jour l'écran
        fetchGames();
      })
      .subscribe();
  }

  function getPlayerPseudo(id: string) {
    const p = allPlayers.find(p => p.id === id);
    return p ? p.pseudo : 'Joueur Inconnu';
  }

  async function handleCreateGame() {
    errorMessage = '';
    try {
      const { error } = await supabase.from('versus_parties').insert({
        joueur1_id: currentUser.id,
        status: 'waiting'
      });
      if (error) throw error;
      // Realtime s'occupera de rafraîchir l'écran
    } catch (e: any) {
      errorMessage = "Impossible de créer la partie.";
    }
  }

  async function handleJoinGame(gameId: string) {
    errorMessage = '';
    try {
      // On rejoint en tant que Joueur 2 et on passe la partie en mode "sélection"
      const { error } = await supabase.from('versus_parties')
        .update({ joueur2_id: currentUser.id, status: 'selecting' })
        .eq('id', gameId);

      if (error) throw error;

      // La redirection se fera via la fonction fetchGames() déclenchée par Realtime !
    } catch (e: any) {
      errorMessage = "Impossible de rejoindre cette partie.";
    }
  }

  async function handleCancelGame() {
    if (!myActiveGame) return;
    errorMessage = '';
    try {
      const { error } = await supabase.from('versus_parties')
        .update({ status: 'cancelled' })
        .eq('id', myActiveGame.id);

      if (error) throw error;
      myActiveGame = null;
    } catch (e: any) {
      errorMessage = "Impossible d'annuler.";
    }
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center text-white min-h-[80vh] pb-20 animate-fade-in">

  <div class="text-center mb-12 flex flex-col items-center">
    <span class="px-4 py-1 rounded-full bg-indigo-500/20 text-indigo-300 text-[10px] font-black uppercase tracking-widest border border-indigo-500/30 mb-4 animate-pulse">
      Mode Multijoueur Actif
    </span>
    <h1 class="text-5xl md:text-7xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-blue-400 via-indigo-400 to-purple-400 mb-2 drop-shadow-[0_0_30px_rgba(99,102,241,0.2)]">
      Versus
    </h1>
    <p class="text-indigo-300/60 uppercase tracking-[0.2em] text-xs font-bold">
      Défiez un autre enquêteur en face à face
    </p>
  </div>

  {#if errorMessage}
    <div class="mb-8 p-4 bg-rose-500/10 border border-rose-500/30 rounded-xl text-center w-full max-w-md">
      <p class="text-xs font-bold text-rose-400 uppercase tracking-widest">{errorMessage}</p>
    </div>
  {/if}

  {#if isLoading}
    <p class="text-indigo-400 animate-pulse uppercase tracking-widest mt-10 text-sm font-bold">Connexion au réseau local...</p>
  {:else}

    {#if myActiveGame}
      <div class="w-full max-w-lg bg-slate-900/80 backdrop-blur-md border border-indigo-500/40 rounded-3xl p-10 flex flex-col items-center text-center shadow-[0_0_40px_rgba(99,102,241,0.15)] animate-fade-in relative overflow-hidden">

        <div class="absolute inset-0 opacity-20 pointer-events-none">
          <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-[200%] aspect-square rounded-full border border-indigo-500/50 animate-[ping_3s_ease-out_infinite]"></div>
          <div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-[100%] aspect-square rounded-full border border-indigo-500/50 animate-[ping_3s_ease-out_infinite_1s]"></div>
        </div>

        <div class="w-20 h-20 bg-indigo-500/10 rounded-full flex items-center justify-center border-2 border-indigo-500/40 mb-6 z-10 animate-pulse">
          <span class="text-3xl">⏳</span>
        </div>

        <h2 class="text-2xl font-black text-indigo-300 uppercase tracking-widest mb-2 z-10">Recherche d'adversaire</h2>
        <p class="text-xs text-indigo-200/60 font-bold uppercase tracking-widest mb-10 z-10">
          Votre partie est visible par les autres joueurs.
        </p>

        <button
          onclick={handleCancelGame}
          class="z-10 text-xs font-black uppercase tracking-widest text-rose-400 bg-rose-500/10 border border-rose-500/30 hover:bg-rose-500/20 px-6 py-3 rounded-xl transition-all cursor-pointer"
        >
          Annuler la partie
        </button>
      </div>

    {:else}
      <div class="grid grid-cols-1 lg:grid-cols-2 gap-8 w-full max-w-5xl">

        <div class="bg-slate-900/60 backdrop-blur-md border border-blue-500/30 rounded-3xl p-8 flex flex-col items-center text-center shadow-lg group hover:border-blue-400/60 transition-colors">
          <div class="w-16 h-16 bg-blue-500/10 rounded-2xl flex items-center justify-center border border-blue-500/30 mb-6 group-hover:scale-110 transition-transform">
            <span class="text-3xl">⚔️</span>
          </div>
          <h2 class="text-xl font-black text-blue-300 uppercase tracking-widest mb-4">Héberger un duel</h2>
          <p class="text-sm text-blue-200/60 mb-8 flex-1">
            Créez une salle d'interrogatoire et attendez qu'un adversaire se présente pour l'affrontement.
          </p>
          <button
            onclick={handleCreateGame}
            class="w-full bg-blue-500/10 border-2 border-blue-500/40 hover:bg-blue-500/20 text-blue-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(59,130,246,0.15)] hover:shadow-[0_0_25px_rgba(59,130,246,0.3)] cursor-pointer"
          >
            Créer une partie
          </button>
        </div>

        <div class="bg-slate-900/60 backdrop-blur-md border border-purple-500/30 rounded-3xl p-8 flex flex-col shadow-lg">
          <h2 class="text-sm font-black text-purple-300 uppercase tracking-widest mb-6 flex items-center gap-3">
            <div class="w-2 h-2 bg-purple-500 rounded-full animate-pulse"></div>
            Parties en attente
          </h2>

          <div class="flex flex-col gap-3 flex-1 overflow-y-auto max-h-[300px] custom-scrollbar pr-2">
            {#if waitingGames.length === 0}
              <div class="flex-1 flex flex-col items-center justify-center py-10 opacity-50 border border-dashed border-purple-500/30 rounded-2xl">
                <span class="text-3xl mb-3">📡</span>
                <p class="text-[10px] font-bold uppercase tracking-widest text-purple-300">Aucun signal détecté</p>
              </div>
            {:else}
              {#each waitingGames as game}
                <div class="flex justify-between items-center bg-slate-950/50 p-4 rounded-2xl border border-purple-500/20 hover:border-purple-500/50 transition-colors animate-fade-in group">
                  <div class="flex flex-col">
                    <span class="text-[9px] text-purple-400/60 font-black uppercase tracking-widest mb-1">Hôte</span>
                    <span class="font-bold text-purple-100">{getPlayerPseudo(game.joueur1_id)}</span>
                  </div>
                  <button
                    onclick={() => handleJoinGame(game.id)}
                    class="bg-purple-500/10 border border-purple-500/40 text-purple-300 px-4 py-2 rounded-xl text-xs font-black uppercase tracking-widest group-hover:bg-purple-500/20 transition-all cursor-pointer"
                  >
                    Rejoindre
                  </button>
                </div>
              {/each}
            {/if}
          </div>
        </div>

      </div>
    {/if}

  {/if}
</div>

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #8b5cf6; border-radius: 4px; }
</style>