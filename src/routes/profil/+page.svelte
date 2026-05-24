<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let user = $state<any>(null);
  let mode = $state<'login' | 'register'>('login');
  let isLoading = $state(false);
  let message = $state({ text: '', type: '' });

  // On ne garde que le pseudo et le mot de passe !
  let pseudo = $state('');
  let password = $state('');

  // L'astuce : on transforme le pseudo en un faux e-mail que Supabase va accepter
 function getFakeEmail(p: string) {
     // 1. Enlève tout ce qui n'est pas une lettre ou un chiffre
     const cleanPseudo = p.toLowerCase().replace(/[^a-z0-9]/g, '');

     // 2. Ajoute ton suffixe sécurisé avant l'arobase
     return `${cleanPseudo}_viewerdl_test@gmail.com`;
   }

  onMount(async () => {
    const { data } = await supabase.auth.getUser();
    user = data.user;

    supabase.auth.onAuthStateChange((_, session) => {
      user = session?.user ?? null;
    });
  });

  async function handleSubmit(e: Event) {
      e.preventDefault();
      isLoading = true;
      message = { text: '', type: '' };

      const email = getFakeEmail(pseudo);

      if (mode === 'register') {
        // 1. On passe le pseudo dans les métadonnées (options.data) pour le trigger !
        const { data: authData, error: authError } = await supabase.auth.signUp({
          email,
          password,
          options: {
            data: {
              pseudo_reel: pseudo
            }
          }
        });

        if (authError) {
          message = { text: "Erreur : " + authError.message, type: 'error' };
        } else {
          // 2. On a PLUS BESOIN de faire un insert manuellement !
          // Le Trigger SQL a déjà créé la ligne dans profil_viewer.
          message = { text: "Inscription réussie ! Connexion en cours...", type: 'success' };

          // Optionnel : on connecte l'utilisateur direct après son inscription
          await supabase.auth.signInWithPassword({ email, password });
          pseudo = '';
          password = '';
        }
      } else {
        // Connexion classique
        const { error } = await supabase.auth.signInWithPassword({ email, password });
        if (error) message = { text: "Pseudo ou mot de passe incorrect.", type: 'error' };
      }

      isLoading = false;
    }

  async function handleLogout() {
    await supabase.auth.signOut();
    pseudo = '';
    password = '';
  }
</script>

<div class="w-full max-w-md mx-auto flex flex-col items-center animate-fade-in z-20">

  {#if user}
    <!-- VUE : CONNECTÉ -->
    <div class="w-full p-8 bg-slate-900/80 backdrop-blur-xl border border-teal-500/30 rounded-3xl shadow-[0_0_30px_rgba(45,212,191,0.15)] text-center">
      <h2 class="text-3xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-teal-300 to-indigo-300 mb-6">
        Transmission établie
      </h2>
      <!-- On affiche le pseudo proprement en retirant le "@viewerdle.local" -->
      <p class="text-indigo-200/70 mb-8">Viewer identifié : <br/><span class="text-teal-300 font-bold tracking-wider text-2xl">{user.email.replace('_viewerdl_test@gmail.com', '')}</span></p>

      <button
        onclick={handleLogout}
        class="w-full py-4 rounded-xl font-bold uppercase tracking-widest text-rose-200 bg-rose-500/10 border border-rose-500/30 hover:bg-rose-500/20 hover:border-rose-400/50 hover:shadow-[0_0_15px_rgba(244,63,94,0.3)] transition-all"
      >
        Déconnexion
      </button>
    </div>

  {:else}
    <!-- VUE : FORMULAIRE -->
    <div class="w-full bg-slate-900/80 backdrop-blur-xl border border-indigo-500/30 rounded-3xl shadow-[0_0_40px_rgba(99,102,241,0.1)] overflow-hidden">

      <div class="flex border-b border-indigo-500/20">
        <button onclick={() => mode = 'login'} class={`flex-1 py-5 text-sm font-bold uppercase tracking-wider transition-all ${mode === 'login' ? 'text-teal-300 bg-indigo-500/10 border-b-2 border-teal-400 shadow-[inset_0_-10px_20px_-10px_rgba(45,212,191,0.3)]' : 'text-indigo-300/50 hover:text-indigo-300 hover:bg-indigo-500/5'}`}>
          Connexion
        </button>
        <button onclick={() => mode = 'register'} class={`flex-1 py-5 text-sm font-bold uppercase tracking-wider transition-all ${mode === 'register' ? 'text-fuchsia-300 bg-indigo-500/10 border-b-2 border-fuchsia-400 shadow-[inset_0_-10px_20px_-10px_rgba(217,70,239,0.3)]' : 'text-indigo-300/50 hover:text-indigo-300 hover:bg-indigo-500/5'}`}>
          Inscription
        </button>
      </div>

      <form onsubmit={handleSubmit} class="p-8 flex flex-col gap-5 relative group">
        <div class="absolute inset-0 bg-gradient-to-b from-transparent to-indigo-500/5 pointer-events-none"></div>

        {#if message.text}
          <div class={`p-4 rounded-xl text-sm font-medium border backdrop-blur-sm ${message.type === 'error' ? 'bg-rose-500/10 border-rose-500/30 text-rose-300' : 'bg-teal-500/10 border-teal-500/30 text-teal-300'}`}>
            {message.text}
          </div>
        {/if}

        <!-- Champ Pseudo (utilisé pour l'inscription ET la connexion) -->
        <div>
          <label class="block text-xs font-bold tracking-widest uppercase text-indigo-300/70 mb-2 ml-2">Pseudo</label>
          <input type="text" bind:value={pseudo} required placeholder="Votre pseudo..." class="w-full p-4 bg-slate-950/50 border border-indigo-500/20 rounded-xl text-indigo-100 placeholder-indigo-300/20 focus:outline-none focus:border-teal-400/50 focus:ring-1 focus:ring-teal-400/50 transition-all" />
        </div>

        <div>
          <label class="block text-xs font-bold tracking-widest uppercase text-indigo-300/70 mb-2 ml-2">Mot de passe</label>
          <input type="password" bind:value={password} required placeholder="••••••••" class="w-full p-4 bg-slate-950/50 border border-indigo-500/20 rounded-xl text-indigo-100 placeholder-indigo-300/20 focus:outline-none focus:border-teal-400/50 focus:ring-1 focus:ring-teal-400/50 transition-all" />
        </div>

        <button
          type="submit"
          disabled={isLoading}
          class={`mt-4 w-full py-4 rounded-xl font-bold uppercase tracking-widest transition-all ${isLoading ? 'opacity-50 cursor-not-allowed' : mode === 'login' ? 'bg-teal-500/20 text-teal-200 border border-teal-500/30 hover:bg-teal-400/30 hover:border-teal-300 hover:shadow-[0_0_20px_rgba(45,212,191,0.4)]' : 'bg-fuchsia-500/20 text-fuchsia-200 border border-fuchsia-500/30 hover:bg-fuchsia-400/30 hover:border-fuchsia-300 hover:shadow-[0_0_20px_rgba(217,70,239,0.4)]'}`}
        >
          {isLoading ? 'Transmission...' : mode === 'login' ? 'S\'identifier' : 'Rejoindre'}
        </button>
      </form>
    </div>
  {/if}
</div>