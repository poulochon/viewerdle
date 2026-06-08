<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { page } from '$app/stores';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import confetti from 'canvas-confetti';

  const gameId = $page.params.id;

  let isLoading = $state(true);
  let currentUser = $state<any>(null);
  let allViewers = $state<any[]>([]);
  let game = $state<any>(null);
  let errorMessage = $state('');

  let gameChannel: any;
  let isComponentMounted = false;

  // --- ÉTATS LOCAUX (Plateau du joueur) ---
  let boardState = $state<Record<string, string>>({}); // 'neutral', 'eliminated', 'supposed'
  let myTargetHidden = $state(false);
  let chatInput = $state('');

  let isProposingMode = $state(false);
  let propositionTarget = $state<string | null>(null);

  // --- ANIMATION POKÉBALL ---
  let isAnimating = $state(false);
  let animationData = $state<any>(null);
  let animationPhase = $state<'pulse' | 'result'>('pulse');
  let lastAnimatedActionId = 0;

  // --- DÉDUCTIONS RÉACTIVES ---
  let isPlayer1 = $derived(currentUser?.id === game?.joueur1_id);
  let isMyTurn = $derived(game?.tour_joueur_id === currentUser?.id);
  let myChoiceId = $derived(isPlayer1 ? game?.choix_joueur1 : game?.choix_joueur2);
  let opponentChoiceId = $derived(isPlayer1 ? game?.choix_joueur2 : game?.choix_joueur1);
  let myTarget = $derived(allViewers.find(v => v.id === myChoiceId));
  let opponentTarget = $derived(allViewers.find(v => v.id === opponentChoiceId)); // Visible uniquement à la fin

  // Logique du tour par tour
  let lastAction = $derived(game?.chat_actions?.[game.chat_actions.length - 1]);
  let needsToAnswer = $derived(lastAction?.type === 'msg' && lastAction?.sender_id !== currentUser?.id && !lastAction?.response);

  onMount(() => {
    isComponentMounted = true;
    initGame();

    return () => {
      isComponentMounted = false;
      if (gameChannel) supabase.removeChannel(gameChannel);
    };
  });

  // Gérer l'animation à chaque mise à jour du chat
  $effect(() => {
    if (game?.chat_actions?.length > 0) {
      const latest = game.chat_actions[game.chat_actions.length - 1];
      if (latest.type === 'proposition' && latest.id !== lastAnimatedActionId) {
        lastAnimatedActionId = latest.id;
        triggerPokeballAnimation(latest);
      }
    }
  });

  function triggerPokeballAnimation(data: any) {
    isAnimating = true;
    animationData = data;
    animationPhase = 'pulse';

    // 3 battements de coeur (3 secondes) puis le résultat
    setTimeout(() => {
      animationPhase = 'result';
      if (data.is_correct && data.sender_id === currentUser.id) {
        confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 } });
      }
      // On ferme l'overlay après 3 secondes de résultat
      setTimeout(() => {
        isAnimating = false;
      }, 3000);
    }, 3000);
  }

  async function initGame() {
    isLoading = true;
    try {
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) { goto('/jouer/versus'); return; }
      currentUser = session.user;

      const { data: players } = await supabase.from('profil_viewer').select('id, pseudo').order('pseudo');
      allViewers = players || [];

      await fetchGameState();
      setupRealtime();
    } catch (e) {
      errorMessage = "Erreur de connexion à la partie.";
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  async function fetchGameState() {
    if (!isComponentMounted) return;
    const { data, error } = await supabase.from('versus_parties').select('*').eq('id', gameId).single();
    if (error || !data) { goto('/jouer/versus'); return; }
    game = data;
  }

  function setupRealtime() {
    gameChannel = supabase.channel(`game_${gameId}`)
      .on('postgres_changes', { event: 'UPDATE', schema: 'public', table: 'versus_parties', filter: `id=eq.${gameId}` },
      (payload) => {
        game = payload.new;
      }).subscribe();
  }

  // --- ACTIONS DE JEU ---

  function toggleBoardState(id: string) {
    const current = boardState[id] || 'neutral';
    if (current === 'neutral') boardState[id] = 'eliminated';
    else if (current === 'eliminated') boardState[id] = 'supposed';
    else boardState[id] = 'neutral';
  }

  async function selectViewer(viewerId: string) {
    try {
      const updateField = isPlayer1 ? { choix_joueur1: viewerId } : { choix_joueur2: viewerId };
      await supabase.from('versus_parties').update(updateField).eq('id', gameId);

      // On re-fetch pour voir si l'autre a aussi choisi
      const { data } = await supabase.from('versus_parties').select('*').eq('id', gameId).single();

      if (data.choix_joueur1 && data.choix_joueur2 && data.status === 'selecting') {
        // Les deux ont choisi, on lance la partie ! Tirage au sort du premier joueur
        const firstTurn = Math.random() > 0.5 ? data.joueur1_id : data.joueur2_id;
        await supabase.from('versus_parties').update({ status: 'playing', tour_joueur_id: firstTurn }).eq('id', gameId);
      }
    } catch (e) { console.warn(e); }
  }

  async function askQuestion() {
    if (!chatInput.trim()) return;
    const text = chatInput.trim();
    chatInput = '';

    const newAction = { id: Date.now(), type: 'msg', sender_id: currentUser.id, content: text, response: null };
    const actions = [...(game.chat_actions || []), newAction];
    const nextTurn = isPlayer1 ? game.joueur2_id : game.joueur1_id;

    await supabase.from('versus_parties').update({ chat_actions: actions, tour_joueur_id: nextTurn }).eq('id', gameId);
  }

  async function answerQuestion(responseStr: string) {
    const actions = [...(game.chat_actions || [])];
    actions[actions.length - 1].response = responseStr;
    // On répond, et ça reste NOTRE tour pour poser une question ou proposer
    await supabase.from('versus_parties').update({ chat_actions: actions }).eq('id', gameId);
  }

  async function makeProposition() {
    if (!propositionTarget) return;

    const targetViewer = allViewers.find(v => v.id === propositionTarget);
    const isCorrect = opponentChoiceId === propositionTarget;

    const newAction = {
      id: Date.now(),
      type: 'proposition',
      sender_id: currentUser.id,
      target_pseudo: targetViewer?.pseudo,
      is_correct: isCorrect
    };

    const actions = [...(game.chat_actions || []), newAction];
    isProposingMode = false;
    propositionTarget = null;

    if (isCorrect) {
      await supabase.from('versus_parties').update({ chat_actions: actions, status: 'finished', gagnant_id: currentUser.id }).eq('id', gameId);
    } else {
      const nextTurn = isPlayer1 ? game.joueur2_id : game.joueur1_id;
      await supabase.from('versus_parties').update({ chat_actions: actions, tour_joueur_id: nextTurn }).eq('id', gameId);
    }
  }

  function getSenderName(senderId: string) {
    return senderId === currentUser.id ? 'Vous' : 'Adversaire';
  }
</script>

{#if isAnimating && animationData}
  <div class="fixed inset-0 z-50 flex items-center justify-center bg-slate-950/90 backdrop-blur-md">
    <div class="flex flex-col items-center text-center">

      {#if animationPhase === 'pulse'}
        <div class="w-32 h-32 rounded-full border-8 border-slate-700 bg-slate-800 flex items-center justify-center animate-pokeball relative overflow-hidden shadow-[0_0_50px_rgba(255,255,255,0.1)]">
          <div class="absolute top-0 w-full h-1/2 bg-rose-500"></div>
          <div class="absolute bottom-0 w-full h-1/2 bg-white"></div>
          <div class="absolute w-full h-2 bg-slate-800"></div>
          <div class="w-10 h-10 rounded-full border-4 border-slate-800 bg-white z-10 flex items-center justify-center">
            <div class="w-4 h-4 rounded-full bg-slate-800 animate-pulse"></div>
          </div>
        </div>
        <h2 class="text-3xl font-black text-white uppercase tracking-widest mt-8 animate-pulse">
          {getSenderName(animationData.sender_id)} propose {animationData.target_pseudo}...
        </h2>

      {:else}
        {#if animationData.is_correct}
          <div class="w-32 h-32 rounded-full bg-emerald-500 flex items-center justify-center shadow-[0_0_80px_rgba(16,185,129,0.5)] animate-bounce">
            <span class="text-6xl">✅</span>
          </div>
          <h2 class="text-5xl font-black text-emerald-400 uppercase tracking-widest mt-8 drop-shadow-lg">C'est GAGNÉ !</h2>
          <p class="text-emerald-200 mt-4 text-xl">L'identité a été découverte !</p>
        {:else}
          <div class="w-32 h-32 rounded-full bg-rose-500 flex items-center justify-center shadow-[0_0_80px_rgba(244,63,94,0.5)] animate-shake">
            <span class="text-6xl">❌</span>
          </div>
          <h2 class="text-5xl font-black text-rose-400 uppercase tracking-widest mt-8 drop-shadow-lg">C'est FAUX !</h2>
          <p class="text-rose-200 mt-4 text-xl">La partie continue...</p>
        {/if}
      {/if}
    </div>
  </div>
{/if}

<div class="w-full max-w-7xl mx-auto p-4 flex flex-col text-white h-[90vh]">

  {#if isLoading || !game}
    <div class="flex-1 flex items-center justify-center">
      <p class="text-indigo-400 animate-pulse uppercase tracking-widest font-bold">Synchronisation de la table...</p>
    </div>

  {:else if game.status === 'selecting'}
    <div class="text-center mb-8 animate-fade-in mt-10">
      <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-blue-400 to-purple-400 mb-2">
        Choisissez votre cible
      </h1>
      <p class="text-indigo-300/60 uppercase tracking-widest text-xs font-bold">
        {#if myChoiceId}
          En attente du choix de l'adversaire...
        {:else}
          Sélectionnez la personne que votre adversaire devra deviner
        {/if}
      </p>
    </div>

    <div class="grid grid-cols-2 md:grid-cols-4 lg:grid-cols-6 gap-3 overflow-y-auto custom-scrollbar p-2 pb-20">
      {#each allViewers as viewer}
        <button
          disabled={myChoiceId !== undefined && myChoiceId !== null}
          onclick={() => selectViewer(viewer.id)}
          class="p-4 rounded-2xl border-2 transition-all cursor-pointer font-bold text-sm
            {myChoiceId === viewer.id ? 'bg-indigo-500/30 border-indigo-400 text-white scale-105 shadow-[0_0_20px_rgba(99,102,241,0.3)]' :
             myChoiceId ? 'bg-slate-900/40 border-slate-700/50 text-slate-500 opacity-50 cursor-not-allowed' :
             'bg-slate-900/80 border-indigo-500/20 text-indigo-200 hover:border-indigo-400 hover:bg-slate-800'}"
        >
          {viewer.pseudo}
        </button>
      {/each}
    </div>

  {:else if game.status === 'playing' || game.status === 'finished'}

    <div class="flex flex-col lg:flex-row gap-6 h-full overflow-hidden animate-fade-in">

      <div class="w-full lg:w-1/3 flex flex-col bg-slate-900/80 backdrop-blur-md border border-indigo-500/30 rounded-3xl overflow-hidden shadow-xl">

        <div class="p-4 border-b border-indigo-500/20 bg-slate-950/50 text-center">
          {#if game.status === 'finished'}
            <span class="text-emerald-400 font-black uppercase tracking-widest text-sm">Partie Terminée</span>
          {:else if isMyTurn}
            <span class="text-amber-400 font-black uppercase tracking-widest text-sm animate-pulse">C'est à votre tour !</span>
          {:else}
            <span class="text-indigo-300/60 font-black uppercase tracking-widest text-xs">Tour de l'adversaire...</span>
          {/if}
        </div>

        <div class="flex-1 overflow-y-auto p-4 flex flex-col gap-3 custom-scrollbar">
          {#each game.chat_actions || [] as action}
            {#if action.type === 'msg'}
              <div class="flex flex-col {action.sender_id === currentUser.id ? 'items-end' : 'items-start'}">
                <span class="text-[9px] text-slate-500 font-bold uppercase tracking-widest mb-1 px-1">
                  {getSenderName(action.sender_id)}
                </span>
                <div class="px-4 py-2 rounded-2xl max-w-[85%] text-sm {action.sender_id === currentUser.id ? 'bg-indigo-500/20 border border-indigo-500/30 text-indigo-100 rounded-tr-sm' : 'bg-slate-800 border border-slate-700 text-slate-200 rounded-tl-sm'}">
                  {action.content}
                </div>
                {#if action.response}
                  <div class="mt-1 px-3 py-1 rounded-xl text-xs font-black uppercase tracking-widest border
                    {action.response === 'Oui' ? 'bg-emerald-500/10 text-emerald-400 border-emerald-500/30' :
                     action.response === 'Non' ? 'bg-rose-500/10 text-rose-400 border-rose-500/30' :
                     'bg-slate-500/10 text-slate-400 border-slate-500/30'}">
                    Réponse : {action.response}
                  </div>
                {/if}
              </div>
            {:else if action.type === 'proposition'}
              <div class="w-full text-center p-3 rounded-xl border border-dashed {action.is_correct ? 'bg-emerald-500/10 border-emerald-500/40 text-emerald-300' : 'bg-rose-500/10 border-rose-500/40 text-rose-300'}">
                <span class="font-bold text-xs uppercase tracking-widest">
                  {getSenderName(action.sender_id)} a proposé {action.target_pseudo}
                </span>
                <br>
                <span class="text-sm font-black mt-1 block">
                  {action.is_correct ? 'C\'EST LA BONNE RÉPONSE ! ✅' : 'C\'EST FAUX ! ❌'}
                </span>
              </div>
            {/if}
          {/each}
        </div>

        {#if game.status === 'playing'}
          <div class="p-4 bg-slate-950/80 border-t border-indigo-500/20">
            {#if isMyTurn}
              {#if needsToAnswer}
                <div class="flex flex-col gap-2">
                  <span class="text-[10px] text-center text-amber-400 font-bold uppercase tracking-widest mb-1">Vous devez répondre :</span>
                  <div class="flex gap-2">
                    <button onclick={() => answerQuestion('Oui')} class="flex-1 bg-emerald-500/10 border border-emerald-500/40 hover:bg-emerald-500/20 text-emerald-400 font-black py-2 rounded-xl transition-all cursor-pointer">OUI</button>
                    <button onclick={() => answerQuestion('Non')} class="flex-1 bg-rose-500/10 border border-rose-500/40 hover:bg-rose-500/20 text-rose-400 font-black py-2 rounded-xl transition-all cursor-pointer">NON</button>
                    <button onclick={() => answerQuestion('Ne sais pas')} class="flex-1 bg-slate-500/10 border border-slate-500/40 hover:bg-slate-500/20 text-slate-400 font-black py-2 rounded-xl transition-all cursor-pointer">NSP</button>
                  </div>
                </div>
              {:else}
                {#if isProposingMode}
                  <div class="flex flex-col gap-2 animate-fade-in">
                    <span class="text-[10px] text-center text-rose-400 font-bold uppercase tracking-widest mb-1">Tenter une résolution</span>
                    <select bind:value={propositionTarget} class="w-full bg-slate-900 border border-rose-500/50 rounded-xl px-4 py-2 text-sm text-white outline-none cursor-pointer">
                      <option value={null} disabled selected>Sélectionnez un viewer...</option>
                      {#each allViewers as v}
                        <option value={v.id}>{v.pseudo}</option>
                      {/each}
                    </select>
                    <div class="flex gap-2 mt-1">
                      <button onclick={() => isProposingMode = false} class="flex-1 bg-slate-800 text-slate-400 text-xs font-bold py-2 rounded-xl cursor-pointer">Annuler</button>
                      <button onclick={makeProposition} disabled={!propositionTarget} class="flex-1 bg-rose-500/20 border border-rose-500/50 text-rose-300 font-black uppercase tracking-widest text-xs py-2 rounded-xl disabled:opacity-50 cursor-pointer">Lancer la Pokéball</button>
                    </div>
                  </div>
                {:else}
                  <div class="flex flex-col gap-2 animate-fade-in">
                    <input type="text" bind:value={chatInput} placeholder="Posez votre question..." class="w-full bg-slate-900 border border-indigo-500/30 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-indigo-400" onkeydown={(e) => e.key === 'Enter' && askQuestion()} />
                    <div class="flex gap-2">
                      <button onclick={askQuestion} disabled={!chatInput.trim()} class="flex-1 bg-indigo-500/20 border border-indigo-500/40 text-indigo-300 font-black uppercase tracking-widest text-xs py-2 rounded-xl disabled:opacity-50 cursor-pointer">Envoyer</button>
                      <button onclick={() => isProposingMode = true} class="flex-1 bg-rose-500/10 border border-rose-500/30 text-rose-400 font-black uppercase tracking-widest text-xs py-2 rounded-xl cursor-pointer">Proposer</button>
                    </div>
                  </div>
                {/if}
              {/if}
            {:else}
              <div class="text-center py-4 opacity-50 pointer-events-none">
                <span class="text-xs uppercase tracking-widest font-bold">Veuillez patienter...</span>
              </div>
            {/if}
          </div>
        {/if}
      </div>

      <div class="w-full lg:w-1/2 flex flex-col bg-slate-900/60 backdrop-blur-md border border-slate-700/50 rounded-3xl p-4 md:p-6 shadow-inner">
        <h2 class="text-xs font-black uppercase tracking-widest text-slate-400 mb-4 text-center">Votre Plateau de Déduction</h2>

        <div class="flex-1 overflow-y-auto custom-scrollbar pr-2">
          <div class="grid grid-cols-3 sm:grid-cols-4 gap-2">
            {#each allViewers as viewer}
              {@const state = boardState[viewer.id] || 'neutral'}
              <button
                onclick={() => toggleBoardState(viewer.id)}
                class="relative aspect-[3/1] flex items-center justify-center rounded-xl border text-xs font-bold transition-all cursor-pointer overflow-hidden
                  {state === 'neutral' ? 'bg-slate-800 border-slate-700 text-slate-300 hover:border-slate-500' :
                   state === 'eliminated' ? 'bg-rose-950/30 border-rose-900/50 text-rose-500/40 opacity-40 grayscale' :
                   'bg-emerald-900/30 border-emerald-500/50 text-emerald-300 shadow-[0_0_15px_rgba(16,185,129,0.15)]'}"
              >
                {#if state === 'eliminated'}
                  <div class="absolute inset-0 flex items-center justify-center">
                    <div class="w-full h-0.5 bg-rose-500/50 rotate-12"></div>
                    <div class="absolute w-full h-0.5 bg-rose-500/50 -rotate-12"></div>
                  </div>
                {/if}
                <span class="truncate px-1 z-10">{viewer.pseudo}</span>
              </button>
            {/each}
          </div>
        </div>
      </div>

      <div class="w-full lg:w-1/6 flex flex-col gap-4">

        <div class="bg-slate-900/80 border border-blue-500/30 rounded-3xl p-4 flex flex-col items-center text-center shadow-lg">
          <span class="text-[10px] font-black uppercase tracking-widest text-blue-400/70 mb-3">Votre Cible</span>
          <button
            onclick={() => myTargetHidden = !myTargetHidden}
            class="w-full aspect-square rounded-2xl border-2 border-dashed flex items-center justify-center text-center p-2 transition-all cursor-pointer
              {myTargetHidden ? 'bg-slate-950 border-slate-700 text-slate-500 hover:border-slate-500' : 'bg-blue-500/10 border-blue-500/50 text-blue-200'}"
          >
            {#if myTargetHidden}
              <span class="text-4xl">🙈</span>
            {:else}
              <span class="text-sm font-black break-words">{myTarget?.pseudo}</span>
            {/if}
          </button>
          <span class="text-[8px] text-slate-500 mt-2 uppercase">Cliquez pour masquer</span>
        </div>

        {#if game.status === 'finished'}
          <div class="bg-slate-900/80 border border-purple-500/30 rounded-3xl p-4 flex flex-col items-center text-center shadow-lg animate-fade-in mt-auto">
            <span class="text-[10px] font-black uppercase tracking-widest text-purple-400/70 mb-3">Cible Adverse</span>
            <div class="w-full aspect-square rounded-2xl bg-purple-500/20 border-2 border-purple-500 text-purple-200 flex items-center justify-center text-center p-2 shadow-[0_0_20px_rgba(168,85,247,0.3)]">
              <span class="text-sm font-black break-words">{opponentTarget?.pseudo}</span>
            </div>
          </div>

          <button onclick={() => goto('/jouer/versus')} class="w-full bg-slate-800 hover:bg-slate-700 text-white font-bold text-xs uppercase py-4 rounded-2xl mt-4 cursor-pointer">
            Quitter la table
          </button>
        {/if}

      </div>
    </div>
  {/if}
</div>

<style>
  .custom-scrollbar::-webkit-scrollbar { width: 4px; }
  .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
  .custom-scrollbar::-webkit-scrollbar-thumb { background: #4f46e5; border-radius: 4px; }

  @keyframes shake {
    0%, 100% { transform: translateX(0); }
    20%, 60% { transform: translateX(-10px); }
    40%, 80% { transform: translateX(10px); }
  }
  .animate-shake {
    animation: shake 0.5s ease-in-out;
  }

  @keyframes pokeballPulse {
    0% { transform: scale(1); }
    25% { transform: scale(1.15) rotate(-5deg); }
    50% { transform: scale(1); }
    75% { transform: scale(1.15) rotate(5deg); }
    100% { transform: scale(1); }
  }
  .animate-pokeball {
    animation: pokeballPulse 1s ease-in-out infinite;
  }
</style>