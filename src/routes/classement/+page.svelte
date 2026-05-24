<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let leaderboard = $state<any[]>([]);
  let isLoading = $state(true);

  onMount(async () => {
    // On récupère les profils triés par score, et on fait une jointure avec leur rang
    const { data, error } = await supabase
      .from('profil_viewer')
      .select(`
        pseudo,
        score_global,
        rank (
          nom_rang,
          icone_url
        )
      `)
      .order('score_global', { ascending: false })
      .limit(50); // On affiche le top 50

    if (!error && data) {
      leaderboard = data;
    }
    isLoading = false;
  });
</script>

<div class="w-full max-w-2xl mx-auto flex flex-col items-center animate-fade-in z-20">

  <div class="text-center mb-10">
    <h1 class="text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-indigo-300 via-purple-300 to-fuchsia-300 mb-4">
      Classement Global
    </h1>
    <p class="text-indigo-200/50 tracking-[0.2em] uppercase text-xs font-semibold">
      Les meilleurs agents de la matrice
    </p>
  </div>

  {#if isLoading}
    <div class="text-indigo-300 animate-pulse uppercase tracking-widest text-sm font-bold mt-10">
      Synchronisation des scores...
    </div>
  {:else if leaderboard.length === 0}
    <p class="text-indigo-200/40 mt-10">Aucun joueur enregistré pour le moment.</p>
  {:else}
    <div class="w-full bg-slate-900/80 backdrop-blur-xl border border-indigo-500/20 rounded-3xl shadow-[0_0_50px_rgba(0,0,0,0.3)] overflow-hidden">

      <div class="grid grid-cols-12 gap-4 px-6 py-4 bg-indigo-500/10 border-b border-indigo-500/20 text-xs font-bold tracking-widest uppercase text-indigo-300/70">
        <div class="col-span-2 text-center">Rang</div>
        <div class="col-span-6">Pseudo</div>
        <div class="col-span-4 text-right">Score Global</div>
      </div>

      <ul class="divide-y divide-indigo-500/10">
        {#each leaderboard as player, index}
          <li class={`grid grid-cols-12 gap-4 px-6 py-4 items-center transition-colors hover:bg-indigo-500/5 ${index === 0 ? 'bg-amber-500/5' : index === 1 ? 'bg-slate-300/5' : index === 2 ? 'bg-amber-700/5' : ''}`}>

            <div class="col-span-2 text-center font-black text-lg">
              {#if index === 0}
                <span class="text-amber-400 drop-shadow-[0_0_10px_rgba(251,191,36,0.4)]">🥇</span>
              {:else if index === 1}
                <span class="text-slate-300">🥈</span>
              {:else if index === 2}
                <span class="text-amber-600">🥉</span>
              {:else}
                <span class="text-indigo-300/40 text-sm font-medium">#{index + 1}</span>
              {/if}
            </div>

            <div class="col-span-6 flex flex-col">
              <span class={`font-bold tracking-wide ${index === 0 ? 'text-amber-300' : 'text-indigo-100'}`}>
                {player.pseudo}
              </span>
              {#if player.rank?.nom_rang}
                <span class="text-[10px] uppercase tracking-wider text-indigo-300/40 font-semibold mt-0.5">
                  {player.rank.nom_rang}
                </span>
              {/if}
            </div>

            <div class="col-span-4 text-right font-mono font-bold text-xl text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-300">
              {player.score_global} <span class="text-xs text-indigo-400/40 font-sans font-normal">pts</span>
            </div>

          </li>
        {/each}
      </ul>
    </div>
  {/if}
</div>