<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import confetti from 'canvas-confetti';

  let isLoading = $state(true);
  let hasWon = $state(false);
  let targetViewer = $state<any>(null);
  let criteres = $state<any[]>([]);
  let allViewers = $state<any[]>([]);
  let guesses = $state<any[]>([]);
  let searchQuery = $state('');
  let showSuggestions = $state(false);
  let victoryInfo = $state<{tentatives: number, pseudo: string} | null>(null);
  let errorMessage = $state('');

  let filteredViewers = $derived(
    searchQuery.trim() === ''
      ? []
      : allViewers.filter(v => v.pseudo.toLowerCase().includes(searchQuery.toLowerCase()) && !guesses.some(g => g.id === v.id))
  );

  onMount(async () => {
    // BARRIÈRE DE SÉCURITÉ : Vérification de la session et des droits 1 ou 3
    const { data: { session } } = await supabase.auth.getSession();
    if (!session) {
      goto('/');
      return;
    }

    const { data: roleCheck } = await supabase
      .from('viewer_droit')
      .select('id_droit')
      .eq('id_viewer', session.user.id)
      .in('id_droit', [1, 3]); // Doit avoir le rôle 1 ou 3

    if (!roleCheck || roleCheck.length === 0) {
      // Pas les droits nécessaires -> Expulsion vers l'accueil
      goto('/');
      return;
    }

    try {
      await initGame();
      await checkAlreadyPlayed();
    } catch (e: any) {
      errorMessage = "Erreur critique de la matrice : " + e.message;
      console.error(e);
    } finally {
      isLoading = false;
    }
  });

  async function initGame() {
    const today = new Date().toISOString().split('T')[0];

    const { data: critData, error: critErr } = await supabase.from('config_caracteristiques').select('*').eq('actif', true).order('ordre', { ascending: true });
    if (critErr) throw critErr;
    criteres = critData || [];

    const { data: viewersData } = await supabase.from('profil_viewer').select('id, pseudo, caracteristiques');
    if (viewersData) allViewers = viewersData;

    const { data: histData } = await supabase.from('historique_cibles').select('id_compte').eq('date_cible', today).maybeSingle();

    if (histData) {
      const { data: targetData } = await supabase.from('profil_viewer').select('id, pseudo, caracteristiques').eq('id', histData.id_compte).single();
      targetViewer = targetData;
    }
  }

  async function checkAlreadyPlayed() {
    const { data: { session } } = await supabase.auth.getSession();
    if (session) {
      const today = new Date().toISOString().split('T')[0];
      const { data } = await supabase
        .from('historique')
        .select('tentatives')
        .eq('id_compte', session.user.id)
        .eq('date_partie', today)
        .maybeSingle();

      if (data) {
        hasWon = true;
        victoryInfo = { tentatives: data.tentatives, pseudo: targetViewer?.pseudo || 'Cible' };
      }
    }
  }

  async function handleGuess(viewer: any) {
    searchQuery = '';
    showSuggestions = false;

    const results = criteres.map(crit => {
      const val = viewer.caracteristiques?.[crit.cle];
      const targetVal = targetViewer.caracteristiques?.[crit.cle];
      const isCorrect = String(val).toLowerCase() === String(targetVal).toLowerCase();
      let hint = (!isCorrect && crit.type_donnee === 'number') ? (Number(val) < Number(targetVal) ? '🔼' : '🔽') : '';
      return { value: val, status: isCorrect ? 'correct' : 'incorrect', hint };
    });

    guesses = [{ id: viewer.id, pseudo: viewer.pseudo, results }, ...guesses];

    if (viewer.id === targetViewer.id) {
      hasWon = true;
      victoryInfo = { tentatives: guesses.length, pseudo: targetViewer.pseudo };
      confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });

      const { data: { session } } = await supabase.auth.getSession();
      if (session) {
        await supabase.from('historique').insert({
          id_compte: session.user.id, victoire: true, tentatives: guesses.length, type_jeu: 'viewerdl'
        });
      }
    }
  }
</script>

<div class="w-full min-h-[80vh] p-4 md:p-10 flex flex-col items-center text-white">
  <h1 class="text-4xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-8">Grille de Jeu</h1>

  {#if errorMessage}
    <div class="p-6 bg-rose-500/20 border border-rose-500 rounded-xl text-rose-300 font-bold">{errorMessage}</div>
  {:else if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest mt-10">Chargement de la session du jour...</p>
  {:else if !targetViewer}
    <p class="text-indigo-400 uppercase tracking-widest mt-10 text-center">Aucune cible initialisée aujourd'hui.<br/>Revenez plus tard.</p>
  {:else}

    {#if !hasWon}
      <div class="w-full max-w-md relative mb-10 z-20">
        <input type="text" bind:value={searchQuery} onfocus={() => showSuggestions = true} placeholder="Entrez un pseudo..." class="w-full bg-slate-900 border border-indigo-500/30 rounded-2xl p-4 text-white outline-none focus:border-teal-400" />
        {#if showSuggestions}
          <div class="absolute w-full bg-slate-900 border border-indigo-500/20 rounded-xl mt-2 overflow-hidden shadow-2xl">
            {#each filteredViewers as v}
              <button onclick={() => handleGuess(v)} class="w-full p-4 text-left hover:bg-teal-500/20 font-bold transition-colors">{v.pseudo}</button>
            {/each}
          </div>
        {/if}
      </div>
    {:else}
      <div class="mb-10 p-8 bg-teal-500/10 border border-teal-500/50 rounded-3xl text-center shadow-[0_0_30px_rgba(45,212,191,0.1)]">
        <h2 class="text-3xl font-black text-teal-300 uppercase tracking-widest mb-4">Transmission établie !</h2>
        <p class="text-indigo-100 text-lg">
          Vous avez déjà identifié le viewer <span class="font-black text-teal-400">{victoryInfo?.pseudo}</span>
          en <span class="font-black text-teal-400">{victoryInfo?.tentatives} {victoryInfo?.tentatives && victoryInfo.tentatives > 1 ? 'tentatives' : 'tentative'}</span>.
        </p>
        <p class="text-indigo-300/60 mt-6 uppercase tracking-widest text-xs font-bold">Revenez demain pour une nouvelle partie !</p>
      </div>
    {/if}

    <div class="flex flex-col gap-3 w-full items-center pb-20">
      {#each guesses as guess}
        <div class="flex gap-3 overflow-x-auto w-full justify-center pb-2">
          <div class="w-40 flex-shrink-0 flex items-center justify-center bg-slate-950 border border-indigo-500/20 rounded-xl p-4 font-black text-sm text-indigo-100">{guess.pseudo}</div>
          {#each guess.results as res}
            <div class="w-32 h-20 flex-shrink-0 rounded-xl flex flex-col items-center justify-center p-2 text-center border {res.status === 'correct' ? 'bg-teal-500/20 border-teal-400' : 'bg-rose-500/10 border-rose-500/30'}">
              <span class="text-xs font-bold break-words">{res.value}</span>
              {#if res.hint}<span class="text-xs mt-1 animate-bounce">{res.hint}</span>{/if}
            </div>
          {/each}
        </div>
      {/each}
    </div>
  {/if}
</div>