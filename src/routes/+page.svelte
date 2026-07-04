<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase, SUPABASE_URL, SUPABASE_KEY } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import { fade, fly } from 'svelte/transition';
  import { backOut } from 'svelte/easing';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0, classiqueCount: 0, anecdoteCount: 0, emoteCount: 0, totalAnecdotes: 0 });
  let authMessage = $state('');

  let messageContent = $state('');
  let isSubmittingMessage = $state(false);
  let messageStatus = $state({ text: '', type: '' });

  let isProfileIncomplete = $state(false);

  let classiqueList = $state<any[]>([]);
  let anecdoteList = $state<any[]>([]);
  let emoteList = $state<any[]>([]);

  // Tableaux pour l'historique des cibles
  let historyClassique = $state<any[]>([]);
  let historyAnecdotes = $state<any[]>([]);
  let historyEmotes = $state<any[]>([]);

  // Variables pour la pop-up du Récap Mensuel
  let showRecapPopup = $state(false);
  let popupKey = $state('');

  function getLocalToday() {
    const d = new Date();
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

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

        const [playersRes, gamesCountRes, todayGamesRes, targetsRes, emotesRes] = await Promise.all([
          fetch(`${SUPABASE_URL}/rest/v1/profil_viewer?select=id,pseudo,indices`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id`, { method: 'HEAD', headers: countHeaders }),
          fetch(`${SUPABASE_URL}/rest/v1/historique?select=id_compte,victoire,tentatives,type_jeu&date_partie=eq.${today}`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/historique_cibles?select=date_cible,id_compte,id_emote,type_jeu&order=date_cible.desc&limit=30`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/emotes_disponibles?select=id,nom`, { headers })
        ]);

        const allPlayers = await playersRes.json();
        const totalGames = gamesCountRes.headers.get('content-range')?.split('/')[1] || 0;
        const todayGames = await todayGamesRes.json();
        const targetsData = await targetsRes.json();
        const allEmotes = await emotesRes.json();

        if (isMounted) {
          const processedClassique = (todayGames || []).filter((g: any) => g.type_jeu === 'viewerdl');
          const processedAnecdote = (todayGames || []).filter((g: any) => g.type_jeu === 'anecdotes');
          const processedEmote = (todayGames || []).filter((g: any) => g.type_jeu === 'emote');

          const totalAnecdotesCount = allPlayers.reduce((acc: number, p: any) => {
            return acc + (p.indices && Array.isArray(p.indices) ? p.indices.length : 0);
          }, 0);

          stats = {
            totalPlayers: allPlayers.length || 0,
            totalGames: Number(totalGames),
            classiqueCount: processedClassique.length,
            anecdoteCount: processedAnecdote.length,
            emoteCount: processedEmote.length,
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
          emoteList = mapPlayers(processedEmote);

          const processHistory = (type: string) => {
            return (targetsData || [])
              .filter((t: any) => t.type_jeu === type && t.date_cible !== today)
              .slice(0, 5)
              .map((t: any) => {
                let targetName = 'Donnée effacée';
                if (type === 'emote') {
                  const emote = allEmotes.find((e: any) => e.id === t.id_emote);
                  if (emote) targetName = emote.nom;
                } else {
                  const player = allPlayers.find((p: any) => p.id === t.id_compte);
                  if (player) targetName = player.pseudo;
                }
                return { date: t.date_cible, nom: targetName };
              });
          };

          historyClassique = processHistory('viewerdl');
          historyAnecdotes = processHistory('anecdotes');
          historyEmotes = processHistory('emote');
        }
      } catch (error) {
        console.warn("Erreur de chargement :", error);
      } finally {
        if (isMounted) isLoading = false;
      }
    };

    fetchStats();

    // ==========================================
    // LOGIQUE DE LA POP-UP ET DU PROFIL
    // ==========================================
    const checkRecapPopup = async () => {
      try {
        const { data: { session } } = await supabase.auth.getSession();

        if (session && isMounted) {
          // --- NOUVEAU : Vérification du profil pour le bandeau rouge ---
          try {
            const [profileRes, critRes] = await Promise.all([
              supabase.from('profil_viewer').select('caracteristiques').eq('id', session.user.id).single(),
              supabase.from('config_caracteristiques').select('cle').eq('actif', true)
            ]);

            if (profileRes.data && critRes.data) {
              const carac = profileRes.data.caracteristiques || {};
              const totalCrit = critRes.data.length;
              let filledCrit = 0;

              for (const crit of critRes.data) {
                const val = carac[crit.cle];
                if (val !== null && val !== undefined && val !== '') filledCrit++;
              }

              // Si plus de 2 critères sont manquants, on active l'alerte
              if (totalCrit - filledCrit > 2) {
                isProfileIncomplete = true;
              }
            }
          } catch (e) {
            console.warn("Erreur vérif profil :", e);
          }
          // -------------------------------------------------------------

          // Date de lancement officiel de la feature recap
          const dateOuverture = new Date('2026-07-01T00:00:00+02:00');
          if (new Date() < dateOuverture) return;

          // On se place sur le mois PRÉCÉDENT pour générer la clé
          const prevMonthDate = new Date();
          prevMonthDate.setDate(0);

          const yyyy = prevMonthDate.getFullYear();
          const mm = String(prevMonthDate.getMonth() + 1).padStart(2, '0');

          popupKey = `recap_seen_${session.user.id}_${yyyy}-${mm}`;

          if (!localStorage.getItem(popupKey)) {
            showRecapPopup = true;
          }
        }
      } catch (error) {
        console.warn("Erreur session recap :", error);
      }
    };
    checkRecapPopup();

    return () => { isMounted = false; };
  });

  function closePopup() {
    showRecapPopup = false;
    if (popupKey) localStorage.setItem(popupKey, 'true');
  }

  function goToRecap() {
    closePopup();
    goto('/jouer/recap');
  }

  async function handlePlayClick() {
    authMessage = '';
    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) { authMessage = "Veuillez vous identifier pour jouer."; return; }
      goto('/jouer');
    } catch (e) { authMessage = "Réseau instable."; }
  }

  async function handleRecapClick() {
    authMessage = '';
    const dateOuverture = new Date('2026-07-01T00:00:00+02:00');
    if (new Date() < dateOuverture) {
      authMessage = "Patience ! La première rétrospective arrivera le 1er Juillet.";
      return;
    }

    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) {
        authMessage = "Veuillez vous identifier pour voir vos statistiques.";
        return;
      }
      goto('/jouer/recap');
    } catch (e) {
      authMessage = "Réseau instable.";
    }
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

<div class="w-full max-w-6xl mx-auto p-4 md:p-10 flex flex-col items-center text-center text-white min-h-[70vh] pb-20 animate-fade-in">

  <h1 class="text-5xl md:text-7xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-10 drop-shadow-[0_0_40px_rgba(99,102,241,0.2)]">
    ViewerDle
  </h1>

  <!-- BANDEAU D'ALERTE PROFIL INCOMPLET -->
  {#if isProfileIncomplete}
    <div in:fly={{ y: -20, duration: 500 }} class="w-full max-w-2xl mx-auto mb-10 bg-rose-500/10 border-2 border-rose-500/50 rounded-2xl p-4 flex items-center gap-5 shadow-[0_0_20px_rgba(244,63,94,0.3)] animate-pulse-slow">
      <span class="text-4xl drop-shadow-[0_0_10px_rgba(244,63,94,0.8)]">⚠️</span>
      <div class="text-left">
        <h3 class="text-rose-400 font-black uppercase tracking-widest text-sm mb-1">
          Attention : Profil Incomplet
        </h3>
        <p class="text-rose-200/80 text-xs leading-relaxed">
          Il te manque plus de 2 caractéristiques. Tu ne pourras pas être tiré au sort comme cible du jour !
          <a href="/jouer/profil" class="inline-block mt-1 underline font-bold text-rose-300 hover:text-white transition-colors">
            Compléter mon profil maintenant ➔
          </a>
        </p>
      </div>
    </div>
  {/if}

  {#if isLoading}
    <p class="text-teal-300 animate-pulse text-xs uppercase tracking-widest font-mono mb-12">Synchronisation des données...</p>
  {:else}

    <!-- STATS PRINCIPALES -->
    <div class="flex flex-wrap gap-4 justify-center mb-16">
      <div class="bg-slate-900/80 border border-indigo-500/30 rounded-3xl p-6 px-8 shadow-[0_0_20px_rgba(99,102,241,0.1)]">
        <span class="block text-4xl font-black text-indigo-400">{stats.totalGames}</span>
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
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 w-full max-w-6xl mb-12">
      <!-- ACTIVITÉ CLASSIQUE -->
      <div class="bg-slate-900/60 backdrop-blur-md border border-teal-500/30 rounded-3xl p-6 flex flex-col gap-4 shadow-lg">
        <h3 class="text-teal-400 font-black uppercase tracking-widest text-sm flex items-center justify-center gap-2">
          Classique <span class="bg-teal-500/20 px-2 py-0.5 rounded text-[10px]">{stats.classiqueCount}</span>
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
          Anecdotes <span class="bg-amber-500/20 px-2 py-0.5 rounded text-[10px]">{stats.anecdoteCount}</span>
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

      <!-- ACTIVITÉ ÉMOTES -->
      <div class="bg-slate-900/60 backdrop-blur-md border border-fuchsia-500/30 rounded-3xl p-6 flex flex-col gap-4 shadow-lg">
        <h3 class="text-fuchsia-400 font-black uppercase tracking-widest text-sm flex items-center justify-center gap-2">
          Émotes <span class="bg-fuchsia-500/20 px-2 py-0.5 rounded text-[10px]">{stats.emoteCount}</span>
        </h3>
        <div class="flex flex-col gap-2 max-h-56 overflow-y-auto custom-scrollbar pr-2 mt-2">
          {#each emoteList as p}
            <div class="flex justify-between items-center bg-slate-950/50 p-2 px-3 rounded-lg text-xs border border-fuchsia-500/10">
              <span class="font-bold text-indigo-200">{p.pseudo}</span>
              <span class="font-mono text-slate-400">{p.tentatives} essais {p.victoire ? '✅' : '❌'}</span>
            </div>
          {:else}
            <p class="text-[10px] text-slate-500 italic py-4">Aucune partie recensée aujourd'hui.</p>
          {/each}
        </div>
      </div>
    </div>

    <!-- HISTORIQUE DES CIBLES (DOSSIERS CLASSÉS) -->
    <div class="w-full max-w-6xl mb-16 bg-slate-900/40 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 shadow-xl">
      <h2 class="text-sm font-black uppercase tracking-widest text-indigo-300/70 mb-8 flex justify-center items-center gap-3">
        Dossiers Classés (Cibles précédentes)
      </h2>

      <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
        <div class="flex flex-col gap-3">
          <h3 class="text-[10px] font-black uppercase tracking-widest text-teal-400/50 text-left pl-2">Classique</h3>
          <div class="flex flex-col gap-1">
            {#each historyClassique as h}
              <div class="flex justify-between items-center text-xs p-2.5 bg-slate-950/30 rounded-lg border border-slate-800/50">
                <span class="text-slate-500 font-mono tracking-widest text-[10px]">{formatDate(h.date)}</span>
                <span class="font-bold text-teal-200/80 truncate ml-4">{h.nom}</span>
              </div>
            {:else}
              <p class="text-[10px] text-slate-500 italic text-left pl-2">Historique vide.</p>
            {/each}
          </div>
        </div>

        <div class="flex flex-col gap-3">
          <h3 class="text-[10px] font-black uppercase tracking-widest text-amber-400/50 text-left pl-2">Anecdotes</h3>
          <div class="flex flex-col gap-1">
            {#each historyAnecdotes as h}
              <div class="flex justify-between items-center text-xs p-2.5 bg-slate-950/30 rounded-lg border border-slate-800/50">
                <span class="text-slate-500 font-mono tracking-widest text-[10px]">{formatDate(h.date)}</span>
                <span class="font-bold text-amber-200/80 truncate ml-4">{h.nom}</span>
              </div>
            {:else}
              <p class="text-[10px] text-slate-500 italic text-left pl-2">Historique vide.</p>
            {/each}
          </div>
        </div>

        <div class="flex flex-col gap-3">
          <h3 class="text-[10px] font-black uppercase tracking-widest text-fuchsia-400/50 text-left pl-2">Émotes</h3>
          <div class="flex flex-col gap-1">
            {#each historyEmotes as h}
              <div class="flex justify-between items-center text-xs p-2.5 bg-slate-950/30 rounded-lg border border-slate-800/50">
                <span class="text-slate-500 font-mono tracking-widest text-[10px]">{formatDate(h.date)}</span>
                <span class="font-bold text-fuchsia-200/80 truncate ml-4">{h.nom}</span>
              </div>
            {:else}
              <p class="text-[10px] text-slate-500 italic text-left pl-2">Historique vide.</p>
            {/each}
          </div>
        </div>
      </div>
    </div>

  {/if}

  <!-- ZONE DE MESSAGE D'ERREUR BOUTONS -->
  {#if authMessage}
    <div transition:fly={{ y: -10, duration: 300 }} class="text-rose-400 font-bold uppercase tracking-widest text-xs mb-6 px-4 py-2 bg-rose-500/10 border border-rose-500/30 rounded-lg shadow-lg">
      {authMessage}
    </div>
  {/if}

  <!-- BOUTONS D'ACTION PRINCIPAUX -->
  <div class="flex flex-col md:flex-row gap-6 mb-20 w-full max-w-2xl justify-center">

    <button onclick={handlePlayClick} class="flex-1 text-sm font-black uppercase text-teal-300 border-2 border-teal-500/30 px-8 py-4 rounded-2xl hover:bg-teal-500/20 hover:scale-105 transition-all shadow-[0_0_20px_rgba(45,212,191,0.1)] cursor-pointer">
      Accéder aux jeux
    </button>

    <button onclick={handleRecapClick} class="flex-1 flex items-center justify-center gap-3 text-sm font-black uppercase text-fuchsia-300 border-2 border-fuchsia-500/30 px-8 py-4 rounded-2xl hover:bg-fuchsia-500/20 hover:scale-105 transition-all shadow-[0_0_20px_rgba(217,70,239,0.1)] cursor-pointer">
      <span class="text-xl block">📦</span> Rétrospective du mois
    </button>

  </div>

  <!-- ZONE DE MESSAGE ADMIN -->
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

<!-- POP-UP RÉCAP MENSUEL ANIMÉE -->
{#if showRecapPopup}
  <div transition:fade={{ duration: 300 }} class="fixed inset-0 bg-slate-950/80 backdrop-blur-md flex items-center justify-center z-50 p-4">
    <div in:fly={{ y: 50, duration: 600, easing: backOut }} class="bg-slate-900 border border-fuchsia-500/40 rounded-3xl p-8 max-w-md w-full relative shadow-[0_0_50px_rgba(217,70,239,0.2)]">

      <!-- Bouton Croix -->
      <button onclick={closePopup} class="absolute top-4 right-4 text-slate-500 hover:text-fuchsia-400 text-xl cursor-pointer transition-colors p-2">✕</button>

      <div class="text-center">
        <span class="text-5xl mb-4 block animate-bounce drop-shadow-[0_0_15px_rgba(217,70,239,0.4)]">📦</span>
        <h2 class="text-2xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-fuchsia-400 to-pink-500 mb-3">
          Ton Récap du Mois !
        </h2>
        <p class="text-sm text-indigo-200/70 leading-relaxed mb-8">
          Le mois s'achève ! Viens découvrir tes statistiques personnelles ainsi que les grands exploits de la communauté.
        </p>

        <div class="flex flex-col gap-3">
          <button onclick={goToRecap} class="w-full bg-gradient-to-r from-fuchsia-500 to-pink-500 hover:opacity-90 text-white font-black uppercase tracking-widest text-xs py-4 rounded-xl shadow-[0_0_20px_rgba(217,70,239,0.4)] cursor-pointer transition-all hover:scale-[1.02]">
            Découvrir mes stats
          </button>
          <button onclick={closePopup} class="w-full bg-slate-800 hover:bg-slate-700 text-slate-400 font-bold uppercase tracking-widest text-[10px] py-3 rounded-xl cursor-pointer transition-all">
            Plus tard
          </button>
        </div>
      </div>
    </div>
  </div>
{/if}

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #334155; border-radius: 4px; }

  .animate-pulse-slow {
    animation: pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }
</style>