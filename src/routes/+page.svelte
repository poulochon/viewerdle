<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase, SUPABASE_URL, SUPABASE_KEY } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0, classiqueCount: 0, anecdoteCount: 0, totalAnecdotes: 0 });
  let authMessage = $state('');

  let messageContent = $state('');
  let isSubmittingMessage = $state(false);
  let messageStatus = $state({ text: '', type: '' });

  let classiqueList = $state<any[]>([]);
  let anecdoteList = $state<any[]>([]);

  // Nouveaux tableaux pour l'historique des cibles
  let historyClassique = $state<any[]>([]);
  let historyAnecdotes = $state<any[]>([]);

  function getLocalToday() {
    const d = new Date();
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

  // Petite fonction pour formater la date en JJ/MM (ex: 06/05)
  function formatDate(dateStr: string) {
    if (!dateStr) return '';
    const parts = dateStr.split('-');
    if (parts.length === 3) return `${parts[2]}/${parts[1]}`;
    return dateStr;
  }

  onMount(() => {
    let isMounted = true;
    isLoading = true;

    const fetchStats = async () => {
      try {
        const headers = { 'apikey': SUPABASE_KEY, 'Authorization': `Bearer ${SUPABASE_KEY}` };
        const countHeaders = { ...headers, 'Prefer': 'count=exact' };
        const today = getLocalToday();

        // Ajout de la récupération des "indices" (pour compter) et de "historique_cibles"
        const [playersRes, gamesCountRes, todayGamesRes, targetsRes] = await Promise.all([
          fetch(`${SUPABASE_URL}/rest/v1/profil_viewer?select=id,pseudo,indices`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id`, { method: 'HEAD', headers: countHeaders }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id_compte,victoire,tentatives,type_jeu&date_partie=eq.${today}`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/historique_cibles?select=date_cible,id_compte,type_jeu&order=date_cible.desc&limit=30`, { headers })
        ]);

        const allPlayers = await playersRes.json();
        const totalGames = gamesCountRes.headers.get('content-range')?.split('/')[1] || 0;
        const todayGames = await todayGamesRes.json();
        const targetsData = await targetsRes.json();

        if (isMounted) {
          const processedClassique = (todayGames || []).filter((g: any) => g.type_jeu === 'viewerdl');
          const processedAnecdote = (todayGames || []).filter((g: any) => g.type_jeu === 'anecdotes');

          // Calcul du nombre total d'anecdotes configurées par les joueurs
          const totalAnecdotesCount = allPlayers.reduce((acc: number, p: any) => {
            return acc + (p.indices && Array.isArray(p.indices) ? p.indices.length : 0);
          }, 0);

          stats = {
            totalPlayers: allPlayers.length || 0,
            totalGames: Number(totalGames),
            classiqueCount: processedClassique.length,
            anecdoteCount: processedAnecdote.length,
            totalAnecdotes: totalAnecdotesCount
          };

          const mapPlayers = (list: any[]) => list.map((game: any) => {
            const player = allPlayers.find((p: any) => p.id === game.id_compte);
            return {
              pseudo: player ? player.pseudo : 'Joueur',
              victoire: (game.victoire === true || String(game.victoire) === 'true'),
              tentatives: game.tentatives
            };
          }).sort((a, b) => {
            if (a.victoire !== b.victoire) return a.victoire ? -1 : 1;
            return a.tentatives - b.tentatives;
          });

          classiqueList = mapPlayers(processedClassique);
          anecdoteList = mapPlayers(processedAnecdote);

          // ------------------------------------------------------------------
          // Traitement de l'historique des cibles
          // ------------------------------------------------------------------
          const processHistory = (type: string) => {
            return (targetsData || [])
              .filter((t: any) => t.type_jeu === type && t.date_cible !== today) // ANTI-SPOIL: On exclut la cible du jour !
              .slice(0, 5) // On garde uniquement les 5 derniers jours
              .map((t: any) => {
                const player = allPlayers.find((p: any) => p.id === t.id_compte);
                return {
                  date: t.date_cible,
                  pseudo: player ? player.pseudo : 'Donnée effacée'
                };
              });
          };

          historyClassique = processHistory('viewerdl');
          historyAnecdotes = processHistory('anecdotes');
        }
      } catch (error) {
        console.warn("Erreur de chargement :", error);
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
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) { authMessage = "Veuillez vous identifier."; return; }
      goto('/jouer');
    } catch (e) { authMessage = "Réseau instable."; }
  }

  async function handleSendMessage(e: Event) {
    e.preventDefault();
    if (!messageContent.trim()) return;
    isSubmittingMessage = true;
    const { error } = await supabase.from('messagerie').insert({ contenu: messageContent.trim() });
    messageStatus = error ? { text: "Erreur", type: 'error' } : { text: "Envoyé !", type: 'success' };
    if (!error) messageContent = '';
    isSubmittingMessage = false;
  }
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center text-center text-white min-h-[70vh] pb-20 animate-fade-in">

  <h1 class="text-5xl md:text-7xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-10 drop-shadow-[0_0_40px_rgba(99,102,241,0.2)]">
    ViewerDle
  </h1>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse text-xs uppercase tracking-widest font-mono mb-12">Synchronisation des données...</p>
  {:else}

    <!-- STATS PRINCIPALES -->
    <div class="flex flex-wrap gap-4 justify-center mb-16">
      <div class="bg-slate-900/80 border border-fuchsia-500/30 rounded-3xl p-6 px-8 shadow-[0_0_20px_rgba(217,70,239,0.1)]">
        <span class="block text-4xl font-black text-fuchsia-400">{stats.totalGames}</span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-200/60 font-bold">Parties Totales</span>
      </div>
      <div class="bg-slate-900/80 border border-teal-500/30 rounded-3xl p-6 px-8 shadow-[0_0_20px_rgba(45,212,191,0.1)]">
        <span class="block text-4xl font-black text-teal-300">{stats.totalPlayers}</span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-200/60 font-bold">Joueurs Inscrits</span>
      </div>
      <div class="bg-slate-900/80 border border-amber-500/30 rounded-3xl p-6 px-8 shadow-[0_0_20px_rgba(245,158,11,0.1)]">
        <span class="block text-4xl font-black text-amber-400">{stats.totalAnecdotes}</span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-200/60 font-bold">Anecdotes Créées</span>
      </div>
    </div>

    <!-- GRILLE DÉTAILLÉE DU JOUR PAR MODE -->
    <div class="grid grid-cols-1 md:grid-cols-2 gap-8 w-full max-w-4xl mb-12">
      <!-- ACTIVITÉ CLASSIQUE -->
      <div class="bg-slate-900/60 backdrop-blur-md border border-teal-500/30 rounded-3xl p-6 flex flex-col gap-4 shadow-lg">
        <h3 class="text-teal-400 font-black uppercase tracking-widest text-sm flex items-center justify-center gap-2">
          Activité Classique <span class="bg-teal-500/20 px-2 py-0.5 rounded text-[10px]">{stats.classiqueCount}</span>
        </h3>
        <div class="flex flex-col gap-2 max-h-56 overflow-y-auto custom-scrollbar pr-2 mt-2">
          {#each classiqueList as p}
            <div class="flex justify-between items-center bg-slate-950/50 p-2 px-3 rounded-lg text-xs border border-teal-500/10">
              <span class="font-bold text-indigo-200">{p.pseudo}</span>
              <span class="font-mono text-slate-400">{p.tentatives} essais {p.victoire ? '✅' : '❌'}</span>
            </div>
          {:else}
            <p class="text-[10px] text-slate-500 italic py-4">Aucune partie recensée aujourd'hui.</p>
          {/each}
        </div>
      </div>

      <!-- ACTIVITÉ ANECDOTES -->
      <div class="bg-slate-900/60 backdrop-blur-md border border-amber-500/30 rounded-3xl p-6 flex flex-col gap-4 shadow-lg">
        <h3 class="text-amber-400 font-black uppercase tracking-widest text-sm flex items-center justify-center gap-2">
          Activité Anecdotes <span class="bg-amber-500/20 px-2 py-0.5 rounded text-[10px]">{stats.anecdoteCount}</span>
        </h3>
        <div class="flex flex-col gap-2 max-h-56 overflow-y-auto custom-scrollbar pr-2 mt-2">
          {#each anecdoteList as p}
            <div class="flex justify-between items-center bg-slate-950/50 p-2 px-3 rounded-lg text-xs border border-amber-500/10">
              <span class="font-bold text-indigo-200">{p.pseudo}</span>
              <span class="font-mono text-slate-400">{p.tentatives} essais {p.victoire ? '✅' : '❌'}</span>
            </div>
          {:else}
            <p class="text-[10px] text-slate-500 italic py-4">Aucune partie recensée aujourd'hui.</p>
          {/each}
        </div>
      </div>
    </div>

    <!-- NOUVEAU BLOC : HISTORIQUE DES CIBLES (DOSSIERS CLASSÉS) -->
    <div class="w-full max-w-4xl mb-16 bg-slate-900/40 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8">
      <h2 class="text-sm font-black uppercase tracking-widest text-indigo-300/70 mb-8 flex justify-center items-center gap-3">
        Dossiers Classés (Cibles précédentes)
      </h2>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
        <!-- Cibles Précédentes Classique -->
        <div class="flex flex-col gap-3">
          <h3 class="text-[10px] font-black uppercase tracking-widest text-teal-400/50 text-left pl-2">Mode Classique</h3>
          <div class="flex flex-col gap-1">
            {#each historyClassique as h}
              <div class="flex justify-between items-center text-xs p-2.5 bg-slate-950/30 rounded-lg border border-slate-800/50">
                <span class="text-slate-500 font-mono tracking-widest text-[10px]">{formatDate(h.date)}</span>
                <span class="font-bold text-teal-200/80">{h.pseudo}</span>
              </div>
            {:else}
              <p class="text-[10px] text-slate-500 italic text-left pl-2">Historique vide.</p>
            {/each}
          </div>
        </div>

        <!-- Cibles Précédentes Anecdotes -->
        <div class="flex flex-col gap-3">
          <h3 class="text-[10px] font-black uppercase tracking-widest text-amber-400/50 text-left pl-2">Mode Anecdotes</h3>
          <div class="flex flex-col gap-1">
            {#each historyAnecdotes as h}
              <div class="flex justify-between items-center text-xs p-2.5 bg-slate-950/30 rounded-lg border border-slate-800/50">
                <span class="text-slate-500 font-mono tracking-widest text-[10px]">{formatDate(h.date)}</span>
                <span class="font-bold text-amber-200/80">{h.pseudo}</span>
              </div>
            {:else}
              <p class="text-[10px] text-slate-500 italic text-left pl-2">Historique vide.</p>
            {/each}
          </div>
        </div>
      </div>
    </div>

  {/if}

  <!-- RESTE DE LA PAGE (Bouton Jouer / Message Admin) -->
  <button onclick={handlePlayClick} class="mb-20 text-sm font-black uppercase text-teal-300 border-2 border-teal-500/30 px-8 py-4 rounded-2xl hover:bg-teal-500/20 hover:scale-105 transition-all shadow-[0_0_20px_rgba(45,212,191,0.1)] cursor-pointer">
    Accéder aux jeux
  </button>

  <div class="w-full max-w-2xl bg-slate-900/60 p-6 md:p-8 rounded-3xl border border-indigo-500/20 shadow-xl">
    <h2 class="text-xs font-black uppercase tracking-widest text-indigo-300/60 mb-6 flex items-center gap-3">
      <div class="w-2 h-2 bg-indigo-500 rounded-full animate-pulse"></div>
      Un message pour les admin ?
    </h2>
    <form onsubmit={handleSendMessage} class="flex flex-col gap-4">
      <textarea bind:value={messageContent} placeholder="Un retour, un bug ou une suggestion ? Laissez un message ici..." rows="3" class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-teal-400 transition-colors"></textarea>

      <div class="flex justify-end items-center gap-4">
        {#if messageStatus.text}
          <span class="text-xs font-bold uppercase tracking-widest {messageStatus.type === 'error' ? 'text-rose-400' : 'text-teal-400'}">
            {messageStatus.text}
          </span>
        {/if}
        <button type="submit" disabled={isSubmittingMessage} class="bg-indigo-500/10 hover:bg-indigo-500/20 text-indigo-300 border border-indigo-500/40 px-6 py-3 rounded-xl text-xs font-bold uppercase tracking-widest transition-all cursor-pointer">
          {isSubmittingMessage ? 'Transmission...' : 'Envoyer'}
        </button>
      </div>
    </form>
  </div>
</div>

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #334155; border-radius: 4px; }
</style>