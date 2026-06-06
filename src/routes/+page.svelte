<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase, SUPABASE_URL, SUPABASE_KEY } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0, classiqueCount: 0, anecdoteCount: 0 });
  let authMessage = $state('');

  let messageContent = $state('');
  let isSubmittingMessage = $state(false);
  let messageStatus = $state({ text: '', type: '' });

  let classiqueList = $state<any[]>([]);
  let anecdoteList = $state<any[]>([]);

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
        const headers = { 'apikey': SUPABASE_KEY, 'Authorization': `Bearer ${SUPABASE_KEY}` };
        const countHeaders = { ...headers, 'Prefer': 'count=exact' };
        const today = getLocalToday();

        const [playersRes, gamesCountRes, todayGamesRes] = await Promise.all([
          fetch(`${SUPABASE_URL}/rest/v1/profil_viewer?select=id,pseudo`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id`, { method: 'HEAD', headers: countHeaders }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id_compte,victoire,tentatives,type_jeu&date_partie=eq.${today}`, { headers })
        ]);

        const allPlayers = await playersRes.json();
        const totalGames = gamesCountRes.headers.get('content-range')?.split('/')[1] || 0;
        const todayGames = await todayGamesRes.json();

        if (isMounted) {
          const processedClassique = (todayGames || []).filter((g: any) => g.type_jeu === 'viewerdl');
          const processedAnecdote = (todayGames || []).filter((g: any) => g.type_jeu === 'anecdotes');

          stats = {
            totalPlayers: allPlayers.length || 0,
            totalGames: Number(totalGames),
            classiqueCount: processedClassique.length,
            anecdoteCount: processedAnecdote.length
          };

          // Fonction de tri optimisée :
          // 1. Victoire d'abord (les gagnants en haut)
          // 2. Nombre de tentatives ascendant (le plus petit nombre d'essais en premier)
          const mapPlayers = (list: any[]) => list.map((game: any) => {
            const player = allPlayers.find((p: any) => p.id === game.id_compte);
            return {
              pseudo: player ? player.pseudo : 'Joueur',
              victoire: (game.victoire === true || String(game.victoire) === 'true'),
              tentatives: game.tentatives
            };
          }).sort((a, b) => {
            // Si l'un a gagné et l'autre non, priorité au gagnant
            if (a.victoire !== b.victoire) return a.victoire ? -1 : 1;
            // Sinon, tri par nombre de tentatives (le plus petit en premier)
            return a.tentatives - b.tentatives;
          });

          classiqueList = mapPlayers(processedClassique);
          anecdoteList = mapPlayers(processedAnecdote);
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

  <h1 class="text-5xl md:text-7xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-10">ViewerDle</h1>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse text-xs uppercase tracking-widest font-mono">Synchronisation des données...</p>
  {:else}
    <div class="flex flex-wrap gap-4 justify-center mb-16">
      <div class="bg-slate-900/80 border border-fuchsia-500/30 rounded-3xl p-6 px-10">
        <span class="block text-4xl font-black text-fuchsia-400">{stats.totalGames}</span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-200/60 font-bold">Parties Totales</span>
      </div>
      <div class="bg-slate-900/80 border border-teal-500/30 rounded-3xl p-6 px-10">
        <span class="block text-4xl font-black text-teal-300">{stats.totalPlayers}</span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-200/60 font-bold">Joueurs Inscrits</span>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-8 w-full max-w-4xl mb-16">

      <div class="bg-slate-900/60 backdrop-blur-md border border-teal-500/30 rounded-3xl p-6 flex flex-col gap-4">
        <h3 class="text-teal-400 font-black uppercase tracking-widest text-sm">Classique ({stats.classiqueCount})</h3>
        <div class="flex flex-col gap-2 max-h-64 overflow-y-auto custom-scrollbar pr-2">
          {#each classiqueList as p}
            <div class="flex justify-between items-center bg-slate-950/50 p-2 px-3 rounded-lg text-xs">
              <span class="font-bold text-indigo-200">{p.pseudo}</span>
              <span class="font-mono text-slate-400">{p.tentatives} essais {p.victoire ? '✅' : '❌'}</span>
            </div>
          {:else}
            <p class="text-[10px] text-slate-500 italic">Aucune partie aujourd'hui</p>
          {/each}
        </div>
      </div>

      <div class="bg-slate-900/60 backdrop-blur-md border border-amber-500/30 rounded-3xl p-6 flex flex-col gap-4">
        <h3 class="text-amber-400 font-black uppercase tracking-widest text-sm">Anecdotes ({stats.anecdoteCount})</h3>
        <div class="flex flex-col gap-2 max-h-64 overflow-y-auto custom-scrollbar pr-2">
          {#each anecdoteList as p}
            <div class="flex justify-between items-center bg-slate-950/50 p-2 px-3 rounded-lg text-xs">
              <span class="font-bold text-indigo-200">{p.pseudo}</span>
              <span class="font-mono text-slate-400">{p.tentatives} essais {p.victoire ? '✅' : '❌'}</span>
            </div>
          {:else}
            <p class="text-[10px] text-slate-500 italic">Aucune partie aujourd'hui</p>
          {/each}
        </div>
      </div>
    </div>
  {/if}

  <button onclick={handlePlayClick} class="mb-20 text-sm font-black uppercase text-teal-300 border-2 border-teal-500/30 px-8 py-4 rounded-2xl hover:bg-teal-500/20 transition-all">
    Accéder aux jeux
  </button>

  <div class="w-full max-w-2xl bg-slate-900/60 p-6 rounded-3xl border border-indigo-500/20">
    <form onsubmit={handleSendMessage} class="flex flex-col gap-4">
      <textarea bind:value={messageContent} placeholder="Un message pour les admins ?" rows="3" class="w-full bg-slate-950 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white"></textarea>
      <button type="submit" class="self-end bg-indigo-500/10 text-indigo-300 px-6 py-2 rounded-xl text-xs font-bold uppercase tracking-widest">Envoyer</button>
    </form>
  </div>
</div>

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #334155; border-radius: 4px; }
</style>