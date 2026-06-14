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

  let isComponentMounted = false;

  // 🎯 Configuration des paliers de déblocage des indices
  const hintThresholds = [4, 8, 15, 20, 25, 30, 35, 40, 45, 50, 60, 70];

  // 💡 Tri automatique des indices de la cible (Simple -> Moyen -> Niche)
  let sortedIndices = $derived(
    (targetViewer?.indices || [])
      .slice()
      .sort((a: any, b: any) => {
        const order: Record<string, number> = { simple: 1, moyen: 2, niche: 3 };
        return (order[a.difficulte] || 99) - (order[b.difficulte] || 99);
      })
  );

  // 🔓 Calcul du nombre d'indices débloqués (Se débloquent tous d'un coup si victoire)
  let unlockedIndicesCount = $derived(
    hasWon
      ? sortedIndices.length
      : hintThresholds.filter(t => guesses.length >= t).length
  );

  let filteredViewers = $derived(
    searchQuery.trim() === ''
      ? []
      : allViewers.filter(v => v.pseudo.toLowerCase().includes(searchQuery.toLowerCase()) && !guesses.some(g => g.id === v.id))
  );

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
          setTimeout(() => reject(new Error("Timeout réseau")), 4000)
        );

        const sessionPromise = supabase.auth.getSession();
        const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

        if (!session) {
          if (isComponentMounted) goto('/');
          return;
        }

        const rolePromise = supabase.from('viewer_droit').select('id_droit').eq('id_viewer', session.user.id).in('id_droit', [1, 3]);
        const { data: roleCheck } = await Promise.race([rolePromise, timeoutPromise]);

        if (!roleCheck || roleCheck.length === 0) {
          if (isComponentMounted) goto('/');
          return;
        }

        if (isComponentMounted) {
          await initGame();
          await checkAlreadyPlayed();
        }

      } catch (e: any) {
        console.warn("Erreur lors du chargement :", e);
        if (isComponentMounted) errorMessage = "Erreur réseau ou délai dépassé : " + e.message;
      } finally {
        if (isComponentMounted) isLoading = false;
      }
    };

    loadGame();
    return () => { isComponentMounted = false; };
  });

  async function initGame() {
    if (!isComponentMounted) return;
    const today = getLocalToday();
    const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout BDD")), 4000));

    const critPromise = supabase.from('config_caracteristiques').select('*').eq('actif', true).order('ordre', { ascending: true });
    const { data: critData, error: critErr } = await Promise.race([critPromise, timeoutPromise]);

    if (critErr) throw critErr;
    if (isComponentMounted) criteres = critData ? critData : [];

    // Récupération avec la colonne "indices"
    let snapPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, caracteristiques, indices').eq('date_jour', today);
    let { data: dailySnapshots } = await Promise.race([snapPromise, timeoutPromise]);

    if ((!dailySnapshots || dailySnapshots.length === 0) && isComponentMounted) {
      const liveViewersPromise = supabase.from('profil_viewer').select('id, pseudo, caracteristiques, indices');
      const { data: liveViewers } = await Promise.race([liveViewersPromise, timeoutPromise]);

      if (liveViewers && liveViewers.length > 0) {
        const payload = liveViewers.map(v => ({
          date_jour: today,
          id_compte: v.id,
          pseudo: v.pseudo,
          caracteristiques: v.caracteristiques || {},
          indices: v.indices || []
        }));

        const insertSnapPromise = supabase.from('joueur_daily_caracteristique').insert(payload);
        const { error: insertSnapErr } = await Promise.race([insertSnapPromise, timeoutPromise]);

        if (insertSnapErr) {
          const retrySnapshotsPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, caracteristiques, indices').eq('date_jour', today);
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
        id: v.id_compte, pseudo: v.pseudo, caracteristiques: v.caracteristiques, indices: v.indices
      }));
    }

    let histPromise = supabase.from('historique_cibles').select('id_compte, caracteristiques, indices').eq('date_cible', today).eq('type_jeu', 'viewerdl').maybeSingle();
    let { data: histData } = await Promise.race([histPromise, timeoutPromise]);

    if (!histData && allViewers.length > 0 && isComponentMounted) {
      const randomIndex = Math.floor(Math.random() * allViewers.length);
      const randomTarget = allViewers[randomIndex];

      const insertTargetPromise = supabase.from('historique_cibles').insert({
        date_cible: today,
        id_compte: randomTarget.id,
        type_jeu: 'viewerdl',
        caracteristiques: randomTarget.caracteristiques,
        indices: randomTarget.indices || []
      });

      const { error: insertErr } = await Promise.race([insertTargetPromise, timeoutPromise]);

      if (insertErr) {
        const retryDataPromise = supabase.from('historique_cibles').select('id_compte, caracteristiques, indices').eq('date_cible', today).eq('type_jeu', 'viewerdl').maybeSingle();
        const { data: retryData } = await Promise.race([retryDataPromise, timeoutPromise]);
        histData = retryData;
      } else {
        histData = { id_compte: randomTarget.id, caracteristiques: randomTarget.caracteristiques, indices: randomTarget.indices || [] };
      }
    }

    if (histData && isComponentMounted) {
      const targetSnapshot = allViewers.find(v => v.id === histData.id_compte);
      targetViewer = {
        id: histData.id_compte,
        pseudo: targetSnapshot ? targetSnapshot.pseudo : 'Joueur Mystère',
        caracteristiques: histData.caracteristiques,
        indices: histData.indices || []
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

          // 1. On interroge la nouvelle table pour récupérer TOUTES les tentatives d'aujourd'hui
          // gte = Greater Than or Equal (supérieur ou égal à aujourd'hui à minuit)
          const propsCheckPromise = supabase.from('historique_proposition')
            .select('id_proposition, is_correct, tentative_num')
            .eq('id_joueur', session.user.id)
            .eq('type_jeu', 'viewerdl')
            .gte('created_at', `${today}T00:00:00`)
            .order('tentative_num', { ascending: false }); // Ordre décroissant pour avoir le plus récent en haut

          const { data: pastPropositions } = await Promise.race([propsCheckPromise, timeoutPromise]);

          if (pastPropositions && pastPropositions.length > 0 && isComponentMounted) {

            let restoredGuesses = [];
            let foundVictory = false;
            let maxTentatives = pastPropositions[0].tentative_num; // Le numéro de la dernière tentative

            // 2. On reconstruit visuellement le plateau ligne par ligne
            for (const prop of pastPropositions) {
              const viewer = allViewers.find(v => v.id === prop.id_proposition);

              if (viewer) {
                // On recalcule les cases (vert/rouge) pour cette ancienne proposition
                const results = criteres.map(crit => {
                  const val = viewer.caracteristiques?.[crit.cle];
                  const targetVal = targetViewer.caracteristiques?.[crit.cle];
                  const isCritCorrect = String(val).toLowerCase() === String(targetVal).toLowerCase();

                  let displayValue = val;
                  if (val === true || String(val).toLowerCase() === 'true') displayValue = 'Vrai';
                  if (val === false || String(val).toLowerCase() === 'false') displayValue = 'Faux';

                  let hint = (!isCritCorrect && crit.type_donnee === 'number') ? (Number(val) < Number(targetVal) ? '🔼' : '🔽') : '';

                  return { value: displayValue, status: isCritCorrect ? 'correct' : 'incorrect', hint };
                });

                // On l'ajoute au tableau
                restoredGuesses.push({ id: viewer.id, pseudo: viewer.pseudo, results });

                if (prop.is_correct) {
                  foundVictory = true;
                }
              }
            }

            // 3. On met à jour l'interface d'un seul coup
            guesses = restoredGuesses;

            // 4. Si la victoire faisait partie des anciennes tentatives, on valide le jeu
            if (foundVictory) {
              hasWon = true;
              victoryInfo = { tentatives: maxTentatives, pseudo: targetViewer.pseudo };
            }
          }
        }
      } catch (error) {
        console.warn("Vérification historique annulée suite à une instabilité réseau.", error);
      }
    }

  async function handleGuess(viewer: any) {
      searchQuery = ''; showSuggestions = false;

      // On vérifie si c'est la bonne réponse
      const isCorrect = viewer.id === targetViewer.id;

      const results = criteres.map(crit => {
        const val = viewer.caracteristiques?.[crit.cle];
        const targetVal = targetViewer.caracteristiques?.[crit.cle];
        const isCritCorrect = String(val).toLowerCase() === String(targetVal).toLowerCase();

        let displayValue = val;
        if (val === true || String(val).toLowerCase() === 'true') displayValue = 'Vrai';
        if (val === false || String(val).toLowerCase() === 'false') displayValue = 'Faux';

        let hint = (!isCritCorrect && crit.type_donnee === 'number') ? (Number(val) < Number(targetVal) ? '🔼' : '🔽') : '';

        return { value: displayValue, status: isCritCorrect ? 'correct' : 'incorrect', hint };
      });

      // On ajoute la tentative visuellement pour le joueur
      guesses = [{ id: viewer.id, pseudo: viewer.pseudo, results }, ...guesses];

      // Mise à jour visuelle instantanée si c'est gagné
      if (isCorrect) {
        hasWon = true;
        victoryInfo = { tentatives: guesses.length, pseudo: targetViewer.pseudo };
        confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });
      }

      // 📡 COMMUNICATION BASE DE DONNÉES EN ARRIÈRE-PLAN
      try {
        const { data: { session } } = await supabase.auth.getSession();
        if (session) {

          // 1. On sauvegarde la proposition dans la nouvelle table analytique
          supabase.from('historique_proposition').insert({
            id_joueur: session.user.id,
            id_proposition: viewer.id,
            is_correct: isCorrect,
            tentative_num: guesses.length,
            type_jeu: 'viewerdl'
          }).then(({error}) => {
             if (error) console.warn("Erreur sauvegarde proposition analytique", error);
          });

          // 2. Si c'est gagné, on enregistre aussi la victoire globale comme avant
          if (isCorrect) {
            supabase.from('historique').insert({
              id_compte: session.user.id,
              victoire: true,
              tentatives: guesses.length,
              type_jeu: 'viewerdl'
            }).then(({error}) => {
               if (error) console.warn("Erreur sauvegarde victoire globale", error);
            });
          }
        }
      } catch (err) {
        console.warn("Erreur d'historisation.", err);
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
      <div class="w-full max-w-md relative mb-8 z-30">
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
          <div class="mb-12 p-8 bg-teal-500/10 border border-teal-500/50 rounded-3xl text-center shadow-[0_0_30px_rgba(45,212,191,0.1)] flex flex-col items-center">
            <h2 class="text-3xl font-black text-teal-300 uppercase tracking-widest mb-4">Vous avez trouvé !</h2>
            <p class="text-indigo-100 text-lg mb-8">
              Vous avez identifié la cible <span class="font-black text-teal-400">{victoryInfo?.pseudo}</span>
              en <span class="font-black text-teal-400">{victoryInfo?.tentatives} {victoryInfo?.tentatives && victoryInfo.tentatives > 1 ? 'tentatives' : 'tentative'}</span>.
            </p>

            <button
              onclick={() => goto('/jouer/anecdotes')}
              class="bg-amber-500/10 border-2 border-amber-500/40 hover:bg-amber-500/20 text-amber-300 font-black uppercase tracking-widest text-sm px-8 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(245,158,11,0.15)] hover:shadow-[0_0_25px_rgba(245,158,11,0.3)] hover:-translate-y-1 cursor-pointer flex items-center gap-3"
            >
              <span class="text-xl">📝</span> Enchaîner avec le mode Anecdotes
            </button>
          </div>
        {/if}

    <div class="w-full max-w-4xl mb-12 bg-slate-900/60 backdrop-blur-md border border-amber-500/20 rounded-3xl p-6 md:p-8 shadow-xl animate-fade-in z-20">
      <h2 class="text-sm font-black uppercase tracking-widest text-amber-400 mb-6 flex items-center gap-3">
        <span class="text-xl">💡</span> Base de données d'indices
      </h2>

      {#if sortedIndices.length === 0}
        <p class="text-slate-500 text-sm italic text-center py-4 border border-dashed border-slate-700 rounded-2xl">
          Cible fantôme : Aucun indice n'est disponible. Vous allez devoir faire sans !
        </p>
      {:else}
        <div class="flex flex-col gap-3">
          {#each sortedIndices as indice, i}
            {#if i < unlockedIndicesCount}
              <div class="flex items-start md:items-center gap-4 bg-slate-950/80 p-4 rounded-2xl border border-amber-500/30 animate-fade-in">
                <span class="px-3 py-1 rounded-lg text-[10px] font-black uppercase tracking-widest border shrink-0
                  {indice.difficulte === 'simple' ? 'bg-emerald-500/10 text-emerald-400 border-emerald-500/40' :
                   indice.difficulte === 'moyen' ? 'bg-amber-500/10 text-amber-400 border-amber-500/40' :
                   'bg-rose-500/10 text-rose-400 border-rose-500/40'}">
                  {indice.difficulte}
                </span>
                <p class="text-amber-100/90 text-sm leading-relaxed">{indice.texte}</p>
              </div>
            {/if}
          {/each}

          {#if !hasWon && unlockedIndicesCount < sortedIndices.length}
            {@const nextThreshold = hintThresholds[unlockedIndicesCount]}
            <div class="flex items-center justify-center gap-3 bg-slate-950/30 p-4 rounded-2xl border border-dashed border-slate-700/50">
              <span class="text-slate-500 text-lg">🔒</span>
              <p class="text-slate-400 text-[10px] md:text-xs font-bold uppercase tracking-widest text-center">
                Prochain indice débloqué à la {nextThreshold}ème tentative
                <span class="text-slate-500 normal-case tracking-normal">(actuellement : {guesses.length})</span>
              </p>
            </div>
          {/if}
        </div>
      {/if}
    </div>

    <div class="flex flex-col gap-3 w-full max-w-4xl mx-auto pb-20">

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

      {#each guesses as guess, rowIndex (guess.id)}
        <div class="flex w-full gap-1.5 md:gap-3 items-stretch">

          <div class="w-1/4 max-w-[90px] md:max-w-[140px] flex-shrink-0 flex items-center justify-center bg-slate-950/80 backdrop-blur-md border border-indigo-500/30 rounded-xl p-2 md:p-4 font-black text-[10px] md:text-sm text-indigo-100 shadow-lg z-10">
            <span class="break-words text-center line-clamp-2 leading-tight">{guess.pseudo}</span>
          </div>

          {#each guess.results as res, i}
            <div
              class="flex-1 min-w-0 flex flex-col items-center justify-center p-1 md:p-3 text-center border rounded-xl shadow-lg
              {res.status === 'correct' ? 'bg-teal-500/20 border-teal-400' : 'bg-rose-500/10 border-rose-500/30'}
              {rowIndex === 0 ? 'flip-card' : ''}"
              style={rowIndex === 0 ? `animation-delay: ${i * 250}ms;` : ''}
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
    0% {
      transform: perspective(400px) rotateX(90deg);
      opacity: 0;
    }
    100% {
      transform: perspective(400px) rotateX(0deg);
      opacity: 1;
    }
  }

  .flip-card {
    opacity: 0;
    animation: flipIn 0.6s ease-out forwards;
  }
</style>