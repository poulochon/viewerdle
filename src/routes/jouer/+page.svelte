<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);

  onMount(() => {
    const checkAccess = async () => {
      try {
        const { data: { session } } = await supabase.auth.getSession();
        if (!session) {
          goto('/');
          return;
        }
        const { data: roleCheck } = await supabase.from('viewer_droit').select('id_droit').eq('id_viewer', session.user.id).in('id_droit', [1, 3]);
        if (!roleCheck || roleCheck.length === 0) {
          goto('/');
          return;
        }
        isLoading = false;
      } catch (e) {
        goto('/');
      }
    };
    checkAccess();
  });
</script>

<div class="w-full max-w-5xl mx-auto p-4 md:p-10 flex flex-col items-center text-white min-h-[80vh] justify-center animate-fade-in pb-20">

  <h1 class="text-4xl md:text-5xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-400 mb-4 text-center">
    Terminaux d'Identification
  </h1>
  <p class="text-indigo-300/60 uppercase tracking-[0.2em] text-xs font-bold mb-16 text-center">
    Sélectionnez votre mode d'investigation
  </p>

  {#if isLoading}
    <div class="flex items-center justify-center h-32">
      <p class="text-teal-300 animate-pulse uppercase tracking-widest text-sm font-bold">
        Vérification des accréditations...
      </p>
    </div>
  {:else}
    <div class="grid grid-cols-1 md:grid-cols-2 gap-8 w-full max-w-4xl">

      <!-- MODE CLASSIQUE -->
      <div class="relative bg-slate-900/80 backdrop-blur-md border border-teal-500/40 rounded-3xl p-8 flex flex-col items-center text-center shadow-[0_0_30px_rgba(45,212,191,0.1)] hover:shadow-[0_0_40px_rgba(45,212,191,0.2)] hover:border-teal-400 hover:-translate-y-2 transition-all group">
        <div class="absolute -top-3 bg-slate-950 px-4 py-1 rounded-full border border-teal-500/40 text-[10px] text-teal-300 font-bold uppercase tracking-widest">
          Opérationnel
        </div>

        <div class="w-16 h-16 bg-teal-500/10 rounded-2xl flex items-center justify-center border border-teal-500/30 mb-6 group-hover:scale-110 transition-transform">
          <span class="text-3xl">🔍</span>
        </div>

        <h2 class="text-2xl font-black text-white uppercase tracking-widest mb-3">Classique</h2>
        <p class="text-sm text-indigo-200/70 mb-8 flex-1 leading-relaxed">
          Identifiez la cible du jour en croisant ses caractéristiques. Déduisez son identité par élimination, essai après essai.
        </p>

        <button
          onclick={() => goto('/jouer/classique')}
          class="w-full bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.15)] group-hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] cursor-pointer"
        >
          Lancer l'investigation
        </button>
      </div>

      <!-- MODE ANECDOTE (A SUIVRE) -->
      <div class="relative bg-slate-900/80 backdrop-blur-md border border-amber-500/40 rounded-3xl p-8 flex flex-col items-center text-center shadow-[0_0_30px_rgba(245,158,11,0.1)] hover:shadow-[0_0_40px_rgba(245,158,11,0.2)] hover:border-amber-400 hover:-translate-y-2 transition-all group">
          <div class="absolute -top-3 bg-slate-950 px-4 py-1 rounded-full border border-amber-500/40 text-[10px] text-amber-300 font-bold uppercase tracking-widest">
            Opérationnel
          </div>

          <div class="w-16 h-16 bg-amber-500/10 rounded-2xl flex items-center justify-center border border-amber-500/30 mb-6 group-hover:scale-110 transition-transform">
            <span class="text-3xl">📝</span>
          </div>

          <h2 class="text-2xl font-black text-white uppercase tracking-widest mb-3">Anecdotes</h2>
          <p class="text-sm text-indigo-200/70 mb-8 flex-1 leading-relaxed">
            Devinez qui se cache derrière ces histoires insolites. Un mode basé sur les secrets et confidences des joueurs.
          </p>

          <button
            onclick={() => goto('/jouer/anecdotes')}
            class="w-full bg-amber-500/10 border-2 border-amber-500/40 hover:bg-amber-500/20 text-amber-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(245,158,11,0.15)] group-hover:shadow-[0_0_25px_rgba(245,158,11,0.3)] cursor-pointer"
          >
            Lancer l'enquête
          </button>
        </div>

    </div>
  {/if}
</div>