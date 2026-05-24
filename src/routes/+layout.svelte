<script lang="ts">
  import '../app.css';
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let { children } = $props();

  let profile = $state<any>(null);
  let isAdmin = $state(false);

  // Fonction pour formater le pseudo (max 5 caractères)
  function formatPseudo(pseudo: string) {
    if (!pseudo) return '';
    return pseudo.length > 5 ? pseudo.slice(0, 5) + '..' : pseudo;
  }

  onMount(async () => {
    const { data: { session } } = await supabase.auth.getSession();
    if (session?.user) {
      await fetchProfile(session.user.id);
    }

    supabase.auth.onAuthStateChange(async (_, session) => {
      if (session?.user) {
        await fetchProfile(session.user.id);
      } else {
        profile = null;
        isAdmin = false;
      }
    });
  });

async function fetchProfile(userId: string) {
    console.log("Tentative de récupération du profil pour :", userId);

    const { data: profileData, error: profileError } = await supabase
      .from('profil_viewer')
      .select('pseudo')
      .eq('id', userId)
      .single();


const { data, error } = await supabase.from('profil_viewer').select('*');

if (error) {
  console.error("❌ Erreur :", error);
} else {
  console.log("✅ Données récupérées ! Voici la table complète :");
  console.table(data); // console.table crée un joli tableau directement dans la console !
}
    // Le mouchard est ici :
    console.log("Résultat Profil:", profileData, "Erreur:", profileError);

    if (profileData && !profileError) {
      profile = profileData;
    }

    const { data: adminCheck, error: adminError } = await supabase
      .from('viewer_droit')
      .select('id_droit')
      .eq('id_viewer', userId)
      .eq('id_droit', 1)
      .maybeSingle();

    console.log("Check Admin:", adminCheck, "Erreur Admin:", adminError);

    isAdmin = !!adminCheck;
  }
</script>

<div class="min-h-screen bg-slate-950 text-slate-200 font-sans relative overflow-hidden flex flex-col">

  <div class="absolute inset-0 bg-[linear-gradient(to_right,#818cf81a_1px,transparent_1px),linear-gradient(to_bottom,#818cf81a_1px,transparent_1px)] bg-[size:2rem_2rem] [mask-image:radial-gradient(ellipse_60%_50%_at_50%_0%,#000_70%,transparent_100%)] pointer-events-none z-0"></div>

  <nav class="relative z-20 w-full px-6 py-4 flex justify-between items-center backdrop-blur-md bg-slate-950/50 border-b border-indigo-500/20">
    <a href="/" class="text-2xl font-black tracking-widest uppercase text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 drop-shadow-sm hover:opacity-80 transition-opacity">
      ViewerDle
    </a>

    <div class="flex items-center gap-8 text-sm font-medium tracking-wide uppercase text-indigo-200/70">
      <a href="/" class="hover:text-teal-300 transition-all">Jouer</a>
      <a href="/classement" class="hover:text-indigo-300 transition-all">Classement</a>

      {#if isAdmin}
        <a href="/admin" class="text-rose-400/80 hover:text-rose-300 hover:shadow-[0_0_15px_rgba(244,63,94,0.4)] transition-all flex items-center gap-2">
          <span class="w-2 h-2 rounded-full bg-rose-500 animate-pulse"></span>
          Admin
        </a>
      {/if}

      {#if profile}
        <a href="/profil" class="text-teal-300 hover:text-teal-200 hover:shadow-[0_0_10px_rgba(94,234,212,0.5)] transition-all border border-teal-500/30 bg-teal-500/10 px-4 py-2 rounded-full font-bold">
          {formatPseudo(profile.pseudo)}
        </a>
      {:else}
        <a href="/profil" class="hover:text-fuchsia-300 transition-all border border-fuchsia-500/20 bg-fuchsia-500/5 px-4 py-2 rounded-full">
          S'identifier
        </a>
      {/if}
      <a href="/faq" class="hover:text-indigo-300 transition-all">FAQ</a>
    </div>
  </nav>

  <main class="relative z-10 flex-1 flex flex-col items-center w-full max-w-4xl mx-auto p-4 pt-20">
    {@render children()}
  </main>
</div>