<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let isLoading = $state(true);
  let stats = $state({ totalPlayers: 0, totalGames: 0 });

  onMount(async () => {
    isLoading = true;

    // Récupération du nombre d'agents (viewers) inscrits
    const { count: playersCount } = await supabase
      .from('profil_viewer')
      .select('*', { count: 'exact', head: true });

    // Récupération du nombre total de parties enregistrées
    const { count: gamesCount } = await supabase
      .from('historique')
      .select('*', { count: 'exact', head: true });

    stats = {
      totalPlayers: playersCount || 0,
      totalGames: gamesCount || 0
    };

    isLoading = false;
  });
</script>

<div class="w-full max-w-3xl mx-auto p-4 md:p-10 flex flex-col items-center text-center text-white min-h-[70vh] justify-center animate-fade-in">

  <h1 class="text-6xl md:text-7xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-4 drop-shadow-[0_0_30px_rgba(99,102,241,0.2)]">
    ViewerDle
  </h1>

  <p class="text-indigo-200/70 max-w-md mx-auto mb-12 leading-relaxed text-sm md:text-base">
    Bienvenue sur le réseau d'identification de la communauté. Analysez les indices, décryptez les caractéristiques et démasquez le viewer mystère du jour. La matrice vous attend.
  </p>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse text-xs uppercase tracking-widest font-mono">
      Calcul des statistiques réseau...
    </p>
  {:else}
    <div class="grid grid-cols-2 gap-4 md:gap-6 w-full max-w-md animate-fade-in">

      <div class="bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-2xl p-6 shadow-xl group hover:border-teal-500/40 transition-colors">
        <span class="block text-4xl md:text-5xl font-black text-teal-300 font-mono mb-2 drop-shadow-[0_0_10px_rgba(45,212,191,0.3)]">
          {stats.totalPlayers}
        </span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-300/50 font-bold">
          Agents Inscrits
        </span>
      </div>

      <div class="bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-2xl p-6 shadow-xl group hover:border-fuchsia-500/40 transition-colors">
        <span class="block text-4xl md:text-5xl font-black text-fuchsia-300 font-mono mb-2 drop-shadow-[0_0_10px_rgba(217,70,239,0.3)]">
          {stats.totalGames}
        </span>
        <span class="text-[10px] uppercase tracking-widest text-indigo-300/50 font-bold">
          Parties Jouées
        </span>
      </div>

    </div>

    <a href="/jouer" class="mt-12 text-xs font-bold uppercase tracking-widest text-teal-300/60 hover:text-teal-300 border border-teal-500/20 hover:border-teal-500/50 hover:bg-teal-500/10 px-6 py-3 rounded-xl transition-all cursor-pointer">
      Accéder à la grille de jeu →
    </a>
  {/if}

</div>