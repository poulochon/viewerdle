<script lang="ts">
  // 🎯 Le viewer mystère à deviner (plus tard, tiré au sort depuis Supabase)
  const targetViewer = { pseudo: "Flogi", caracteristiques: { role: "Modérateur", anciennete: "2 ans" } };

  // Notre base de données temporaire
  const fakeViewers = [
    { pseudo: "Zera", caracteristiques: { role: "VIP", anciennete: "1 an" } },
    { pseudo: "Kameto", caracteristiques: { role: "Viewer", anciennete: "3 mois" } },
    { pseudo: "Antoine", caracteristiques: { role: "Viewer", anciennete: "5 ans" } }
  ];

  let searchQuery = $state("");
  let suggestions = $state<typeof fakeViewers>([]);
  // Historique des tentatives du joueur
  let guesses = $state<typeof fakeViewers>([]);

  function handleInput() {
    if (searchQuery.length < 1) {
      suggestions = [];
      return;
    }
    const query = searchQuery.toLowerCase();

    // On propose les viewers qui matchent ET qu'on n'a pas encore essayés
    suggestions = fakeViewers.filter(v =>
      v.pseudo.toLowerCase().includes(query) &&
      !guesses.find(g => g.pseudo === v.pseudo)
    );
  }

  function selectViewer(viewer: typeof fakeViewers[0]) {
    // On ajoute le viewer choisi tout en haut de la liste des essais
    guesses = [viewer, ...guesses];
    searchQuery = "";
    suggestions = [];
  }

  // 🎨 Fonction qui compare une valeur et renvoie le design correspondant
  function getMatchClass(value: string, targetValue: string) {
    if (value === targetValue) {
      // Correct : Menthe lumineuse
      return "bg-teal-500/20 border-teal-400/50 text-teal-200 shadow-[0_0_15px_rgba(45,212,191,0.2)]";
    }
    // Incorrect : Rose poudré / Rouge doux
    return "bg-rose-500/20 border-rose-400/50 text-rose-200";
  }
</script>

<div class="flex flex-col items-center mb-12 text-center w-full">
  <h1 class="text-6xl md:text-8xl font-black uppercase tracking-tighter text-transparent bg-clip-text bg-gradient-to-br from-teal-200 via-indigo-300 to-fuchsia-300 drop-shadow-[0_0_20px_rgba(165,180,252,0.2)] mb-4">
    ViewerDle
  </h1>
  <p class="text-indigo-200/50 tracking-[0.3em] uppercase text-xs font-semibold">
    Identifie la cible du jour
  </p>
</div>

<!-- Zone de recherche -->
<div class="relative w-full max-w-lg z-20">
  <div class="relative group">
    <div class="absolute -inset-0.5 bg-gradient-to-r from-teal-400 to-fuchsia-400 rounded-2xl blur opacity-20 group-hover:opacity-40 transition duration-1000"></div>

    <input
      type="text"
      bind:value={searchQuery}
      oninput={handleInput}
      placeholder="Entrez un pseudo..."
      class="relative w-full p-4 pl-6 text-lg bg-slate-900/80 backdrop-blur-xl border border-indigo-500/30 rounded-2xl text-indigo-100 placeholder-indigo-300/30 focus:outline-none focus:border-teal-300/50 focus:ring-1 focus:ring-teal-300/50 shadow-2xl transition-all"
    />
  </div>

  {#if suggestions.length > 0}
    <ul class="absolute w-full mt-3 bg-slate-900/90 backdrop-blur-xl border border-indigo-500/30 rounded-2xl shadow-[0_0_30px_rgba(0,0,0,0.5)] overflow-hidden">
      {#each suggestions as viewer}
        <li>
          <button
            class="w-full text-left px-6 py-4 hover:bg-indigo-500/20 focus:bg-indigo-500/20 transition-all cursor-pointer flex justify-between items-center group"
            onclick={() => selectViewer(viewer)}
          >
            <span class="font-bold text-indigo-100 group-hover:text-teal-200 transition-colors">{viewer.pseudo}</span>
            <span class="text-xs font-medium tracking-wider uppercase text-fuchsia-300/70 border border-fuchsia-500/20 px-2 py-1 rounded-full">
              {viewer.caracteristiques.role}
            </span>
          </button>
        </li>
      {/each}
    </ul>
  {/if}
</div>

<!-- 🎮 Plateau de jeu (Historique des essais) -->
{#if guesses.length > 0}
  <div class="w-full max-w-2xl mt-16 flex flex-col gap-3 relative z-10">

    <!-- En-têtes de colonnes -->
    <div class="grid grid-cols-3 gap-4 px-2 text-center text-[10px] sm:text-xs font-bold tracking-widest uppercase text-indigo-300/50 mb-2">
      <div>Pseudo</div>
      <div>Rôle</div>
      <div>Ancienneté</div>
    </div>

    <!-- Lignes de tentatives -->
    {#each guesses as guess}
      <div class="grid grid-cols-3 gap-2 sm:gap-4 animate-fade-in">

        <!-- Case Pseudo -->
        <div class={`flex items-center justify-center p-3 sm:p-4 rounded-xl border backdrop-blur-md font-bold transition-all ${getMatchClass(guess.pseudo, targetViewer.pseudo)}`}>
          <span class="truncate">{guess.pseudo}</span>
        </div>

        <!-- Case Rôle -->
        <div class={`flex items-center justify-center p-3 sm:p-4 rounded-xl border backdrop-blur-md font-medium text-sm transition-all ${getMatchClass(guess.caracteristiques.role, targetViewer.caracteristiques.role)}`}>
          <span class="truncate">{guess.caracteristiques.role}</span>
        </div>

        <!-- Case Ancienneté -->
        <div class={`flex items-center justify-center p-3 sm:p-4 rounded-xl border backdrop-blur-md font-medium text-sm transition-all ${getMatchClass(guess.caracteristiques.anciennete, targetViewer.caracteristiques.anciennete)}`}>
          <span class="truncate">{guess.caracteristiques.anciennete}</span>
        </div>

      </div>
    {/each}
  </div>
{/if}