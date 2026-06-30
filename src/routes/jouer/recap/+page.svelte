<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';
  import { fade, fly } from 'svelte/transition';
  import { backOut } from 'svelte/easing';

  let isLoading = $state(true);
  let recapData = $state<any>(null);
  let nomMois = $state('');
  let errorMessage = $state('');

  // Calcule le 1er et le dernier jour du MOIS PRÉCÉDENT
    function getPreviousMonthBoundaries() {
      const d = new Date();
      d.setDate(0); // Règle la date sur le dernier jour du mois précédent

      const year = d.getFullYear();
      const month = d.getMonth() + 1;
      const lastDay = d.getDate();

      const debut = `${year}-${String(month).padStart(2, '0')}-01`;
      const fin = `${year}-${String(month).padStart(2, '0')}-${String(lastDay).padStart(2, '0')}`;

      const nomsMois = ['Janvier', 'Février', 'Mars', 'Avril', 'Mai', 'Juin', 'Juillet', 'Août', 'Septembre', 'Octobre', 'Novembre', 'Décembre'];
      const nom = `${nomsMois[d.getMonth()]} ${year}`;

      return { debut, fin, nom };
    }

  onMount(async () => {
    try {
      const dateOuverture = new Date('2026-07-01T00:00:00+02:00');
        if (new Date() < dateOuverture) { goto('/'); return; }

        const { data: { session } } = await supabase.auth.getSession();
        if (!session) { goto('/'); return; }

        // On utilise la NOUVELLE fonction
        const { debut, fin, nom } = getPreviousMonthBoundaries();
        nomMois = nom;

        const { data, error } = await supabase.rpc('get_monthly_recap', {
          date_debut: debut,
          date_fin: fin,
          id_utilisateur: session.user.id
        });

      if (error) throw error;
      recapData = data;
    } catch (err) {
      errorMessage = "Impossible de générer le récapitulatif.";
    } finally {
      setTimeout(() => { isLoading = false; }, 300);
    }
  });
</script>

