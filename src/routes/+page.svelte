<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase, SUPABASE_URL, SUPABASE_KEY } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0, todayGames: 0 });
  let authMessage = $state('');

  let messageContent = $state('');
  let isSubmittingMessage = $state(false);
  let messageStatus = $state({ text: '', type: '' });

  // Nouvelles variables pour la popup des joueurs du jour
  let todayPlayersList = $state<any[]>([]);
  let showTodayPopup = $state(false);

  // Fonction pour obtenir la date du jour au format YYYY-MM-DD
  function getLocalToday() {
    const d = new Date();
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

  onMount(() => {
    let isMounted = true;
    isLoading = true;

    const fetchStats = async () => {
      try {
        const headers = {
          'apikey': SUPABASE_KEY,
          'Authorization': `Bearer ${SUPABASE_KEY}`
        };
        const countHeaders = { ...headers, 'Prefer': 'count=exact' };

        const today = getLocalToday();

        // 🚀 BYPASS HTTP MULTIPLE : On lance les 3 requêtes en même temps
        const [playersRes, gamesCountRes, todayGamesRes] = await Promise.all([
          // 1. On récupère la liste des id et pseudos pour faire le lien avec les parties du jour
          fetch(`${SUPABASE_URL}/rest/v1/profil_viewer?select=id,pseudo`, { headers }),
          // 2. On compte juste le nombre total de parties (ultra rapide)
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id`, { method: 'HEAD', headers: countHeaders }),
          // 3. On récupère toutes les données des parties d'aujourd'hui
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id_compte,victoire,tentatives&date_partie=eq.${today}`, { headers })
        ]);

        const allPlayers = await playersRes.json();
        const totalGames = gamesCountRes.headers.get('content-range')?.split('/')[1] || 0;
        const todayGames = await todayGamesRes.json();

        if (isMounted) {
          stats = {
            totalPlayers: allPlayers.length || 0,
            totalGames: Number(totalGames),
            todayGames: todayGames.length || 0
          };

          // On croise les données : on associe le pseudo de profil_viewer à l'id_compte de l'historique
          todayPlayersList = (todayGames || []).map((game: any) => {
            const player = allPlayers.find((p: any) => p.id === game.id_compte);
            return {
              pseudo: player ? player.pseudo : 'Joueur Inconnu',
              victoire: game.victoire,
              tentatives: game.tentatives
            };
          });

          // On trie la liste (les gagnants d'abord, puis par nombre de tentatives)
          todayPlayersList.sort((a, b) => {
            const aWin = a.victoire === true || String(a.victoire) === 'true';
            const bWin = b.victoire === true || String(b.victoire) === 'true';
            if (aWin !== bWin) return aWin ? -1 : 1;
            return a.tentatives - b.tentatives;
          });
        }
      } catch (error) {
        console.warn("Erreur de chargement HTTP direct :", error);
      } finally {
        if (isMounted) isLoading = false;
      }
    };

    fetchStats();

    return () => { isMounted = false; };
  });

  async function handlePlayClick() {
    authMessage = '';
    try {
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout")), 5000));
      const sessionPromise = supabase.auth.getSession();

      const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

      if (!session) {
        authMessage = "Veuillez d'abord vous identifier pour accéder au jeu.";
        return;
      }

      const rolePromise = supabase.from('viewer_droit').select('id_droit').eq('id_viewer', session.user.id).in('id_droit', [1, 3]);
      const { data: roleCheck } = await Promise.race([rolePromise, timeoutPromise]);

      if (!roleCheck || roleCheck.length === 0) {
        authMessage = "Accès refusé : Votre niveau d'accréditation est insuffisant.";
        return;
      }

      goto('/jouer');
    } catch (error) {
      authMessage = "Le réseau est instable. Veuillez réessayer.";
    }
  }

  async function handleSendMessage(e: Event) {
    e.preventDefault();
    messageStatus = { text: '', type: '' };
    if (!messageContent.trim()) {
      messageStatus = { text: "Le message est vide.", type: 'error' };
      return;
    }
    isSubmittingMessage = true;
    try {
      const { error } = await supabase.from('messagerie').insert({ contenu: messageContent.trim() });
      if (error) {
        messageStatus = { text: "Erreur : " + error.message, type: 'error' };
      } else {
        messageStatus = { text: "Message transmis avec succès.", type: 'success' };
        messageContent = '';
      }
    } catch (err) {
      messageStatus = { text: "Erreur inattendue du réseau.", type: 'error' };
    } finally {
      isSubmittingMessage = false;
    }
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
    <div class="flex flex-col md:flex-row gap-6 w-full max-w-4xl justify-center items-stretch mb-16 animate-fade-in">

      <div class="w-full relative bg-slate-900/80 backdrop-blur-md border border-fuchsia-500/30 rounded-3xl p-8 shadow-[0_0_30px_rgba(217,70,239,0.1)] group hover:border-fuchsia-400 transition-colors flex flex-col justify-center items-center">
        <div class="absolute -top-3 left-1/2 -translate-x-1/2 bg-slate-950 px-4 py-1 rounded-full border border-fuchsia-500/30 text-[10px] text-fuchsia-300 font-bold uppercase tracking-widest">
          Activité globale
        </div>
        <span class="block text-5xl md:text-6xl font-black text-fuchsia-400 font-mono mb-2 drop-shadow-[0_0_15px_rgba(217,70,239,0.4)]">
          {stats.totalGames}
        </span>
        <span class="text-xs uppercase tracking-widest text-indigo-200/60 font-bold">
          Parties Jouées
        </span>
      </div>

      <div class="w-full relative bg-slate-900/60 backdrop-blur-md border border-teal-500/20 rounded-3xl p-8 shadow-xl group hover:border-teal-500/40 transition-colors flex flex-col justify-center items-center">
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

      <button
        onclick={() => showTodayPopup = true}
        class="w-full relative bg-slate-900/70 backdrop-blur-md border border-amber-500/30 rounded-3xl p-8 shadow-[0_0_30px_rgba(245,158,11,0.1)] group hover:border-amber-400 hover:bg-slate-800/80 hover:scale-[1.02] transition-all cursor-pointer flex flex-col justify-center items-center"
      >
        <div class="absolute -top-3 left-1/2 -translate-x-1/2 bg-slate-950 px-4 py-1 rounded-full border border-amber-500/30 text-[10px] text-amber-300 font-bold uppercase tracking-widest whitespace-nowrap">
          Aujourd'hui
        </div>
        <span class="block text-5xl md:text-6xl font-black text-amber-400 font-mono mb-2 drop-shadow-[0_0_10px_rgba(245,158,11,0.3)] group-hover:scale-110 transition-transform">
          {stats.todayGames}
        </span>
        <span class="text-xs uppercase tracking-widest text-indigo-200/60 font-bold group-hover:-translate-y-1 transition-transform">
          Joueurs Actifs
        </span>
        <span class="absolute bottom-3 text-[9px] text-amber-400/70 uppercase tracking-widest opacity-0 group-hover:opacity-100 transition-opacity">
          Voir la liste
        </span>
      </button>

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

{#if showTodayPopup}
  <div class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-950/80 backdrop-blur-sm animate-fade-in">
    <div class="absolute inset-0 cursor-pointer" onclick={() => showTodayPopup = false}></div>

    <div class="relative bg-slate-900 border border-amber-500/30 rounded-3xl p-6 w-full max-w-md shadow-[0_0_40px_rgba(245,158,11,0.15)] flex flex-col max-h-[80vh] animate-fade-in">

      <h3 class="text-xl font-black uppercase tracking-widest text-amber-400 mb-6 text-center drop-shadow-[0_0_10px_rgba(245,158,11,0.3)]">
        Activité du jour
      </h3>

      <div class="flex-1 overflow-y-auto pr-2 custom-scrollbar space-y-2 mb-6">
        {#if todayPlayersList.length === 0}
          <p class="text-slate-400 text-sm text-center py-6 italic">
            Aucun joueur n'a encore tenté sa chance aujourd'hui.
          </p>
        {:else}
          {#each todayPlayersList as player}
            <div class="flex justify-between items-center bg-slate-800/50 rounded-xl p-3 border border-slate-700/50 hover:border-slate-600 transition-colors">
              <span class="font-bold text-indigo-200">{player.pseudo}</span>
              <div class="flex items-center gap-3">
                <span class="text-xs font-mono text-slate-400 bg-slate-900 px-2 py-1 rounded-md border border-slate-700">
                  {player.tentatives} {player.tentatives > 1 ? 'essais' : 'essai'}
                </span>
                <span class="text-xl drop-shadow-md">
                  {(player.victoire === true || String(player.victoire) === 'true') ? '✅' : '❌'}
                </span>
              </div>
            </div>
          {/each}
        {/if}
      </div>

      <button
        onclick={() => showTodayPopup = false}
        class="w-full py-4 rounded-xl bg-slate-800/80 hover:bg-slate-700 text-white font-bold uppercase tracking-widest text-xs transition-colors border border-slate-600 cursor-pointer"
      >
        Fermer le registre
      </button>

    </div>
  </div>
{/if}