<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let isLoading = $state(true);
  let errorMessage = $state('');

  onMount(() => {
    let isMounted = true;

    const checkAuth = async () => {
      try {
        const { data: { session } } = await supabase.auth.getSession();
        if (!session) {
          if (isMounted) goto('/');
          return;
        }

        // Vérification des droits (optionnel, selon ta logique de base)
        const { data: roleCheck } = await supabase.from('viewer_droit')
          .select('id_droit')
          .eq('id_viewer', session.user.id)
          .in('id_droit', [1, 3]);

        if (!roleCheck || roleCheck.length === 0) {
          if (isMounted) goto('/');
          return;
        }

      } catch (e) {
        console.warn("Erreur d'authentification", e);
        if (isMounted) errorMessage = "Erreur de connexion au serveur.";
      } finally {
        if (isMounted) isLoading = false;
      }
    };

    checkAuth();

    return () => { isMounted = false; };
  });
</script>

<div class="w-full min-h-[80vh] p-4 md:p-10 flex flex-col items-center justify-center text-white pb-20 animate-fade-in">

  <div class="text-center mb-16">
    <h1 class="text-4xl md:text-6xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-400 to-fuchsia-400 mb-4 drop-shadow-lg">
      Terminaux de jeu
    </h1>
    <p class="text-indigo-200/60 uppercase tracking-[0.2em] text-xs md:text-sm font-bold">
      Sélectionnez votre protocole d'investigation
    </p>
  </div>

  {#if isLoading}
    <p class="text-teal-300 animate-pulse uppercase tracking-widest font-mono text-sm">
      Vérification des accréditations...
    </p>
  {:else if errorMessage}
    <div class="p-6 bg-rose-500/20 border border-rose-500 rounded-xl text-rose-300 font-bold max-w-xl text-center">
      {errorMessage}
    </div>
  {:else}

    <div class="grid grid-cols-1 md:grid-cols-2 gap-8 w-full max-w-5xl mx-auto">

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

      <div class="relative bg-slate-900/80 backdrop-blur-md border border-purple-500/40 rounded-3xl p-8 flex flex-col items-center text-center shadow-[0_0_30px_rgba(168,85,247,0.1)] hover:shadow-[0_0_40px_rgba(168,85,247,0.2)] hover:border-purple-400 hover:-translate-y-2 transition-all group">
        <div class="absolute -top-3 bg-slate-950 px-4 py-1 rounded-full border border-purple-500/40 text-[10px] text-purple-300 font-bold uppercase tracking-widest">
          Opérationnel
        </div>

        <div class="w-16 h-16 bg-purple-500/10 rounded-2xl flex items-center justify-center border border-purple-500/30 mb-6 group-hover:scale-110 transition-transform">
          <span class="text-3xl">⚔️</span>
        </div>

        <h2 class="text-2xl font-black text-white uppercase tracking-widest mb-3">Versus</h2>
        <p class="text-sm text-indigo-200/70 mb-8 flex-1 leading-relaxed">
          Affrontez un autre enquêteur en temps réel dans un face-à-face impitoyable inspiré du célèbre jeu "Qui est-ce ?".
        </p>

        <button
          onclick={() => goto('/jouer/versus')}
          class="w-full bg-purple-500/10 border-2 border-purple-500/40 hover:bg-purple-500/20 text-purple-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(168,85,247,0.15)] group-hover:shadow-[0_0_25px_rgba(168,85,247,0.3)] cursor-pointer"
        >
          Rejoindre le Lobby
        </button>
      </div>

      <div class="relative bg-slate-900/80 backdrop-blur-md border border-fuchsia-500/40 rounded-3xl p-8 flex flex-col items-center text-center shadow-[0_0_30px_rgba(217,70,239,0.1)] hover:shadow-[0_0_40px_rgba(217,70,239,0.2)] hover:border-fuchsia-400 hover:-translate-y-2 transition-all group">
        <div class="absolute -top-3 bg-slate-950 px-4 py-1 rounded-full border border-fuchsia-500/40 text-[10px] text-fuchsia-300 font-bold uppercase tracking-widest animate-pulse">
          Nouveau
        </div>

        <div class="w-16 h-16 bg-fuchsia-500/10 rounded-2xl flex items-center justify-center border border-fuchsia-500/30 mb-6 group-hover:scale-110 transition-transform">
          <span class="text-3xl">🖼️</span>
        </div>

        <h2 class="text-2xl font-black text-white uppercase tracking-widest mb-3">Émotes</h2>
        <p class="text-sm text-indigo-200/70 mb-8 flex-1 leading-relaxed">
          Décodez l'émote mystère du jour. L'image, d'abord floutée, se dévoilera un peu plus à chaque tentative échouée.
        </p>

        <button
          onclick={() => goto('/jouer/emote')}
          class="w-full bg-fuchsia-500/10 border-2 border-fuchsia-500/40 hover:bg-fuchsia-500/20 text-fuchsia-300 font-black uppercase tracking-widest text-sm px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(217,70,239,0.15)] group-hover:shadow-[0_0_25px_rgba(217,70,239,0.3)] cursor-pointer"
        >
          Lancer le décodage
        </button>
      </div>

    </div>
  {/if}
</div>