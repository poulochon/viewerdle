<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import confetti from 'canvas-confetti';

  let isLoading = $state(true);
  let hasWon = $state(false);
  let targetEmote = $state<any>(null);
  let allEmotes = $state<any[]>([]);
  let guesses = $state<any[]>([]);
  let searchQuery = $state('');
  let showSuggestions = $state(false);
  let victoryInfo = $state<{tentatives: number, nom: string} | null>(null);
  let errorMessage = $state('');

  let isComponentMounted = false;

  // 🌫️ Calcul dynamique du flou (Commence à 25px, diminue de 5px à chaque essai)
  let blurAmount = $derived(
    hasWon ? 0 : Math.max(0, 25 - (guesses.length * 5))
  );

  let filteredEmotes = $derived(
    searchQuery.trim() === ''
      ? []
      : allEmotes.filter(e => e.nom.toLowerCase().includes(searchQuery.toLowerCase()) && !guesses.some(g => g.id === e.id))
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
        const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout réseau")), 4000));
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

    // 1. Récupération des émotes
    const emotesPromise = supabase.from('emotes_disponibles').select('id, nom, url_image').eq('actif', true);
    const { data: emotesData, error: emotesErr } = await Promise.race([emotesPromise, timeoutPromise]);

    if (emotesErr || !emotesData || emotesData.length === 0) {
      errorMessage = "Impossible de charger la base de données des émotes.";
      return;
    }

    if (isComponentMounted) {
      allEmotes = emotesData;
    }

    // 2. Vérification de la cible historique (avec la colonne id_emote)
    let histPromise = supabase.from('historique_cibles').select('id_emote').eq('date_cible', today).eq('type_jeu', 'emote').maybeSingle();
    let { data: histData } = await Promise.race([histPromise, timeoutPromise]);

    // 3. Tirage aléatoire (Automatique au chargement)
    if (!histData && allEmotes.length > 0 && isComponentMounted) {
      const randomIndex = Math.floor(Math.random() * allEmotes.length);
      const randomTarget = allEmotes[randomIndex];

      const insertTargetPromise = supabase.from('historique_cibles').insert({
        date_cible: today,
        id_emote: randomTarget.id,
        type_jeu: 'emote'
      });

      const { error: insertErr } = await Promise.race([insertTargetPromise, timeoutPromise]);

      if (insertErr) {
        console.warn("Erreur d'initialisation cible", insertErr);
        const retryDataPromise = supabase.from('historique_cibles').select('id_emote').eq('date_cible', today).eq('type_jeu', 'emote').maybeSingle();
        const { data: retryData } = await Promise.race([retryDataPromise, timeoutPromise]);
        histData = retryData;
      } else {
        histData = { id_emote: randomTarget.id };
      }
    }

    // 4. On charge la cible
    if (histData && isComponentMounted) {
      targetEmote = allEmotes.find(e => e.id === histData.id_emote);
      if (!targetEmote) {
        errorMessage = "Cible introuvable dans la base de données actuelle.";
      }
    }
  }

  async function checkAlreadyPlayed() {
    if (!isComponentMounted) return;
    try {
      const sessionPromise = supabase.auth.getSession();
      const timeoutPromise = new Promise<any>((_, reject) => setTimeout(() => reject(new Error("Timeout verif")), 3000));
      const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

      if (session && targetEmote) {
        const today = getLocalToday();

        const histCheckPromise = supabase.from('historique').select('tentatives').eq('id_compte', session.user.id).eq('date_partie', today).eq('type_jeu', 'emote').maybeSingle();

        const propsCheckPromise = supabase.from('historique_proposition')
          .select('id_emote_proposition, is_correct, tentative_num')
          .eq('id_joueur', session.user.id)
          .eq('type_jeu', 'emote')
          .gte('created_at', `${today}T00:00:00`)
          .order('tentative_num', { ascending: false });

        const [{ data: histData }, { data: pastPropositions }] = await Promise.all([
          Promise.race([histCheckPromise, timeoutPromise]).catch(() => ({ data: null })),
          Promise.race([propsCheckPromise, timeoutPromise]).catch(() => ({ data: null }))
        ]);

        if (!isComponentMounted) return;

        if (pastPropositions && pastPropositions.length > 0) {
          let restoredGuesses = [];
          for (const prop of pastPropositions) {
            const emote = allEmotes.find(e => e.id === prop.id_emote_proposition);
            if (emote) {
              restoredGuesses.push({ id: emote.id, nom: emote.nom, url_image: emote.url_image, isCorrect: prop.is_correct });
            }
          }
          guesses = restoredGuesses;
        }

        if (histData) {
          hasWon = true;
          victoryInfo = { tentatives: histData.tentatives, nom: targetEmote.nom };
        } else if (pastPropositions && pastPropositions.some(p => p.is_correct)) {
          hasWon = true;
          victoryInfo = { tentatives: pastPropositions[0].tentative_num, nom: targetEmote.nom };
        }
      }
    } catch (error) {
      console.warn("Vérification historique annulée suite à une instabilité réseau.", error);
    }
  }

  async function handleGuess(emote: any) {
    searchQuery = ''; showSuggestions = false;

    const isCorrect = emote.id === targetEmote.id;

    guesses = [{ id: emote.id, nom: emote.nom, url_image: emote.url_image, isCorrect }, ...guesses];

    if (isCorrect) {
      hasWon = true;
      victoryInfo = { tentatives: guesses.length, nom: targetEmote.nom };
      confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });
    }

    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (session) {
        supabase.from('historique_proposition').insert({
          id_joueur: session.user.id,
          id_emote_proposition: emote.id,
          is_correct: isCorrect,
          tentative_num: guesses.length,
          type_jeu: 'emote'
        }).then(({error}) => {
           if (error) console.warn("Erreur sauvegarde proposition analytique", error);
        });

        if (isCorrect) {
          supabase.from('historique').insert({
            id_compte: session.user.id,
            victoire: true,
            tentatives: guesses.length,
            type_jeu: 'emote'
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
  <h1 class="text-4xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-fuchsia-400 to-pink-500 mb-2">Devine l'Émote</h1>
  <p class="text-fuchsia-200/50 uppercase tracking-[0.2em] text-xs font-bold mb-8">L'image se dévoile à chaque erreur</p>

  {#if errorMessage}
    <div class="p-6 bg-rose-500/20 border border-rose-500 rounded-xl text-rose-300 font-bold max-w-xl text-center">{errorMessage}</div>
  {:else if isLoading}
    <p class="text-fuchsia-300 animate-pulse uppercase tracking-widest mt-10">Chargement de la bibliothèque visuelle...</p>
  {:else if !targetEmote}
    <p class="text-rose-400 uppercase tracking-widest mt-10 text-center">En attente de la cible... Veuillez rafraîchir.</p>
  {:else}

    <div class="mb-10 p-4 bg-slate-900/60 backdrop-blur-md border border-fuchsia-500/30 rounded-3xl shadow-[0_0_30px_rgba(217,70,239,0.15)] animate-fade-in relative overflow-hidden group">

      {#if !hasWon}
        <div class="absolute inset-0 bg-[url('data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNDAiIGhlaWdodD0iNDAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PGRlZnM+PHBhdHRlcm4gaWQ9ImdyaWQiIHdpZHRoPSI0MCIgaGVpZ2h0PSI0MCIgcGF0dGVyblVuaXRzPSJ1c2VyU3BhY2VPblVzZSI+PHBhdGggZD0iTSA0MCAwIEwgMCAwIDAgNDAiIGZpbGw9Im5vbmUiIHN0cm9rZT0icmdiYSgyNTUsMjU1LDI1NSwwLjA1KSIgc3Ryb2tlLXdpZHRoPSIxIi8+PC9wYXR0ZXJuPjwvZGVmcz48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSJ1cmwoI2dyaWQpIiAvPjwvc3ZnPg==')] pointer-events-none opacity-50 z-10"></div>
      {/if}

      <div class="w-48 h-48 md:w-64 md:h-64 rounded-2xl overflow-hidden flex items-center justify-center bg-slate-950 relative">
        {#if targetEmote?.url_image}
          <img
            src={targetEmote.url_image}
            alt="Émote mystère"
            class="w-full h-full object-contain p-4"
            style="filter: blur({blurAmount}px) {hasWon ? 'none' : 'grayscale(30%)'}; transition: filter 0.8s ease-in-out;"
          />
        {:else}
          <span class="text-fuchsia-500/50 text-4xl">?</span>
        {/if}
      </div>

      {#if !hasWon}
        <div class="absolute bottom-2 right-4 z-20 text-[10px] font-black uppercase tracking-widest text-fuchsia-300/50 bg-slate-950/80 px-3 py-1 rounded-full border border-fuchsia-500/20 backdrop-blur-md">
          Flou : {blurAmount}px
        </div>
      {/if}
    </div>

    {#if !hasWon}
      <div class="w-full max-w-md relative mb-12 z-30">
        <input type="text" bind:value={searchQuery} onfocus={() => showSuggestions = true} placeholder="Entrez le nom de l'émote..." class="w-full bg-slate-900 border border-fuchsia-500/30 rounded-2xl p-4 text-white outline-none focus:border-fuchsia-400 shadow-[0_0_15px_rgba(217,70,239,0.1)] transition-all" />
        {#if showSuggestions}
          <div class="absolute w-full bg-slate-900 border border-fuchsia-500/20 rounded-xl mt-2 overflow-hidden shadow-2xl max-h-60 overflow-y-auto custom-scrollbar">
            {#each filteredEmotes as e}
              <button onclick={() => handleGuess(e)} class="w-full p-4 text-left hover:bg-fuchsia-500/20 transition-colors cursor-pointer flex items-center">
                <span class="font-bold text-sm tracking-wide text-indigo-100">{e.nom}</span>
              </button>
            {/each}
          </div>
        {/if}
      </div>
    {:else}
      <div class="mb-12 p-8 bg-emerald-500/10 border border-emerald-500/50 rounded-3xl text-center shadow-[0_0_30px_rgba(16,185,129,0.1)] animate-fade-in flex flex-col items-center">
        <h2 class="text-3xl font-black text-emerald-300 uppercase tracking-widest mb-4">Émote décodée !</h2>
        <p class="text-emerald-100 text-lg mb-8">
          C'était bien l'émote <span class="font-black text-emerald-400">{victoryInfo?.nom}</span> ! Tu as trouvé
          en <span class="font-black text-emerald-400">{victoryInfo?.tentatives} {victoryInfo?.tentatives && victoryInfo.tentatives > 1 ? 'tentatives' : 'tentative'}</span>.
        </p>

        <button
          onclick={() => goto('/jouer')}
          class="bg-slate-800 border-2 border-slate-700 hover:bg-slate-700 text-slate-300 font-black uppercase tracking-widest text-sm px-8 py-4 rounded-xl transition-all cursor-pointer flex items-center gap-3"
        >
          <span class="text-xl">🏠</span> Retour aux modes de jeu
        </button>
      </div>
    {/if}

    {#if guesses.length > 0 && !hasWon}
      <div class="w-full max-w-md mt-4">
        <h3 class="text-[10px] font-black uppercase tracking-widest text-slate-500 mb-4 text-center">Tentatives précédentes</h3>
        <div class="flex flex-col gap-2">
          {#each guesses as guess, i (guess.id + i)}
            <div class="flex items-center justify-between p-3 bg-slate-900/50 border border-slate-700/50 rounded-xl animate-fade-in flip-card">
              <div class="flex items-center gap-4">
                <img src={guess.url_image} alt={guess.nom} class="w-6 h-6 object-contain grayscale opacity-70" />
                <span class="text-slate-300 font-bold text-sm">{guess.nom}</span>
              </div>
              <span class="text-rose-500/50 text-xs uppercase tracking-widest font-black">
                {guess.isCorrect ? 'Trouvé' : 'Faux'}
              </span>
            </div>
          {/each}
        </div>
      </div>
    {/if}

  {/if}
</div>

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #d946ef; border-radius: 4px; }

  @keyframes flipIn {
    0% { transform: perspective(400px) rotateX(90deg); opacity: 0; }
    100% { transform: perspective(400px) rotateX(0deg); opacity: 1; }
  }

  .flip-card {
    animation: flipIn 0.4s ease-out forwards;
  }
</style>