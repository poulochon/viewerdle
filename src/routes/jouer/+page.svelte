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

  // Sécurité anti-freeze : contrôle d'existence du composant
  let isComponentMounted = false;

  let filteredViewers = $derived(
    searchQuery.trim() === ''
      ? []
      : allViewers.filter(v => v.pseudo.toLowerCase().includes(searchQuery.toLowerCase()) && !guesses.some(g => g.id === v.id))
  );

  // Fonction utilitaire pour avoir YYYY-MM-DD en heure LOCALE
  function getLocalToday() {
    const d = new Date();
    const year = d.getFullYear();
    const month = String(d.getMonth() + 1).padStart(2, '0');
    const day = String(d.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

  onMount(() => {
    isComponentMounted = true;
    isLoading = true;

    const loadGame = async () => {
      try {
        const timeoutPromise = new Promise<any>((_, reject) =>
          setTimeout(() => reject(new Error("Timeout de connexion au réseau")), 4000)
        );

        // 1. Vérification de la session
        const sessionPromise = supabase.auth.getSession();
        const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

        if (!session) {
          if (isComponentMounted) goto('/');
          return;
        }

        // 2. Vérification de l'accréditation
        const rolePromise = supabase.from('viewer_droit').select('id_droit').eq('id_viewer', session.user.id).in('id_droit', [1, 3]);
        const { data: roleCheck } = await Promise.race([rolePromise, timeoutPromise]);

        if (!roleCheck || roleCheck.length === 0) {
          if (isComponentMounted) goto('/');
          return;
        }

        // 3. Initialisation du jeu
        if (isComponentMounted) {
          await initGame();
          await checkAlreadyPlayed();
        }

      } catch (e: any) {
        console.warn("Erreur lors du chargement initial :", e);
        if (isComponentMounted) errorMessage = "Erreur réseau ou délai d'attente dépassé : " + e.message;
      } finally {
        if (isComponentMounted) isLoading = false;
      }
    };

    loadGame();

    // NETTOYAGE à la destruction
    return () => {
      isComponentMounted = false;
    };
  });

  async function initGame() {
    if (!isComponentMounted) return;
    const today = getLocalToday();

    const timeoutPromise = new Promise<any>((_, reject) =>
      setTimeout(() => reject(new Error("Le réseau met trop de temps à répondre")), 4000)
    );

    // 1. Récupération des critères de jeu
    const critPromise = supabase.from('config_caracteristiques').select('*').eq('actif', true).order('ordre', { ascending: true });
    const { data: critData, error: critErr } = await Promise.race([critPromise, timeoutPromise]);

    if (critErr) throw critErr;
    if (isComponentMounted) criteres = critData ? critData : [];

    // ==========================================
    // 2. GESTION DU SNAPSHOT QUOTIDIEN
    // ==========================================

    let snapPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, caracteristiques').eq('date_jour', today);
    let { data: dailySnapshots, error: snapErr } = await Promise.race([snapPromise, timeoutPromise]);

    // S'il n'y a pas de snapshot, on le crée
    if ((!dailySnapshots || dailySnapshots.length === 0) && isComponentMounted) {
      console.log("Premier chargement du jour : Génération du Snapshot global...");

      const liveViewersPromise = supabase.from('profil_viewer').select('id, pseudo, caracteristiques');
      const { data: liveViewers } = await Promise.race([liveViewersPromise, timeoutPromise]);

      if (liveViewers && liveViewers.length > 0) {
        const payload = liveViewers.map(v => ({
          date_jour: today,
          id_compte: v.id,
          pseudo: v.pseudo,
          caracteristiques: v.caracteristiques || {}
        }));

        const insertSnapPromise = supabase.from('joueur_daily_caracteristique').insert(payload);
        const { error: insertSnapErr } = await Promise.race([insertSnapPromise, timeoutPromise]);

        if (insertSnapErr) {
          const retrySnapshotsPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, caracteristiques').eq('date_jour', today);
          const { data: retrySnapshots } = await Promise.race([retrySnapshotsPromise, timeoutPromise]);
          dailySnapshots = retrySnapshots || [];
        } else {
          dailySnapshots = payload;
        }
      } else {
        dailySnapshots = [];
      }
    }

    if (isComponentMounted) {
      allViewers = dailySnapshots.map((v: any) => ({
        id: v.id_compte,
        pseudo: v.pseudo,
        caracteristiques: v.caracteristiques
      }));
    }

    // ==========================================
    // 3. GESTION DE LA CIBLE DU JOUR
    // ==========================================

    let histPromise = supabase.from('historique_cibles').select('id_compte, caracteristiques').eq('date_cible', today).eq('type_jeu', 'viewerdl').maybeSingle();
    let { data: histData } = await Promise.race([histPromise, timeoutPromise]);

    if (!histData && allViewers.length > 0 && isComponentMounted) {
      console.log("Initialisation de la cible du jour...");

      const randomIndex = Math.floor(Math.random() * allViewers.length);
      const randomTarget = allViewers[randomIndex];

      const insertTargetPromise = supabase.from('historique_cibles').insert({
        date_cible: today,
        id_compte: randomTarget.id,
        type_jeu: 'viewerdl',
        caracteristiques: randomTarget.caracteristiques
      });

      const { error: insertErr } = await Promise.race([insertTargetPromise, timeoutPromise]);

      if (insertErr) {
        const retryDataPromise = supabase.from('historique_cibles').select('id_compte, caracteristiques').eq('date_cible', today).eq('type_jeu', 'viewerdl').maybeSingle();
        const { data: retryData } = await Promise.race([retryDataPromise, timeoutPromise]);
        histData = retryData;
      } else {
        histData = {
          id_compte: randomTarget.id,
          caracteristiques: randomTarget.caracteristiques
        };
      }
    }

    // 4. Assignation finale
    if (histData && isComponentMounted) {
      const targetSnapshot = allViewers.find(v => v.id === histData.id_compte);
      targetViewer = {
        id: histData.id_compte,
        pseudo: targetSnapshot ? targetSnapshot.pseudo : 'Joueur Mystère',
        caracteristiques: histData.caracteristiques
      };
    }
  }

  async function checkAlreadyPlayed() {
    if (!isComponentMounted) return;

    try {
      const sessionPromise = supabase.auth.getSession();
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout verif")), 3000));
      const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

      if (session && targetViewer) {
        const today = getLocalToday();

        const histCheckPromise = supabase.from('historique').select('tentatives').eq('id_compte', session.user.id).eq('date_partie', today).maybeSingle();
        const { data } = await Promise.race([histCheckPromise, timeoutPromise]);

        if (data && isComponentMounted) {
          hasWon = true;
          victoryInfo = { tentatives: data.tentatives, pseudo: targetViewer.pseudo };
        }
      }
    } catch (error) {
      console.warn("Vérification historique annulée suite à une instabilité réseau.", error);
    }
  }

  async function handleGuess(viewer: any) {
    searchQuery = '';
    showSuggestions = false;

    const results = criteres.map(crit => {
      // Valeurs figées
      const val = viewer.caracteristiques?.[crit.cle];
      const targetVal = targetViewer.caracteristiques?.[crit.cle];
      const isCorrect = String(val).toLowerCase() === String(targetVal).toLowerCase();

      let displayValue = val;
      if (val === true || String(val).toLowerCase() === 'true') displayValue = 'Vrai';
      if (val === false || String(val).toLowerCase() === 'false') displayValue = 'Faux';

      let hint = (!isCorrect && crit.type_donnee === 'number') ? (Number(val) < Number(targetVal) ? '🔼' : '🔽') : '';

      return { value: displayValue, status: isCorrect ? 'correct' : 'incorrect', hint };
    });

    guesses = [{ id: viewer.id, pseudo: viewer.pseudo, results }, ...guesses];

    if (viewer.id === targetViewer.id) {
      hasWon = true;
      victoryInfo = { tentatives: guesses.length, pseudo: targetViewer.pseudo };
      confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });

      try {
        const { data: { session } } = await supabase.auth.getSession();
        if (session) {
          // On n'attend pas la fin de l'insertion pour ne pas bloquer l'expérience utilisateur
          supabase.from('historique').insert({
            id_compte: session.user.id, victoire: true, tentatives: guesses.length, type_jeu: 'viewerdl'
          }).then(({error}) => {
             if (error) console.warn("Erreur lors de l'enregistrement de la victoire", error);
          });
        }
      } catch (err) {
        console.warn("Impossible de sauvegarder l'historique.", err);
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
      <div class="w-full max-w-md relative mb-12 z-20">
        <input type="text" bind:value={searchQuery} onfocus={() => showSuggestions = true} placeholder="Entrez un pseudo..." class="w-full bg-slate-900 border border-indigo-500/30 rounded-2xl p-4 text-white outline-none focus:border-teal-400 shadow-[0_0_15px_rgba(99,102,241,0.1)] transition-all" />
        {#if showSuggestions}
          <div class="absolute w-full bg-slate-900 border border-indigo-500/20 rounded-xl mt-2 overflow-hidden shadow-2xl">
            {#each filteredViewers as v}
              <button onclick={() => handleGuess(v)} class="w-full p-4 text-left hover:bg-teal-500/20 font-bold transition-colors cursor-pointer">{v.pseudo}</button>
            {/each}
          </div>
        {/if}
      </div>
    {:else}
      <div class="mb-12 p-8 bg-teal-500/10 border border-teal-500/50 rounded-3xl text-center shadow-[0_0_30px_rgba(45,212,191,0.1)]">
        <h2 class="text-3xl font-black text-teal-300 uppercase tracking-widest mb-4">Vous avez trouvé !</h2>
        <p class="text-indigo-100 text-lg">
          Vous avez identifié la cible <span class="font-black text-teal-400">{victoryInfo?.pseudo}</span>
          en <span class="font-black text-teal-400">{victoryInfo?.tentatives} {victoryInfo?.tentatives && victoryInfo.tentatives > 1 ? 'tentatives' : 'tentative'}</span>.
        </p>
        <p class="text-indigo-300/60 mt-6 uppercase tracking-widest text-xs font-bold">Revenez demain pour une nouvelle partie !</p>
      </div>
    {/if}

    <div class="flex flex-col gap-3 w-full max-w-4xl mx-auto pb-20 mt-4">

      {#if guesses.length > 0}
        <div class="flex w-full gap-1.5 md:gap-3 items-end mb-1 px-1 animate-fade-in">

          <div class="w-1/4 max-w-[90px] md:max-w-[140px] flex-shrink-0 text-center">
            <span class="text-[9px] md:text-[11px] font-black uppercase tracking-widest text-indigo-300/50">
              Joueur
            </span>
          </div>

          {#each criteres as crit}
            <div class="flex-1 min-w-0 flex justify-center text-center">
              <span class="text-[8px] md:text-[10px] font-black uppercase tracking-widest text-teal-300/50 break-words leading-tight line-clamp-2">
                {crit.label}
              </span>
            </div>
          {/each}

        </div>
      {/if}

      {#each guesses as guess}
        <div class="flex w-full gap-1.5 md:gap-3 items-stretch">

          <div class="w-1/4 max-w-[90px] md:max-w-[140px] flex-shrink-0 flex items-center justify-center bg-slate-950/80 backdrop-blur-md border border-indigo-500/30 rounded-xl p-2 md:p-4 font-black text-[10px] md:text-sm text-indigo-100 shadow-lg z-10">
            <span class="break-words text-center line-clamp-2 leading-tight">{guess.pseudo}</span>
          </div>

          {#each guess.results as res, i}
            <div
              class="flex-1 min-w-0 flex flex-col items-center justify-center p-1 md:p-3 text-center border rounded-xl shadow-lg flip-card
              {res.status === 'correct' ? 'bg-teal-500/20 border-teal-400' : 'bg-rose-500/10 border-rose-500/30'}"
              style="animation-delay: {i * 150}ms;"
            >
              <span class="text-[9px] md:text-xs font-bold text-white break-words whitespace-normal leading-tight">
                {res.value}
              </span>
              {#if res.hint}
                <span class="text-[10px] md:text-xs mt-1 animate-bounce">{res.hint}</span>
              {/if}
            </div>
          {/each}

        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  @keyframes flipIn {
    0% { transform: perspective(400px) rotateX(90deg); opacity: 0; }
    100% { transform: perspective(400px) rotateX(0deg); opacity: 1; }
  }

  .flip-card {
    opacity: 0;
    animation: flipIn 0.5s ease-out forwards;
  }
</style>