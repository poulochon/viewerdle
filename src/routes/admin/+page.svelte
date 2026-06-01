<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  // --- ÉTATS ---
  let isLoading = $state(true);
  let isSaving = $state(false);
  let message = $state({ text: '', type: '' });
  let activeTab = $state<'criteres' | 'rangs' | 'enums' | 'messages'>('criteres');

  // Tableaux de données
  let criteres = $state<any[]>([]);
  let ranks = $state<any[]>([]);
  let inboxMessages = $state<any[]>([]); // Tableau des messages

  // Paniers de suppression
  let criteresToDelete = $state<string[]>([]);
  let ranksToDelete = $state<number[]>([]);

  // Variables temporaires pour les inputs des options d'énumération
  let optionInputs = $state<Record<number, string>>({});

  // Sécurité anti-freeze : contrôle d'existence du composant
  let isComponentMounted = false;

  // --- INITIALISATION SÉCURISÉE ---
  onMount(() => {
    isComponentMounted = true;
    isLoading = true;

    const initializeAdmin = async () => {
      try {
        const timeoutPromise = new Promise<any>((_, reject) =>
          setTimeout(() => reject(new Error("Timeout initialisation")), 3000)
        );

        // 1. Récupération de la session
        const sessionPromise = supabase.auth.getSession();
        const { data: { session } } = await Promise.race([sessionPromise, timeoutPromise]);

        if (!session) {
          if (isComponentMounted) goto('/');
          return;
        }

        // 2. Vérification des droits admin
        const adminCheckPromise = supabase
          .from('viewer_droit')
          .select('id_droit')
          .eq('id_viewer', session.user.id)
          .eq('id_droit', 1)
          .maybeSingle();

        const { data: adminCheck } = await Promise.race([adminCheckPromise, timeoutPromise]);

        if (!adminCheck) {
          if (isComponentMounted) goto('/');
          return;
        }

        // 3. Récupération des données
        if (isComponentMounted) {
          await fetchData();
        }

      } catch (error) {
        console.warn("Erreur lors de l'accès au panneau admin au réveil de l'onglet :", error);
        if (isComponentMounted) {
          message = { text: "Le réseau est instable. Tentative de reconnexion...", type: 'error' };
        }
      } finally {
        if (isComponentMounted) isLoading = false;
      }
    };

    initializeAdmin();

    // NETTOYAGE : Coupe instantanément les processus asynchrones si on change de page
    return () => {
      isComponentMounted = false;
    };
  });

  // --- RÉCUPÉRATION DES DONNÉES (SÉCURISÉE) ---
  async function fetchData() {
    if (!isComponentMounted) return;
    isLoading = true;
    criteresToDelete = [];
    ranksToDelete = [];
    optionInputs = {};

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout chargement données")), 4000)
      );

      const dbRequests = Promise.all([
        supabase.from('config_caracteristiques').select('*').order('ordre', { ascending: true }),
        supabase.from('rank').select('*').order('nombre_points', { ascending: true }),
        supabase.from('messagerie').select('*').order('id', { ascending: false })
      ]);

      const [criteresReq, ranksReq, messagesReq] = await Promise.race([dbRequests, timeoutPromise]);

      if (isComponentMounted) {
        if (criteresReq.data) {
          criteres = criteresReq.data.map(c => ({
            ...c,
            options: c.options || []
          }));
        }
        if (ranksReq.data) ranks = ranksReq.data;
        if (messagesReq.data) inboxMessages = messagesReq.data;
      }
    } catch (error) {
      console.warn("Erreur réseau lors de la récupération des tables :", error);
      if (isComponentMounted) {
        message = { text: "Erreur de synchronisation avec la base de données.", type: 'error' };
      }
    } finally {
      if (isComponentMounted) isLoading = false;
    }
  }

  // --- LOGIQUE : CRITÈRES & ÉNUMÉRATIONS ---
  function addCritere() {
    criteres = [...criteres, {
      cle: '',
      label: '',
      type_donnee: 'text',
      ordre: criteres.length + 1,
      actif: true,
      options: []
    }];
  }

  function removeCritere(index: number) {
    const target = criteres[index];
    if (target.cle && target.cle.trim() !== '') {
      criteresToDelete = [...criteresToDelete, target.cle];
    }
    criteres = criteres.filter((_, i) => i !== index);
  }

  function addOption(critereIndex: number) {
    const newVal = optionInputs[critereIndex]?.trim();
    if (newVal) {
      if (!criteres[critereIndex].options) criteres[critereIndex].options = [];
      if (!criteres[critereIndex].options.includes(newVal)) {
        criteres[critereIndex].options = [...criteres[critereIndex].options, newVal];
      }
      optionInputs[critereIndex] = '';
    }
  }

  function removeOption(critereIndex: number, optionIndex: number) {
    criteres[critereIndex].options = criteres[critereIndex].options.filter((_, i) => i !== optionIndex);
  }

  async function saveCriteres() {
    if (!isComponentMounted) return;
    isSaving = true;
    message = { text: '', type: '' };

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout sauvegarde critères")), 4000)
      );

      if (criteresToDelete.length > 0) {
        const deletePromise = supabase.from('config_caracteristiques').delete().in('cle', criteresToDelete);
        const { error: delError } = await Promise.race([deletePromise, timeoutPromise]);
        if (delError) {
          if (isComponentMounted) message = { text: "Erreur de suppression : " + delError.message, type: 'error' };
          return;
        }
      }

      const dataToSave = criteres.filter(c => c.cle.trim() !== '' && c.label.trim() !== '');
      const upsertPromise = supabase.from('config_caracteristiques').upsert(dataToSave);
      const { error: upsertError } = await Promise.race([upsertPromise, timeoutPromise]);

      if (upsertError) {
        if (isComponentMounted) message = { text: "Erreur d'enregistrement : " + upsertError.message, type: 'error' };
      } else {
        if (isComponentMounted) {
          message = { text: "Modifications des critères déployées !", type: 'success' };
          await fetchData();
        }
      }
    } catch (error) {
      console.warn("Erreur sauvegarde critères :", error);
      if (isComponentMounted) message = { text: "Le réseau a mis trop de temps à répondre.", type: 'error' };
    } finally {
      if (isComponentMounted) isSaving = false;
    }
  }

  // --- LOGIQUE : RANGS ---
  function addRank() {
    ranks = [...ranks, { nom_rang: '', nombre_points: 0 }];
  }

  function removeRank(index: number) {
    const target = ranks[index];
    if (target.id) ranksToDelete = [...ranksToDelete, target.id];
    ranks = ranks.filter((_, i) => i !== index);
  }

  async function saveRanks() {
    if (!isComponentMounted) return;
    isSaving = true;
    message = { text: '', type: '' };

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout sauvegarde rangs")), 4000)
      );

      if (ranksToDelete.length > 0) {
        const deletePromise = supabase.from('rank').delete().in('id', ranksToDelete);
        const { error: delError } = await Promise.race([deletePromise, timeoutPromise]);
        if (delError) {
          if (isComponentMounted) message = { text: "Erreur de suppression : " + delError.message, type: 'error' };
          return;
        }
      }

      const validRanks = ranks.filter(r => r.nom_rang.trim() !== '');
      const ranksToUpdate = validRanks.filter(r => r.id);
      const ranksToInsert = validRanks.filter(r => !r.id);

      if (ranksToUpdate.length > 0) {
        const upsertPromise = supabase.from('rank').upsert(ranksToUpdate);
        const { error } = await Promise.race([upsertPromise, timeoutPromise]);
        if (error) {
          if (isComponentMounted) message = { text: "Erreur MAJ : " + error.message, type: 'error' };
          return;
        }
      }

      if (ranksToInsert.length > 0) {
        const insertPromise = supabase.from('rank').insert(ranksToInsert);
        const { error } = await Promise.race([insertPromise, timeoutPromise]);
        if (error) {
          if (isComponentMounted) message = { text: "Erreur Création : " + error.message, type: 'error' };
          return;
        }
      }

      if (isComponentMounted) {
        message = { text: "Mise à jour du système de rangs effectuée !", type: 'success' };
        await fetchData();
      }
    } catch (error) {
      console.warn("Erreur sauvegarde rangs :", error);
      if (isComponentMounted) message = { text: "Délai de transmission dépassé par le réseau.", type: 'error' };
    } finally {
      if (isComponentMounted) isSaving = false;
    }
  }

  // --- LOGIQUE : MESSAGERIE ---
  async function deleteMessage(id: number) {
    if (!isComponentMounted) return;
    if (!confirm("Voulez-vous vraiment supprimer ce message définitivement ?")) return;

    try {
      const timeoutPromise = new Promise<any>((_, reject) =>
        setTimeout(() => reject(new Error("Timeout suppression message")), 3000)
      );

      const deletePromise = supabase.from('messagerie').delete().eq('id', id);
      const { error } = await Promise.race([deletePromise, timeoutPromise]);

      if (error) {
        if (isComponentMounted) message = { text: "Erreur lors de la suppression : " + error.message, type: 'error' };
      } else {
        if (isComponentMounted) {
          inboxMessages = inboxMessages.filter(m => m.id !== id);
          message = { text: "Message supprimé avec succès.", type: 'success' };
        }
      }
    } catch (error) {
      console.warn("Erreur suppression message :", error);
      if (isComponentMounted) message = { text: "Action interrompue suite à une instabilité réseau.", type: 'error' };
    }
  }