<div class="w-full max-w-5xl mx-auto p-4 md:p-10 text-white min-h-[70vh] pb-20 flex flex-col items-center overflow-hidden">

  <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-fuchsia-400 via-purple-400 to-indigo-400 text-center mb-2 drop-shadow-[0_0_20px_rgba(217,70,239,0.3)]">
    Archives Mensuelles
  </h1>
  <p class="text-indigo-200/50 uppercase tracking-[0.2em] text-xs font-bold text-center mb-12">
    Rétrospective de {nomMois}
  </p>

  {#if isLoading}
    <div in:fade={{duration: 300}} out:fade={{duration: 200}} class="flex flex-col items-center justify-center mt-20 gap-4">
      <div class="w-10 h-10 border-4 border-fuchsia-500/30 border-t-fuchsia-500 rounded-full animate-spin"></div>
      <p class="text-fuchsia-400 animate-pulse uppercase tracking-widest font-mono text-xs">Compilation du protocole...</p>
    </div>

  {:else if errorMessage}
    <div in:fly={{y: 20, duration: 500}} class="bg-rose-500/20 border border-rose-500/50 text-rose-300 p-6 rounded-2xl text-center">
      {errorMessage}
    </div>

  {:else if recapData}

    <!-- MESSAGE INTRODUCTIF -->
    <div in:fly={{ y: 30, duration: 800, delay: 100 }} class="text-center max-w-2xl mb-16 bg-slate-900/60 p-6 md:p-8 rounded-3xl border border-indigo-500/20 shadow-xl">
      <p class="text-indigo-100/80 text-sm md:text-base leading-relaxed">
        👋 Bienvenue dans le récapitulatif de <span class="font-black text-fuchsia-400">{nomMois}</span>.
        La communauté a été très active ce mois-ci, voyons ensemble vos plus grands accomplissements !
      </p>
    </div>

    <!-- ================= RECAP PERSONNEL ================= -->
    <div class="w-full mb-16">
      <h2 in:fade={{ duration: 600, delay: 800 }} class="text-xs font-black uppercase tracking-[0.2em] text-fuchsia-400 mb-6 w-full text-center md:text-left pl-2 flex items-center justify-center md:justify-start gap-3">
        <span class="w-2 h-2 bg-fuchsia-500 rounded-full"></span>
        Analyse Personnelle
      </h2>

      <div class="grid grid-cols-1 sm:grid-cols-2 gap-6 w-full">
        <div in:fly={{ y: 50, duration: 800, delay: 1200, easing: backOut }} class="bg-gradient-to-br from-slate-900 to-slate-950 border border-fuchsia-500/30 p-8 rounded-3xl flex items-center justify-between shadow-[0_0_30px_rgba(217,70,239,0.1)] hover:scale-[1.02] transition-transform">
          <div>
            <span class="text-5xl font-black block text-transparent bg-clip-text bg-gradient-to-r from-fuchsia-400 to-pink-300">
              {recapData.perso_parties}
            </span>
            <span class="text-[10px] uppercase font-black text-fuchsia-200/50 tracking-widest mt-2 block">Parties terminées</span>
          </div>
          <span class="text-5xl drop-shadow-[0_0_15px_rgba(217,70,239,0.4)]">🎮</span>
        </div>

        <div in:fly={{ y: 50, duration: 800, delay: 1400, easing: backOut }} class="bg-gradient-to-br from-slate-900 to-slate-950 border border-purple-500/30 p-8 rounded-3xl flex items-center justify-between shadow-[0_0_30px_rgba(168,85,247,0.1)] hover:scale-[1.02] transition-transform">
          <div>
            <span class="text-5xl font-black block text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-indigo-300">
              {recapData.perso_selectionne} <span class="text-2xl">fois</span>
            </span>
            <span class="text-[10px] uppercase font-black text-purple-200/50 tracking-widest mt-2 block">Sélectionné comme cible</span>
          </div>
          <span class="text-5xl drop-shadow-[0_0_15px_rgba(168,85,247,0.4)]">🎯</span>
        </div>
      </div>
    </div>

    <!-- ================= RECAP GLOBAL ================= -->
    <div class="w-full">
      <h2 in:fade={{ duration: 600, delay: 2200 }} class="text-xs font-black uppercase tracking-[0.2em] text-teal-400 mb-6 w-full text-center md:text-left pl-2 flex items-center justify-center md:justify-start gap-3">
        <span class="w-2 h-2 bg-teal-500 rounded-full"></span>
        Rapport Global de la Communauté
      </h2>

      <div in:fade={{ duration: 800, delay: 2400 }} class="w-full bg-slate-900/60 backdrop-blur-md border border-teal-500/20 rounded-3xl p-6 md:p-10 flex flex-col items-center mb-12 shadow-2xl">

        <!-- Total des parties (Tous modes) -->
        <div in:fly={{ y: 40, scale: 0.9, duration: 800, delay: 2600, easing: backOut }} class="text-center mb-12 bg-slate-950/50 py-6 px-12 rounded-3xl border border-slate-800">
          <span class="text-6xl font-black text-teal-300 block drop-shadow-[0_0_20px_rgba(45,212,191,0.3)]">
            {recapData.total_commu}
          </span>
          <span class="text-[10px] uppercase tracking-widest font-black text-teal-200/50 mt-2 block">
            Parties disputées collectivement (Tous modes)
          </span>
        </div>

        <!-- RECORDS SUR UNE JOURNÉE -->
        <div in:fly={{ y: 30, duration: 800, delay: 2900, easing: backOut }} class="w-full mb-12">
          <h3 class="text-[11px] font-black uppercase tracking-widest text-pink-400 mb-6 text-center">
            🔥 Les Acharnés d'une journée (Max tentatives)
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-3 gap-6">

            <!-- Record Classique -->
            <div class="bg-slate-950/60 border border-teal-500/30 p-5 rounded-2xl text-center shadow-[0_0_15px_rgba(45,212,191,0.1)]">
              <span class="text-[10px] uppercase font-bold text-teal-400 block mb-2 tracking-widest">Classique</span>
              {#if recapData.max_classique.pseudo}
                <span class="font-black text-slate-200 text-lg block">{recapData.max_classique.pseudo}</span>
                <span class="font-mono text-teal-200/60 text-xs">{recapData.max_classique.score} essais</span>
              {:else}
                <span class="text-slate-500 text-xs italic">Aucun record</span>
              {/if}
            </div>

            <!-- Record Anecdotes -->
            <div class="bg-slate-950/60 border border-amber-500/30 p-5 rounded-2xl text-center shadow-[0_0_15px_rgba(245,158,11,0.1)]">
              <span class="text-[10px] uppercase font-bold text-amber-400 block mb-2 tracking-widest">Anecdotes</span>
              {#if recapData.max_anecdotes.pseudo}
                <span class="font-black text-slate-200 text-lg block">{recapData.max_anecdotes.pseudo}</span>
                <span class="font-mono text-amber-200/60 text-xs">{recapData.max_anecdotes.score} essais</span>
              {:else}
                <span class="text-slate-500 text-xs italic">Aucun record</span>
              {/if}
            </div>

            <!-- Record Emotes -->
            <div class="bg-slate-950/60 border border-fuchsia-500/30 p-5 rounded-2xl text-center shadow-[0_0_15px_rgba(217,70,239,0.1)]">
              <span class="text-[10px] uppercase font-bold text-fuchsia-400 block mb-2 tracking-widest">Émotes</span>
              {#if recapData.max_emotes.pseudo}
                <span class="font-black text-slate-200 text-lg block">{recapData.max_emotes.pseudo}</span>
                <span class="font-mono text-fuchsia-200/60 text-xs">{recapData.max_emotes.score} essais</span>
              {:else}
                <span class="text-slate-500 text-xs italic">Aucun record</span>
              {/if}
            </div>
          </div>
        </div>

        <!-- LES 3 PODIUMS GLOBAUX -->
        <div class="grid grid-cols-1 md:grid-cols-3 gap-8 w-full border-t border-slate-800/60 pt-10">

          <!-- PODIUM 1 : PLUS GRAND NOMBRE DE PARTIES JOUÉES (Assiduité globale) -->
          <div in:fly={{ y: 40, duration: 800, delay: 3300, easing: backOut }} class="bg-slate-950/40 p-6 rounded-2xl border border-slate-800/80 hover:border-amber-500/30 transition-colors relative">
            <!-- Couronne pour le 1er ! -->
            <div class="absolute -top-3 -right-2 text-2xl rotate-12 drop-shadow-lg">👑</div>
            <h4 class="text-[10px] font-black uppercase tracking-wider text-amber-400 mb-6 flex flex-col items-center gap-1 text-center">
              <span>🏆 Les plus Assidus</span>
              <span class="text-[8px] text-amber-500/60">(Parties jouées, tous modes)</span>
            </h4>
            <div class="flex flex-col gap-3">
              {#each recapData.top_joueurs as j, idx}
                <div class="flex justify-between items-center text-xs p-3 bg-slate-900/80 border border-slate-800 rounded-xl {idx === 0 ? 'border-amber-500/40 shadow-[0_0_10px_rgba(245,158,11,0.1)]' : ''}">
                  <div class="flex items-center gap-3">
                    <span class="font-black text-slate-500 text-sm">#{idx+1}</span>
                    <span class="font-bold {idx === 0 ? 'text-amber-300' : 'text-slate-300'}">{j.pseudo}</span>
                  </div>
                  <span class="font-mono text-slate-400 bg-slate-950 px-2 py-1 rounded">{j.score} p.</span>
                </div>
              {:else}
                <p class="text-[10px] text-slate-500 italic">Données insuffisantes.</p>
              {/each}
            </div>
          </div>

          <!-- PODIUM 2 : PLUS DE TENTATIVES CUMULÉES -->
          <div in:fly={{ y: 40, duration: 800, delay: 3500, easing: backOut }} class="bg-slate-950/40 p-6 rounded-2xl border border-slate-800/80 hover:border-rose-500/30 transition-colors">
            <h4 class="text-[10px] font-black uppercase tracking-wider text-rose-400 mb-6 flex flex-col items-center gap-1 text-center">
              <span>🔥 Pro du hasard</span>
              <span class="text-[8px] text-rose-500/60">(Cumul propositions, tous modes)</span>
            </h4>
            <div class="flex flex-col gap-3">
              {#each recapData.top_tentatives as t, idx}
                <div class="flex justify-between items-center text-xs p-3 bg-slate-900/80 border border-slate-800 rounded-xl {idx === 0 ? 'border-rose-500/40 shadow-[0_0_10px_rgba(243,113,113,0.1)]' : ''}">
                  <div class="flex items-center gap-3">
                    <span class="font-black text-slate-500 text-sm">#{idx+1}</span>
                    <span class="font-bold {idx === 0 ? 'text-rose-300' : 'text-slate-300'}">{t.pseudo}</span>
                  </div>
                  <span class="font-mono text-slate-400 bg-slate-950 px-2 py-1 rounded">{t.score} clics</span>
                </div>
              {:else}
                <p class="text-[10px] text-slate-500 italic">Données insuffisantes.</p>
              {/each}
            </div>
          </div>

          <!-- PODIUM 3 : CIBLES RÉCURRENTES (Inclut maintenant les émotes !) -->
          <div in:fly={{ y: 40, duration: 800, delay: 3700, easing: backOut }} class="bg-slate-950/40 p-6 rounded-2xl border border-slate-800/80 hover:border-indigo-500/30 transition-colors">
            <h4 class="text-[10px] font-black uppercase tracking-wider text-indigo-400 mb-6 flex flex-col items-center gap-1 text-center">
              <span>🎯 Cibles Récurrentes</span>
              <span class="text-[8px] text-indigo-500/60">(Joueurs & Émotes)</span>
            </h4>
            <div class="flex flex-col gap-3">
              {#each recapData.top_cibles as c, idx}
                <div class="flex justify-between items-center text-xs p-3 bg-slate-900/80 border border-slate-800 rounded-xl {idx === 0 ? 'border-indigo-500/40 shadow-[0_0_10px_rgba(99,102,241,0.1)]' : ''}">
                  <div class="flex items-center gap-3">
                    <span class="font-black text-slate-500 text-sm">#{idx+1}</span>
                    <span class="font-bold {idx === 0 ? 'text-indigo-300' : 'text-slate-300'}">{c.pseudo}</span>
                  </div>
                  <span class="font-mono text-slate-400 bg-slate-950 px-2 py-1 rounded">{c.score} fois</span>
                </div>
              {:else}
                <p class="text-[10px] text-slate-500 italic">Données insuffisantes.</p>
              {/each}
            </div>
          </div>

        </div>
      </div>
    </div>

    <!-- BOUTON RETOUR -->
    <button in:fade={{ duration: 1000, delay: 4500 }} onclick={() => goto('/')} class="mt-8 bg-slate-800/50 border border-slate-700/50 hover:bg-slate-700 hover:border-slate-500 text-slate-300 text-xs font-black uppercase tracking-widest p-4 px-10 rounded-2xl transition-all cursor-pointer shadow-lg">
      Retour à l'accueil
    </button>
  {/if}
</div>