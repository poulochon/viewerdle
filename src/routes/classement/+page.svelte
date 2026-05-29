<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let isLoading = $state(true);
  let leaderboard = $state<any[]>([]);

  // Tableau de la suite de Fibonacci (de F0 à F10)
  const fibonacci = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];

  // Calcul dynamique des points pour le graphique (de 1 à 10 essais)
  const chartData = Array.from({ length: 10 }, (_, i) => {
    const essais = i + 1;
    const index = Math.max(0, 11 - essais);
    const fVal = fibonacci[index] || 0;
    const points = Math.round((fVal + 2) / 2);
    return { essais, points };
  });

  // Pour le SVG : on trouve la valeur max pour mettre à l'échelle le graphique
  const maxPoints = Math.max(...chartData.map(d => d.points));

  onMount(async () => {
    isLoading = true;

    const { data, error } = await supabase
      .from('profil_viewer')
      .select('pseudo, historique(victoire, tentatives)');

    if (data) {
      const calculatedScores = data.map(player => {
        let score = 0;
        let wins = 0;
        let totalTentatives = 0;

        if (player.historique && Array.isArray(player.historique)) {
          player.historique.forEach((game: any) => {
            if (game.victoire === true || game.victoire === 'true') {
              wins++;
              const tries = Number(game.tentatives) || 0;
              totalTentatives += tries;

              const index = Math.max(0, 11 - tries);
              const fVal = fibonacci[index] || 0;
              const points = Math.round((fVal + 2) / 2);
              score += points;
            }
          });
        }

        return {
          pseudo: player.pseudo || 'Agent Inconnu',
          score,
          wins,
          avgTries: wins > 0 ? (totalTentatives / wins).toFixed(1) : '-'
        };
      });

      leaderboard = calculatedScores.sort((a, b) => {
        if (b.score !== a.score) return b.score - a.score;
        return a.pseudo.localeCompare(b.pseudo);
      });
    }

    isLoading = false;
  });
</script>