</script>

<div class="w-full max-w-5xl mx-auto flex flex-col items-center animate-fade-in z-20 pb-20">

  <div class="text-center mb-8">
    <h1 class="text-4xl font-black uppercase tracking-widest text-transparent bg-clip-text bg-gradient-to-r from-rose-400 to-orange-300 flex items-center justify-center gap-3">
      <span class="w-3 h-3 rounded-full bg-rose-500 animate-pulse"></span>
      Panneau d'Administration
    </h1>
    <p class="text-rose-200/50 tracking-[0.2em] uppercase text-xs font-semibold mt-2">
      Modification du cœur du jeu
    </p>
  </div>

  {#if isLoading}
    <div class="text-indigo-300 animate-pulse uppercase tracking-widest text-sm font-bold mt-10">
      Connexion à la bdd...
    </div>
  {:else}
    <div class="w-full bg-slate-900/80 backdrop-blur-xl border border-rose-500/30 rounded-3xl shadow-[0_0_40px_rgba(244,63,94,0.1)] overflow-hidden">

      <div class="grid grid-cols-2 md:grid-cols-4 border-b border-rose-500/20">
        <button
          onclick={() => { activeTab = 'criteres'; message = {text: '', type: ''}; }}
          class={`py-4 text-xs md:text-sm font-bold uppercase tracking-wider transition-all ${activeTab === 'criteres' ? 'text-teal-300 bg-rose-500/10 border-b-2 border-teal-400 shadow-[inset_0_-10px_20px_-10px_rgba(45,212,191,0.3)]' : 'text-indigo-300/50 hover:text-rose-300 hover:bg-rose-500/5'}`}
        >
          Critères
        </button>
        <button
          onclick={() => { activeTab = 'enums'; message = {text: '', type: ''}; }}
          class={`py-4 text-xs md:text-sm font-bold uppercase tracking-wider transition-all ${activeTab === 'enums' ? 'text-amber-300 bg-rose-500/10 border-b-2 border-amber-400 shadow-[inset_0_-10px_20px_-10px_rgba(251,191,36,0.3)]' : 'text-indigo-300/50 hover:text-rose-300 hover:bg-rose-500/5'}`}
        >
          Énumérations
        </button>
        <button
          onclick={() => { activeTab = 'rangs'; message = {text: '', type: ''}; }}
          class={`py-4 text-xs md:text-sm font-bold uppercase tracking-wider transition-all ${activeTab === 'rangs' ? 'text-fuchsia-300 bg-rose-500/10 border-b-2 border-fuchsia-400 shadow-[inset_0_-10px_20px_-10px_rgba(217,70,239,0.3)]' : 'text-indigo-300/50 hover:text-rose-300 hover:bg-rose-500/5'}`}
        >
          Rangs
        </button>
        <button
          onclick={() => { activeTab = 'messages'; message = {text: '', type: ''}; }}
          class={`py-4 text-xs md:text-sm font-bold uppercase tracking-wider transition-all ${activeTab === 'messages' ? 'text-cyan-300 bg-rose-500/10 border-b-2 border-cyan-400 shadow-[inset_0_-10px_20px_-10px_rgba(34,211,238,0.3)]' : 'text-indigo-300/50 hover:text-rose-300 hover:bg-rose-500/5'}`}
        >
          Messagerie
          {#if inboxMessages.length > 0}
            <span class="ml-2 bg-cyan-500 text-slate-900 px-2 py-0.5 rounded-full text-[10px] font-black">{inboxMessages.length}</span>
          {/if}
        </button>
      </div>

      {#if message.text}
        <div class="p-4 mx-6 mt-6 rounded-xl text-sm font-medium border backdrop-blur-sm bg-slate-950/50 text-center {message.type === 'error' ? 'border-rose-500/50 text-rose-300' : 'border-teal-500/50 text-teal-300'}">
          {message.text}
        </div>
      {/if}

      {#if activeTab === 'criteres'}
        <div class="p-6 md:p-8 flex flex-col gap-4 animate-fade-in">

          <div class="hidden md:grid grid-cols-12 gap-4 px-4 text-[10px] font-bold tracking-widest uppercase text-indigo-300/50">
            <div class="col-span-2">Clé (Unique)</div>
            <div class="col-span-3">Label (Nom Public)</div>
            <div class="col-span-3">Type de Donnée</div>
            <div class="col-span-2 text-center">Ordre</div>
            <div class="col-span-1 text-center">Actif</div>
            <div class="col-span-1"></div>
          </div>

          {#each criteres as critere, index}
            <div class="grid grid-cols-1 md:grid-cols-12 gap-4 items-center bg-slate-950/50 p-4 rounded-xl border border-indigo-500/20 group hover:border-teal-400/30 transition-all">
              <input type="text" bind:value={critere.cle} placeholder="ex: role_viewer" class="md:col-span-2 w-full bg-slate-900 border border-indigo-500/20 rounded p-2 text-sm text-teal-200 font-mono focus:outline-none focus:border-teal-400/50"/>
              <input type="text" bind:value={critere.label} placeholder="ex: Rôle sur Twitch" class="md:col-span-3 w-full bg-transparent text-indigo-100 focus:outline-none focus:text-teal-200 font-bold px-2"/>

              <select bind:value={critere.type_donnee} class="md:col-span-3 w-full bg-slate-900 border border-indigo-500/20 rounded p-2 text-sm text-indigo-200 focus:outline-none focus:border-teal-400/50 cursor-pointer">
                <option value="text">Texte brut</option>
                <option value="number">Nombre entier/décimal</option>
                <option value="boolean">Booléen (Oui/Non)</option>
                <option value="date">Format Date</option>
                <option value="enumeration">Énumération (Choix)</option>
              </select>

              <input type="number" bind:value={critere.ordre} class="md:col-span-2 w-full bg-slate-900 border border-indigo-500/20 rounded p-2 text-center text-indigo-100 focus:outline-none focus:border-teal-400/50 font-mono"/>
              <div class="md:col-span-1 flex justify-center"><input type="checkbox" bind:checked={critere.actif} class="accent-teal-400 w-5 h-5 cursor-pointer" /></div>
              <div class="md:col-span-1 flex justify-center"><button onclick={() => removeCritere(index)} class="text-rose-400/40 hover:text-rose-400 p-2 cursor-pointer">✕</button></div>
            </div>
          {/each}

          <button onclick={addCritere} class="w-full py-4 mt-2 border border-dashed border-indigo-500/30 text-indigo-300/70 hover:text-teal-300 hover:bg-teal-500/5 rounded-xl font-bold uppercase tracking-widest text-xs cursor-pointer">+ Nouveau critère</button>
          <button onclick={saveCriteres} disabled={isSaving} class="mt-8 w-full py-4 rounded-xl font-bold uppercase tracking-widest bg-teal-500/20 text-teal-200 border border-teal-500/30 hover:bg-teal-400/30 transition-all cursor-pointer">Sauvegarder les critères</button>
        </div>
      {/if}

      {#if activeTab === 'enums'}
        <div class="p-6 md:p-8 flex flex-col gap-6 animate-fade-in">
          <p class="text-xs text-indigo-300/60 uppercase tracking-widest text-center font-bold mb-4">
            Gérez ici les choix possibles pour vos critères de type "Énumération".
          </p>

          {#each criteres as critere, index}
            {#if critere.type_donnee === 'enumeration'}
              <div class="flex flex-col bg-slate-950/50 p-5 rounded-2xl border border-amber-500/20">

                <div class="flex items-center gap-3 border-b border-indigo-500/20 pb-3 mb-4">
                  <span class="text-amber-400 font-black tracking-widest uppercase">{critere.label || critere.cle || 'Critère sans nom'}</span>
                  <span class="text-[10px] bg-slate-900 text-indigo-400 px-2 py-1 rounded-md font-mono">{critere.cle}</span>
                </div>

                <div class="flex flex-wrap gap-2 mb-4">
                  {#if !critere.options || critere.options.length === 0}
                    <span class="text-sm text-indigo-400/40 italic">Aucun choix configuré pour le moment.</span>
                  {/if}

                  {#each critere.options || [] as option, optIndex}
                    <div class="flex items-center gap-2 bg-amber-500/10 border border-amber-500/30 text-amber-100 px-3 py-1.5 rounded-full text-sm font-medium group">
                      {option}
                      <button onclick={() => removeOption(index, optIndex)} class="text-amber-400/50 hover:text-rose-400 transition-colors cursor-pointer">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
                          <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
                        </svg>
                      </button>
                    </div>
                  {/each}
                </div>

                <div class="flex gap-2">
                  <input
                    type="text"
                    bind:value={optionInputs[index]}
                    onkeydown={(e) => { if (e.key === 'Enter') addOption(index); }}
                    placeholder="Ex: Rouge, Bleu..."
                    class="flex-1 bg-slate-900 border border-indigo-500/20 rounded-xl p-3 text-sm text-indigo-100 focus:outline-none focus:border-amber-400/50"
                  />
                  <button
                    onclick={() => addOption(index)}
                    class="bg-amber-500/20 hover:bg-amber-500/30 text-amber-300 font-bold px-5 rounded-xl border border-amber-500/30 transition-colors cursor-pointer"
                  >
                    Ajouter
                  </button>
                </div>
              </div>
            {/if}
          {/each}

          <button onclick={saveCriteres} disabled={isSaving} class="mt-4 w-full py-4 rounded-xl font-bold uppercase tracking-widest bg-amber-500/20 text-amber-200 border border-amber-500/30 hover:bg-amber-400/30 transition-all cursor-pointer">
            {isSaving ? 'Synchronisation BDD...' : 'Sauvegarder les choix'}
          </button>
        </div>
      {/if}

      {#if activeTab === 'rangs'}
        <div class="p-6 md:p-8 flex flex-col gap-4 animate-fade-in">
          {#each ranks as rank, index}
            <div class="grid grid-cols-12 gap-4 items-center bg-slate-950/50 p-3 rounded-xl border border-indigo-500/20">
              <input type="text" bind:value={rank.nom_rang} placeholder="Ex: Diamant" class="col-span-7 bg-transparent text-indigo-100 focus:outline-none focus:text-fuchsia-200 font-bold px-2"/>
              <input type="number" bind:value={rank.nombre_points} placeholder="0" class="col-span-4 bg-slate-900 border border-indigo-500/20 rounded p-2 text-center text-indigo-100 font-mono focus:outline-none focus:border-fuchsia-400/50"/>
              <button onclick={() => removeRank(index)} class="col-span-1 flex justify-center text-rose-400/50 hover:text-rose-400 cursor-pointer">✕</button>
            </div>
          {/each}
          <button onclick={addRank} class="w-full py-4 mt-2 border border-dashed border-indigo-500/30 text-indigo-300/70 hover:text-fuchsia-300 hover:bg-fuchsia-500/5 rounded-xl font-bold uppercase text-xs cursor-pointer">+ Nouveau grade</button>
          <button onclick={saveRanks} disabled={isSaving} class="mt-8 w-full py-4 rounded-xl font-bold uppercase tracking-widest bg-fuchsia-500/20 text-fuchsia-200 border border-fuchsia-500/30 hover:bg-fuchsia-400/30 cursor-pointer">Sauvegarder le classement</button>
        </div>
      {/if}

      {#if activeTab === 'messages'}
        <div class="p-6 md:p-8 flex flex-col gap-4 animate-fade-in">

          {#if inboxMessages.length === 0}
            <div class="py-12 flex flex-col items-center justify-center border border-dashed border-indigo-500/30 rounded-2xl bg-slate-950/30">
              <span class="text-4xl mb-4">📭</span>
              <p class="text-indigo-300/50 uppercase tracking-widest text-sm font-bold text-center">
                Aucun message en attente.<br/>Le réseau est silencieux.
              </p>
            </div>
          {:else}
            <p class="text-xs text-indigo-300/60 uppercase tracking-widest font-bold mb-2">
              Messages des utilisateurs ({inboxMessages.length})
            </p>

            <div class="flex flex-col gap-4">
              {#each inboxMessages as msg}
                <div class="bg-slate-950/50 p-5 rounded-2xl border border-cyan-500/20 hover:border-cyan-500/50 transition-all flex flex-col md:flex-row justify-between items-start md:items-center gap-4 group">

                  <div class="flex-1 flex flex-col gap-2 w-full">
                    <div class="flex items-center gap-3">
                      <span class="bg-cyan-500/10 border border-cyan-500/30 text-cyan-300 px-2 py-0.5 rounded text-[10px] font-mono font-bold uppercase tracking-widest">
                        Ticket #{msg.id}
                      </span>
                    </div>
                    <p class="text-indigo-100 text-sm md:text-base leading-relaxed whitespace-pre-wrap pl-1">
                      {msg.contenu}
                    </p>
                  </div>

                  <button
                    onclick={() => deleteMessage(msg.id)}
                    class="self-end md:self-auto flex items-center justify-center p-3 rounded-xl bg-rose-500/10 text-rose-400 border border-rose-500/30 hover:bg-rose-500/20 hover:text-rose-300 transition-colors cursor-pointer group-hover:opacity-100 md:opacity-50"
                    title="Supprimer ce message"
                  >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                    </svg>
                  </button>

                </div>
              {/each}
            </div>
          {/if}

        </div>
      {/if}

    </div>
  {/if}
</div>