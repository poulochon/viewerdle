<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase, SUPABASE_URL, SUPABASE_KEY } from '$lib/supabaseClient';

  let isLoading = $state(true);
  let leaderboard = $state<any[]>([]);
  let visibleRanks = $state<any[]>([]);

  let isComponentMounted = false;

  const fibonacci = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];
  const chartData = Array.from({ length: 10 }, (_, i) => {
    const essais = i + 1;
    const index = Math.max(0, 11 - essais);
    const fVal = fibonacci[index] || 0;
    return { essais, points: Math.round((fVal + 2) / 2) };
  });

  const maxPoints = Math.max(...chartData.map(d => d.points));
  const paddingX = 60; const paddingY = 40;
  const widthX = 1000 - (paddingX * 2); const heightY = 250 - (paddingY * 2);
  const getX = (index: number) => paddingX + (index / 9) * widthX;
  const getY = (points: number) => 250 - paddingY - ((points / maxPoints) * heightY);

  // Dictionnaire de style pour les médailles
  function getMedalDisplay(categorie: string) {
    const medailles: Record<string, { icon: string, title: string, color: string }> = {
      'og': { icon: '🌟', title: 'Pionnier OG', color: 'text-orange-400 drop-shadow-[0_0_2px_rgba(251,146,60,0.8)]' },
      'createur': { icon: '🛠️', title: 'Créateur', color: 'text-teal-400' },
      'streameuse': { icon: '👑', title: 'Streameuse', color: 'text-pink-400' },
      'assiduite': { icon: '🥇', title: 'Top Assidu', color: 'text-yellow-400' },
      'acharnes': { icon: '🔥', title: 'Pro du Hasard', color: 'text-rose-400' }
    };
    return medailles[categorie] || { icon: '🏅', title: categorie, color: 'text-slate-400' };
  }

  onMount(() => {
    isComponentMounted = true;
    isLoading = true;

    const loadLeaderboard = async () => {
      try {
        const headers = { 'apikey': SUPABASE_KEY, 'Authorization': `Bearer ${SUPABASE_KEY}` };

        // Ajout de user_medals dans le select
        const [rankRes, playerRes] = await Promise.all([
          fetch(`${SUPABASE_URL}/rest/v1/rank?select=*&order=nombre_points.desc`, { headers }),
          fetch(`${SUPABASE_URL}/rest/v1/profil_viewer?select=pseudo,historique(victoire,tentatives),user_medals(categorie,rank)`, { headers })
        ]);

        const ranks = await rankRes.json() || [];
        const playerData = await playerRes.json();

        if (isComponentMounted && playerData && !playerData.error) {

          const calculatedScores = playerData.map((player: any) => {
            let score = 0; let wins = 0; let totalTentatives = 0;

            if (player.historique && Array.isArray(player.historique)) {
              player.historique.forEach((game: any) => {
                if (game.victoire === true || game.victoire === 'true') {
                  wins++;
                  const tries = Number(game.tentatives) || 0;
                  totalTentatives += tries;
                  const index = Math.max(0, 11 - tries);
                  score += Math.round(((fibonacci[index] || 0) + 2) / 2);
                }
              });
            }

            const currentRank = ranks.find((r: any) => score >= r.nombre_points) || ranks[ranks.length - 1] || null;

            return {
              pseudo: player.pseudo || 'Joueur Inconnu',
              score,
              wins,
              avgTries: wins > 0 ? (totalTentatives / wins).toFixed(1) : '-',
              rankName: currentRank?.nom_rang || 'Non classé',
              rankIcon: currentRank?.icone_url || null,
              userMedals: player.user_medals || [] // Récupération des médailles
            };
          });

          leaderboard = calculatedScores.sort((a: any, b: any) => {
            if (b.score !== a.score) return b.score - a.score;
            return a.pseudo.localeCompare(b.pseudo);
          });

          const highestScoreReached = leaderboard.length > 0 ? Math.max(...leaderboard.map(p => p.score)) : 0;
          const highestRankReached = ranks.find((r: any) => highestScoreReached >= r.nombre_points) || ranks[ranks.length - 1];

          if (highestRankReached) {
            visibleRanks = [...ranks].reverse().filter(r => r.nombre_points <= highestRankReached.nombre_points);
          }
        }
      } catch (error) {
        console.warn("Erreur HTTP classement :", error);
      } finally {
        if (isComponentMounted) isLoading = false;
      }
    };

    loadLeaderboard();
    return () => { isComponentMounted = false; };
  });
