<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import confetti from 'canvas-confetti';

  let isLoading = $state(true);
  let hasWon = $state(false);
  let targetViewer = $state<any>(null);
  let allViewers = $state<any[]>([]);
  let guesses = $state<any[]>([]);
  let searchQuery = $state('');
  let showSuggestions = $state(false);
  let victoryInfo = $state<{tentatives: number, pseudo: string} | null>(null);
  let errorMessage = $state('');

  let hasPlayedClassic = $state(false);
  let hasPlayedEmote = $state(false); // NOUVEAU : Variable pour le mode emote

  let isComponentMounted = false;

  // 💡 Tri automatique des indices de la cible (Simple -> Moyen -> Niche)
  let sortedIndices = $derived(
    (targetViewer?.indices || [])
      .slice()
      .sort((a: any, b: any) => {
        const order: Record<string, number> = { simple: 1, moyen: 2, niche: 3 };
        return (order[a.difficulte] || 99) - (order[b.difficulte] || 99);
      })
  );

  // 🔓 Calcul du nombre d'indices débloqués (1 au début, +1 tous les 3 échecs)
  let unlockedIndicesCount = $derived(
    hasWon
      ? sortedIndices.length
      : Math.min(sortedIndices.length, 1 + Math.floor(guesses.length / 3))
  );

  // ⏳ Calcul du nombre d'essais restants avant le prochain indice
  let essaisAvantProchain = $derived(
    3 - (guesses.length % 3)
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
          if (!errorMessage) await checkAlreadyPlayed();
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

    // 1. Récupération de la snapshot du jour
    let snapPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, indices').eq('date_jour', today);
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
          const retrySnapshotsPromise = supabase.from('joueur_daily_caracteristique').select('id_compte, pseudo, indices').eq('date_jour', today);
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
        id: v.id_compte, pseudo: v.pseudo, indices: v.indices
      }));
    }

    // 2. Vérification de la cible historique
    let histPromise = supabase.from('historique_cibles').select('id_compte, indices').eq('date_cible', today).eq('type_jeu', 'anecdotes').maybeSingle();
    let { data: histData } = await Promise.race([histPromise, timeoutPromise]);

    // 3. Tirage d'une nouvelle cible si besoin
    if (!histData && allViewers.length > 0 && isComponentMounted) {
      const eligibleTargets = allViewers.filter(v => v.indices && v.indices.length > 0);

      if (eligibleTargets.length === 0) {
        errorMessage = "Impossible de lancer le jeu : aucun joueur n'a configuré d'indices secrets dans son profil !";
        return;
      }

      const randomIndex = Math.floor(Math.random() * eligibleTargets.length);
      const randomTarget = eligibleTargets[randomIndex];

      const insertTargetPromise = supabase.from('historique_cibles').insert({
        date_cible: today,
        id_compte: randomTarget.id,
        type_jeu: 'anecdotes',
        indices: randomTarget.indices
      });

      const { error: insertErr } = await Promise.race([insertTargetPromise, timeoutPromise]);

      if (insertErr) {
        const retryDataPromise = supabase.from('historique_cibles').select('id_compte, indices').eq('date_cible', today).eq('type_jeu', 'anecdotes').maybeSingle();
        const { data: retryData } = await Promise.race([retryDataPromise, timeoutPromise]);
        histData = retryData;
      } else {
        histData = { id_compte: randomTarget.id, indices: randomTarget.indices };
      }
    }

    // 4. Initialisation
    if (histData && isComponentMounted) {
      const targetSnapshot = allViewers.find(v => v.id === histData.id_compte);
      targetViewer = {
        id: histData.id_compte,
        pseudo: targetSnapshot ? targetSnapshot.pseudo : 'Joueur Mystère',
        indices: histData.indices || []
      };
    }
  }

  // 🔄 Chargement de l'état persistant depuis la table historique_proposition et vérification des autres modes
  async function checkAlreadyPlayed() {
    if (!isComponentMounted) return;
    try {
      const sessionPromise = supabase.auth.getSession();
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout verif")), 3000));
      const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

      if (session && targetViewer) {
        const today = getLocalToday();

        // NOUVEAU : On vérifie s'il a déjà gagné le mode Classique OU le mode Émote aujourd'hui
        const { data: playedGames } = await supabase.from('historique')
          .select('type_jeu')
          .eq('id_compte', session.user.id)
          .eq('date_partie', today)
          .in('type_jeu', ['viewerdl', 'emote']);

        if (playedGames) {
          hasPlayedClassic = playedGames.some((g: any) => g.type_jeu === 'viewerdl');
          hasPlayedEmote = playedGames.some((g: any) => g.type_jeu === 'emote');
        }

        // Vérification de la persistance des essais sur le mode Anecdotes
        const propsCheckPromise = supabase.from('historique_proposition')
          .select('id_proposition, is_correct, tentative_num')
          .eq('id_joueur', session.user.id)
          .eq('type_jeu', 'anecdotes')
          .gte('created_at', `${today}T00:00:00`)
          .order('tentative_num', { ascending: false });

        const { data: pastPropositions } = await Promise.race([propsCheckPromise, timeoutPromise]);

        if (pastPropositions && pastPropositions.length > 0 && isComponentMounted) {

          let restoredGuesses = [];
          let foundVictory = false;
          let maxTentatives = pastPropositions[0].tentative_num;

          for (const prop of pastPropositions) {
            const viewer = allViewers.find(v => v.id === prop.id_proposition);
            if (viewer) {
              restoredGuesses.push({ id: viewer.id, pseudo: viewer.pseudo, isCorrect: prop.is_correct });
              if (prop.is_correct) foundVictory = true;
            }
          }

          guesses = restoredGuesses;

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

  // 📊 Sauvegarde de chaque essai dans la base d'analytics
  async function handleGuess(viewer: any) {
    searchQuery = ''; showSuggestions = false;

    const isCorrect = viewer.id === targetViewer.id;

    // Mise à jour visuelle instantanée
    guesses = [{ id: viewer.id, pseudo: viewer.pseudo, isCorrect }, ...guesses];

    if (isCorrect) {
      hasWon = true;
      victoryInfo = { tentatives: guesses.length, pseudo: targetViewer.pseudo };
      confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });
    }

    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (session) {

        // 1. Enregistrement analytique
        supabase.from('historique_proposition').insert({
          id_joueur: session.user.id,
          id_proposition: viewer.id,
          is_correct: isCorrect,
          tentative_num: guesses.length,
          type_jeu: 'anecdotes'
        }).then(({error}) => {
           if (error) console.warn("Erreur sauvegarde proposition analytique", error);
        });

        // 2. Enregistrement victoire globale si trouvé
        if (isCorrect) {
          supabase.from('historique').insert({
            id_compte: session.user.id,
            victoire: true,
            tentatives: guesses.length,
            type_jeu: 'anecdotes'
          }).then(({error}) => {
             if (error) console.warn("Erreur sauvegarde victoire", error);
          });
        }
      }
    } catch (err) {
      console.warn("Impossible de sauvegarder l'historique.", err);
    }
  }
</script>

<div class="w-full min-h-[80vh] p-4 md:p-10 flex flex-col items-center text-white pb-20">
  <h1 class="text-4xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-amber-300 to-rose-400 mb-2">Anecdotes</h1>
  <p class="text-amber-200/50 uppercase tracking-[0.2em] text-xs font-bold mb-8">Démasquez la cible grâce à ses secrets</p>

  {#if errorMessage}
    <div class="p-6 bg-rose-500/20 border border-rose-500 rounded-xl text-rose-300 font-bold max-w-xl text-center">{errorMessage}</div>
  {:else if isLoading}
    <p class="text-amber-300 animate-pulse uppercase tracking-widest mt-10">Récupération des dossiers confidentiels...</p>
  {:else if !targetViewer}
    <p class="text-rose-400 uppercase tracking-widest mt-10 text-center">Aucune cible initialisée aujourd'hui.<br/>Revenez plus tard.</p>
  {:else}

    {#if !hasWon}
      <!-- BARRE DE RECHERCHE -->
      <div class="w-full max-w-md relative mb-12 z-30">
        <input type="text" bind:value={searchQuery} onfocus={() => showSuggestions = true} placeholder="Entrez le pseudo suspecté..." class="w-full bg-slate-900 border border-amber-500/30 rounded-2xl p-4 text-white outline-none focus:border-amber-400 shadow-[0_0_15px_rgba(245,158,11,0.1)] transition-all" />
        {#if showSuggestions}
          <div class="absolute w-full bg-slate-900 border border-amber-500/20 rounded-xl mt-2 overflow-hidden shadow-2xl">
            {#each filteredViewers as v}
              <button onclick={() => handleGuess(v)} class="w-full p-4 text-left hover:bg-amber-500/20 font-bold transition-colors cursor-pointer">{v.pseudo}</button>
            {/each}
          </div>
        {/if}
      </div>
    {:else}
      <!-- MESSAGE DE VICTOIRE AVEC BOUTONS DYNAMIQUES -->
      <div class="mb-12 p-8 bg-emerald-500/10 border border-emerald-500/50 rounded-3xl text-center shadow-[0_0_30px_rgba(16,185,129,0.1)] animate-fade-in flex flex-col items-center">
        <h2 class="text-3xl font-black text-emerald-300 uppercase tracking-widest mb-4">Dossier classé !</h2>
        <p class="text-emerald-100 text-lg mb-8">
          Vous avez démasqué <span class="font-black text-emerald-400">{victoryInfo?.pseudo}</span>
          en <span class="font-black text-emerald-400">{victoryInfo?.tentatives} {victoryInfo?.tentatives && victoryInfo.tentatives > 1 ? 'tentatives' : 'tentative'}</span>.
        </p>

        <!-- CASCADE DE BOUTONS -->
        {#if !hasPlayedClassic}
          <button
            onclick={() => goto('/jouer/classique')}
            class="bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest text-sm px-8 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.15)] hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] hover:-translate-y-1 cursor-pointer flex items-center gap-3"
          >
            <span class="text-xl">🔍</span> Enchaîner avec le mode Classique
          </button>
        {:else if !hasPlayedEmote}
          <button
            onclick={() => goto('/jouer/emote')}
            class="bg-fuchsia-500/10 border-2 border-fuchsia-500/40 hover:bg-fuchsia-500/20 text-fuchsia-300 font-black uppercase tracking-widest text-sm px-8 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(217,70,239,0.15)] hover:shadow-[0_0_25px_rgba(217,70,239,0.3)] hover:-translate-y-1 cursor-pointer flex items-center gap-3"
          >
            <span class="text-xl">🖼️</span> Enchaîner avec le mode Émotes
          </button>
        {:else}
          <button
            onclick={() => goto('/jouer')}
            class="bg-slate-800 border-2 border-slate-700 hover:bg-slate-700 text-slate-300 font-black uppercase tracking-widest text-sm px-8 py-4 rounded-xl transition-all cursor-pointer flex items-center gap-3"
          >
            <span class="text-xl">🏠</span> Retour aux modes de jeu
          </button>
        {/if}
      </div>
    {/if}

    <!-- AFFICHAGE DES INDICES -->
    <div class="w-full max-w-2xl flex flex-col gap-4 mb-12">
      {#each sortedIndices as indice, i}
        {#if i < unlockedIndicesCount}
          <div class="flex items-center gap-4 bg-slate-900/80 backdrop-blur-md p-6 rounded-3xl border border-amber-500/30 shadow-[0_0_20px_rgba(245,158,11,0.05)] animate-fade-in">
            <div class="w-12 h-12 rounded-full bg-amber-500/10 border border-amber-500/30 flex items-center justify-center shrink-0">
              <span class="text-amber-400 font-black text-xl">#{i + 1}</span>
            </div>
            <div>
              <span class="px-2 py-0.5 rounded text-[9px] font-black uppercase tracking-widest border mb-2 inline-block
                  {indice.difficulte === 'simple' ? 'bg-emerald-500/10 text-emerald-400 border-emerald-500/40' :
                   indice.difficulte === 'moyen' ? 'bg-amber-500/10 text-amber-400 border-amber-500/40' :
                   'bg-rose-500/10 text-rose-400 border-rose-500/40'}">
                  {indice.difficulte}
              </span>
              <p class="text-amber-100 text-lg leading-relaxed">{indice.texte}</p>
            </div>
          </div>
        {/if}
      {/each}

      <!-- CADENAS POUR LE PROCHAIN INDICE -->
      {#if !hasWon && unlockedIndicesCount < sortedIndices.length}
        <div class="flex items-center justify-center gap-3 bg-slate-900/40 p-5 rounded-3xl border border-dashed border-slate-600/50">
          <span class="text-slate-500 text-xl">🔒</span>
          <p class="text-slate-400 text-xs font-bold uppercase tracking-widest text-center">
            Prochain indice débloqué dans <span class="text-amber-400 font-black text-base mx-1">{essaisAvantProchain}</span> essai{essaisAvantProchain > 1 ? 's' : ''}
          </p>
        </div>
      {/if}

      {#if !hasWon && unlockedIndicesCount === sortedIndices.length}
        <p class="text-center text-rose-400/60 text-xs font-bold uppercase tracking-widest mt-4">
          Tous les indices sont révélés. À vous de jouer !
        </p>
      {/if}
    </div>

    <!-- HISTORIQUE DES TENTATIVES -->
    {#if guesses.length > 0 && !hasWon}
      <div class="w-full max-w-md mt-4">
        <h3 class="text-[10px] font-black uppercase tracking-widest text-slate-500 mb-4 text-center">Historique des pistes écartées</h3>
        <div class="flex flex-col gap-2">
          {#each guesses as guess, i (guess.id + i)}
            <div class="flex items-center justify-between p-3 bg-rose-500/5 border border-rose-500/20 rounded-xl animate-fade-in flip-card">
              <span class="text-rose-200 font-bold text-sm line-clamp-1">{guess.pseudo}</span>
              <span class="text-rose-500/50 text-xs uppercase tracking-widest font-black">
                {guess.isCorrect ? 'Démasqué' : 'Erreur'}
              </span>
            </div>
          {/each}
        </div>
      </div>
    {/if}

  {/if}
</div>

<style>
  @keyframes flipIn {
    0% { transform: perspective(400px) rotateX(90deg); opacity: 0; }
    100% { transform: perspective(400px) rotateX(0deg); opacity: 1; }
  }

  .flip-card {
    animation: flipIn 0.4s ease-out forwards;
  }
</style>