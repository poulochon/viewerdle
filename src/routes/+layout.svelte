<script lang="ts">
  import '../app.css';
  let { children } = $props();
</script>

<!-- Conteneur global sombre -->
<div class="min-h-screen bg-slate-950 text-slate-200 font-sans relative overflow-hidden flex flex-col">

  <!-- Effet de grille rétro-futuriste subtile en fond -->
  <div class="absolute inset-0 bg-[linear-gradient(to_right,#818cf81a_1px,transparent_1px),linear-gradient(to_bottom,#818cf81a_1px,transparent_1px)] bg-[size:2rem_2rem] [mask-image:radial-gradient(ellipse_60%_50%_at_50%_0%,#000_70%,transparent_100%)] pointer-events-none"></div>

  <!-- Navbar Glassmorphism -->
  <nav class="relative z-10 w-full px-6 py-4 flex justify-between items-center backdrop-blur-md bg-slate-950/50 border-b border-indigo-500/20">
    <!-- Logo -->
    <a href="/" class="text-2xl font-black tracking-widest uppercase text-transparent bg-clip-text bg-gradient-to-r from-teal-300 via-indigo-300 to-fuchsia-300 drop-shadow-sm hover:opacity-80 transition-opacity">
      ViewerDle
    </a>

    <!-- Menu -->
    <div class="flex gap-8 text-sm font-medium tracking-wide uppercase text-indigo-200/70">
      <a href="/" class="hover:text-teal-300 hover:shadow-[0_0_10px_rgba(94,234,212,0.5)] transition-all">Jouer</a>
      <a href="/" class="hover:text-indigo-300 transition-all">Classement</a>
      <a href="/profil" class="hover:text-fuchsia-300 transition-all">Profil</a>
    </div>
  </nav>

  <!-- Contenu de la page -->
  <main class="relative z-10 flex-1 flex flex-col items-center w-full max-w-4xl mx-auto p-4 pt-20">
    {@render children()}
  </main>
</div>