</script>

<div class="w-full max-w-5xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20">

  <div class="text-center mb-12">
    <h1 class="text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-amber-300 via-orange-400 to-rose-400 mb-2 drop-shadow-[0_0_30px_rgba(251,191,36,0.2)]">
      Classement
    </h1>
    <p class="text-amber-200/50 tracking-[0.3em] uppercase text-xs font-bold">
      L'élite du réseau
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
        <div class="grid grid-cols-12 gap-2 md:gap-4 p-4 md:p-6 border-b border-amber-500/20 text-[9px] md:text-[10px] font-black uppercase tracking-widest text-amber-300/50 text-center items-center">
          <div class="col-span-1 text-center">#</div>
          <div class="col-span-3 text-center">Palier</div>
          <div class="col-span-3 text-left pl-2">Joueur</div>
          <div class="col-span-2">Score</div>
          <div class="col-span-1">Vict.</div>
          <div class="col-span-2">Moy.</div>
        </div>

        <div class="flex flex-col">
          {#each leaderboard as player, index}
            <div class="grid grid-cols-12 gap-2 md:gap-4 p-3 md:p-4 items-center border-b border-indigo-500/10 hover:bg-amber-500/5 transition-colors group">
              <div class="col-span-1 flex justify-center">
                {#if index === 0 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-6 h-6 md:w-8 md:h-8 rounded-full bg-amber-500/20 border border-amber-400 text-amber-300 text-xs md:text-sm font-black shadow-[0_0_15px_rgba(251,191,36,0.4)]">1</span>
                {:else if index === 1 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-6 h-6 md:w-8 md:h-8 rounded-full bg-slate-300/20 border border-slate-300 text-slate-200 text-xs md:text-sm font-black shadow-[0_0_15px_rgba(203,213,225,0.2)]">2</span>
                {:else if index === 2 && player.score > 0}
                  <span class="inline-flex items-center justify-center w-6 h-6 md:w-8 md:h-8 rounded-full bg-orange-700/20 border border-orange-600 text-orange-400 text-xs md:text-sm font-black shadow-[0_0_15px_rgba(234,88,12,0.2)]">3</span>
                {:else}
                  <span class="inline-flex items-center justify-center w-6 h-6 md:w-8 md:h-8 text-indigo-300/40 text-xs md:text-sm font-bold">{index + 1}</span>
                {/if}
              </div>

              <div class="col-span-3 flex items-center justify-center gap-1 md:gap-2">
                {#if player.rankIcon}
                  <img src={player.rankIcon} alt="" class="w-4 h-4 md:w-6 md:h-6 object-contain" />
                {/if}
                <span class="text-[9px] md:text-[11px] font-black uppercase tracking-widest {player.score === 0 ? 'text-indigo-300/30' : 'text-amber-200/80'} truncate">
                  {player.rankName}
                </span>
              </div>

              <div class="col-span-3 text-left flex items-center gap-1.5 pl-2 truncate w-full">
                <a href="/utilisateur/{player.pseudo}" class="font-bold text-xs md:text-sm truncate hover:underline hover:text-teal-300 transition-colors {player.score === 0 ? 'text-indigo-300/40' : 'text-indigo-100 group-hover:text-amber-200'}" title="Consulter le dossier de {player.pseudo}">
                  {player.pseudo}
                </a>

                {#if player.userMedals && player.userMedals.length > 0}
                  <div class="flex items-center gap-0.5 shrink-0 pt-0.5">
                    {#each player.userMedals as medal}
                      {@const style = getMedalDisplay(medal.categorie)}
                      <span
                        class="text-[11px] md:text-sm cursor-help {style.color} hover:scale-125 transition-transform"
                        title="{style.title}"
                      >
                        {style.icon}
                      </span>
                    {/each}
                  </div>
                {/if}
              </div>

              <div class="col-span-2 text-center font-mono font-black text-sm md:text-lg {player.score === 0 ? 'text-indigo-300/30' : 'text-amber-400'}">
                {player.score}
              </div>

              <div class="col-span-1 text-center font-bold text-xs md:text-sm {player.wins === 0 ? 'text-indigo-300/30' : 'text-indigo-300'}">
                {player.wins}
              </div>

              <div class="col-span-2 text-center font-mono text-[10px] md:text-xs {player.avgTries === '-' ? 'text-indigo-300/30' : 'text-teal-300/70'}">
                {player.avgTries}
              </div>
            </div>
          {/each}
        </div>
      </div>
    {/if}

    <div class="w-full bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 shadow-xl mb-12">
      <h2 class="text-xs font-black uppercase tracking-widest text-indigo-300/60 mb-6 flex items-center gap-3">
        <div class="w-2 h-2 bg-indigo-500 rounded-full animate-pulse"></div>
        Paliers Découverts par la Communauté
      </h2>
      <div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-4">
        {#each visibleRanks as rank}
          <div class="flex items-center gap-3 p-3 bg-slate-950/60 border border-indigo-500/10 rounded-2xl shadow-inner group hover:border-amber-500/30 transition-colors">
            <div class="w-10 h-10 bg-slate-900 border border-indigo-500/10 rounded-xl flex items-center justify-center p-1.5 shadow-md">
              {#if rank.icone_url}
                <img src={rank.icone_url} alt="" class="w-full h-full object-contain" />
              {:else}
                <div class="w-2 h-2 bg-indigo-500/30 rounded-full"></div>
              {/if}
            </div>
            <div class="flex flex-col min-w-0">
              <span class="text-xs font-black uppercase tracking-wide text-indigo-100 truncate group-hover:text-amber-300 transition-colors">{rank.nom_rang}</span>
              <span class="text-[10px] font-mono text-indigo-400/60 font-semibold">{rank.nombre_points} pts requis</span>
            </div>
          </div>
        {/each}

        <div class="flex items-center gap-3 p-3 bg-slate-950/20 border border-dashed border-indigo-500/10 rounded-2xl opacity-40">
          <div class="w-10 h-10 border border-dashed border-indigo-500/20 rounded-xl flex items-center justify-center font-black text-indigo-300/40">?</div>
          <div class="flex flex-col">
            <span class="text-xs font-bold uppercase tracking-wide text-indigo-300/40">Palier Inconnu</span>
            <span class="text-[10px] font-mono text-indigo-400/20">🔒 ??? pts</span>
          </div>
        </div>
      </div>
    </div>

    <div class="w-full max-w-2xl bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 flex flex-col items-center shadow-2xl">
      <h3 class="text-xs font-black uppercase tracking-widest text-indigo-300/60 mb-8 text-center">
        Courbe de récompense <br/><span class="text-[10px] font-mono text-teal-400/50">Fibonacci</span>
      </h3>

      <div class="relative w-full h-52 md:h-68">
        <svg viewBox="0 0 1000 250" class="w-full h-full overflow-visible" preserveAspectRatio="none">
          {#each [0, 0.25, 0.5, 0.75, 1] as yRatio}
            <line
              x1="30"
              y1={250 - paddingY - (heightY * yRatio)}
              x2="970"
              y2={250 - paddingY - (heightY * yRatio)}
              stroke="rgba(99, 102, 241, 0.1)"
              stroke-width="1"
              stroke-dasharray="4"
            />
          {/each}

          <polyline
            fill="none"
            stroke="url(#gradientLine)"
            stroke-width="4"
            points={chartData.map((d, i) => `${getX(i)},${getY(d.points)}`).join(' ')}
          />

          {#each chartData as d, i}
            <g transform="translate({getX(i)}, {getY(d.points)})">
              <circle r="6" fill="#14b8a6" class="shadow-[0_0_10px_#14b8a6]" />
              <circle r="3" fill="#ffffff" />
              <text x="0" y="-15" fill="#fbcb3b" font-size="13" font-weight="black" text-anchor="middle" class="font-mono">
                {d.points}
              </text>
            </g>

            <text x={getX(i)} y="242" fill="rgba(129, 140, 248, 0.5)" font-size="12" font-weight="black" text-anchor="middle" class="font-mono">
              {d.essais}
            </text>
          {/each}

          <defs>
            <linearGradient id="gradientLine" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" stop-color="#14b8a6" />
              <stop offset="100%" stop-color="#818cf8" />
            </linearGradient>
          </defs>
        </svg>
      </div>

      <p class="text-[10px] uppercase tracking-widest text-indigo-400/40 mt-6">Axe X : Tentatives | Axe Y : Points octroyés</p>
    </div>

  {/if}
</div>