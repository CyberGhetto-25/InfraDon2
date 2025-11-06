<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

// --- Type du document
interface InfradonDoc {
  _id?: string
  _rev?: string
  title: string
  description: string
  created_at: string
  status: string
  author: string
}

// --- Références
const storage = ref<PouchDB.Database<InfradonDoc> | null>(null)
const docs = ref<InfradonDoc[]>([])
const formTitle = ref('')
const formDescription = ref('')
const selectedDoc = ref<InfradonDoc | null>(null)
const deletingIds = ref<Set<string>>(new Set())



// --- Connexion à la base
const initDatabase = () => {
   console.log('=> Connexion à la base locale + distante avec réplication')

  // 1️⃣ Base locale (dans le navigateur)
  const localDB = new PouchDB<InfradonDoc>('test_infradon2_local')

  // 2️⃣ Base distante (CouchDB)
  const remoteDB = new PouchDB<InfradonDoc>(
    'http://admin:admin@127.0.0.1:5984/test_infradon2'
  )

  // 3️⃣ Première réplication descendante (remote → local)
  localDB
    .replicate.from(remoteDB)
    .on('complete', () => {
      console.log('✅ Initial pull terminé (remote → local)')
    })
    .on('error', (err) => console.error('❌ Erreur initial pull:', err))

  // 4️⃣ Synchronisation bidirectionnelle continue (local ⇄ remote)
  localDB
    .sync(remoteDB, { live: true, retry: true })
    .on('change', (info) => {
      console.log('🔄 Sync change', info)
    })
    .on('paused', (err) => {
      if (err) console.warn('⏸️ Sync paused (err)', err)
    })
    .on('active', () => console.log('▶️ Sync active'))
    .on('error', (err) => console.error('❌ Sync error:', err))

  // 5️⃣ On conserve la base locale pour le CRUD existant
  storage.value = localDB
  console.log('✅ Base locale prête et synchronisée avec CouchDB')
}

// --- Récupération des docs
const fetchData = () => {
  if (!storage.value) return

  console.log('=> Récupération des documents...')
  storage.value
    .allDocs({ include_docs: true })
    .then((res: any) => {
      docs.value = res.rows
        .filter((r: any) => r.doc && !String(r.id).startsWith('_design/'))
        .map((r: any) => r.doc)
      console.log('✅ Données récupérées :', docs.value.length)
    })
    .catch((err: any) => console.error('❌ Erreur fetchData:', err))
}

// --- Créer un doc (POST)
const createDoc = () => {
  if (!storage.value) return

  const newDoc: InfradonDoc = {
    title: formTitle.value || 'Fake document ' + Date.now(),
    description:
      formDescription.value ||
      'Lorem ipsum dolor sit amet, généré automatiquement pour test.',
    created_at: new Date().toISOString(),
    status: 'draft',
    author: 'Marc Bridy'
  }

  storage.value
    .post(newDoc)
    .then((res: any) => {
      console.log('✅ Document créé:', res)
      formTitle.value = ''
      formDescription.value = ''
      fetchData()
    })
    .catch((err: any) => console.error('❌ Erreur création doc:', err))
}

// --- Sélectionner un doc à modifier
const selectDoc = (doc: InfradonDoc) => {
  selectedDoc.value = { ...doc }
  formTitle.value = doc.title
  formDescription.value = doc.description
}

// --- Mettre à jour un doc (PUT)
const updateDoc = () => {
  if (!storage.value || !selectedDoc.value) {
    console.warn('Aucun document sélectionné pour mise à jour')
    return
  }

  const updatedDoc: InfradonDoc = {
    ...selectedDoc.value,
    title: formTitle.value || selectedDoc.value.title,
    description: formDescription.value || selectedDoc.value.description,
    status: 'updated'
  }

  storage.value
    .put(updatedDoc)
    .then((res: any) => {
      console.log('✅ Document mis à jour:', res)
      selectedDoc.value = null
      formTitle.value = ''
      formDescription.value = ''
      fetchData()
    })
    .catch((err: any) => console.error('❌ Erreur update doc:', err))
}

// --- Supprimer un doc (DELETE)
const deleteDoc = async (doc: InfradonDoc) => {
  if (!storage.value) return
  if (!doc._id || !doc._rev) {
    console.warn('Doc sans _id/_rev, impossible de supprimer')
    return
  }

  const ok = window.confirm(`Supprimer définitivement « ${doc.title} » ?`)
  if (!ok) return

  try {
    deletingIds.value.add(doc._id)

    // Nettoyer le formulaire si c'était le doc en cours d’édition
    if (selectedDoc.value?._id === doc._id) {
      selectedDoc.value = null
      formTitle.value = ''
      formDescription.value = ''
    }

    await storage.value.remove(doc._id, doc._rev)
    console.log('🗑️ Document supprimé:', doc._id)
    await fetchData()
  } catch (err) {
    console.error('❌ Erreur suppression doc:', err)
    alert('Erreur lors de la suppression. Regarde la console pour les détails.')
  } finally {
    deletingIds.value.delete(doc._id)
  }
}

// --- Lifecycle
onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <div style="padding: 2rem;">
    <h1>📦 Gestion des documents Infradon2</h1>

    <!-- Formulaire -->
    <section style="margin-top: 1.5rem; margin-bottom: 2rem;">
      <h2>{{ selectedDoc ? '✏️ Modifier un document' : '➕ Créer un document' }}</h2>
      <input
        type="text"
        v-model="formTitle"
        placeholder="Titre"
        style="display:block;margin:0.5rem 0;padding:0.5rem;width:300px;"
      />
      <textarea
        v-model="formDescription"
        placeholder="Description"
        style="display:block;margin:0.5rem 0;padding:0.5rem;width:300px;height:80px;"
      ></textarea>

      <button
        @click="selectedDoc ? updateDoc() : createDoc()"
        style="padding:0.5rem 1rem;margin-right:0.5rem;"
      >
        {{ selectedDoc ? '💾 Mettre à jour' : '📤 Créer' }}
      </button>
      <button
        v-if="selectedDoc"
        @click="
          selectedDoc = null;
          formTitle = '';
          formDescription = '';
        "
        style="padding:0.5rem 1rem;"
      >
        ❌ Annuler
      </button>
    </section>

    <!-- Liste des documents -->
    <section>
      <h2>🗂️ Liste des documents</h2>
      <div
        v-for="doc in docs"
        :key="doc._id"
        style="border:1px solid #555;border-radius:8px;padding:1rem;margin-bottom:1rem;"
      >
        <h3>{{ doc.title }}</h3>
        <p>{{ doc.description }}</p>
        <p>
          <small>
            Auteur: {{ doc.author }} | Status: {{ doc.status }} |
            Créé le: {{ new Date(doc.created_at).toLocaleString() }}
          </small>
        </p>

        <button
          @click="selectDoc(doc)"
          style="padding:0.3rem 0.8rem;margin-right:0.4rem;"
        >
          ✏️ Modifier
        </button>

        <button
          @click="deleteDoc(doc)"
          :disabled="deletingIds.has(doc._id!)"
          style="padding:0.3rem 0.8rem;"
        >
          {{ deletingIds.has(doc._id!) ? '… Suppression' : '🗑️ Supprimer' }}
        </button>
      </div>
    </section>
  </div>
</template>
