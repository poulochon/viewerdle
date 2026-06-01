<script lang="ts">
  import '../app.css';
  import { version } from '../../package.json';
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let { children } = $props();

  let profile = $state<any>(null);
  let isAdmin = $state(false);

  // Sécurité anti-freeze : contrôle d'existence du composant
  let isComponentMounted = false;

  // Fonction pour formater le pseudo (max 5 caractères)
  function formatPseudo(pseudo: string) {
    if (!pseudo) return '';
    return pseudo.length > 5 ? pseudo.slice(0, 5) + '..' : pseudo;
  }

  onMount(() => {
    isComponentMounted = true;

    // 1. Chargement initial de la session
    const initSession = async () => {
      try {
        const { data: { session } } = await supabase.auth.getSession();
        if (session?.user && isComponentMounted) {
          await fetchProfile(session.user.id);
        }
      } catch (error) {
        console.warn("Erreur lors de la récupération initiale de la session :", error);
      }
    };
    initSession();

    // 2. Écouteur des changements de session (Connexion / Déconnexion)
    const { data: { subscription } } = supabase.auth.onAuthStateChange(async (_, session) => {
      if (!isComponentMounted) return; // On ignore si la page est détruite

      if (session?.user) {
        await fetchProfile(session.user.id);
      } else {
        profile = null;
        isAdmin = false;
      }
    });

    // NETTOYAGE CRUCIAL : C'est cette ligne qui évite le gel du navigateur au réveil de l'onglet !
    return () => {
      isComponentMounted = false;
      subscription.unsubscribe();
    };
  });

  async function fetchProfile(userId: string) {
    if (!isComponentMounted) return;

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout de la barre de navigation")), 4000)
      );

      // On lance les deux requêtes en parallèle pour gagner du temps
      const dbRequests = Promise.all([
        supabase.from('profil_viewer').select('pseudo').eq('id', userId).maybeSingle(),
        supabase.from('viewer_droit').select('id_droit').eq('id_viewer', userId).eq('id_droit', 1).maybeSingle()
      ]);

      // Course contre le chrono
      const [profileRes, adminRes] = await Promise.race([dbRequests, timeoutPromise]);

      if (isComponentMounted) {
        // Mise à jour du profil
        if (profileRes.data && !profileRes.error) {
          profile = profileRes.data;
        } else {
          profile = null;
        }

        // Mise à jour des droits Admin
        isAdmin = !!adminRes.data;
      }
    } catch (error) {
      console.warn("Délai dépassé ou erreur réseau pour le profil global :", error);
      // En cas de timeout, on conserve le profil existant s'il y en a déjà un en mémoire
    }
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