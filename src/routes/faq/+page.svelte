<script lang="ts">
  // Importation de l'animation de Svelte
  import { slide } from 'svelte/transition';

  // Importation directe de notre fichier JSON
  import faqData from '$lib/data/faq.json';

  // État pour savoir quelle question est ouverte (null = tout est fermé)
  let openIndex = $state<number | null>(null);

  // Fonction pour ouvrir/fermer une question
  function toggleQuestion(index: number) {
    if (openIndex === index) {
      openIndex = null; // Si on clique sur celle déjà ouverte, on la ferme
    } else {
      openIndex = index; // Sinon, on ouvre la nouvelle
    }
  }
</script>

<div class="w-full max-w-3xl mx-auto flex flex-col items-center animate-fade-in z-20 pb-20">

  <div class="text-center mb-12">
    <h1 class="text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 mb-4">
      À Propos & F.A.Q
    </h1>
  </div>

  <div class="w-full flex flex-col gap-4">
    {#each faqData as item, index}
      <div class="bg-slate-900/80 backdrop-blur-xl border border-indigo-500/20 rounded-2xl overflow-hidden transition-all duration-300 {openIndex === index ? 'shadow-[0_0_30px_rgba(99,102,241,0.15)] border-indigo-400/50' : 'hover:border-indigo-400/30'}">

        <button
          onclick={() => toggleQuestion(index)}
          class="w-full px-6 py-5 flex items-center justify-between text-left focus:outline-none cursor-pointer"
        >
          <span class="font-bold tracking-wide {openIndex === index ? 'text-teal-300' : 'text-indigo-100'}">
            {item.question}
          </span>

          <span class="text-indigo-400/50 transition-transform duration-300 {openIndex === index ? 'rotate-180 text-teal-400' : ''}">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M19 9l-7 7-7-7" />
            </svg>
          </span>
        </button>

        {#if openIndex === index}
          <div transition:slide={{ duration: 300 }} class="px-6 pb-6 pt-0">
            <div class="w-full h-px bg-gradient-to-r from-transparent via-indigo-500/20 to-transparent mb-4"></div>
            <p class="text-indigo-200/80 leading-relaxed text-sm">
              {item.reponse}
            </p>
          </div>
        {/if}

      </div>
    {/each}
  </div>

  <div class="mt-12 text-center p-6 border border-dashed border-indigo-500/30 rounded-2xl bg-indigo-500/5">
    <p class="text-indigo-300/70 text-sm">
      Une autre question ? C'est encore une alpha donc ca viendra peut etre un jour
    </p>
  </div>

</div>