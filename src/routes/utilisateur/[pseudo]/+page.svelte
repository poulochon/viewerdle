<script lang="ts">
  import { onMount } from 'svelte';
  import { page } from '$app/stores';
  import { supabase } from '$lib/supabaseClient';

  let pseudo = $derived($page.params.pseudo);

  let isLoading = $state(true);
  let userProfile = $state<any>(null);
  let criteres = $state<any[]>([]);
  let history = $state<any[]>([]);
  let maxTries = $state(1);

  // Sécurité anti-freeze : contrôle d'existence du composant
  let isComponentMounted = false;

  // Variables pour le classement
  let userRank = $state<number | string>("-");
  let userScore = $state(0);
  const fibonacci = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];

  // Variables du calendrier
  const currentDate = new Date();
  let selectedMonth = $state(currentDate.getMonth());
  let selectedYear = $state(currentDate.getFullYear());
  const monthNames = ["Janvier", "Février", "Mars", "Avril", "Mai", "Juin", "Juillet", "Août", "Septembre", "Octobre", "Novembre", "Décembre"];
  const availableYears = Array.from({ length: currentDate.getFullYear() - 2024 + 1 }, (_, i) => 2024 + i).reverse();
  const weekDays = ["Lun", "Mar", "Mer", "Jeu", "Ven", "Sam", "Dim"];

  let calendarGrid = $derived.by(() => {
    const grid = [];
    const firstDay = new Date(selectedYear, selectedMonth, 1);
    const lastDay = new Date(selectedYear, selectedMonth + 1, 0);
    let startingDay = firstDay.getDay() === 0 ? 6 : firstDay.getDay() - 1;

    for (let i = 0; i < startingDay; i++) grid.push(null);

    for (let i = 1; i <= lastDay.getDate(); i++) {
      const yyyy = selectedYear;
      const mm = String(selectedMonth + 1).padStart(2, '0');
      const dd = String(i).padStart(2, '0');
      const dateStr = `${yyyy}-${mm}-${dd}`;

      const game = history.find(h => h.date_partie === dateStr);

      grid.push({
        dayNumber: i,
        dateStr: dateStr,
        played: !!game,
        won: game?.victoire === true || game?.victoire === 'true',
        tries: game?.tentatives || 0
      });
    }
    return grid;
  });

  onMount(() => {
    isComponentMounted = true;

    // NETTOYAGE : Détruit le statut si on change de page
    return () => {
      isComponentMounted = false;
    };
  });

  $effect(() => {
    if (pseudo) fetchUserProfile(pseudo);
  });

  async function fetchUserProfile(targetPseudo: string) {
    isLoading = true;

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout réseau chargement profil")), 4000)
      );

      // 1. Récupération du profil principal
      const userPromise = supabase
        .from('profil_viewer')
        .select('id, pseudo, caracteristiques')
        .ilike('pseudo', targetPseudo)
        .maybeSingle();

      const { data: userData } = await Promise.race([userPromise, timeoutPromise]);

      if (!userData) {
        if (isComponentMounted) userProfile = null;
        return;
      }

      if (isComponentMounted) userProfile = userData;

      // 2. Requêtes parallèles pour accélérer le chargement (Critères + Classement + Historique perso)
      const dbRequests = Promise.all([
        supabase.from('config_caracteristiques').select('cle, label').eq('actif', true).order('ordre'),
        supabase.from('profil_viewer').select('id, pseudo, historique(victoire, tentatives)'),
        supabase.from('historique').select('date_partie, tentatives, victoire').eq('id_compte', userData.id).eq('type_jeu', 'viewerdl').order('date_partie', { ascending: false })
      ]);

      const [critRes, allUsersRes, histRes] = await Promise.race([dbRequests, timeoutPromise]);

      if (isComponentMounted) {
        // --- Assignation des critères ---
        criteres = critRes.data || [];

        // --- Calcul du Classement ---
        const allUsers = allUsersRes.data;
        if (allUsers) {
          const scores = allUsers.map(u => {
            let score = 0;
            if (u.historique && Array.isArray(u.historique)) {
              u.historique.forEach((game: any) => {
                if (game.victoire === true || game.victoire === 'true') {
                  const tries = Number(game.tentatives) || 0;
                  const index = Math.max(0, 11 - tries);
                  score += Math.round(((fibonacci[index] || 0) + 2) / 2);
                }
              });
            }
            return { id: u.id, pseudo: u.pseudo, score };
          });

          scores.sort((a, b) => b.score !== a.score ? b.score - a.score : a.pseudo.localeCompare(b.pseudo));

          const rankIndex = scores.findIndex(s => s.id === userData.id);
          if (rankIndex !== -1 && scores[rankIndex].score > 0) {
            userRank = rankIndex + 1;
            userScore = scores[rankIndex].score;
          } else {
            userRank = "Non classé";
            userScore = 0;
          }
        }

        // --- Assignation de l'Historique ---
        history = histRes.data || [];
        maxTries = history.length > 0 ? Math.max(...history.map(h => h.tentatives)) : 1;
      }

    } catch (error) {
      console.warn("Erreur ou timeout lors du chargement du profil :", error);
      if (isComponentMounted && !userProfile) {
        userProfile = null; // Force l'affichage de "Dossier Introuvable" en cas d'échec total
      }
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  function getHeatmapStyle(tries: number) {
    if (tries === 0) return 'background-color: rgba(30, 41, 59, 0.4); border-color: rgba(51, 65, 85, 0.3); color: rgba(148, 163, 184, 0.2);';
    if (maxTries <= 1) return 'background-color: rgba(16, 185, 129, 0.8); border-color: rgb(52, 211, 153); box-shadow: 0 0 10px rgba(16, 185, 129, 0.4); color: white;';

    const hue = 120 - ((tries - 1) / (maxTries - 1)) * 120;
    return `background-color: hsla(${hue}, 70%, 45%, 0.8); border-color: hsla(${hue}, 80%, 60%, 1); box-shadow: 0 0 10px hsla(${hue}, 80%, 50%, 0.3); color: white;`;
  }

  function formatValue(val: any) {
    if (val === true || String(val).toLowerCase() === 'true') return 'Vrai';
    if (val === false || String(val).toLowerCase() === 'false') return 'Faux';
    return val !== undefined && val !== null ? val : 'Non défini';
  }
</script>

<div class="w-full max-w-5xl mx-auto p-4 md:p-10 flex flex-col items-center animate-fade-in pb-20 text-white">

  {#if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest text-sm font-bold mt-20">Recherche du dossier joueur ...</p>
  {:else if !userProfile}
    <div class="text-center mt-20 p-8 bg-rose-500/10 border border-rose-500/30 rounded-3xl">
      <h1 class="text-4xl font-black text-rose-400 mb-2">Dossier Introuvable</h1>
      <p class="text-rose-300/70">Aucun joueur portant le pseudonyme "{pseudo}" n'a été identifié.</p>
      <a href="/" class="inline-block mt-6 px-6 py-2 bg-slate-900 border border-indigo-500/30 rounded-xl text-indigo-300 hover:text-teal-300 transition-colors">Retour à l'accueil</a>
    </div>
  {:else}

    <div class="text-center mb-12 w-full flex flex-col items-center">
      <div class="inline-flex items-center justify-center w-24 h-24 bg-slate-900 border-2 border-teal-500/50 rounded-full shadow-[0_0_20px_rgba(45,212,191,0.2)] mb-4 relative">
        <span class="text-4xl font-black text-teal-300">{userProfile.pseudo.charAt(0).toUpperCase()}</span>

        {#if userRank !== "Non classé"}
          <div class="absolute -bottom-2 -right-4 bg-slate-950 border border-amber-500/50 text-amber-400 text-xs font-black px-3 py-1 rounded-full shadow-[0_0_10px_rgba(251,191,36,0.3)]">
            #{userRank}
          </div>
        {/if}
      </div>

      <h1 class="text-5xl md:text-6xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 drop-shadow-md">
        {userProfile.pseudo}
      </h1>

      <div class="flex gap-4 mt-3">
        <p class="text-indigo-300/50 uppercase tracking-[0.3em] text-xs font-bold">
          ID Joueur: {userProfile.id.split('-')[0]}
        </p>
        <span class="text-indigo-300/20">•</span>
        <p class="text-teal-300/70 uppercase tracking-[0.3em] text-xs font-bold">
          Score Global: {userScore} pts
        </p>
      </div>
    </div>

    <div class="grid grid-cols-1 lg:grid-cols-12 gap-8 w-full items-start">

      <div class="lg:col-span-5 bg-slate-900/60 backdrop-blur-md border border-indigo-500/20 rounded-3xl p-6 md:p-8 shadow-xl">
        <h2 class="text-sm font-black uppercase tracking-widest text-indigo-300/60 mb-6 flex items-center gap-3">
          <div class="w-2 h-2 bg-indigo-500 rounded-full animate-pulse"></div>
          Caractéristiques
        </h2>

        <div class="flex flex-col gap-3">
          {#each criteres as crit}
            <div class="flex justify-between items-center p-3 bg-slate-950/50 border border-indigo-500/10 rounded-xl">
              <span class="text-xs uppercase tracking-widest text-indigo-200/50">{crit.label}</span>
              <span class="text-sm font-bold text-teal-300 text-right max-w-[60%] break-words">
                {formatValue(userProfile.caracteristiques?.[crit.cle])}
              </span>
            </div>
          {/each}
        </div>
      </div>

      <div class="lg:col-span-7 bg-slate-900/60 backdrop-blur-md border border-fuchsia-500/20 rounded-3xl p-6 md:p-8 shadow-xl flex flex-col">

        <div class="flex flex-col sm:flex-row justify-between items-center mb-8 gap-4">
          <h2 class="text-sm font-black uppercase tracking-widest text-fuchsia-300/60 flex items-center gap-3">
            <div class="w-2 h-2 bg-fuchsia-500 rounded-full animate-pulse"></div>
            Historique Mensuel
          </h2>

          <div class="flex gap-2">
            <select bind:value={selectedMonth} class="bg-slate-950 border border-fuchsia-500/30 text-fuchsia-200 text-xs font-bold uppercase tracking-widest rounded-lg px-3 py-2 outline-none focus:border-fuchsia-400 cursor-pointer">
              {#each monthNames as month, index}
                <option value={index}>{month}</option>
              {/each}
            </select>

            <select bind:value={selectedYear} class="bg-slate-950 border border-fuchsia-500/30 text-fuchsia-200 text-xs font-bold uppercase tracking-widest rounded-lg px-3 py-2 outline-none focus:border-fuchsia-400 cursor-pointer">
              {#each availableYears as year}
                <option value={year}>{year}</option>
              {/each}
            </select>
          </div>
        </div>

        <div class="grid grid-cols-7 gap-1 md:gap-2 mb-2">
          {#each weekDays as day}
            <div class="text-center text-[10px] md:text-xs font-black uppercase tracking-widest text-indigo-300/40 pb-2">{day}</div>
          {/each}
        </div>

        <div class="grid grid-cols-7 gap-1 md:gap-2">
          {#each calendarGrid as day}
            {#if day === null}
              <div class="h-12 md:h-16 rounded-xl bg-transparent"></div>
            {:else}
              <div class="relative h-12 md:h-16 rounded-xl flex flex-col items-center justify-center border transition-all {day.played ? 'hover:scale-105 cursor-default' : 'opacity-60'}" style={getHeatmapStyle(day.tries)}>
                <span class="absolute top-1 left-1.5 md:top-1.5 md:left-2 text-[8px] md:text-[10px] font-bold opacity-70">{day.dayNumber}</span>
                {#if day.played}
                  <span class="text-sm md:text-xl font-black mt-2 drop-shadow-md">{day.tries}</span>
                {/if}
              </div>
            {/if}
          {/each}
        </div>

        <div class="w-full flex justify-between items-center mt-8 pt-6 border-t border-indigo-500/10">
          <span class="text-[10px] uppercase tracking-widest text-indigo-300/40">1 Essai</span>
          <div class="h-1.5 flex-1 mx-4 rounded-full bg-gradient-to-r from-emerald-500 via-yellow-500 to-rose-500 opacity-50"></div>
          <span class="text-[10px] uppercase tracking-widest text-indigo-300/40">{maxTries} Essais</span>
        </div>

      </div>
    </div>
  {/if}
</div>