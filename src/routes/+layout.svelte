<script lang="ts">
  import '../app.css';
  import { version, changelog } from '../../package.json';
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let { children } = $props();

  let profile = $state<any>(null);
  let isAdmin = $state(false);
  let showUpdatePopup = $state(false);
  let isComponentMounted = false;

  function formatPseudo(pseudo: string) {
    if (!pseudo) return '';
    return pseudo.length > 5 ? pseudo.slice(0, 5) + '..' : pseudo;
  }

  onMount(() => {
    isComponentMounted = true;

    // Gestion de la modale de mise à jour
    const savedVersion = localStorage.getItem('viewerdle_version');
    if (savedVersion !== version && changelog) {
      showUpdatePopup = true;
    }

    // Écouteur global Supabase (Il gère le premier chargement ET les reconnexions)
    const { data: { subscription } } = supabase.auth.onAuthStateChange(async (event, session) => {
      if (!isComponentMounted) return;

      if (session?.user) {
        // LE BOUCLIER ANTI-FREEZE
        if (!profile) {
          await fetchProfile(session.user.id);
        }
      } else {
        profile = null;
        isAdmin = false;
      }
    });

    return () => {
      isComponentMounted = false;
      subscription.unsubscribe();
    };
  });

  async function fetchProfile(userId: string) {
    if (!isComponentMounted) return;

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout navigation")), 10000)
      );

      const dbRequests = Promise.all([
        supabase.from('profil_viewer').select('pseudo').eq('id', userId).maybeSingle(),
        supabase.from('viewer_droit').select('id_droit').eq('id_viewer', userId).eq('id_droit', 1).maybeSingle()
      ]);

      const [profileRes, adminRes] = await Promise.race([dbRequests, timeoutPromise]);

      if (isComponentMounted) {
        if (profileRes.data && !profileRes.error) profile = profileRes.data;
        isAdmin = !!adminRes.data;
      }
    } catch (error) {
      console.warn("Délai dépassé pour le layout :", error);
    }
  }

  function closeUpdatePopup() {
    localStorage.setItem('viewerdle_version', version);
    showUpdatePopup = false;
  }
</script>

<div class="min-h-screen bg-slate-950 text-slate-200 font-sans relative overflow-hidden flex flex-col">

  <div class="absolute inset-0 bg-[linear-gradient(to_right,#818cf81a_1px,transparent_1px),linear-gradient(to_bottom,#818cf81a_1px,transparent_1px)] bg-[size:2rem_2rem] [mask-image:radial-gradient(ellipse_60%_50%_at_50%_0%,#000_70%,transparent_100%)] pointer-events-none z-0"></div>

  <nav class="relative z-20 w-full px-6 py-4 flex justify-between items-center backdrop-blur-md bg-slate-950/50 border-b border-indigo-500/20">
    <a href="/" class="flex items-start gap-2 hover:scale-105 transition-all cursor-pointer group">
      <span class="font-black text-teal-300 text-xl md:text-2xl leading-none group-hover:text-teal-100 transition-colors drop-shadow-[0_0_10px_rgba(45,212,191,0.3)]">
        ViewerDle
      </span>
      <span class="px-2 py-0.5 text-[9px] font-bold uppercase tracking-widest text-teal-400 bg-teal-500/10 border border-teal-500/30 rounded-full shadow-[0_0_10px_rgba(45,212,191,0.2)]">
        v{version}
      </span>
    </a>

    <div class="flex items-center gap-8 text-sm font-medium tracking-wide uppercase text-indigo-200/70">
      <a href="/jouer" class="hover:text-teal-300 transition-all">Jouer</a>
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

{#if showUpdatePopup}
  <div class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-950/80 backdrop-blur-sm animate-fade-in">
    <div class="relative w-full max-w-lg bg-slate-900 border border-fuchsia-500/40 rounded-3xl p-8 shadow-[0_0_50px_rgba(217,70,239,0.15)] flex flex-col items-center text-center">

      <div class="absolute -top-4 bg-slate-950 px-4 py-1.5 rounded-full border border-fuchsia-500/50 text-xs font-black uppercase tracking-widest text-fuchsia-400 shadow-[0_0_15px_rgba(217,70,239,0.3)]">
        Version {version}
      </div>

      <h2 class="text-2xl font-black uppercase tracking-widest text-teal-300 mb-6 mt-2">
        Nouvelle version !
      </h2>

      <div class="w-full bg-slate-950/50 border border-indigo-500/20 rounded-xl p-5 mb-8 text-left text-sm text-indigo-200/80 leading-relaxed">
          {@html changelog.replace(/\\n|\n/g, '<br />')}
        </div>

      <button
        onclick={closeUpdatePopup}
        class="w-full bg-teal-500/10 border-2 border-teal-500/40 hover:bg-teal-500/20 text-teal-300 font-black uppercase tracking-widest px-6 py-4 rounded-xl transition-all shadow-[0_0_15px_rgba(45,212,191,0.1)] hover:shadow-[0_0_25px_rgba(45,212,191,0.3)] cursor-pointer"
      >
        J'ai compris, fermer
      </button>

    </div>
  </div>
{/if}