<div class="w-full max-w-4xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20">

  <div class="text-center mb-12">
    <h1 class="text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-amber-300 via-orange-400 to-rose-400 mb-2 drop-shadow-[0_0_30px_rgba(251,191,36,0.2)]">
      Classement
    </h1>
    <p class="text-amber-200/50 tracking-[0.3em] uppercase text-xs font-bold">
      L'élite d'internet
    </p>
  </div>

  {#if isLoading}
    <p class="text-amber-300 animate-pulse uppercase tracking-widest text-sm font-bold mt-10">
      Extraction des données de la bdd...
    </p>
  {:else}

    {#if leaderboard.length === 0}
      <div class="p-8 border border-indigo-500/30 rounded-3xl bg-slate-900/50 text-center w-full mb-12">
        <p class="text-indigo-300/60 uppercase tracking-widest">Le réseau est vide.</p>
      </div>
    {:else}
      <div class="w-full bg-slate-900/80 backdrop-blur-xl border border-amber-500/20 rounded-3xl shadow-[0_0_40px_rgba(251,191,36,0.05)] overflow-hidden mb-12">

        <div class="grid grid-cols-12 gap-4 p-6 border-b border-amber-500/20 text-[10px] font-black uppercase tracking-widest text-amber-300/50 text-center items-center">
          <div class="col-span-2 text-left pl-4">Rang</div>
          <div class="col-span-4 text-left">Agent</div>
          <div class="col-span-2">Score</div>
          <div class="col-span-2">Victoires</div>
          <div class="col-span-2">Moy. Essais</div>
        </div>

        <div class="flex flex-col">
          {#each leaderboard as player, index}
            <div class="grid grid-cols-12 gap-4 p-4 items-center border-b border-indigo-500/10 hover:bg-amber-500/5 transition-colors group">

              <div class="col-span-2 text-left pl-4">
                {#if index === 0 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-8 h-8 rounded-full bg-amber-500/20 border border-amber-400 text-amber-300 font-black shadow-[0_0_15px_rgba(251,191,36,0.4)]">1</span>
                {:else if index === 1 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-8 h-8 rounded-full bg-slate-300/20 border border-slate-300 text-slate-200 font-black shadow-[0_0_15px_rgba(203,213,225,0.2)]">2</span>
                {:else if index === 2 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-8 h-8 rounded-full bg-orange-700/20 border border-orange-600 text-orange-400 font-black shadow-[0_0_15px_rgba(234,88,12,0.2)]">3</span>
                {:else}
                  <span class="inline-flex items-center justify-center w-8 h-8 text-indigo-300/40 font-bold">{index + 1}</span>
                {/if}
              </div>

              <div class="col-span-4 text-left font-bold text-sm truncate">
                <a
                  href="/utilisateur/{player.pseudo}"
                  class="hover:underline hover:text-teal-300 transition-colors {player.score === 0 ? 'text-indigo-300/40' : 'text-indigo-100 group-hover:text-amber-200'}"
                  title="Consulter le dossier agent de {player.pseudo}"
                >
                  {player.pseudo}
                </a>
              </div>

              <div class="col-span-2 text-center font-mono font-black text-lg {player.score === 0 ? 'text-indigo-300/30' : 'text-amber-400'}">
                {player.score}
              </div>

              <div class="col-span-2 text-center font-bold text-sm {player.wins === 0 ? 'text-indigo-300/30' : 'text-indigo-300'}">
                {player.wins}
              </div>

              <div class="col-span-2 text-center font-mono text-xs {player.avgTries === '-' ? 'text-indigo-300/30' : 'text-teal-300/70'}">
                {player.avgTries}
              </div>

            </div>
          {/each}
        </div>
      </div>
    {/if}

    <div class="w-full max-w-2xl bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 flex flex-col items-center shadow-2xl">
      <h3 class="text-xs font-black uppercase tracking-widest text-indigo-300/60 mb-8 text-center">
        Courbe de récompense <br/><span class="text-[10px] font-mono text-teal-400/50">Fibonacci</span>
      </h3>

      <div class="relative w-full h-48 md:h-64">
        <svg viewBox="0 0 1000 250" class="w-full h-full overflow-visible" preserveAspectRatio="none">

          {#each [0, 0.25, 0.5, 0.75, 1] as yRatio}
            <line
              x1="0"
              y1={250 - (250 * yRatio)}
              x2="1000"
              y2={250 - (250 * yRatio)}
              stroke="rgba(99, 102, 241, 0.1)"
              stroke-width="1"
              stroke-dasharray="4"
            />
          {/each}

          <polyline
            fill="none"
            stroke="url(#gradientLine)"
            stroke-width="4"
            points={chartData.map((d, i) => `${(i / 9) * 1000},${250 - ((d.points / maxPoints) * 200)}`).join(' ')}
          />

          {#each chartData as d, i}
            <g transform="translate({(i / 9) * 1000}, {250 - ((d.points / maxPoints) * 200)})">
              <circle r="6" fill="#14b8a6" class="shadow-[0_0_10px_#14b8a6]" />
              <circle r="3" fill="#ffffff" />
              <text x="0" y="-15" fill="#fbcb3b" font-size="14" font-weight="bold" text-anchor="middle" class="font-mono">
                {d.points}
              </text>
            </g>
          {/each}

          <defs>
            <linearGradient id="gradientLine" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#14b8a6" /> <stop offset="100%" stop-color="#818cf8" /> </linearGradient>
          </defs>
        </svg>

        <div class="absolute -bottom-6 w-full flex justify-between px-[2%]">
          {#each chartData as d}
            <span class="text-[10px] md:text-xs font-bold text-indigo-400/60 w-8 text-center -ml-4">
              {d.essais}
            </span>
          {/each}
        </div>
      </div>

      <p class="text-[10px] uppercase tracking-widest text-indigo-400/40 mt-12">Axe X : Tentatives | Axe Y : Points octroyés</p>
    </div>

  {/if}
</div>